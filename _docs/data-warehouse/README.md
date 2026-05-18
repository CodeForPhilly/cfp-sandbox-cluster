# Data Warehouse

## Access via Metabase

Metabase is available at <https://metabase.sandbox.k8s.phl.io/>

To be invited, post a request including your email address to the [`#cfp-data-pipeline` Slack channel](https://codeforphilly.org/chat/cfp-data-pipeline)

## Access via psql

The PostgreSQL database port is exposed publically via a `NodePort` service, which means that port number is open on all nodes in the cluster and automatically proxied to PostgreSQL's port regardless of which node it is actually running on.

As of this writing, one such IP is `45.33.79.76` but this is subject to change when that node is eventually recycled, so the warehouse could be connected to for administration with a command like:

```bash
psql -h 45.33.79.76 -p 30432 -U admin -d warehouse
```

The password can be found in VaultWarden under the `cfp-data-pipeline` organization within the item `admin @ postgresql database`.
