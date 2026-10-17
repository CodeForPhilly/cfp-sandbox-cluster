# Reference

## Directory map

| Path | What it is | Rule |
|---|---|---|
| `<app>/` | The app's workspace: `release-values.yaml` (helm) or `kustomization.yaml` (kustomize) | **Lens input.** Only the chart values or kustomization belong here; other files are dropped by the render. |
| `<app>.secrets/` | SealedSecrets for the app's namespace | Raw manifests. Seal against the cluster you're deploying to. |
| `<app>.cnpg/` | The app's `DatabaseRole` + `Database` on `shared-cluster` | Raw manifests; declare `namespace: cloudnative-pg`. A kustomize app may use `<app>/cnpg/` instead. |
| `_infra/` | cert-manager, Envoy Gateway, the cnpg operator and `shared-cluster`, backups | Shared. Projects don't edit it; in particular, **no per-tenant `managed.roles`** on the Cluster. |
| `_gateways/<app>.yaml` | Public hostname(s), TLS, and for helm apps the HTTPRoute | One per app. |
| `_admins/<app>.yaml` | Namespace + `deployment-admin` ServiceAccount/Role | Scoped to the app's namespace. |
| `_docs/` | This site | Published on every push. |
| `.holo/sources/<name>.toml` | Upstream repo + ref | Pin to a tag. |
| `.holo/branches/k8s-manifests/<app>/` | Which upstream files land in `<app>/` | |
| `.holo/lenses/<app>.toml` | How `<app>/` renders | |

The `_` prefix means "not a workload namespace" and is dropped in the projected tree. The `.suffix` directories exist because `<app>/` is spoken for by the lens.

## shared-cluster facts

| | |
|---|---|
| Service | `shared-cluster-rw.cloudnative-pg.svc.cluster.local:5432` (`-ro` for replicas) |
| Version | PostgreSQL 18, pinned by image digest in `_infra/cloudnative-pg/shared-cluster.yaml`; PostGIS 3.6 and pgvector 0.8 available |
| Instances | 2, automatic failover |
| Connections | `max_connections` 100; each role gets `connectionLimit: 20` |
| Backups | Barman Cloud plugin → `gs://cnpg-<cluster>`; nightly 04:00 UTC + WAL; 30-day retention |
| Operator | CloudNativePG, chart in `_infra/cloudnative-pg/operator` |

## Gotchas

- **Secret before role.** A `DatabaseRole` reconciled before its `passwordSecret` exists is marked failed and not retried. Same PR is enough; otherwise nudge with `kubectl annotate cluster -n cloudnative-pg shared-cluster cnpg.io/reconciliationLoop="$(date +%s)" --overwrite`.
- **Pod before Secret.** A changed SealedSecret and a Deployment in the same deploy can race by a second; the pod restarts once.
- **Not a superuser.** Extensions go on the `Database`, not in migrations.
- **Statistics aren't restored.** `n_live_tup` reads 0 after a physical restore; compare with `count(*)`.
- **`DEBUG=false` is true to Django** when the app does `bool(os.environ.get("DEBUG"))`; set it empty instead.
- **Deleting a file deletes the resource.** Including your database objects (the role itself survives thanks to `retain`; the `Database` object's data does too, but check before you clean up).
