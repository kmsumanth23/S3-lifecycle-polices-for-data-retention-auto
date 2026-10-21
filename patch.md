# Combined patch - PATCH 02 + PATCH 03, three edits

Apply in this order. Edit 3 is a file replace; edits 1 and 2 are anchored inserts.

**PATCH 02 Part 3 is deliberately dropped.** It would have rewritten `Set bucket names`
in `aws-storage-bucket-create.yml`. You told your reviewer that file is untouched, and
that statement is worth more than removing one duplicated string literal. Cost:
`hclsw-hss-bkt-` exists in two places. Raise it as a separate cleanup PR later if anyone
minds.

---

## Edit 1 of 3 - aws-client-infra-provision-core/aws-client-vars-base.yml

**Anchor:** the existing WAF block.

```yaml
# Waf default values
waf_log_group: "aws-waf-logs-{{ client_code }}-{{ client_env }}"
waf_log_retention: 14
waf_policies_client: []
waf_rule_exceptions: []
```

Insert immediately after it. This is the complete final block - if you already added part
of it, reconcile against this rather than appending again.

```yaml
# S3 bucket lifecycle policy defaults
# Feature is off until a client sets s3_lifecycle_manage: true
s3_bucket_name_prefix: "hclsw-hss-bkt-"
s3_lifecycle_manage: false
s3_lifecycle_check_only: false
s3_lifecycle_rules_default: []
s3_lifecycle_aws_cli: /usr/local/bin/aws
s3_lifecycle_valid_storage_classes:
  - STANDARD_IA
  - ONEZONE_IA
  - INTELLIGENT_TIERING
  - GLACIER_IR
  - GLACIER
  - DEEP_ARCHIVE
s3_lifecycle_ia_storage_classes:
  - STANDARD_IA
  - ONEZONE_IA
s3_lifecycle_ia_min_days: 30
s3_lifecycle_rule_id_max_length: 255
```

`s3_bucket_name_prefix` is the new line versus the earlier patch. Without it, edit 3
fails with an undefined variable.

---

## Edit 2 of 3 - the root provisioning play (AWS Client Environment Build)

**Anchor:** `- name: include tasks to manage the bucket lifecycle configuration`,
immediately after `- name: Include the create bucket task in aws`.

### Replace this

```yaml
    - name: include tasks to manage the bucket lifecycle configuration
      include_tasks: "tasks/aws-storage-bucket-lifecycle.yml"
      tags:
        - s3-lifecycle
```

### With this

```yaml
    - name: include tasks to manage the bucket lifecycle configuration
      include_tasks: "tasks/aws-storage-bucket-lifecycle.yml"
      loop: "{{ client_buckets }}"
      loop_control:
        loop_var: bucket_info
        label: "{{ bucket_info.name }}"
      when: client_buckets is defined and client_buckets is not none
      tags:
        - bucket
        - s3-lifecycle
```

Why each line:

- `loop` + `loop_var: bucket_info` - `bucket_info` is loop-scoped and does not survive
  the create loop. Without this the task file has no bucket. Gate off it is a silent
  no-op; gate on it fails with `'bucket_info' is undefined`.
- `when:` - the same guard your create include already carries. Without it a client with
  no `client_buckets` fails on `loop:` over an undefined variable.
- `bucket` tag - measured: `--tags bucket` gives 3 buckets created, 0 lifecycle applied.
  Safe to add because `s3_lifecycle_manage` defaults to false. Drop it only if you want
  lifecycle to run exclusively via `--tags s3-lifecycle`.

---

## Edit 3 of 3 - aws-client-infra-provision-core/tasks/aws-storage-bucket-lifecycle.yml

Full replace with the file shipped alongside this patch. Two changes versus what you
currently have:

1. Derives `s3_lifecycle_bucket_name` from `bucket_info` instead of reading
   `bucket_name`. `bucket_name` is a `set_fact` from inside the create task file; as a
   sibling loop at the root it holds the **last** bucket's name for every iteration.
   Measured before the fix: a rule declared for `np-dataload` was written to
   `hclsw-hss-bkt-hxtx-np-donotdelete`, and the play reported `failed=0` because the
   read-back verification read the same wrong bucket.
2. Adds an aws CLI presence check. `run_once: true` selects an arbitrary host and these
   plays include Windows instances, so a missing CLI now aborts up front naming the host
   instead of failing per bucket.

---

## Verify the patch landed

```
# edit 1
grep -n "s3_bucket_name_prefix" aws-client-infra-provision-core/aws-client-vars-base.yml

# edit 2
grep -A8 "include tasks to manage the bucket lifecycle configuration" <root play>.yml

# edit 3  - must print nothing at all
grep -n "bucket_name" aws-client-infra-provision-core/tasks/aws-storage-bucket-lifecycle.yml \
  | grep -v s3_lifecycle_bucket_name | grep -v s3_bucket_name_prefix
```

Do not use `grep -c "{{ bucket_name }}"` for this. It returns 0 while
`[bucket_name ~ '...']` style references are still present - that exact blind spot hid a
bug in the preview path through two shipped revisions.

Then the real check, which no grep can do - run with
`{"s3_lifecycle_manage": true, "s3_lifecycle_check_only": true}` and confirm the
`would apply` lines each name a **different** bucket. The same name repeated means edit 3
did not land.
