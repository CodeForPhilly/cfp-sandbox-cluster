# Shared PostgreSQL

Both clusters run one PostgreSQL cluster for everyone, `shared-cluster` in the `cloudnative-pg` namespace, managed by [CloudNativePG](https://cloudnative-pg.io). Projects get their own user and database on it rather than running Postgres themselves.

**What you get**

- PostgreSQL 18 with PostGIS and pgvector available, two instances with automatic failover.
- Reachable from inside the cluster at `shared-cluster-rw.cloudnative-pg.svc.cluster.local:5432` (`-ro` for a read replica).
- Nightly backups plus continuous archiving, kept 30 days, stored off-Linode in Google Cloud Storage, and tested by restore.
- Operator upgrades, image pinning and certificates handled centrally in `_infra/cloudnative-pg/`.

**The three steps**

1. **[Provision a database](provision.md)**: four files in the cluster repo give you a user and an empty database. None of them touches shared infrastructure.
2. **[Connect your app](connect.md)**: read host, database, user and password from configuration, for helm and for kustomize apps, and land the change in the right order.
3. **[Data and backups](data-and-backups.md)**: get a `psql` shell, load existing data, and know what the shared backups do and don't cover.

The running example is a made-up `example-app` with a Postgres user and database both named `example`. Substitute your app's name.
