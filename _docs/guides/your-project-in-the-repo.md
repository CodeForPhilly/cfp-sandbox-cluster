# Your project in the repo

A project is a small set of files in predictable places. Here they are for a made-up `example-app`; yours look the same with the name swapped.

```
.holo/sources/example-app.toml          ← where your code comes from, pinned to a tag
.holo/branches/k8s-manifests/example-app/helm-chart.toml
.holo/lenses/example-app.toml           ← how it's rendered (helm or kustomize)
example-app/release-values.yaml         ← your helm values (or kustomization.yaml)
example-app.secrets/*.yaml              ← SealedSecrets for your namespace
example-app.cnpg/*.yaml                 ← your database user + database (next page)
_gateways/example-app.yaml              ← your public hostname and TLS
_admins/example-app.yaml                ← namespace + a deployment-admin role for your team
```

## What you own

Everything with your app's name in it is yours to change through a PR. The three `.holo/` files rarely change after setup; `release-values.yaml`, `.secrets/` and `.cnpg/` are where day-to-day changes happen.

Directories starting with `_` (`_infra/`, `_gateways/`, `_admins/`) are cluster-wide. You own your file inside `_gateways/` and `_admins/`; `_infra/` (cert-manager, Envoy Gateway, the Postgres operator) you should never need to touch.

## The one rule about `<app>/`

`example-app/` is the **input to a lens**. For a helm app that means: the chart is fetched into it from your repo, `release-values.yaml` is applied, and the rendered output *replaces* the directory in the projected tree. Anything else you put in that directory is not rendered and silently disappears.

That's why things that belong to your app but aren't part of the chart live in **sibling directories** with a suffix: `example-app.secrets/` and `example-app.cnpg/`. Those are raw manifests, applied as written, and routed to the right namespace by the `namespace:` they declare.

(A kustomize app can get away with a subdirectory instead, because kustomize passes raw resources through; balancer keeps its database files in `balancer/cnpg/` and lists them in its kustomization. Either way is fine; the sibling directory works for both.)

## Pins

`.holo/sources/example-app.toml` names a git ref in your repo. **Make it a tag.** Tracking `refs/heads/master` means a merge in your repo can change what's deployed, or break the cluster build, with no PR here to review it. Cut a release, bump the pin, open the PR: that's a deploy.

## Your namespace

Your app runs in a namespace with its name. `_admins/<app>.yaml` gives your team a `deployment-admin` ServiceAccount there, enough to see pods, read logs, `exec` and port-forward. It does not reach other namespaces, which matters for the database pages: your Postgres lives in `cloudnative-pg`, and you reach it from inside your own namespace.

Next: [Shared PostgreSQL](database/), starting with [Provision a database](database/provision.md).
