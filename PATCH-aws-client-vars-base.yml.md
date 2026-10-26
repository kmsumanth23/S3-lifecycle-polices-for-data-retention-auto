# Final changes — Shawn's review, 26 Oct

Two code changes were asked for. Both done, plus the removals they imply.

| Asked | Status |
|---|---|
| Remove the overall feature flag | Done — `s3_lifecycle_manage` gone |
| Empty `lifecycle_rules` means remove all rules | Done |
| Raise the PR | Ready |

## Correction to the meeting notes — read before you implement anything

Gemini's **Next steps #2** says:

> "Ensure the automation **ignores** lifecycle configuration when the rule list is empty."

That is the opposite of what was agreed, and it contradicts Gemini's own Decisions
section in the same document. The transcript is authoritative — Shawn at 00:09:57 and
again at 00:13:34:

> "an empty life cycle rule literally means **all rules should be removed**" … "where if
> life cycle rules was **not present** … then it's just **ignored**. There's a nuance and
> a difference there."

So two different cases, opposite behaviour. Implementing the Gemini wording literally
would have built the wrong thing. Worth correcting in the notes so nobody works from it.

---

## The three semantics now implemented

| Client config | Behaviour | API calls |
|---|---|---|
| no `lifecycle_rules` key | bucket ignored entirely | **zero** |
| `lifecycle_rules: []` | every lifecycle rule removed from the bucket | GET + DELETE |
| `lifecycle_rules:` with entries | bucket config replaced with exactly these | GET + PUT + GET |

The first row is why removing the flag is safe. Existing clients have no
`lifecycle_rules` keys anywhere, so their next provisioning run makes **zero S3 calls**
and nothing changes. Verified with a call-logging wrapper: two buckets each holding a
hand-made rule, neither declaring the key, run completed with 0 aws invocations and both
rules intact.

---

## Edit 1 of 3 — `aws-client-vars-base.yml`

Replace the whole `# S3 bucket lifecycle policy defaults` block with this. Two variables
are **removed**: `s3_lifecycle_manage` and `s3_lifecycle_rules_default`.

```yaml
# S3 bucket lifecycle policy defaults
# Lifecycle is driven per bucket by the presence of lifecycle_rules in the client config
s3_bucket_name_prefix: "hclsw-hss-bkt-"
s3_lifecycle_check_only: false
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

### On `s3_lifecycle_check_only` — be ready to defend keeping it

Shawn's objection was specific: the flag "deviates from our standard design pattern" and
is "one more thing that's going to muddy up our client configs." That applies to
`s3_lifecycle_manage`, which was a per-client gate living in client config.

`s3_lifecycle_check_only` is different in kind: it never goes in a client config. It is
passed as a job extra var for one run, like `--check`, and prints the document the
automation would write without writing it. If Shawn wants it gone too, say so and I will
strip it — but it is the thing that lets a human read a policy before it touches a
client, and it is not in the client config surface he objected to.

---

## Edit 2 of 3 — `aws-storage-bucket-lifecycle.yml`

Full replace with the file shipped alongside this patch. Changes from the version you
demoed:

- Outer gate is now `when: bucket_info.lifecycle_rules is defined` instead of the
  feature flag.
- `s3_lifecycle_rules_default`, `lifecycle_use_default` and `lifecycle_purge` are gone.
  Empty list does what `lifecycle_purge` used to.
- New task **Report what this bucket has declared** — prints either the rule count or a
  warning that every rule is about to be removed.
- New task **Report the rules that an empty lifecycle_rules will remove** — names them
  before the delete, so there is a record in the job output.
- The adopt guard no longer runs when the list is empty. See below.
- `check_only` previews the removal path as well as the apply path.

### One design judgement you should know about

The adopt guard asks "is this rule on the bucket but absent from config?" Against an
**empty** list every existing rule answers yes, so the guard would have blocked the very
removal Shawn asked for. It now skips when the list is empty, because an explicit
`lifecycle_rules: []` **is** the deliberate declaration — in Shawn's words, "you're
literally saying there that there should be no life cycle rules."

The guard still fires normally when rules are declared and the bucket holds something
else. The practical effect:

| Situation | Guard |
|---|---|
| Rules declared, bucket holds a rule not in config | **blocks**, names it, prints paste-ready config |
| `lifecycle_rules: []`, bucket holds rules | proceeds, lists what it removed |
| Key absent, bucket holds rules | never looks |

So `lifecycle_rules: []` is a loaded operation with no second confirmation. It lists what
it removed and `check_only` previews it, but someone writing `[]` against the wrong
bucket will clear it. That follows directly from the agreed semantics; flagging it so the
behaviour is a choice and not a surprise.

---

## Edit 3 of 3 — the root play include

Unchanged from what you have, and still correct. For completeness:

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

**Still unanswered and worth settling before the PR merges:** what tags the job
templates pass. Measured — `--tags bucket` creates the buckets and applies zero
lifecycle, silently. The `bucket` tag above is what prevents that. It did not come up in
the meeting.

---

## Client config, after the change

```yaml
client_buckets:
  - name: "{{ client_env }}-dailyjob"
    description: "Daily job bucket"
    location: "{{ aws_region }}"
    # no lifecycle_rules key - bucket ignored, zero API calls

  - name: "{{ client_env }}-dataload"
    description: "Data Load bucket"
    location: "{{ aws_region }}"
    lifecycle_rules:
      - id: transition-to-ia
        status: Enabled
        prefix: data/
        transitions:
          - days: 30
            storage_class: STANDARD_IA
          - days: 90
            storage_class: GLACIER
        expiration_days: 365
      - id: cleanup-incomplete-uploads
        status: Enabled
        abort_incomplete_multipart_upload_days: 7

  - name: "{{ client_env }}-velero"
    description: "Data bucket for velerobackups"
    location: "{{ aws_region }}"
    lifecycle_rules: []     # explicit: remove any lifecycle rules from this bucket

  - name: "{{ client_env }}-donotdelete"
    description: "donotdelete bucket"
    location: "{{ aws_region }}"
    # key omitted - never managed, never read
```

Note the difference between velero and donotdelete above. Both end up with no lifecycle
rules, but `[]` actively removes anything present while an omitted key never looks. For
`donotdelete` the omitted key is the safer expression of intent. Worth a sentence in the
PR description.

---

## Regression after the rewrite

Against a file-backed stub of the S3 API.

```
N1  key absent: ignored, 0 API calls                PASS
N2  declared / empty / absent in one run            PASS
N3  adopt guard still blocks declared-rule case     PASS
N3b paste-ready hint still shown                    PASS
N4  empty list removes rules without needing adopt  PASS
N5  empty list, nothing there: clean no-op          PASS
N6  check_only previews the removal                 PASS
N7  check_only previews an apply, 0 writes          PASS
N8  bad configs still rejected                      2/2
N9  missing CLI blocks early                        PASS
N10 root-play layout, correct bucket each           PASS
```

## Re-test in AWS before the PR

The semantics changed, so the earlier live run no longer covers the code. Shortest
useful pass on hxsa/awsawxtest:

1. Remove `s3_lifecycle_manage` from the client env file. Omit `lifecycle_rules`
   entirely on one bucket, declare rules on another, set `[]` on a third that currently
   has rules.
2. Run with `s3_lifecycle_check_only: true`. Confirm the omitted bucket is skipped, one
   shows "would apply", and the third shows "would REMOVE every lifecycle rule".
3. Run for real. Confirm with `get-bucket-lifecycle-configuration` on all three.
4. The adopt guard has still never fired in AWS. Plant a rule with
   `plant-unmanaged-rule.sh` and confirm it aborts. That is the one outstanding test.
