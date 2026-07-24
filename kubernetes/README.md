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
│   └── cnpg/                    # CloudNativePG operator
└── apps/
    ├── argocd-apps/             # one ArgoCD Application manifest per app/env
    │   ├── cnpg/prod/cnpg.yaml
    │   └── zammad/prod/zammad.yaml
    └── zammad/prod/             # the actual source ArgoCD syncs: helm values,
        ├── namespace.yaml       # the CNPG Cluster CR, and a kustomization tying
        ├── cnpg-cluster.yaml    # them together via kustomize's helmCharts field
        ├── zammad-values.yaml
        └── kustomization.yaml
```

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

### Not yet wired up (deferred on purpose)

- **Secrets**: `zammad-values.yaml` references `zammad-pg-app` (the Secret CNPG
  auto-generates for the Cluster CR), so Zammad's DB password needs no manual
  secret today. Broader secrets management (sealed-secrets vs. External
  Secrets Operator + AWS Secrets Manager) is a separate follow-up.
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
