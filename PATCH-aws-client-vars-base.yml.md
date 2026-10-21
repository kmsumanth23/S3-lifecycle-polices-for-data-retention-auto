# PATCH: aws-client-infra-provision-core/aws-client-vars-base.yml

Anchor: the existing WAF defaults block, which reads

```yaml
# Waf default values
waf_log_group: "aws-waf-logs-{{ client_code }}-{{ client_env }}"
waf_log_retention: 14
waf_policies_client: []
waf_rule_exceptions: []
```

Insert the block below immediately after it. Same pattern as waf_policies_client:
empty by default, overridden per client.

```yaml
# S3 bucket lifecycle policy defaults
# Feature is off until a client sets s3_lifecycle_manage: true
s3_lifecycle_manage: false
s3_bucket_name_prefix: "hclsw-hss-bkt-"
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
grep -c "{{ bucket_name }}" aws-client-infra-provision-core/tasks/aws-storage-bucket-lifecycle.yml   # must be 0
```
