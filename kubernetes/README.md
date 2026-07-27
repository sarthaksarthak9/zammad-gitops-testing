# Infra

This repository consist of the internal Kubernetes infrastructure that is currently deployed in CloudRaft AWS account.

## Current Setup
A k3s VM is running in the AWS account. It has a public IP that is accessible using `*.app.cloudraft.io`.  

## Current Applications

### Compass

### n8n

### Zammad - Helpdesk

## GitOps layout

This directory is managed by ArgoCD using an app-of-apps pattern (mirrors the
`cloudraftio/turboraft` accelerator, adapted for the single k3s node):

```
kubernetes/
├── platform/                    # cluster-wide components, applied manually once
│   ├── argocd/                  # ArgoCD itself (bootstraps everything below)
│   ├── cnpg/                    # CloudNativePG operator + Barman Cloud Plugin
│   └── external-secrets/        # External Secrets Operator (ESO)
└── apps/
    ├── argocd-apps/             # one ArgoCD Application manifest per app/env
    │   ├── cnpg/prod/cnpg.yaml
    │   ├── external-secrets/prod/external-secrets.yaml
    │   └── zammad/prod/zammad.yaml
    └── zammad/prod/             # the actual source ArgoCD syncs: helm values,
        ├── namespace.yaml               # CNPG Cluster CR + backup ObjectStore,
        ├── cnpg-cluster.yaml            # the ESO SecretStore + ExternalSecret,
        ├── cnpg-backup.yaml             # and a kustomization tying them all
        ├── secretstore.yaml             # together via kustomize's helmCharts
        ├── externalsecret-pg-backup-s3.yaml
        ├── zammad-values.yaml
        └── kustomization.yaml
```

### CNPG: HA + S3 backup

`zammad-pg` runs **2 instances** (primary + streaming replica) for HA, and backs
up to S3 (`s3://cloudraft-cnpg-backup/zammad-pg`, `us-east-1`, 7 day retention)
via the **Barman Cloud Plugin** — recent CNPG (1.26+) deprecated the in-tree
`spec.backup.barmanObjectStore` field in favor of this separate plugin, so
`platform/cnpg/kustomization.yaml` installs it alongside the operator itself.
Two things this needs that aren't automated here:

- **cert-manager** must already exist in the cluster (the plugin's prerequisite;
  already true on the live k3s VM).
- **`zammad-pg-backup-s3-creds`** — a Secret in the `zammad` namespace with keys
  `ACCESS_KEY_ID` / `ACCESS_SECRET_KEY` for the `cloudraft-cnpg-backup` bucket.
  Referenced by `cnpg-backup.yaml`. Materialized automatically by ESO — see below,
  not created directly in this repo.

### Secrets: External Secrets Operator (ESO)

Real secret values never live in git. `platform/external-secrets/` installs
ESO; `apps/zammad/prod/secretstore.yaml` + `externalsecret-pg-backup-s3.yaml`
pull the actual S3 backup credentials from **AWS Secrets Manager**
(`zammad-pg-backup-s3-creds`, `us-east-1`) and materialize them as a
Kubernetes Secret of the same name — which is exactly what `cnpg-backup.yaml`
expects.

Auth: the `SecretStore` has no `auth` block on purpose — it relies on the k3s
EC2 instance's attached IAM role (`zammad-eso-role`, scoped to
`secretsmanager:GetSecretValue`/`DescribeSecret` on just this one secret's
ARN) rather than any static AWS key living in the cluster. That role has to
be attached to whatever instance runs this — if the VM is ever rebuilt, this
is the one manual step that doesn't self-heal from git (see "rebuild" note
in `platform/argocd`'s bootstrap section conceptually — same idea applies
here).

The AWS-side prerequisites (not in this repo, already done for the current
VM):
- IAM user `zammad-cnpg-backup`, policy scoped to `s3:PutObject/GetObject/
  DeleteObject/ListBucket` on `cloudraft-cnpg-backup`, with an access key.
- That access key's `ACCESS_KEY_ID`/`ACCESS_SECRET_KEY` stored as the
  Secrets Manager secret `zammad-pg-backup-s3-creds`.
- IAM role `zammad-eso-role` (policy above) attached as the EC2 instance
  profile on the k3s VM.

**Note**: `cnpg` and `external-secrets` are independent Applications under
the recursive `root-app.yaml`, so on a from-scratch bootstrap, `zammad`'s
`SecretStore`/`ExternalSecret` (which need ESO's CRDs) or its CNPG `Cluster`
(which needs the CNPG CRDs) may show transient sync errors until those
platform Applications finish first — ArgoCD's `selfHeal` retries until they
land, so this resolves on its own without intervention.

### Bootstrap order (manual, one-time, on the k3s VM)

1. `kubectl apply -k platform/argocd` — installs ArgoCD itself. This step can't
   be done by ArgoCD (it doesn't exist yet), so it's applied directly.
2. Register this repo in ArgoCD (repo credentials — see "Not yet wired up" below).
3. `kubectl apply -f platform/argocd/root-app.yaml` — the root Application. It
   watches `apps/argocd-apps/**` recursively, so ArgoCD auto-registers every
   Application manifest it finds there (currently `cnpg` and `zammad`).
4. From here on, ArgoCD owns sync: edits under `apps/zammad/prod/` or
   `platform/cnpg/` land automatically (`prune: true, selfHeal: true`).

### Tested on minikube (2026-07-24)

Validated against a real ArgoCD instance (not just local rendering) by pushing
this directory to a throwaway public repo and syncing from there:

- `platform/argocd` installs cleanly; the `kustomize.buildOptions` value lands
  in `argocd-cm` as expected.
- `apps/zammad/prod` renders the full, correct resource set (all Deployments/
  StatefulSets, Ingress, and the `zammad-pg` CNPG `Cluster` CR) — matches what
  actually runs on the live k3s VM.
- `platform/cnpg`: the CNPG operator's `clusters.postgresql.cnpg.io` CRD is
  large enough that client-side `kubectl apply` rejects it (the full manifest
  duplicated into the `last-applied-configuration` annotation exceeds
  Kubernetes' 256KiB annotation limit) — a known CNPG+ArgoCD issue. Fixed by
  adding `syncOptions: [ServerSideApply=true]` to `cnpg.yaml`, which applies
  without the annotation.

Note: the 2-instance HA + S3 backup config and the ESO setup (above) were
added after this test run and have **not** been validated end-to-end yet —
they need real S3/Secrets Manager credentials, cert-manager, and the EC2
instance role, none of which the minikube test had.

### Not yet wired up (deferred on purpose)

- **DB password secret**: `zammad-values.yaml` still references
  `zammad-pg-app` (the Secret CNPG auto-generates for the Cluster CR
  directly), not ESO — only the S3 backup credentials go through ESO so far.
  Moving the DB password through ESO too (or sealed-secrets) is a separate
  follow-up.
- **Repo credentials for ArgoCD** — needs a deploy key or token added to
  ArgoCD so it can pull this (private) repo; not automated here.
- **Adopting the existing live Zammad release** — Zammad is currently running
  on the k3s VM via a manual `helm upgrade --install` (see
  `../../cloudraft-zammad-poc/SETUP.md`). Cutting it over to ArgoCD management
  needs care (matching values exactly so Argo doesn't prune/recreate the live
  CNPG cluster or PVCs) — not done as part of this pass.
- **Monitoring/alerting** (Uptime Kuma checks, incident.io/spike.sh, Telegram/
  Slack notification) and **Compass/n8n** Application manifests — out of scope
  here, follow-up work.
