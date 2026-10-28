# platform

What ArgoCD deploys onto the EKS cluster for the Build Cache Platform SRE portfolio project.

Full story: see `docs/PLAN.md` in the parent `sre_portfolio/` workspace (not part of this repo).

Empty until phase 2 (`infra`'s `bootstrap` module installs ArgoCD and points it at this repo) —
that's intentional, not a gap.

## Layout

- `apps/build-cache-node/` — the Develocity Build Cache Node (phase 3).
- `apps/demo-spring-app/` — Spring Boot app built with Gradle, exercises the cache (phase 3).
- `apps/monitoring/` — kube-prometheus-stack + dashboards (phase 4).
- `apps/velero/` — scheduled backups to S3 (phase 5).
- `docs/runbooks/`, `docs/postmortems/`, `docs/slos.md` — written artifacts (phases 4-6).

## Hard rule

Nothing changes on the cluster except through a commit here that ArgoCD syncs. No `kubectl apply`
or `helm install` by hand, except inside `infra`'s bootstrap module.
