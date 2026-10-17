# Deploying your project on the Civic Cloud

This is the path for a project team bringing an app onto a CodeForPhilly cluster, or adding something to an app that's already here. It assumes you know your own app and Docker, and nothing about this repo.

There are two clusters, each with its own repo: **sandbox** ([cfp-sandbox-cluster](https://github.com/CodeForPhilly/cfp-sandbox-cluster), `*.sandbox.k8s.phl.io`) and **live** ([cfp-live-cluster](https://github.com/CodeForPhilly/cfp-live-cluster), `*.live.k8s.phl.io` and real domains). They have the same layout. Everything lands on sandbox first.

Read in order the first time; each page is short.

1. **[How deploys work](how-deploys-work.md)**: nothing is applied by hand. You change files, a pipeline turns them into manifests, and a second pull request shows you exactly what will change before it does.
2. **[Your project in the repo](your-project-in-the-repo.md)**: the handful of files that make up a project here, which ones you own, and the one rule about where raw manifests can go.
3. **[Add a database](add-a-database.md)**: a PostgreSQL user and database on the shared cluster, in four files, none of them shared infrastructure.
4. **[Connect your app](connect-your-app.md)**: reading the connection settings and password, for helm and for kustomize apps, and the order to land things in.
5. **[Data and backups](data-and-backups.md)**: getting a `psql` shell, loading existing data, and what the backups do and don't give you.
6. **[Reference](reference.md)**: the directory map and the gotchas, for when you already know the shape.

The running example is **paws-data-pipeline**, which is moving from a Postgres container inside its own pod to the shared cluster. Substitute your app's name.

If you get stuck, ask in `#civic-cloud` on the Code for Philly Slack.
