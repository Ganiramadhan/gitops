# ganipedia-gitops

Single source of truth for Kubernetes deployment manifests across the
Ganipedia cluster. This repo contains **only Kubernetes manifests** —
application source code lives in its own repository per app.

ArgoCD watches this repo via an `ApplicationSet` and auto-deploys every
app defined under `apps/`.

## Why a separate repo?

Application CI pipelines (Jenkins/GitHub Actions) only need **write access
here**, never to the application source code repos. This keeps the blast
radius small if a CI credential ever leaks, and keeps app commit history
free of automated deploy noise.

```
┌───────────────────┐  build  ┌────────────────────┐  sync  ┌─────────────┐
│  app source repo   │────────▶│   this repo         │───────▶│   cluster   │
│  (e.g. api-saku)   │ CI only │  (manifests only)   │ GitOps │  (ArgoCD)   │
└───────────────────┘         └────────────────────┘        └─────────────┘
    read-only                   write (CI bumps              pull-based
                                 image tag here)
```

## Structure

```
apps/
  <app-name>/
    deploy.yaml      # Namespace, Deployment, Service, Ingress, HPA,
                      # PDB, ResourceQuota, LimitRange, NetworkPolicy
```

Each subfolder under `apps/` becomes one ArgoCD `Application` automatically
— no manual registration needed.

## How deployment works

1. Developer pushes code to the app repo
2. CI builds the image and pushes it to the registry
3. CI clones **this repo**, bumps the `image:` tag in
   `apps/<app-name>/deploy.yaml`, commits with `[skip ci]`, and pushes
4. ArgoCD's `ApplicationSet` (`ganipedia-apps`) detects the change and
   auto-syncs the matching `Application`
5. The cluster pulls the new image and performs a rolling update

No manual `kubectl apply` is involved at any point.

## Adding a new app

1. Create `apps/<new-app-name>/deploy.yaml` with the full manifest set
   (Namespace, ResourceQuota, LimitRange, NetworkPolicy, Deployment,
   Service, Ingress, HPA, PDB — follow the pattern used by existing apps)
2. Commit and push
3. ArgoCD's `ApplicationSet` picks it up automatically within minutes

No ArgoCD CLI command or manual `Application` CRD is required.

## Rolling back a deploy

```bash
git revert <bad-commit-sha>
git push
```

ArgoCD detects the reverted state and re-syncs the cluster to the previous
image tag automatically.

## Secrets policy

**Never commit real secrets here** — not even placeholders. Application
secrets (DB credentials, JWT keys, API tokens) are managed exclusively via
HashiCorp Vault and injected into pods at runtime through the Vault Agent
Injector. Manifests only reference Vault paths via pod annotations
(`vault.hashicorp.com/*`), never plaintext values.

## Current apps

| App | Namespace | Public URL |
|---|---|---|
| `pare-moments` | `pare-moments` | https://moments.ganipedia.com |
| `api-saku-pay` | `saku-pay-staging` | https://api-saku.ganipedia.com |
