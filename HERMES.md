# HERMES.md

## Repository Identity

```yaml
repository_name: hermes-agent-commons
repository_owner: KrzysiekSko
manifest_role: canonical_repository_agent_manifest
manifest_version: 1.0
```

## Language Governance

```yaml
repository_language: pl-PL
operator_communication_language: pl-PL
agent_analysis_language: pl-PL
artifact_language_policy:
  preserve_existing_artifact_language: true
  new_governance_artifacts_language: pl-PL
  code_identifiers_translation: forbidden
upstream_contribution_language_policy:
  rule: use_target_repository_language
mixed_language_policy:
  rule: preserve_artifact_language
  mass_translation: forbidden
language_precedence:
  - explicit_operator_instruction
  - repository_HERMES_md
  - established_repository_documentation_language
  - existing_artifact_language
  - pl-PL
```

## Authority Model

The repository-level `HERMES.md` is the canonical source of repository-specific agent rules. A local Hermes copy may exist only as a synchronized runtime replica.

```yaml
authority_precedence:
  - operator_explicit_instruction
  - repository_HERMES_md
  - synchronized_local_HERMES_replica
  - Hermes_defaults
```

## Agent Operating Rules

```yaml
agent_behavior:
  evidence_first: true
  fail_closed: true
  no_false_pass: true
  explicit_gate_status: true
  preserve_baseline: true
  mutation_requires_authorization: true
```

Agents MUST distinguish between `VERIFIED`, `PARTIALLY_VERIFIED`, `NOT_VERIFIED`, and `CONTRADICTED`. A `PASS` MUST NOT be declared without evidence.

## Change Control

Repository mutations MUST follow:

```text
PRE → AUTHORIZE → CHANGE → VERIFY → EVIDENCE → CLOSE
```

Direct mutation of protected or canonical branches is forbidden unless explicitly authorized. Preferred workflow: `branch → change → verification → PR → merge → post-merge read-back`.

## Baseline Protection

```yaml
baseline_policy:
  historical_audit_results_immutable: true
  retrospective_baseline_rewrite: forbidden
  remediation_separate_from_baseline: true
```

New findings MUST be recorded separately and MUST NOT silently modify previously closed audit results.

## Secret Handling

```yaml
secret_policy:
  repository_manifest_secrets: forbidden
  credentials: forbidden
  tokens: forbidden
  private_keys: forbidden
  secret_fragments: forbidden
```

Credentials, tokens, API keys, private keys, authentication headers, and secret fragments MUST NOT be placed in this manifest, evidence, logs, commits, pull requests, or persistent agent messages.

## Trusted Agent Group

```yaml
trusted_agent_group:
  enabled: true
  repository_context_sharing: true
  governance_sharing: true
  evidence_sharing: true
  secret_sharing: false
  credential_sharing: false
  repository_language: pl-PL
  agent_communication_language: pl-PL
```

## Manifest Synchronization

```yaml
manifest_sync:
  repository_copy: canonical
  local_copy: replica
  verify_before_use: true
  drift_detection: required
  automatic_repository_overwrite: forbidden
  conflict_policy: fail_closed
```

On repository access, locate the repository manifest and local runtime replica, calculate both hashes, compare versions/hashes, and classify the result as `MATCH`, `REPOSITORY_NEWER`, `LOCAL_NEWER_OR_DIFFERENT`, or `CONFLICT`. A local replica MUST NOT automatically overwrite the repository manifest.

## Evidence Requirements

```yaml
evidence:
  branch:
  commit_sha:
  artifact_path:
  artifact_blob_sha:
  pull_request:
  verification_result:
  post_merge_commit_sha:
  post_merge_blob_sha:
```

Evidence containing secrets is forbidden.

## STOP Conditions

Agents MUST stop and request operator direction when:

```text
AUTHORIZATION = NOT_VERIFIED
MANIFEST_CONFLICT = TRUE
BASELINE_MUTATION_REQUIRED = TRUE
SECRET_EXPOSURE_RISK = TRUE
REPOSITORY_LANGUAGE = UNRESOLVED
SCOPE_EXPANSION_REQUIRED = TRUE
```

## Repository-Specific Scope

This repository is the shared Polish knowledge, governance, documentation, audit, and operational context repository for trusted Hermes Agent instances.

```yaml
repository_context:
  primary_language: pl-PL
  governance_language: pl-PL
  operator_language: pl-PL
  trusted_agent_context_exchange: allowed
  secret_exchange: forbidden
```
