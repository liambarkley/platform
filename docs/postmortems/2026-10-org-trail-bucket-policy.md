# Postmortem: organization CloudTrail failed to apply (bucket policy missing a prefix)

**Date:** October 28, 2026
**Status:** Resolved
**Severity:** Low (build-time failure, no live service impact - the project has no running
workloads yet)

This is blameless by design: the goal is to understand why the first apply failed and why a
plain retry wouldn't have fixed it, not to assign fault for writing the policy that way.

## Summary

The first `terraform apply` of `infra/log-archive` failed creating the organization CloudTrail
trail (`aws_cloudtrail.org`) with `InsufficientS3BucketPolicyException`. The trail's S3 bucket
policy only granted `s3:PutObject` under `AWSLogs/<org-id>/*`. CloudTrail, even for an
organization trail, also writes management-account-specific objects (digest files and the
management account's own log files) under `AWSLogs/<management-account-id>/*` - a second,
separate prefix the policy didn't cover. Adding that prefix to the same statement fixed it;
the bucket and every other resource were untouched.

## Timeline

1. `infra/landing-zone` applied successfully, creating the `log-archive`, `shared-services` and
   `platform-dev` accounts.
2. `infra/log-archive` applied: the S3 bucket, versioning, public-access block, encryption and
   lifecycle configuration, and the bucket policy all created successfully.
3. `aws_cloudtrail.org` failed: `InsufficientS3BucketPolicyException`.
4. Root cause diagnosed: the `CloudTrailWrite` statement's `resources` list had one entry,
   `arn:aws:s3:::build-cache-org-trail/AWSLogs/<org-id>/*`. CloudTrail needed a second entry,
   `arn:aws:s3:::build-cache-org-trail/AWSLogs/<management-account-id>/*`, which didn't exist.
5. Fix applied: added the second prefix to the same statement, built from
   `data.aws_caller_identity.management.account_id` rather than hardcoded, so it doesn't drift
   if this is ever applied in a different account. Conditions, bucket name, and every other
   resource were left unchanged.
6. Verified with a real `terraform plan` before reapplying: `aws_s3_bucket_policy.org_trail`
   would update **in-place** (the one new `Resource` entry), `aws_cloudtrail.org` would be
   **created** (the resource that previously failed) - nothing replaced.
7. Reapplied. The trail is logging (`IsLogging=true`) to `build-cache-org-trail` in
   log-archive.

## Why a retry wouldn't have helped

This wasn't a transient AWS-side issue (eventual consistency, a race, a rate limit) - every one
of those would be a reasonable first guess for an "it failed, try again" response, and none of
them was the actual cause here. The bucket policy was deterministically missing a required
grant. Retrying the same `apply` without changing the policy would have failed the same way
every time, since nothing about the failure depended on timing - it depended on what the
policy actually said.

## Root cause

Organization trails have a less obvious requirement than single-account trails: CloudTrail
writes to **two** distinct `AWSLogs/` prefixes under an org trail, not one -
`AWSLogs/<org-id>/*` for member-account logs, and `AWSLogs/<management-account-id>/*` for the
management account's own logs and digest files. The original policy was written by analogy to
a typical single-account trail's bucket policy, which only ever needs the account-ID prefix -
the org-ID prefix was added correctly, but the management-account prefix that org trails
*also* need wasn't.

## Prevention: how to catch this before `apply`

The policy document is a `data "aws_iam_policy_document"` resource, computed purely from local
values with no AWS API calls - `terraform plan` (or even `validate`) can render its `.json`
output without ever touching real infrastructure. Two concrete follow-ups, not yet
implemented:

1. **A `terraform test` (native `.tftest.hcl`) assertion** on
   `data.aws_iam_policy_document.org_trail_bucket.json` in `infra/log-archive`, asserting the
   rendered JSON contains two distinct `AWSLogs/` prefixes in the `CloudTrailWrite` statement's
   `Resource` list - one matching the org ID, one matching the management account ID. This
   would have caught the bug at `plan` time, in CI, with no AWS credentials needed.
2. **A `conftest`/OPA rule** in `infra/policy/`, run against `terraform plan`'s JSON output,
   generalizing the same check: for any `aws_s3_bucket_policy` resource whose bucket is also
   referenced by an `aws_cloudtrail` resource with `is_organization_trail = true`, require the
   policy's `PutObject` statement to list at least two distinct `AWSLogs/<id>/*` resource
   patterns. More effort than the `terraform test` above, but catches the same mistake in any
   future organization-trail bucket policy, not just this one.

Either is a small, contained addition - deliberately not bundled into this PR to keep it
scoped to what was asked.
