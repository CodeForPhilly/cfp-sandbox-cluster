# Provision a database

A user and an empty database on `shared-cluster`, in four files in the cluster repo. None of them is in `_infra/`. (New here? Start with the [overview](README.md).)

## 1. One password, sealed twice

Your Postgres user needs a password. cnpg reads it from a Secret in `cloudnative-pg` to set the role's password; your app reads it from a Secret in your own namespace. Same password, two SealedSecrets.

Run this with `KUBECONFIG` pointing at the cluster you're adding to (sandbox first); sealed values only decrypt on the cluster that sealed them.

```bash
APP=example-app; ROLE=example
PW=$(openssl rand -base64 48 | tr -dc A-Za-z0-9 | head -c 40)
for ns in cloudnative-pg $APP; do
  kubectl create secret generic $ROLE-db-credentials -n $ns \
    --type kubernetes.io/basic-auth \
    --from-literal=username=$ROLE --from-literal=password="$PW" \
    --dry-run=client -o yaml \
  | kubeseal --controller-name sealed-secrets --controller-namespace sealed-secrets --format yaml \
  > $ns.secrets/$ROLE-db-credentials.yaml
done
unset PW
```

That writes `cloudnative-pg.secrets/example-db-credentials.yaml` and `example-app.secrets/example-db-credentials.yaml`. You never need to write the password down; once deployed, anyone with access to your namespace can read it back from the Secret.

## 2. The user: `example-app.cnpg/role.yaml`

A `DatabaseRole` is your Postgres user. cnpg creates it on `shared-cluster` and keeps it matching this spec, password included.

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: DatabaseRole
metadata:
  name: example
  namespace: cloudnative-pg        # required: cnpg wants this next to the Cluster
spec:
  cluster:
    name: shared-cluster
  name: example
  login: true
  connectionLimit: 20              # the cluster allows 100 in total; be a good neighbour
  passwordSecret:
    name: example-db-credentials
  databaseRoleReclaimPolicy: retain   # deleting this file does not drop the role
```

## 3. The database: `example-app.cnpg/database.yaml`

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: example
  namespace: cloudnative-pg
spec:
  cluster:
    name: shared-cluster
  name: example
  owner: example
  extensions:                      # only if you need them
    - name: postgis
      ensure: present
```

!!! note "Extensions go here, not in your migrations"
    Your role is not a superuser, so `CREATE EXTENSION` in a migration will fail. Declare the extension on the `Database` and the operator installs it before your app connects. Django, Rails, Prisma and the rest are happy once it exists.

## 4. Open the PR

Commit the four files, run the projection diff ([how](../how-deploys-work.md#seeing-the-projection-yourself)), and open the PR. You should see exactly four additions, all under `cloudnative-pg/`: a `SealedSecret`, a `DatabaseRole`, a `Database`, and a second `SealedSecret` under your namespace. Merge it and its deploy PR.

Then check it took:

```bash
kubectl -n cloudnative-pg get databaserole example -o jsonpath='{.status}'
kubectl -n cloudnative-pg get database example -o jsonpath='{.status.applied}{"\n"}'
```

You now have an empty database named `example`, owned by a user named `example`, reachable from inside the cluster at `shared-cluster-rw.cloudnative-pg.svc.cluster.local`. Nothing uses it yet.

!!! warning "Secret before role"
    If cnpg reconciles the `DatabaseRole` before its Secret exists, it marks the role failed and does not retry on its own. Landing both in one PR is enough. If it happens anyway: `kubectl annotate cluster -n cloudnative-pg shared-cluster cnpg.io/reconciliationLoop="$(date +%s)" --overwrite`, and check `.status` again.

Next: [Connect your app](connect.md).
