# karpenter

The Karpenter controller, plus a single default `NodePool`/`EC2NodeClass`. Built in phase 2,
as the deliberately-deferred second half of `infra/modules/eks`'s Karpenter work - that module
only creates the AWS-side IAM/SQS/EventBridge plumbing Karpenter needs; everything here is a
Kubernetes object, which belongs in this repo under `CLAUDE.md`'s hard rule (nothing changes on
the cluster except through a commit here that ArgoCD syncs).

## Layout

- `application.yaml` - a multi-source ArgoCD `Application`: one source is the `karpenter` Helm
  chart straight from AWS's public OCI registry (`public.ecr.aws/karpenter`), the other is this
  repo's own `manifests/` directory.
- `manifests/ec2nodeclass.yaml` - how Karpenter should launch nodes (AMI family, the
  Terraform-created instance profile, subnet/security-group selectors by tag).
- `manifests/nodepool.yaml` - what Karpenter is allowed to launch (instance families, spot
  preferred with on-demand fallback, a small 8 vCPU / 16 GiB cap, aggressive consolidation).

## Pod Identity, not IRSA

The Helm values carry no `eks.amazonaws.com/role-arn` annotation on the service account - the
controller's IAM role is bound to it via EKS Pod Identity
(`infra/modules/eks`'s `aws_eks_pod_identity_association`), keyed on namespace + service
account name alone. That's the whole reason this is two ArgoCD sources instead of one: IRSA
would need the role's ARN (and the AWS account ID inside it) written into this repo's Helm
values, which `CLAUDE.md` rules out.

## `replicas: 1`, not the chart's default of 2

The system node group this runs on (`nodeSelector: node-pool: system` - it can't schedule onto
nodes it provisions itself) is two `t3.small` instances, already tight on EKS's 11-pod-per-node
limit after the EKS system daemonsets and ArgoCD (see `docs/PLAN.md`'s phase 2 notes - this is
exactly the failure mode that broke ArgoCD's first install). One replica fits comfortably in
the headroom that's actually left; two replicas, with the chart's default pod anti-affinity
forcing them onto different nodes, risks one getting stuck `Pending` forever on the node that's
already full.

## Status

Written, not yet synced - the root Application (`infra/modules/bootstrap`) only watches
`platform/apps`, which this directory is now under, so the next ArgoCD reconciliation picks it
up on its own. No `kubectl apply` needed.
