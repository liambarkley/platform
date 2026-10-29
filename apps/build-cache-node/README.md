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

Deployed, internal-only still (the `Ingress` for ALB exposure is written in
`manifests/ingress.yaml` but not yet applied - needs the AWS Load Balancer Controller's IAM
Terraform applied first). Confirmed functionally working - not just "pod is Running":
`PUT`/`GET` against `/cache/<hash>` from inside the cluster round-tripped real data (Gradle's
documented HTTP build cache protocol), both returning `200`, and `/ping` (the documented,
always-unauthenticated health endpoint - confirmed directly, since every other path including
`/` returns `401` without real credentials) is what both the pod's own probes and the Ingress's
ALB health check use.

Getting the StatefulSet healthy took three real, serial bugs on the live pod - the app
rejecting a `ConfigMap`-mounted config file as read-only, the fix for that silently not
applying because the StatefulSet controller stalled mid-rollout (needed a manual `kubectl
delete pod` to unstick, not a manifest change), and the access-control warning itself. None
affected the StorageClass/PVC/image choices, which worked on the first attempt. No `TLS` -
plain HTTP on the ALB's own AWS-generated DNS name, since this project has no domain to get an
ACM certificate for; a deliberate, documented trade-off (credentials travel in cleartext), not
an oversight.
