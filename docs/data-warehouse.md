# Data Warehouse

## Access

### Metabase

Metabase is available at <https://metabase.sandbox.k8s.phl.io/>

To be invited, post a request including your email address to the [`#cfp-data-pipeline` Slack channel](https://codeforphilly.org/chat/cfp-data-pipeline)

### PostgreSQL Admin

The PostgreSQL database port is exposed publically via a `NodePort` service, which means that port number is open on all nodes in the cluster and automatically proxied to PostgreSQL's port regardless of which node it is actually running on.

As of this writing, one such IP is `45.33.79.76` but this is subject to change when that node is eventually recycled, so the warehouse could be connected to for administration with a command like:

```bash
psql -h 45.33.79.76 -p 30432 -U admin -d warehouse
```

The password can be found in VaultWarden under the `cfp-data-pipeline` organization within the item `admin @ postgresql database`.

## Initial setup

1. Create `warehouse` database in PostgreSQL:

    ```sql
    CREATE DATABASE warehouse;
    ```

2. Create `readonly` warehouse role:

    ```sql
    CREATE ROLE readonly;

    -- Grant access to existing tables
    GRANT USAGE ON SCHEMA public TO readonly;
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

    -- Grant access to future tables
    ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;
    ```

3. Set up user for `metabase`:

    ```sql
    CREATE USER metabase WITH ENCRYPTED PASSWORD 'GENERATE_A_PASSWORD_AND_SAVE_IN_VAULT';
    GRANT readonly TO metabase;
    ```
