# platform

What ArgoCD deploys onto the EKS cluster for the Build Cache Platform SRE portfolio project.

Full story: see `docs/PLAN.md` in the parent `sre_portfolio/` workspace (not part of this repo).

`infra`'s `bootstrap` module installs ArgoCD and points it at this repo's `apps/` directory
(phase 2) — `apps/karpenter/` is the first thing actually in it.

## Layout

- `apps/karpenter/` — the Karpenter controller + a default `NodePool`/`EC2NodeClass` (phase 2,
  deferred from `infra/modules/eks` until there was a live cluster to validate against).
- `apps/aws-load-balancer-controller/` — provisions ALBs for `Ingress` objects (phase 3, added
  to expose `build-cache-node` to the internet).
- `apps/build-cache-node/` — the Develocity Build Cache Node (phase 3), live and
  internet-reachable via an ALB.
- `apps/demo-spring-app/` — Spring Boot app, deployed from the separate `demo-spring-app` repo's
  published image, backed by `infra/modules/rds`'s Postgres (phase 3). Its own CI is where the
  cache actually gets exercised - this just deploys the result.
- `apps/monitoring/` — kube-prometheus-stack + dashboards (phase 4).
- `apps/velero/` — scheduled backups to S3 (phase 5).
- `docs/runbooks/`, `docs/postmortems/`, `docs/slos.md` — written artifacts (phases 4-6).

## Hard rule

Nothing changes on the cluster except through a commit here that ArgoCD syncs. No `kubectl apply`
or `helm install` by hand, except inside `infra`'s bootstrap module.

## Git hooks

This repo is public. `scripts/git-hooks/pre-push` runs `gitleaks` against the full history plus
two custom checks (a tracked `.tfstate`/`.tfvars` file, a bare 12-digit AWS account ID) on every
push - see `infra/README.md`'s "Git hooks" for the full rationale, which applies here too even
though this repo hasn't needed either check yet.

Not wired up by git on its own - each clone needs to opt in once:

```
git config core.hooksPath scripts/git-hooks
```

Requires `gitleaks` (`brew install gitleaks`).
