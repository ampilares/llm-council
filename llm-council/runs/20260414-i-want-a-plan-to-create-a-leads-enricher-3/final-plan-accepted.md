# Plan

## Overview
Build a batch Python CLI that reads a lead CSV, preserves trusted existing contact fields, enriches only missing or explicitly unverified fields, and writes an enriched CSV plus audit artifacts. The pipeline should normalize names/addresses, query approved sources through a pluggable provider layer, verify candidates with field-specific checks, and export per-field confidence and status so weak matches remain reviewable instead of silently accepted.

## Scope
- In:
  - CSV ingestion with flexible column mapping for `name`, `address`, and optional existing contact columns
  - Enrichment of missing `phone`, `mobile`, `email`, `linkedin_url`, `facebook_url`, `instagram_url`
  - Lead normalization, person/business classification, candidate discovery, verification, and deterministic per-field confidence scoring
  - Output of enriched CSV, evidence JSONL, run summary, and unresolved/conflicted review files
  - Resume, caching, deduplication, dry-run cost estimation, and provider feature flags
  - Automated tests plus small-batch calibration against labeled sample rows
- Out:
  - CRM sync, outbound messaging, or workflow automation
  - CAPTCHA bypass, proxy rotation, or scraping flows that violate source terms
  - Broad PII discovery beyond the requested contact details
  - Guaranteed fill rate; unresolved fields should remain blank with evidence

## Phases
### Phase 1: Specification and Guardrails
**Goal**: Lock the data contract, source policy, and safe defaults before implementation.

#### Task 1.1: Define schemas, statuses, and source approval matrix
- Location: `docs/leads-enricher-spec.md`, `src/leads_enricher/schema.py`
- Description: Define canonical input/output fields, evidence schema, verification statuses, confidence bands, and the approved-source matrix.
- Estimated Tokens: 1000
- Dependencies: None
- Steps:
  - Define canonical columns and placeholder handling for values like `N/A`, `unknown`, and whitespace.
  - Define per-field outputs such as `value`, `status`, `confidence`, `source_count`, and `source_summary`.
  - Define approved source categories, allowed verification methods, and prohibited collection patterns.
- Acceptance Criteria:
  - Spec covers every exported column and status.
  - Existing values are not treated as verified unless verification metadata says so.
  - Source policy is explicit enough to implement providers without policy ambiguity.

#### Task 1.2: Scaffold CLI, config, and row-state pipeline
- Location: `src/leads_enricher/cli.py`, `src/leads_enricher/config.py`, `src/leads_enricher/pipeline.py`
- Description: Create the CLI entrypoint, configuration model, and resumable pipeline stages.
- Estimated Tokens: 1200
- Dependencies: Task 1.1
- Steps:
  - Add CLI arguments for input path, output directory, provider selection, `--dry-run`, `--resume`, `--skip-existing`, and `--overwrite-unverified`.
  - Define row states such as `loaded`, `normalized`, `searched`, `verified`, `scored`, `written`.
  - Add run IDs and checkpoint metadata for safe reruns.
- Acceptance Criteria:
  - CLI runs against a sample CSV and produces placeholder artifacts.
  - Dry run estimates rows, fields, providers, and approximate API cost.
  - Resume metadata is stable across restarts.

### Phase 2: Normalization and Candidate Discovery
**Goal**: Turn raw rows into reliable match keys and gather structured candidates from approved sources.

#### Task 2.1: Normalize leads and build match keys
- Location: `src/leads_enricher/normalize.py`, `src/leads_enricher/matchers.py`, `tests/test_normalize.py`
- Description: Normalize names, addresses, and lead type so lookups and matching are consistent.
- Estimated Tokens: 1400
- Dependencies: Task 1.2
- Steps:
  - Normalize casing, punctuation, business suffixes, address abbreviations, and country/region formats.
  - Classify rows as likely person or business to adjust matching strictness.
  - Generate normalized lookup keys and deduplication keys for caching.
- Acceptance Criteria:
  - Equivalent rows map to the same normalized keys.
  - Partial or malformed addresses degrade gracefully.
  - Duplicate leads can reuse prior discovery results.

#### Task 2.2: Implement provider registry and approved adapters
- Location: `src/leads_enricher/providers/`, `src/leads_enricher/search.py`, `tests/test_providers.py`
- Description: Add a provider layer that returns structured candidates and evidence from approved source types.
- Estimated Tokens: 1800
- Dependencies: Task 2.1
- Steps:
  - Define a provider interface returning candidate objects with `field`, `value`, `source`, `source_url`, `matched_name`, `matched_address`, and raw match context.
  - Implement adapters for the approved source categories: primary enrichment API, search API for social/profile discovery, and website/domain extraction for business leads.
  - Add rate limiting, retries, timeouts, feature flags, and local caching by normalized lead key.
- Acceptance Criteria:
  - Providers return normalized candidate objects, not free-form text.
  - Provider failures are isolated and logged without aborting the run.
  - Cached duplicate rows do not trigger duplicate external lookups.

### Phase 3: Verification and Decisioning
**Goal**: Verify candidate quality, resolve conflicts, and export only explainable values.

#### Task 3.1: Implement field-specific verification rules
- Location: `src/leads_enricher/verify.py`, `src/leads_enricher/rules.py`, `tests/test_verify.py`
- Description: Verify phones, emails, and social URLs using corroboration rules appropriate to each field.
- Estimated Tokens: 1700
- Dependencies: Task 2.2
- Steps:
  - Validate phones by format, region, type, duplicate checks, and cross-source consistency.
  - Validate emails by syntax, domain relevance, MX/DNS checks, and corroboration with website/domain or multiple sources; do not treat SMTP acceptance as proof of ownership.
  - Validate social profiles by URL pattern, name match, and location/address corroboration where available.
- Acceptance Criteria:
  - Each candidate is marked `verified`, `corroborated`, `unverified`, `conflicted`, or `unresolved`.
  - Weak single-source guesses stay low confidence.
  - Conflicting candidates remain in evidence output instead of being discarded.

#### Task 3.2: Score candidates and select winners
- Location: `src/leads_enricher/scoring.py`, `src/leads_enricher/select.py`, `tests/test_scoring.py`
- Description: Assign deterministic per-field confidence and decide whether to export a value or leave the field blank.
- Estimated Tokens: 1500
- Dependencies: Task 3.1
- Steps:
  - Score by source reliability, name match strength, address match strength, cross-source agreement, and verification outcome.
  - Map numeric scores to stable bands such as `high`, `medium`, and `low`.
  - Export only the highest-confidence non-conflicting candidate above threshold; otherwise leave blank and route to unresolved/conflicted review output.
- Acceptance Criteria:
  - Same evidence always yields the same selected value and score.
  - Low-confidence candidates never overwrite blanks by default.
  - Score explanations can be reconstructed from stored evidence.

### Phase 4: Outputs and Safe Operation
**Goal**: Make the run auditable, idempotent, and usable in production.

#### Task 4.1: Write enriched CSV and audit artifacts
- Location: `src/leads_enricher/output.py`, `src/leads_enricher/report.py`, `tests/test_output.py`
- Description: Produce reviewer-friendly outputs for downstream use and audit.
- Estimated Tokens: 1200
- Dependencies: Task 3.2
- Steps:
  - Write enriched CSV preserving original columns and appending enrichment values, statuses, and confidence columns.
  - Write evidence JSONL with all candidates, checks, and final decisions.
  - Write run summary plus separate unresolved/conflicted CSVs for manual review.
- Acceptance Criteria:
  - Output CSV is usable without reading internal logs.
  - Evidence file explains every populated field.
  - Uncertain rows are easy to isolate and review.

#### Task 4.2: Add resume, overwrite controls, and observability
- Location: `src/leads_enricher/pipeline.py`, `src/leads_enricher/logging.py`, `tests/test_idempotency.py`
- Description: Prevent accidental data loss and make failures diagnosable.
- Estimated Tokens: 1100
- Dependencies: Task 4.1
- Steps:
  - Add deterministic row IDs, checkpoints, and `resume-run` behavior.
  - Enforce safe defaults: skip verified existing values, only overwrite explicitly unverified values when requested.
  - Add structured logs with row ID, provider, retry count, and final outcome.
- Acceptance Criteria:
  - Re-running the same input is idempotent.
  - Resume mode only processes incomplete or failed rows.
  - Operators can tell why a row was skipped, conflicted, or failed.

### Phase 5: Validation and Calibration
**Goal**: Prove the pipeline works on realistic data before wider use.

#### Task 5.1: Build fixtures and end-to-end tests
- Location: `tests/fixtures/leads/`, `tests/test_pipeline_e2e.py`, `tests/test_conflicts.py`
- Description: Add representative datasets and regression coverage for success, failure, and ambiguity.
- Estimated Tokens: 1400
- Dependencies: Task 4.2
- Steps:
  - Create fixture CSVs for complete rows, sparse rows, duplicates, ambiguous names, bad addresses, and mixed geographies.
  - Add snapshot tests for enriched CSV, evidence JSONL, and unresolved/conflicted outputs.
  - Add failure-path tests for provider timeouts, malformed responses, and partial writes.
- Acceptance Criteria:
  - E2E tests validate expected outputs for representative datasets.
  - Conflict cases stay unresolved or flagged.
  - Test suite fails on scoring or overwrite regressions.

#### Task 5.2: Calibrate thresholds and document rollout
- Location: `docs/leads-enricher-runbook.md`, `docs/confidence-calibration.md`
- Description: Tune confidence thresholds on labeled samples and document safe rollout.
- Estimated Tokens: 1000
- Dependencies: Task 5.1
- Steps:
  - Manually label a small sample set with known-good and known-bad outcomes.
  - Compare predicted confidence bands to observed accuracy and adjust thresholds or weights.
  - Document pilot-run procedure, provider costs, and rollback steps before full-batch use.
- Acceptance Criteria:
  - Confidence bands correspond to measured accuracy on the labeled sample.
  - Runbook supports pilot rollout on a small CSV before full execution.
  - Thresholds and default providers are documented and versioned.

## Testing Strategy
- Unit test normalization, placeholder handling, phone parsing, email validation, social URL parsing, and score calculation.
- Contract test each provider adapter with mocked responses to ensure stable candidate objects and source metadata.
- Add end-to-end tests that assert enriched CSV, evidence JSONL, unresolved/conflicted outputs, and idempotent reruns.
- Add failure-path tests for timeouts, malformed provider responses, conflicting candidates, and checkpoint recovery.
- Run a labeled pilot set to measure precision by field and tune export thresholds before production use.

## Risks
- Source quality risk: providers may return stale or mismatched data. Mitigation: require corroboration and leave weak matches blank.
- Compliance risk: some sources may restrict automated collection or reuse. Mitigation: enforce an approval matrix and make providers independently disableable.
- Identity collision risk: common names or shared addresses can create false matches. Mitigation: separate person/business rules and score name/address independently.
- Verification risk: syntax, MX, and liveness checks are weak signals, not proof. Mitigation: cap confidence unless multiple independent signals agree.
- Cost/rate-limit risk: large batches can exhaust provider quotas. Mitigation: dry-run estimates, caching, deduplication, and resumable runs.

## Rollback Plan
- Never modify the input CSV; write all artifacts to a run-specific output directory.
- Disable a bad provider via config and rerun only unresolved or affected rows.
- Delete suspect output artifacts for the affected run ID and reprocess from the last clean checkpoint.
- If scoring thresholds are too permissive, raise thresholds and rerun without changing original input data.

## Edge Cases
- Duplicate names at different addresses
- Multiple businesses at one address or suite
- Rows with partial, malformed, or international addresses
- Existing placeholder values such as `N/A` or `unknown`
- Personal and business leads mixed in one file
- Landline/mobile ambiguity or identical numbers in both fields
- Social profiles with correct name but wrong geography
- Provider outages, rate limits, or HTML changes breaking extraction
- Large CSVs where caching and checkpointing materially affect cost

## Open Questions
- Which source providers are already approved and budgeted?
- Is the input mostly businesses, mostly people, or mixed?
- What geographies must phone and address normalization support?
- Is there already a labeled sample set for confidence calibration?