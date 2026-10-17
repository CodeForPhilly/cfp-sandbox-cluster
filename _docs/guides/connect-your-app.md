# Connect your app

Your app needs five things: host, port, database, user, password. Four are plain configuration; the password comes from the Secret you sealed into your namespace.

```
host      shared-cluster-rw.cloudnative-pg.svc.cluster.local   (-ro for a read replica)
port      5432
database  paws
user      paws
password  Secret paws-db-credentials, key "password"
```

## Helm apps

Make the chart read these from values, with the password from a Secret. In `templates/deployment.yaml`:

```yaml
env:
  - name: PGHOST
    value: {{ .Values.database.host }}
  - name: PGDATABASE
    value: {{ .Values.database.name }}
  - name: PGUSER
    value: {{ .Values.database.user }}
  - name: PGPASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ .Values.database.passwordSecret }}
        key: password
```

If the chart currently runs its own Postgres container (PAWS did, as a sidecar in the app pod), remove it, or keep it behind a `database.embedded: true` default so local dev is unchanged. Cut a release and bump `.holo/sources/<app>.toml` to the new tag.

Then set the values in the cluster repo, `paws-data-pipeline/release-values.yaml`:

```yaml
database:
  embedded: false
  host: shared-cluster-rw.cloudnative-pg.svc.cluster.local
  name: paws
  user: paws
  passwordSecret: paws-db-credentials
```

## Kustomize apps

See `balancer/` for a worked example: a `balancer.env` file rendered into a ConfigMap for the non-secret settings, the password merged into the app's existing SealedSecret with `kubeseal --merge-into`, and the `cnpg/` files included from the kustomization. One thing to know: a ConfigMap generated with a stable name does not restart your pod when it changes, so balancer carries a pod-template annotation it bumps on purpose.

## Order of operations

1. Database PR (previous page). Empty database exists.
2. App change in your repo, tagged; pin bump here.
3. [Load your data](data-and-backups.md#loading-existing-data) if you have any.
4. Values PR. The app rolls onto the shared cluster.

Steps 1 and 4 are small PRs in the cluster repo; step 2 is in yours. Doing it on sandbox first means step 4 on live is a copy.

!!! note "Expect one restart if you change a sealed password"
    The deploy applies your SealedSecret and your Deployment within the same second, and the sealed-secrets controller takes about a second to write the Secret. A pod that starts in that gap reads the old value and restarts once. Harmless, and not a sign something is wrong.

Next: [Data and backups](data-and-backups.md).
