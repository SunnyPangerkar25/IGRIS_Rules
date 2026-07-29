# AISDR SageMaker Rules — What Changed

**Date:** 2026-07-28 · 6 use cases → 7 rule files · validate + tests pass · not deployed

---

## AWS SageMaker - AI Estate Enumeration (Broad Resource Discovery)

**Split into two rules. Original file deleted.**

| | |
|---|---|
| **Before** | One rule. Key was `entity_id`, built by `eval(entity_id = coalesce(sessionIssuer.arn, userIdentity.arn))`. |
| **After** | Two rules, each keyed on `actor.user.uid` (real field, renamed `principal_uid`). |

New rules:

| Rule name | File | Filter added | Severity |
|---|---|---|---|
| AWS SageMaker - AI Estate Enumeration by Long-Term Credential | `aws-sagemaker-estate-enumeration-iam-user.yaml` | `actor.user.uid: AIDA*` | High (was Medium) |
| AWS SageMaker - AI Estate Enumeration by Assumed Role | `aws-sagemaker-estate-enumeration-assumed-role.yaml` | `actor.user.uid: AROA*` | Medium |

Also: removed `eval(op_breadth = distinct_ops)`; `distinct_ops` is now the first aggregate. Thresholds unchanged (`distinct_ops >= 5` or `event_count >= 50`).

---

## AWS SageMaker - Notebook Instance Created (Direct Internet Exposure Risk)

`aws-sagemaker-notebook-internet-exposure.yaml`

| | |
|---|---|
| **Before** | Key = `entity_id` from `eval(coalesce(...))` |
| **After** | Key = `principal_uid` (`actor.user.uid`) |

Detection logic unchanged. Severity High, unchanged.

---

## AWS SageMaker - Presigned Notebook URL Generated

`aws-sagemaker-presigned-notebook-url.yaml`

| | |
|---|---|
| **Before** | Key = `entity_id` from `eval(coalesce(...))` |
| **After** | Key = `principal_uid` (`actor.user.uid`) |

Detection logic unchanged. Severity High, unchanged.

---

## AWS SageMaker - Notebook Lifecycle Configuration or Role Modified

`aws-sagemaker-notebook-lifecycle-config.yaml`

| | |
|---|---|
| **Before** | Key = `entity_id` from `eval(coalesce(...))` |
| **After** | Key = `principal_uid` (`actor.user.uid`) |

Detection logic unchanged. Severity High, unchanged.

---

## AWS SageMaker - Model or Inference Endpoint Deployed or Modified

`aws-sagemaker-model-endpoint-deployment.yaml`

| | |
|---|---|
| **Before** | Key = `entity_id` from `eval(coalesce(...))` |
| **After** | Key = `principal_uid` (`actor.user.uid`) |

Detection logic unchanged. Severity Medium, unchanged.

---

## AWS SageMaker - Training or Processing Job Created

`aws-sagemaker-training-processing-job-created.yaml`

| | |
|---|---|
| **Before** | Key = `entity_id` from `eval(coalesce(...))` |
| **After** | Key = `principal_uid` (`actor.user.uid`) |

Detection logic unchanged. Severity Medium, unchanged.

---

## The change, in full (applied to all 7)

**Before**

```
| eval(entity_id = coalesce(unmapped.userIdentity.%json.sessionContext.sessionIssuer.arn, unmapped.userIdentity.%json.arn))
| stats ...
  by ig_tenant_id, %ingest.import_rule_name,
     entity_id, unmapped.userIdentity.%json.type, ...
```
```yaml
    - label: "incident_key"
      value: "entity_id"
    - label: "entity_id"
      value: "{{@alert.results_table.rows[0].entity_id}}"
```

**After**

```
| stats ...
  by ig_tenant_id, %ingest.import_rule_name,
     actor.user.uid,
     unmapped.userIdentity.%json.arn,
     unmapped.userIdentity.%json.sessionContext.sessionIssuer.arn,
     unmapped.userIdentity.%json.type, ...
| rename(
    actor.user.uid as principal_uid,
    unmapped.userIdentity.%json.arn as principal_arn,
    unmapped.userIdentity.%json.sessionContext.sessionIssuer.arn as role_arn,
    ...
  )
```
```yaml
    - label: "incident_key"
      value: "principal_uid"
    - label: "key_entity"
      value: "{{principal_uid}}"
```

**Why:** `entity_id` was not a real field — it was created by `eval` at query time, so nothing downstream could resolve or pivot on it. `actor.user.uid` is a real OCSF-mapped field, populated for both IAM users (`AIDA…`) and assumed roles (`AROA…:session`).

**Side effects:**
- Old ARNs kept as context columns `principal_arn` and `role_arn`.
- 6th alert label renamed `entity_id` → `key_entity`.
- Every test `dataset_inline` row now includes `actor.user.uid`.
- `@index={ … }` now uses `81fe91d5-1dee-4321-8700-0d1c4f441770`.
