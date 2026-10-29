# aws-load-balancer-controller

Watches `Ingress` objects and provisions ALBs for them - needed to expose
`apps/build-cache-node` to GitHub Actions. Phase 3.

## Pod Identity, not IRSA

Same reasoning as `apps/karpenter`: the controller's IAM role is bound to its service account
via EKS Pod Identity (`infra/modules/eks`'s `aws_eks_pod_identity_association`), so no role ARN
(and the account ID inside it) needs to live in this repo's Helm values.

## IAM policy

`infra/modules/eks/policies/aws-load-balancer-controller.json` is the exact, unmodified policy
AWS publishes for this controller - kept as a static file rather than hand-written in HCL, so
it's trivially diffable against upstream if it ever changes.

## `replicaCount: 1`, no node pinning

Same class of constraint as Karpenter's `replicas: 1` - the system node group had essentially
no free pod slots left when this was written (confirmed via `kubectl get pods -A`). Unlike
Karpenter, though, this controller doesn't provision its own capacity, so there's no
bootstrapping circularity in letting Karpenter provision a node for it if nothing existing
fits - no `nodeSelector` here, deliberately.

## Status

Written, not yet applied - needs `infra/modules/eks`'s IAM/Pod Identity Terraform applied
first (Liam's call, not Claude Code's).
