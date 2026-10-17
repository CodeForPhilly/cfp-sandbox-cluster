# How deploys work

The short version: **you edit files on `main`, and a pipeline applies them to the cluster.** Nobody runs `kubectl apply` by hand, and nothing is in the cluster that isn't in the repo.

## Workspace, projection, deploy

The files on `main` are the **workspace**. They aren't Kubernetes manifests yet; they're the inputs: a pin to your app's repo and tag, your helm values or kustomization, your sealed secrets.

A tool called [hologit](https://github.com/JarvusInnovations/hologit) **projects** the workspace into plain manifests. It fetches your app's chart or kustomization from the pinned tag, renders it with your values, and files every resource into a predictable path: `<namespace>/<Kind>/<name>.yaml`. That rendered tree is what gets applied.

```
you push to main
  → Build workflow projects the tree          → branch  releases/k8s-manifests
  → a "Deploy releases/k8s-manifests …" PR opens by itself, with a kubectl diff in its body
  → someone merges it                          → branch  deploys/k8s-manifests
  → Deploy workflow runs kubectl apply
```

Two things follow from this that are worth internalising on day one:

!!! note "The deploy PR is your preview"
    Its body is `kubectl diff` against the real cluster: every line that will change. Read it even when your change is one value. Lenses and admission webhooks add things you didn't type, and this is where you see them.

!!! warning "Deleting a file deletes the resource"
    The deploy's last step removes anything that disappeared from the projected tree. Delete your app's directory and the app is gone on the next deploy. This is a feature, but it means "clean up this old file" is a production change.

## Seeing the projection yourself

You don't have to wait for the pipeline. From a clone with `git holo` installed:

```bash
git fetch origin
SHA=$(git holo project k8s-manifests-github 2>&1 | tail -1)
git diff --name-status origin/deploys/k8s-manifests "$SHA"   # which resources change
git show "$SHA":balancer/Deployment/balancer.yaml           # look at any one of them
```

Every PR to these repos includes that diff in its description. It's the one review step that catches a values typo before it becomes a deploy.

## Sandbox first

Both clusters have the same layout on purpose. The habit is: land the change on sandbox, look at it running, then open the same change on live. Most live PRs are a sandbox PR with a hostname changed.

Next: [Your project in the repo](your-project-in-the-repo.md).
