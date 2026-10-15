# S3 lifecycle policy automation - reference

Engine, semantics and traps. Nothing here is in a code comment.

## Files

| File | Change |
|---|---|
| `aws-client-infra-provision-core/tasks/aws-storage-bucket-lifecycle.yml` | new |
| `aws-client-infra-provision-core/tasks/aws-storage-bucket-create.yml` | one `include_tasks` appended |
| `aws-client-infra-provision-core/aws-client-vars-base.yml` | defaults block appended |
| `clients/<client>/<env>/aws-client-vars-env.yml` | `s3_lifecycle_manage` + per-bucket `lifecycle_rules` |

## Engine

Whole-document replace via `aws s3api put-bucket-lifecycle-configuration`, CLI on the
host, STS creds as `AWS_*` env vars, `run_once: true` because lifecycle is account-level.
No new collection dependency.

The S3 API has no "add one rule" call - `PutBucketLifecycleConfiguration` replaces the
entire document. `amazon.aws.s3_lifecycle` simulates per-rule merge with GET-modify-PUT,
which means a rule deleted from config is never deleted from the bucket and the config
file silently stops being the truth. Replace was chosen to avoid that. The cost of
replace is that it would destroy rules added by hand, which is what the adopt guard is for.

## Rule resolution

Per bucket, in order:

1. `bucket_info.lifecycle_rules` is defined -> use it, even if `[]`
2. else `bucket_info.lifecycle_use_default: true` -> use `s3_lifecycle_rules_default`
3. else -> no rules

Inheritance from the estate default is **explicit opt-in**. A bucket that says nothing
never acquires a rule because somebody edited a shared var. Deliberate, because
`velero` and `donotdelete` are in the standard bucket set.

## An empty rule set means "not managed", not "delete everything"

A resolved rule set of `[]` skips the bucket entirely. It does **not** clear an existing
lifecycle configuration. To actually clear one, set `lifecycle_purge: true` on the bucket,
which issues `delete-bucket-lifecycle`.

Consequence: removing rule 2 of 2 from config works declaratively; removing the last
remaining rule needs the purge flag. This is the one place config is not fully
declarative, and it is traded deliberately for not wiping a bucket that merely forgot
to opt in.

## The adopt guard

Before writing, the task reads the bucket's current configuration and compares rule IDs
to what config declares. Any rule on the bucket that config does not declare aborts the
run with the rule IDs named. Override per bucket with `lifecycle_adopt: true`, which
discards them.

This forces the hand-made rules currently on your buckets into git the first time each
bucket is touched. On a bucket with no policy there is nothing unmanaged and no friction.

## A failed read is never treated as "no rules"

A `get-bucket-lifecycle-configuration` that fails with anything other than
`NoSuchLifecycleConfiguration` aborts. Without this, a wrong CLI path or a missing
`s3:GetLifecycleConfiguration` permission returns a non-zero rc, the existing-rules list
comes back empty, the adopt guard sees nothing unmanaged, and the task overwrites a
populated bucket. That hole existed in the first draft and was found by test S9.

## Validation performed before any AWS call

Rule ID present, non-empty, unique per bucket, <= 255 chars. `status` in
Enabled/Disabled. At least one action per rule. Every `storage_class` in
`s3_lifecycle_valid_storage_classes`. Transitions to STANDARD_IA / ONEZONE_IA at least
`s3_lifecycle_ia_min_days` (30). Transition days strictly ascending. `expiration_days`
greater than every transition day.

AWS rejects most of these with unhelpful errors, some only after a delay.

## Verification

After the PUT the task reads the configuration back from AWS and asserts the served
rule IDs equal the declared rule IDs. A PUT that is accepted but not served fails the
run. Per CLAUDE.md section 11: verify at the destination, not at the service.

## What a green run does NOT prove

- **That the rules ever fire.** S3 evaluates lifecycle once a day, asynchronously.
  A correct document is attached; nothing has happened yet. Confirm days later with
  `aws s3api list-object-versions` counts, or S3 Storage Lens.
- **That the retention is what the client wanted.** The automation applies what the
  config says. A wrong `expiration_days` is applied perfectly and deletes real data.
  `s3_lifecycle_check_only: true` exists so a human reads the document first.
- **Cost direction.** Transitioning many small objects to IA/Glacier raises the bill:
  IA bills a 128KB minimum per object, and transition requests are charged per object.
  Check average object size before adding transitions.

## Minimum storage durations - early deletion is still billed

STANDARD_IA / ONEZONE_IA 30 days, GLACIER_IR 90, GLACIER 90, DEEP_ARCHIVE 180.
`transition to GLACIER at 90, expiration_days: 100` bills 90 days of Glacier for 10 days
of use. Not validated against, because it is a cost mistake and not an API error.

## Bucket-specific hazards in the standard set

- **velero** - Velero manages its own retention and keeps a catalog. An S3 expiration
  rule deletes objects out from under it; Velero then lists backups whose data is gone
  and restores fail. Keep `lifecycle_rules: []`, or noncurrent-version rules only.
- **donotdelete** - the name is the requirement. `lifecycle_rules: []`.
- **dailyjob** - safest pilot. Versioned since creation with no lifecycle, so it is
  probably holding every version of every object ever written.
- **dataload** - check object sizes before transitioning.

## Known gaps

- Every bucket gets `versioning: true` from the create task, so
  `noncurrent_version_*` rules are safe here. A backfill play that does not run
  bucket creation first must check versioning itself, or those rules are accepted and
  silently never fire.
- `tmp_directory` resolves `lookup('env','TMPDIR')` on the AWX EE, but the document is
  written on the host. If TMPDIR is set on the EE to a path absent on the host, `copy`
  fails. Override `tmp_directory` if that happens.
- The document is written in API V2 `Filter` form throughout. Mixing V2 `Filter` with
  V1 top-level `Prefix` in one document is a runtime error, so do not hand-edit a
  bucket's policy into V1 form afterwards.
- `description` and `location` in `client_buckets` appear unconsumed by the create
  task. Grep before assuming either way.
- No `set_stats` digest yet. Worth adding for the backfill campaign so a workflow can
  aggregate which buckets got which policies across clients.
