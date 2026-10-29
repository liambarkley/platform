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

## Access control - temporary, read before touching this

`manifests/configmap.yaml` sets `anonymousLevel: readwrite` right now. That's only acceptable
because the `Service` is `ClusterIP` - nothing outside the cluster can reach it. The node's own
startup log says why this can't stay that way: *"anonymous access ... may allow anyone to write
malicious cache entries."* Before any ALB/Ingress makes this internet-reachable, this needs
real user credentials instead (`cache.accessControl.users`, a salted password hash via `java
-jar build-cache-node-21.2.jar hash`, the password itself in a Secret - never committed).

## Status

Deployed, internal-only, confirmed functionally working - not just "pod is Running":
`PUT`/`GET` against `/cache/<hash>` from inside the cluster round-tripped real data
(Gradle's documented HTTP build cache protocol), both returning `200`. **Not yet reachable
from GitHub Actions** - that needs an ALB/Ingress (the AWS Load Balancer Controller isn't
installed yet either) and the real auth described above. Tracked as the next concrete step in
`docs/PLAN.md`'s phase 3 section, not done here.

Getting here took three real, serial bugs on the live pod - the app rejecting a `ConfigMap`-
mounted config file as read-only, the fix for that silently not applying because the
StatefulSet controller stalled mid-rollout (needed a manual `kubectl delete pod` to unstick,
not a manifest change), and the access-control warning itself. None affected the
StorageClass/PVC/image choices, which worked on the first attempt.
