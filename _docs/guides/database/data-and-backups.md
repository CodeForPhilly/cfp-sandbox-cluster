# Data and backups

## Getting a psql shell

Your team's access covers your namespace, not `cloudnative-pg`, so the way in is a throwaway pod in your namespace with the credentials loaded:

```bash
kubectl -n paws-data-pipeline run psql --rm -it --restart=Never --image=postgres:18 \
  --overrides='{"spec":{"containers":[{"name":"psql","image":"postgres:18","stdin":true,"tty":true,
    "command":["psql"],
    "env":[{"name":"PGHOST","value":"shared-cluster-rw.cloudnative-pg.svc.cluster.local"},
           {"name":"PGDATABASE","value":"paws"},
           {"name":"PGUSER","valueFrom":{"secretKeyRef":{"name":"paws-db-credentials","key":"username"}}},
           {"name":"PGPASSWORD","valueFrom":{"secretKeyRef":{"name":"paws-db-credentials","key":"password"}}}]}]}}'
```

The same pod with `["sleep","3600"]` as the command is a workbench for `pg_dump` and `pg_restore`; `kubectl cp` files in and out.

## Loading existing data

Dump the old database in custom format, then restore **as your own user**, so you own everything you load:

```bash
# from wherever can reach the old database
pg_dump -Fc --no-owner --no-privileges -h OLDHOST -U OLDUSER -d OLDDB -f app.dump

# from the workbench pod, connected as paws
pg_restore -l app.dump | grep -vE " (EXTENSION|COMMENT - EXTENSION) " > list
pg_restore --no-owner --no-privileges -L list -d paws app.dump
```

The `grep -v EXTENSION` matters: the dump wants to `CREATE EXTENSION`, your user can't, and the extension already exists from `database.yaml`. For a 50 MB database this takes seconds.

Then compare. Use exact counts, not `pg_stat` estimates (statistics aren't carried by a restore):

```sql
select relname, (xpath('/row/c/text()', query_to_xml(
  format('select count(*) as c from %I.%I', schemaname, relname), false, true, '')))[1]::text::int
from pg_stat_user_tables order by 1;
```

Run it on both sides and `diff` the output.

## What the backups give you

- A nightly base backup at 04:00 UTC and continuous WAL archiving in between, so any point in the last 30 days is recoverable.
- Stored in Google Cloud Storage, in a CodeForPhilly GCP project, deliberately not on Linode: they survive losing the Linode account.
- Tested: both clusters have been restored into a scratch cluster and compared table by table.

## What they don't give you

Backups are **per cluster**, not per database. Every database on `shared-cluster` shares one backup stream. Getting *your* database back to last Tuesday means recovering the whole cluster into a temporary cnpg `Cluster` (about two minutes) and dumping your database out of it. That's an operator task: ask in `#civic-cloud` with the time you want, and keep in mind the other tenants' data is in the same recovery.

If you want your own logical dumps too, the workbench pod above plus a `CronJob` in your namespace is all it takes, and `pg_dump` output is the portable form if your project ever leaves.

That's the end of the database section. [Reference](../reference.md) has the directory map and gotchas in one place.
