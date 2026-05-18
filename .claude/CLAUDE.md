# cfp-sandbox-cluster — agent instructions

GitOps repo for the CodeForPhilly sandbox Kubernetes cluster on Linode LKE. Source-projected via hologit. Don't edit deployed branches directly — change the workspace, project, push the result.

## Source pipeline

```
JarvusInnovations/cluster-template
  └─ civic-cloud/cluster-template
      └─ this repo (cfp-sandbox-cluster)
          └─ projected branches → live cluster
```

This repo pulls civic-cloud via `.holo/sources/civic-cloud.toml`.

To refresh a holosource: `git holo source fetch <name>`. **Never** `git fetch <upstream-url> <refspec>` — that auto-pulls upstream tags into local `refs/tags/` and pollutes the tag namespace.

## How projection works

- **Workspace files** are what humans edit. `.holo/branches/<branch>/` configs map workspace paths to source content.
- `git holo project <branch>` runs the pipeline and prints a tree SHA on the last stdout line. Inspect with `git ls-tree -r <SHA>` or `git diff <ref> <SHA>`.
- Two branches matter:
  - `k8s-manifests` — manifests only
  - `k8s-manifests-github` — manifests + GitHub Actions workflows (overlays on top of `k8s-manifests`)
- Deploy lifecycle: push to `main` → `Build k8s-manifests` workflow → `releases/k8s-manifests` → Deploy PR auto-opens → merge → `deploys/k8s-manifests` → `K8s: Deploy k8s-manifests` workflow → `kubectl apply` to cluster.
- The deploy workflow's "Apply manifests: deleted resources" step removes anything that disappears from the projection. Drop a file from the workspace → resource deleted on next deploy.

### Lenses

`.holo/lenses/<name>.toml` describes per-source transformations:

- **helm3** — renders a chart against the app's `release-values.yaml`
- **kustomize** — builds a kustomization
- **k8s-normalize** — routes flat manifests into the `<namespace>/<Kind>/<name>.yaml` layout

Cluster-scoped resources land in `_/<Kind>/<name>.yaml`.

## Directory map

| Path | Purpose |
|---|---|
| `_infra/` | Cluster-level infra (cert-manager issuers, envoy-gateway config, cnpg cluster) |
| `_gateways/` | Per-app Gateway + HTTPRoute pairs, one file per app |
| `<app>/` | App workspace — `release-values.yaml` for helm, `kustomization.yaml` for kustomize |
| `<app>.secrets/` | SealedSecrets for that namespace |
| `.holo/sources/` | Holosource pins (URL + ref) |
| `.holo/branches/<branch>/` | Holomappings — source content → workspace path |
| `.holo/lenses/` | Lens configs |

`_` prefix means "not a workload namespace." Workspace convention; projected tree drops it.

## Standing patterns

Established post-Envoy migration. Mimic these.

### Per-app routing

Each public-facing app gets `_gateways/<app>.yaml` containing:

- `Gateway` in the app's namespace with HTTPS listener on the app's hostname, `cert-manager.io/cluster-issuer: letsencrypt-prod` annotation, certificateRef to `<app>-gw-tls`
- `HTTPRoute` with `parentRefs` attached **only** to the per-app Gateway (no `main-gateway`)

HTTP (port 80) is handled globally by `_infra/envoy-gateway/http-redirect.yaml` — a single `HTTPRoute` on `main-gateway` that 301s everything to HTTPS. ACME challenge paths bypass it via Gateway API conflict resolution (cert-manager creates an `Exact`-path HTTPRoute per challenge).

### Cert Secret naming

`<app>-gw-tls` — to avoid collision with legacy `<app>-tls`.

### Database resources (cnpg)

The shared `cloudnative-pg/Cluster/shared-cluster` is the only PostgreSQL cluster. To add a database:

- `Database` CR lives in `cloudnative-pg` namespace (cnpg requires same-namespace reference to the Cluster — not configurable)
- Workspace file can live wherever organizationally makes sense (e.g. `<app>/cnpg/database.yaml`). k8s-normalize routes by `metadata.namespace` at projection time.
- Add a `managed.roles[<app>]` entry to the Cluster CR with `passwordSecret: name: <app>-db-credentials`
- Create the SealedSecret `<app>-db-credentials` in `cloudnative-pg` namespace before cnpg can fully provision the role

### Envoy Gateway

- `EnvoyProxy` resource has `mergeGateways: true` — every Gateway shares one Envoy data plane and one LoadBalancer. **Do not disable.**
- `GatewayClass` is named `eg`
- The shared HTTP `main-gateway` lives in `envoy-gateway-system`; per-app Gateways attach implicitly via the merged data plane.

## Before pushing a PR — required QA

Run a local projection and diff it against the deployed tree. **No PR ships without this.**

```bash
# 1. Commit everything first
git status   # must be clean

# 2. Fetch and project against the deploy branch's layout
git fetch origin
SHA=$(git holo project k8s-manifests-github 2>&1 | tail -1)

# 3. Diff
git diff --name-status origin/deploys/k8s-manifests "$SHA"
git diff --stat origin/deploys/k8s-manifests "$SHA"

# 4. Spot-check content for changed files
git show "$SHA":<path>
```

If using `git holo project --working` to test uncommitted changes, project `k8s-manifests` (not `-github`) and expect the deploy workflow files to show as deletions — they live in `k8s-manifests-github`, not `k8s-manifests`. Harmless noise. Committing first is usually simpler.

The diff is the definitive preview. Read it carefully — admission webhooks add defaults that show up here (cnpg adds `databaseReclaimPolicy: retain`, HTTPRoutes get default `PathPrefix: /` matches, etc.) and side effects of changed helm values can surface as unrelated-looking ConfigMap or Deployment edits (e.g. `ingress.enabled: false` clearing `server.domain` on grafana).

## Common operations

### Add a new app

1. Create workspace dir + resources (chart values or kustomize)
2. Add holomapping at `.holo/branches/k8s-manifests/<app>/`
3. Add lens config at `.holo/lenses/<app>.toml` if applicable
4. Add `_gateways/<app>.yaml` if it needs external HTTPS
5. Run the projection + diff (above)

### Bump an upstream chart version

Edit the version pin in `.holo/sources/<name>.toml` → `git holo source fetch <name>` → project → diff.

For chart versions owned by the upstream chain: bump in cluster-template or civic-cloud → wait for release → bump civic-cloud pin here.

### Disable an Ingress on a helm-managed app

`<app>/release-values.yaml`: `ingress.enabled: false`. If the chart inferred its public hostname from `ingress.hosts[0]` (historical examples: grafana → `grafana.ini.server.domain`, metabase → `MB_SITE_URL`), set it directly via the chart's other values. Verify via render diff.

## Cluster context

Things not in any single grep-able file:

- **`shared-cluster` (cnpg)**: PG 18.0 + PostGIS 3.6.3 + pgvector 0.8.2 + pgaudit 18.0 (available, not preloaded). 2 instances. Image `ghcr.io/cloudnative-pg/postgis:18-3-system-trixie`. Storage class `linode-block-storage-retain`.
- **Envoy LB**: external IP for the cluster. Wildcard DNS `*.sandbox.k8s.phl.io` resolves there. Linode LKE supports LoadBalancer hairpin natively (in-cluster pods can reach the LB external IP).
- **No hairpin-proxy, no ingress-nginx**: both were decommissioned in the May 2026 Envoy migration. Don't reintroduce.
- **DNS for some non-wildcard hostnames** (e.g. `sandbox.balancerproject.org`, `test.pawsdp.org`) was retired during the migration. Those hostnames don't work anymore.

## Guardrails

Take these only with explicit user authorization:

- `kubectl apply/delete/patch` against shared-infra namespaces: `kube-system`, `cert-manager`, `cloudnative-pg`, `envoy-gateway-system`, `sealed-secrets`
- Force-pushes to `releases/k8s-manifests` or `deploys/k8s-manifests`
- Merging upstream release PRs (cluster-template, civic-cloud) — user handles these
- Restarting deployments in shared namespaces
- Modifying `Cluster/shared-cluster` (especially `instances`, storage, `managed.roles`)

Editing workspace files in this repo and opening PRs are fine without per-action approval.

## Known external issues

- **hologit shallow-clone race** ([JarvusInnovations/hologit#450](https://github.com/JarvusInnovations/hologit/issues/450)) — `Build k8s-manifests` intermittently fails with `fatal: shallow file has changed since we read it`. Rerun the workflow.

For repo-local issues, check the open issue list directly — anything I'd list here will rot.

## References

- Migration umbrella: [#130](https://github.com/CodeForPhilly/cfp-sandbox-cluster/issues/130)
- Live-cluster equivalent plan: [cfp-live-cluster#144](https://github.com/CodeForPhilly/cfp-live-cluster/issues/144)
- Upstream cluster-template: <https://github.com/JarvusInnovations/cluster-template>
- civic-cloud cluster-template: <https://github.com/civic-cloud/cluster-template>
- Hologit: <https://github.com/JarvusInnovations/hologit>
