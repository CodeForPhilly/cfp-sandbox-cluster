# Philly Civic Cloud

CodeForPhilly runs two Kubernetes clusters on Linode, **sandbox** and **live**, and each is described completely by a git repo. Change the repo, and a pipeline changes the cluster; nothing is applied by hand, and nothing runs that isn't in the repo. Shared pieces (TLS, routing, PostgreSQL, backups) are run once for everyone, so a project brings its container image and a few files, and gets a hostname, a certificate, a database and a backup without building any of it.

| | Sandbox | Live |
|---|---|---|
| Repo | [cfp-sandbox-cluster](https://github.com/CodeForPhilly/cfp-sandbox-cluster) | [cfp-live-cluster](https://github.com/CodeForPhilly/cfp-live-cluster) |
| Hostnames | `*.sandbox.k8s.phl.io` | `*.live.k8s.phl.io`, plus real domains |
| Purpose | land it here first | production |

## Where to start

**I'm on a project team and want my app running here.**
Start with the [Guides](guides/): how deploys work, where your project lives in the repo, and how to add and use a database. About twenty minutes, in order.

**My app is already here and I need a database.**
Skip to [Add a database](guides/add-a-database.md) and [Connect your app](guides/connect-your-app.md). Four files, no shared infrastructure touched.

**I need to change a value, bump a version, or rotate a secret.**
[Your project in the repo](guides/your-project-in-the-repo.md) shows which files are yours, and [How deploys work](guides/how-deploys-work.md) shows how to preview the change before it lands.

**I'm operating the cluster.**
[Operations](operations/) covers upgrades, the echo service for checking connectivity, and the shared components. The agent-facing notes in each repo's `.claude/CLAUDE.md` hold the current state of the cluster and the hard-won lessons.

**I want to understand the pieces.**
[Development](development/) describes the architecture inherited from the upstream cluster template; [Reference](guides/reference.md) is the one-page directory map and gotcha list for these clusters.

Questions go to `#civic-cloud` on the Code for Philly Slack.
