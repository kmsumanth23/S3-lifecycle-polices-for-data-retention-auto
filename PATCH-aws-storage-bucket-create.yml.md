# PATCH: aws-client-infra-provision-core/tasks/aws-storage-bucket-create.yml

Anchor: the LAST task in the file, `- name: Display the creation result`.
Append the new task immediately after it, before the trailing comment banner.

## Before

```yaml
- name: Display the creation result
  debug:
    var: s3_create_result
  tags:
    - debug
    - never

# *** The values in this file should not be modified directly                      ***
# *** If overrides are needed, they should be done in client environment vars file ***
```

## After

```yaml
- name: Display the creation result
  debug:
    var: s3_create_result
  tags:
    - debug
    - never

- name: include tasks to manage the bucket lifecycle configuration
  include_tasks: "{{ base_src_infra_provision }}/tasks/aws-storage-bucket-lifecycle.yml"
  tags:
    - bucket
    - s3-lifecycle

# *** The values in this file should not be modified directly                      ***
# *** If overrides are needed, they should be done in client environment vars file ***
```

CONFIRM BEFORE APPLYING: the `include_tasks` path. I used
`{{ base_src_infra_provision }}` because aws-client-vars-base.yml already defines it
and CLAUDE.md shows clouds-common reaching into this repo by variable. If a bare
`include_tasks: aws-storage-bucket-lifecycle.yml` already resolves from inside this
file in your layout, use that instead - it is shorter and has no var dependency.
