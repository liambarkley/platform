# build-cache-node

The Develocity Build Cache Node - the service this whole project is built around. Phase 3.

## Licence terms (the open question from `docs/PLAN.md`, now resolved)

Free to self-host, standalone, no license key or Develocity account registration required for
basic operation - confirmed against the official docs before writing any of this. Image:
`gradle/build-cache-node:21.2` on Docker Hub. Registering a node with a full Develocity
installation is optional and only unlocks extra features (centralized monitoring, cache entry
replication across multiple nodes) that this project doesn't use.

**Known, deliberate limitation**: the Build Cache Node is deprecated - no longer distributed
or supported after December 31, 2026, with Develocity Edge as the stated successor. Used here
anyway: it's still freely available today, works for this project's timeline, and the
deprecation itself is a reasonable thing to be able to speak to directly (what it is, why it's
EOL, what the real migration path would be) rather than a gap to hide. Documented in the main
README's "honest limits" too.

## Layout

- `application.yaml` - the ArgoCD Application, plain manifests (no Helm chart exists for this).
- `manifests/storageclass.yaml` - a `gp3` StorageClass using the EBS CSI driver
  (`ebs.csi.aws.com`). Needed because EKS's own default `gp2` StorageClass uses the legacy
  in-tree provisioner, which no longer actually works on this cluster's Kubernetes version
  (1.31) - confirmed present-but-broken via `kubectl get storageclass` before writing this.
- `manifests/statefulset.yaml` - a single replica (this project doesn't use the
  multi-node-replication feature, which needs Develocity registration anyway), with a 10Gi
  `gp3` volume matching the node's own default max cache size.
- `manifests/service.yaml` - `ClusterIP`, internal only for now.

## Access control

`manifests/configmap.yaml` sets `anonymousLevel: read` - anyone can pull cache entries with no
credentials (normal for a build cache; a PR build from a fork shouldn't need a secret just to
read). Writing requires the `ci` user, whose password hash is in that ConfigMap.

**How that hash was generated, for the record (and for rotating it later) - no plain password
ever appears in this log, by design:**

1. Liam generated a password locally: `printf '%s%s!\n' "$(openssl rand -hex 16)" "$(openssl
   rand -hex 1 | cut -c1 | tr 'abcdef' '123456')"` - 32 hex chars plus a digit and `!` to
   satisfy typical complexity rules.
2. Hashed it on the live pod, interactively, so the plain password was only ever typed into a
   real terminal prompt - never piped, echoed, logged, or seen by Claude Code:
   ```
   kubectl exec -it build-cache-node-0 -n build-cache-node -c build-cache-node -- \
     java -jar /build-cache-node.jar hash
   ```
   (Piping the password via `echo | kubectl exec -i` does not work here - the tool uses Java's
   `Console.readPassword()`, which requires a real TTY and silently fails against piped stdin.
   `-it`, not `-i`, is the fix.)
3. The command printed a `"salt:hash"` string - that's what's in `manifests/configmap.yaml`.
   It's a one-way hash, safe to commit.
4. The plain password itself was never pasted to Claude Code and isn't recorded in this repo
   anywhere - Liam has it, for a future GitHub Actions secret (the CI workflow step, not done
   yet).

To rotate: repeat steps 1-3 with a new password, replace the hash in the ConfigMap, update
whatever secret store CI ends up using.

Before this, briefly (while the `Service` was still `ClusterIP`-only, nothing outside the
cluster could reach it), the config ran with `anonymousLevel: readwrite` just to validate the
cache worked at all - gone now, replaced by the above.

## Status

**Live and internet-reachable**, confirmed end-to-end from outside the cluster entirely (not
`kubectl exec`, an actual `curl` from outside AWS) against the ALB's own DNS name:
- `GET /ping` → `200 SUCCESS`, unauthenticated.
- `GET /cache/<hash>` with no credentials → `200`, returning real previously-cached data -
  anonymous read works.
- `PUT /cache/<hash>` with no credentials → `403` - anonymous write is correctly blocked.

No `TLS` - plain HTTP on the ALB's own AWS-generated DNS name, since this project has no
domain to get an ACM certificate for; a deliberate, documented trade-off (the `ci` user's
credentials travel in cleartext), not an oversight.

Real bugs hit getting here, beyond the StatefulSet ones (ConfigMap-mounted file rejected as
read-only, a stalled rollout needing a manual pod delete, the access-control warning):
1. The AWS Load Balancer Controller crash-looped: `failed to fetch VPC ID from instance
   metadata ... context deadline exceeded`. Root cause confirmed via `aws ec2
   describe-instances`: EKS's default launch template caps IMDS at one network hop, but a pod
   is one hop further out than the host - fixed with a custom launch template on the system
   node group (`infra/modules/eks`) raising it to 2, applied to Karpenter's `EC2NodeClass` too.
2. That fix forced a full node group replacement (Terraform's `aws_eks_node_group` treats
   adding a `launch_template` as `ForceNew`, even though the underlying AWS API supports an
   in-place update) - temporary downtime for everything scheduled there, recovered on its own
   once the new nodes came up.
3. Right after the new nodes came up, the Ingress briefly failed with a TLS error calling the
   controller's own admission webhook (`certificate signed by unknown authority`) - transient,
   self-resolved within a couple of minutes without intervention, consistent with the
   controller's webhook cert still catching up after its own restart.
