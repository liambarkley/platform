# Postmortem: five real bugs between "cluster created" and "ArgoCD + Karpenter healthy"

**Date:** October 29, 2026
**Status:** Resolved
**Severity:** Low (build-time only - no live service impact; the project has no running
workloads yet)

Blameless by design: the point is understanding why each failure happened and why guessing at
a fix (instead of reading the cluster's actual state) would have wasted a cycle, not assigning
fault for the original code.

## Summary

Phase 2's "done when" bar - one `terraform apply` producing a cluster with ArgoCD syncing an
empty `platform/apps` - was met in a single `terraform apply`, with no manual `kubectl apply`
or hand-run `helm install` at any point. Getting there took five separate bugs, found and
fixed one at a time, each confirmed against the live, partially-applied cluster before the fix
was written. None required destroying or recreating the cluster - every fix was either a
Terraform update-in-place or a Kubernetes-side reconciliation once the real blocker was gone.

Same cluster, same session, also covers Karpenter's bring-up (`platform/apps/karpenter`),
which happened right after - the same cluster made that straightforward to debug the same way.

## Stage 1: the `exec` plugin had no credentials of its own

**Symptom:** `helm_release.argocd` failed: *"Kubernetes cluster unreachable: the server has
asked for the client to provide credentials."* The EKS cluster, node group, and every addon had
already applied fine.

**Root cause:** the `kubernetes`/`helm` Terraform providers authenticate via an `exec` plugin
that runs `aws eks get-token` as a plain subprocess. That subprocess only inherits the ambient
shell environment - not the `profile` argument given to this module's own `aws` provider
blocks, and not a `-var` passed to `terraform apply`. With `AWS_PROFILE` unset in the shell,
the subprocess had no credentials to authenticate with.

**Fix:** pass the same credentials (`aws_profile`, or the static triple for CI) into the
`exec` block's own `env` map, so authentication no longer depends on what happens to be
exported in whoever's shell runs `apply`.

## Stage 2: "cluster creator gets admin" didn't actually happen

**Symptom:** same error, after Stage 1's fix.

**Root cause:** `aws eks list-access-entries` on the live cluster showed only the EKS
service-linked role and the Karpenter node role. AWS's docs say
`bootstrap_cluster_creator_admin_permissions` defaults to granting the cluster creator an admin
access entry - empirically, on this apply, it didn't produce one for the identity that actually
ran `terraform apply`.

**Fix:** stopped relying on that implicit mechanism. Added an explicit `aws_eks_access_entry` +
`aws_eks_access_policy_association` (`AmazonEKSClusterAdminPolicy`) for whoever holds the
AdministratorAccess Identity Center permission set, looked up by name pattern via
`data.aws_iam_roles` so the account ID in the matched role's ARN never has to be written into
the repo. Also just better practice regardless of what caused the original gap - a reviewable
resource, not a side effect of who happened to click apply.

## Stage 3: `t3.small`'s 11-pod limit couldn't fit ArgoCD

**Symptom:** `helm_release.argocd` got created this time, but timed out waiting for health.
`argocd-server`/`repo-server`/`notifications-controller` stuck `Pending`
(`FailedScheduling: 1 Too many pods`); `dex-server` crash-looping on a secret `argocd-server`
generates on its own first boot - which never happened, since `argocd-server` never started.

**Root cause:** EKS caps pods per node by ENI/IP math, not just CPU/memory - `t3.small` caps
out at 11. The EKS system daemonsets alone (`vpc-cni`, `kube-proxy`, `coredns` x2,
`ebs-csi-controller` x2, `ebs-csi-node`, `pod-identity-agent`) already used 8 of those on the
single system node, leaving 3 free - not enough for `argo-cd`'s 7 pods. CPU and memory were
never the constraint (checked: 9% CPU, 7% memory allocated on the less-full node).

**Fix:** scaled the system node group to 2 nodes (`min`/`desired`/`max` all set to 2 - this
group exists to run ArgoCD and Karpenter's controller, not to autoscale).

## Stage 4: `root-app` was syncing the next app's raw manifests too

**Symptom:** `platform/apps/karpenter`'s first real sync: *"The Kubernetes API could not find
karpenter.k8s.aws/EC2NodeClass ... Make sure the CRD is installed."* Retried repeatedly, never
recovered on its own.

**Root cause:** `root-app`'s directory source recursed into all of `apps/` with no `include`
filter. It therefore applied `apps/karpenter/manifests/*.yaml` (the `EC2NodeClass`/`NodePool`)
directly, in the same sync wave as creating the nested `karpenter` Application that was
supposed to install those CRDs via its Helm source. The CRDs didn't exist yet when `root-app`
tried to apply objects of that kind.

**Fix:** `directory.include: "*/application.yaml"` on `root-app`'s source. It now only ever
creates per-app Application objects, one per `apps/<name>/` directory - never that app's own
manifests, which belong to the nested Application to sync.

## Stage 5: Karpenter's own controller was missing two IAM permissions

**Symptom:** the `karpenter` Application synced and its pod started - Pod Identity
authentication itself worked immediately, no issue there. But `EC2NodeClass.status.conditions`
showed `AMIsReady=Unknown` ("failed to discover any AMIs for alias `al2023@latest`", no
explicit AWS error attached), and an explicit `AccessDenied` on `iam:ListInstanceProfiles` from
a controller that runs unconditionally regardless of whether any `EC2NodeClass` actually uses
Karpenter-managed instance profiles (this one doesn't - see `infra/modules/eks/README.md`).

**Root cause, two separate gaps in the same IAM policy:**
- `ssm:GetParameter` was scoped to `arn:aws:ssm:*:<our-account-id>:parameter/aws/service/*` -
  the obvious-looking choice. AWS's own EKS-optimized-AMI SSM parameters (what alias resolution
  actually reads) live under an **account-less** ARN namespace
  (`arn:aws:ssm:*::parameter/aws/service/*`). The mismatch didn't produce an explicit
  `AccessDenied` - it just matched nothing, so the query silently returned no AMIs.
- `iam:ListInstanceProfiles`/`GetInstanceProfile`/`TagInstanceProfile` were never granted at
  all - no part of the original policy anticipated the garbage-collection controller running by
  default.

**Fix:** corrected the SSM resource ARN; added the three IAM read actions (deliberately not
`Create`/`Delete`/`AddRoleToInstanceProfile` - the one instance profile that exists is
Terraform's to manage, not Karpenter's).

Confirmed resolved by watching `EC2NodeClass.status.conditions` flip to all-`True`
(`SubnetsReady`/`SecurityGroupsReady`/`InstanceProfileReady` were already `True` before this
stage - the subnet/security-group selectors and instance profile wiring in
`platform/apps/karpenter` were correct from the very first attempt) and the `karpenter`
Application's health going `Healthy`, with no pod restart needed - IAM policy changes take
effect on the next API call, not on a delay.

## Common thread across all five

Every fix came from reading the cluster's actual, current state (`kubectl get`/`describe`,
`aws eks list-access-entries`, controller logs) - never from re-reading documentation harder or
guessing a second time. Two of the five (Stage 2's access entry, Stage 5's SSM ARN) are cases
where AWS's documented default behavior, or the "obvious" way to scope a resource, didn't match
what actually happened - the kind of gap that's expensive to find by inspection and cheap to
find by just asking the live system what it thinks is going on.

## Prevention: how to catch some of these before `apply`

Not implemented yet, kept small and specific rather than "add more tests" generically:

1. **A short runbook step**, not yet written (candidate for `platform/docs/runbooks/`): before
   sizing *any* new EKS node group, check the instance type's max-pods limit
   (`aws ec2 describe-instance-types` or the published `eni-max-pods.txt`) against what's
   actually expected to run there, not just CPU/memory. Stage 3 was entirely avoidable with a
   five-minute check before picking `t3.small`.
2. **A `terraform test` on `modules/eks`** asserting `aws_eks_access_entry.cluster_admin` and
   `aws_eks_access_policy_association.cluster_admin` are always planned for at least one
   principal, so a future change can't silently drop admin access the way the implicit
   mechanism did in Stage 2.
3. **No good automated check for Stage 1 or Stage 4** - both are "the exec plugin's
   environment" and "ArgoCD's directory scanning," neither of which has a meaningful
   plan-time signal in this project's current tooling. Documented in
   `infra/modules/bootstrap/main.tf` and the chart's own comments instead, for the next person
   (or the next phase) to read before changing either.
