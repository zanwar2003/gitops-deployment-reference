# GitOps Deployment Reference

A reference implementation of the "workload repo" half of a two-repo (app-of-apps) GitOps
pattern: Argo CD Applications that point at Kustomize overlays, layered on a shared base, with a
separate `external-secrets/` example showing how secrets stay out of source control entirely.
This mirrors the pattern I work with day to day running a multi-source Argo CD platform on AKS â
rewritten from scratch here as a clean, standalone reference rather than any employer's actual
configuration.

## Why a two-repo (app-of-apps) split

Splitting "platform" concerns (installing Argo CD itself, ingress controllers, secret-store
identities) from "workload" concerns (what applications get deployed, with what config, to which
environment) keeps the blast radius of a workload change small: an application team can merge a
change to their own overlay without touching anything that could affect the cluster's platform
components, and vice versa. This repo represents the **workload** side only.

## Architecture

```
apps/
  sample-api.yaml            # Argo CD Application CR â points Argo CD at overlays/<env>

base/
  deployment.yaml            # environment-agnostic Deployment spec
  service.yaml                # environment-agnostic Service spec
  kustomization.yaml          # declares the base resources

overlays/
  dev/kustomization.yaml       # patches: 1 replica, dev image tag, dev-specific env vars
  staging/kustomization.yaml   # patches: 2 replicas, staging image tag
  production/kustomization.yaml # patches: 5 replicas, prod image tag, resource limits raised

external-secrets/
  secret-store.yaml            # SecretStore pointing at an external secrets backend (placeholder)
  external-secret.yaml         # ExternalSecret that syncs a named secret into the namespace
```

Each environment overlay is applied via `kustomize build overlays/<env>`, which layers
environment-specific patches on top of the shared `base/` â so a change to `base/deployment.yaml`
(e.g. adding a new health-check probe) automatically applies to every environment, while a
change to `overlays/production/kustomization.yaml` (e.g. raising the replica count) stays scoped
to production only.

## Sync policy

The Argo CD Application in `apps/sample-api.yaml` sets:

- `syncPolicy.automated.prune: true` â resources removed from Git are removed from the cluster,
  instead of being silently left behind.
- `syncPolicy.automated.selfHeal: true` â a manual `kubectl edit` against a live resource gets
  reverted back to what's declared in Git on the next reconcile loop, rather than drifting
  silently.

Together these are what make the cluster state an accurate reflection of the repo rather than
something that can quietly diverge from it over time.

## Secrets

`external-secrets/` shows the pattern for keeping secrets out of Git entirely: a `SecretStore`
resource declares where the real secret values live (a cloud key vault, in practice), and an
`ExternalSecret` resource declares which named secret should be synced into the namespace and
under what Kubernetes Secret name â the actual secret *values* never appear in this repo, only
the wiring that says where to fetch them from.

## Validating locally

You don't need a live cluster to validate this repo â `kustomize build` renders the final YAML
for a given environment so you can review exactly what would be applied before it ever reaches
Argo CD:

```bash
kustomize build overlays/dev
kustomize build overlays/staging
kustomize build overlays/production
```

Diffing the rendered output between environments is a fast way to confirm an overlay patch did
what you expected, without needing cluster access at all.
