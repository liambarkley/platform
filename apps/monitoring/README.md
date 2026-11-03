# monitoring

Phase 4 (Observability and SLOs). `kube-prometheus-stack` plus the pieces it can't provide on
its own: a CloudWatch exporter standing in for the Build Cache Node's missing metrics, two SLO
burn-rate `PrometheusRule`s, and the Alertmanager wiring that turns a threatened SLO into a
real email.

## Why there's a CloudWatch exporter at all

The Build Cache Node has no Prometheus endpoint - confirmed against the official BCN 21.2
manual, which documents `/ping` and nothing else under monitoring/metrics/observability.
"Centralized monitoring" (mentioned in `platform/apps/build-cache-node/README.md`) is a
Develocity-registration-only feature this project deliberately doesn't use (phase 3). So cache
hit rate and request latency come from the ALB's own free CloudWatch metrics instead, via
`manifests/cloudwatch-exporter.yaml` (YACE - yet-another-cloudwatch-exporter):

- **Cache hit rate** is a proxy, not a true count: `GET` returning `200` (hit) vs `404` (miss)
  at the ALB, i.e. `HTTPCode_Target_2XX_Count` vs the rest of `RequestCount`. The real hit/miss
  decision happens inside the Build Cache Node, which doesn't surface it - this is the closest
  available signal without that endpoint.
- **Latency** is `TargetResponseTime` (Average and p95), published by the ALB at 1-minute
  resolution for free (no AWS "detailed monitoring" charge, unlike EC2's).

YACE discovers the ALB by its `project=build-cache` tag, not a hardcoded ARN - that ARN changes
every time the AWS Load Balancer Controller recreates the Ingress (confirmed missing entirely
until this phase; fixed in `apps/build-cache-node/manifests/ingress.yaml`, same session).

## The two SLOs

Both score the Build Cache Node from `cloudwatch-exporter`'s metrics - see
`manifests/slo-burn-rate-rules.yaml` for the full recording/alerting rules and the comments on
why `avg_over_time` is used instead of `rate()`/`increase()` (YACE's "Sum" metrics are
periodic gauge snapshots, not counters).

| SLO | Objective | Window |
| --- | --- | --- |
| Availability | 99.5% of requests are not 5xx | 30 days |
| p95 latency | At most 5% of 1-minute windows have p95 > 300ms | 30 days |

The latency SLO is a deliberate simplification: a "fraction of good requests" SLI needs a
per-request histogram, which the Build Cache Node doesn't expose and CloudWatch's pre-aggregated
`TargetResponseTime` can't reconstruct. Scoring "did this 1-minute window's own p95 breach the
threshold" as a binary indicator is the honest alternative given what's actually available - not
presented as equivalent to a true latency histogram SLO.

Both use the Google SRE Workbook's standard multi-window, multi-burn-rate shape: a fast pair
(1h + 5m windows, 2m `for`, pages) and a slow pair (6h + 30m windows, 15m `for`, tickets) - the
short window in each pair stops a brief blip from paging on its own, since it clears before the
long window confirms a real sustained burn.

## Alertmanager -> real email

Not SMTP: the first version of this routed through SES's SMTP endpoint (an IAM user + access
key, since SMTP AUTH has no role-based equivalent) and failed on a real apply - this account's
landing-zone SCP (`workloads-guardrails` -> `DenyIamUserAndAccessKeyCreation`,
`infra/landing-zone/scps.tf`) flatly denies `iam:CreateUser`/`iam:CreateAccessKey` across the
whole Workloads OU, a guardrail set up deliberately in phase 1. See
`infra/modules/alerting/README.md` for the full story.

Alertmanager's `receivers` here use a plain `webhook_configs` instead, POSTing to
`infra/modules/alerting`'s API Gateway URL. A small Lambda behind that URL calls SES's
`SendEmail` **API** (not SMTP) using its own execution role - a real IAM role, which the SCP
doesn't touch at all. No long-lived credential exists anywhere in this path, and Alertmanager
itself needs no AWS permissions or Pod Identity role any more - it's an unauthenticated HTTPS
POST to a bridge that holds the real credential.

**Two manual steps**, same shape as `demo-spring-app`'s `BUILD_CACHE_URL`:

1. Clicking the SES verification email (`infra/modules/alerting/README.md`) - sends fail,
   loudly, in the Lambda's own CloudWatch Logs, until that identity shows Verified.
2. Copying `infra/modules/alerting`'s `webhook_url` output into this file's
   `alertmanager.config.receivers` `webhook_configs` `url` (currently a `REPLACE-ME`
   placeholder) - it's AWS-generated and changes every time `envs/dev` is destroyed and
   recreated, and unlike the old SMTP password there's no secret-file trick available for a
   plain webhook URL. `scripts/up.sh` prints it for exactly this reason.

## Cost, flagged per CLAUDE.md

- Prometheus's `gp3` PVC: 8Gi, ~US$0.64/month if left running continuously - a few cents per
  session in practice, since `envs/dev` is destroyed between sessions (docs/PLAN.md's cost
  section), same as every other EBS volume in this project.
- Grafana has no persistent volume (`persistence.enabled: false`) - dashboards are code
  (this directory), nothing would be lost restarting the pod.
- `cloudwatch-exporter`'s `GetMetricData`/`ListMetrics` calls: at this request volume and a
  60s scrape interval, a few cents a month at most (CloudWatch's first 1M `GetMetricData`
  requests per metric per month are free; this project uses a handful of metrics on one ALB).
- No new EC2 capacity expected - Prometheus/Alertmanager/Grafana/YACE's combined requests are
  small enough to fit the existing system node group or a single small Karpenter-provisioned
  node, not confirmed live yet (first real `kubectl get pods -n monitoring` after Liam applies
  this will show whether that held).

## Done when (docs/PLAN.md's phase 4 bar)

Not yet confirmed - needs a real apply, which per CLAUDE.md's hard rule only Liam runs. Once
applied: force a 5xx/latency breach against the Build Cache Node (or just wait for the SES
verification email and lower a threshold temporarily to prove the pipe end-to-end) and confirm
a real email lands.
