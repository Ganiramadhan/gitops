# ganipedia-gitops

> Single source of truth for Kubernetes deployment manifests across the
> Ganipedia cluster, managed via [ArgoCD](https://argo-cd.readthedocs.io/).

This repository contains **only Kubernetes manifests** — no application
source code. Each app's source lives in its own repository and is built by
its own CI pipeline.

## Table of Contents

- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [How Deployment Works](#how-deployment-works)
- [Adding a New App](#adding-a-new-app)
- [Manifest Conventions](#manifest-conventions)
- [Rolling Back a Deploy](#rolling-back-a-deploy)
- [Secrets Policy](#secrets-policy)
- [CI Integration](#ci-integration)
- [Troubleshooting](#troubleshooting)
- [Current Apps](#current-apps)

## Architecture

```
┌───────────────────┐  build  ┌────────────────────┐  sync  ┌─────────────┐
│  app source repo   │────────▶│   this repo         │───────▶│   cluster   │
│  (e.g. api-saku)   │ CI only │  (manifests only)   │ GitOps │  (ArgoCD)   │
└───────────────────┘         └────────────────────┘        └─────────────┘
    read-only                   write (CI bumps              pull-based
                                 image tag here)                sync
```

**Why a separate repo instead of keeping manifests next to app code?**

| Concern | Mono-repo (code + manifest together) | This repo (separated) |
|---|---|---|
| CI write access | Needs push access to app source repo | Needs push access to **this repo only** |
| Blast radius if CI credential leaks | Attacker can modify app source code | Attacker can only modify deploy config (revert in one `git revert`) |
| Commit history | Mixed: feature commits + automated deploy commits | Clean separation; app history stays readable |
| Multi-app consistency | Hard to compare configs across apps | All manifests visible in one place |

## Repository Structure

```
apps/
  <app-name>/
    deploy.yaml   # Namespace, ResourceQuota, LimitRange, NetworkPolicy,
                  # Deployment, Service, Ingress, HPA, PDB — all in one file
```

Each subfolder under `apps/` is picked up automatically by ArgoCD's
`ApplicationSet` (`ganipedia-apps`) and becomes its own `Application`.
**No manual `Application` CRD or `argocd app create` command is ever
needed** — the folder structure IS the registration.

## How Deployment Works

1. Developer pushes code to the app's own repository
2. CI (Jenkins) builds the Docker image and pushes it to the registry,
   tagged as `<build-number>-<short-commit-sha>`
3. CI clones **this repo**, bumps the `image:` line in
   `apps/<app-name>/deploy.yaml`, and pushes a commit tagged `[skip ci]`
   (prevents the CI pipeline from re-triggering on its own commit)
4. ArgoCD's `ApplicationSet` detects the change (poll interval ~3 min, or
   instant via webhook if configured) and marks the `Application` as
   `OutOfSync`
5. Because `syncPolicy.automated.selfHeal: true`, ArgoCD applies the change
   immediately without manual approval
6. Kubernetes performs a rolling update; old pods terminate only after new
   pods pass readiness checks

No human ever runs `kubectl apply` against this cluster for app deploys.

## Adding a New App

1. Create `apps/<new-app-name>/deploy.yaml` following the
   [Manifest Conventions](#manifest-conventions) below
2. Commit and push
3. Within minutes, `kubectl -n argocd get applications` will show a new
   `Application` with the same name as the folder

## Manifest Conventions

Every `apps/<name>/deploy.yaml` MUST include, in this order:

1. `Namespace` — with `pod-security.kubernetes.io/audit` and `/warn`
   labels set to at least `baseline`
2. `ResourceQuota` — hard caps on total CPU/memory/pod count for the
   namespace (required: this cluster runs on constrained hardware)
3. `LimitRange` — default + max resources per container
4. `NetworkPolicy` — `default-deny-ingress` plus an explicit `allow-from-*`
   rule scoped to the minimum required source (e.g. `kube-system` for
   Traefik-fronted apps)
5. `Deployment` — must set `securityContext` (`runAsNonRoot`, dropped
   capabilities, `seccompProfile: RuntimeDefault`) and resource
   requests/limits on every container
6. `Service`, `Ingress` (with `cert-manager.io/cluster-issuer` annotation
   for TLS), `HorizontalPodAutoscaler`, `PodDisruptionBudget` as needed

If the app needs secrets, reference Vault via pod annotations
(see [Secrets Policy](#secrets-policy)) — do not add a `Secret` object here.

If the app pulls from a private registry, reference an `imagePullSecrets`
entry; the actual `docker-registry` Secret is created out-of-band per
namespace (not tracked in Git).

## Rolling Back a Deploy

```bash
git log --oneline apps/<app-name>/deploy.yaml   # find the bad commit
git revert <bad-commit-sha>
git push
```

ArgoCD detects the reverted state and re-syncs the cluster to the previous
image tag automatically — no cluster access required to roll back.

## Secrets Policy

**Never commit real secrets here — not even as placeholders.** Application
secrets (DB credentials, JWT keys, API tokens) are managed exclusively in
HashiCorp Vault and injected into pods at runtime via the Vault Agent
Injector. Manifests reference Vault paths only through pod annotations
(`vault.hashicorp.com/agent-inject-*`), never plaintext values.

## CI Integration

CI pipelines need exactly one credential scoped to this repository (ideally
a fine-grained GitHub token limited to `ganipedia-gitops` only, **never**
a token with access to application source repos). The pipeline's only
responsibility toward this repo is:

```bash
git clone https://<token>@github.com/Ganiramadhan/gitops.git
cd gitops
sed -i "s|image:.*<app-name>.*|image: <registry>/<app-name>:<tag>|" apps/<app-name>/deploy.yaml
git commit -am "chore: deploy <app-name>:<tag> [skip ci]"
git push origin main
```

CI must never run `kubectl apply`, hold a kubeconfig, or have any direct
access to the cluster — ArgoCD is the only system that applies changes.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ImagePullBackOff` / `no basic auth credentials` | k3s (containerd) has no registry login; `docker login` on the CI host does not carry over | Create a `docker-registry` Secret in the target namespace and reference it via `imagePullSecrets` |
| ArgoCD app stuck "Syncing" indefinitely | `argocd-application-controller` OOMKilled (too many tracked apps for its memory limit) | Check `kubectl top pods -n argocd`; raise controller memory limit |
| New pod doesn't pick up a manifest annotation change | ArgoCD synced but pod template hash didn't change | Confirm the diff actually touched `spec.template`, not just metadata outside it |
| HPA won't scale past N replicas | `ResourceQuota` in the app's namespace is capping total CPU/memory | Check `kubectl describe resourcequota -n <namespace>` before raising `maxReplicas` |

## Current Apps

| App | Namespace | Public URL |
|---|---|---|
| `pare-moments` | `pare-moments` | https://moments.ganipedia.com |
| `api-saku-pay` | `saku-pay-staging` | https://api-saku.ganipedia.com |
