# demo-spring-app

Deploys the image `demo-spring-app` (its own repo - https://github.com/liambarkley/demo-spring-app)
publishes, backed by `infra/modules/rds`'s Postgres instance. Phase 3.

The app's actual job - proving the Build Cache Node works - happens in that repo's CI, not
here. This deployment exists for architectural completeness: a real workload, deployed by
ArgoCD, talking to a real database, the same way `build-cache-node` is.

## Layout

- `application.yaml` - the ArgoCD Application, plain manifests.
- `manifests/serviceaccount.yaml` - bound to a Pod Identity role (`infra/modules/eks`'s
  `aws_eks_pod_identity_association.demo_app`) scoped to exactly two things: reading the RDS
  address and secret-ARN SSM parameters, and reading the RDS master password itself.
- `manifests/deployment.yaml` - an init container that fetches both at pod startup (never
  anything written here), then the app container.
- `manifests/service.yaml` - `ClusterIP`.

## Why an init container, not a Secrets Store CSI Driver or similar

Simpler, and this project already had the pattern from `build-cache-node`'s config file
problem: fetch what's needed into files on a shared `emptyDir`, let the main container read
them. Installing another cluster-wide controller just for one app's two values wasn't
proportionate.

## No environment-specific value here, deliberately

`DB_ADDRESS_PARAM` and `DB_SECRET_ARN_PARAM` are fixed, deterministic SSM parameter *names*
(built only from `infra/envs/dev`'s environment name, `build-cache-dev`) - never the RDS
instance's actual address, or the Secrets Manager secret's own ARN. Both of those are
AWS-generated and change every time `infra/envs/dev` is destroyed and recreated; the fixed
parameter names don't. See `infra/modules/rds`.

## GHCR image visibility

`demo-spring-app`'s GitHub Container Registry package needs to be public for the cluster to
pull it with no credentials - it's private by default even though the repo itself is public.
One-time setting, not something Terraform or this repo manages.
