# System Invariants

Authoritative hard rules for the governed system. This file contains only invariants. Capability requirements, platform choices, and architectural decisions belong in `REQUIREMENTS.md`, `ARCHITECTURE.md`, `CONSISTENCY.md`, or `PROBLEM.md`.

---

## Invariant Completeness Framework

Every invariant in this document must be evaluable against the following checklist.

**Example:** For any [object] at [scope], exactly/at-most/never [hard rule over time], otherwise [reject, fail closed, degrade, or reconcile].

### Required Questions

1. **What object is this about?**  
   Anchor the invariant to a concrete entity or evaluation unit.

2. **At what scope does it hold?**  
   Define the consistency boundary.

3. **What must never be violated?**  
   State the hard, falsifiable rule.

4. **How many are allowed?**  
   State cardinality explicitly.

5. **What happens over time?**  
   Define immutability, monotonicity, ordering, or version progression.

6. **What happens when things break?**  
   Define reject, fail-closed, degrade, reconcile, or explicit failed state behavior.

7. **How is it enforced?**  
   State the enforcement mechanism without turning the invariant into implementation detail.

### Director-Level Extension

8. **What tradeoff does this invariant force?**  
   Record the cost as a side note, not as part of the invariant itself.

> Any invariant that cannot be answered clearly across 1-7 is incomplete.

---

## Format Convention

Each invariant below uses this structure:

- **Invariant:** single hard-rule sentence
- **Enforcement:** how the system guarantees the invariant
- **Tradeoff:** the cost accepted to keep the invariant true

---

## Correct Data

### Deterministic Decisions

- **Invariant:** Given the same request identity, snapshot version, policy version, captured experiment references, and availability inputs, routing must never produce a different experiment assignment or routing decision for the same admitted request lineage.  
  **Enforcement:** Immutable decision records, immutable published policy versions, immutable captured experiment references, and replay against recorded identifiers; if replay-critical inputs cannot be proven, replay fails explicitly.  
  **Tradeoff:** Version capture and immutable publication add operational overhead and make emergency fixes require new versions.

- **Invariant:** An admitted request must never have zero authoritative decision records or more than one authoritative decision record.  
  **Enforcement:** Stable `decision_id`, uniqueness on `tenant_id + idempotency_key`, immutable persistence semantics, and rejection if the authoritative record cannot be persisted exactly once.  
  **Tradeoff:** This adds duplicate-collapse and write-ordering constraints to the request path.

- **Invariant:** Published routing-policy state and captured experiment-reference state must never be mutated in place after publication or capture.  
  **Enforcement:** Immutable persistence, monotonic version validation, publish-time capture, snapshot construction from published identifiers only, and publication failure if the immutable version set cannot be recorded completely.  
  **Tradeoff:** Corrections require new versions instead of in-place edits.

### Tenant And Audit Integrity

- **Invariant:** Governed records must never cross tenant boundaries in read, write, join, or projection behavior.  
  **Enforcement:** Tenant-scoped schema fields, authorization checks, tenant filters on reads and writes, projection contracts keyed by tenant-bound stable identifiers, and fail-closed behavior when tenant identity cannot be proven.  
  **Tradeoff:** This increases indexing, query, and policy complexity.

- **Invariant:** Audit records must never be mutated after creation or removed before the required retention window expires.  
  **Enforcement:** Append-only audit storage, retention policy enforcement, immutable event payloads, auditable control-plane workflows, and operation failure or explicit incomplete-control-plane state when required audit persistence fails.  
  **Tradeoff:** This increases storage and retention-management cost.

- **Invariant:** Governed downstream records for an admitted request must never be emitted without the stable identifiers required to join them back to the same request lineage.  
  **Enforcement:** Schema contracts keyed by `decision_id` and related stable identifiers, projection validation, projection-status inspection APIs, and explicit incomplete-projection state when required join keys are missing.  
  **Tradeoff:** This creates tighter cross-service schema coupling.

- **Invariant:** A successful execution outcome must never be persisted unless it identifies the exact producer that served the request.  
  **Enforcement:** Outcome schema requirements for internal `model_version + manifest_uri` and external provider/model identity, validation before terminal outcome persistence, and explicit platform-failure recording when producer identity cannot be proven.  
  **Tradeoff:** This adds hot-path metadata capture cost.

### Training And Artifact Integrity

- **Invariant:** A post-validation training run must never publish a mixed or incomplete artifact set for a model version.  
  **Enforcement:** Manifest-last write discipline, artifact presence validation, consistency checks, and publication or registration only after local completeness; if any required artifact is missing or inconsistent, the run fails and no manifest is published.  
  **Tradeoff:** This increases training latency because partial outputs cannot be exposed early.

- **Invariant:** Canonical artifact URIs for a published model version must never point anywhere other than that version's published artifact set and must never use a non-canonical storage form.  
  **Enforcement:** Manifest-driven publication, URI validation, immutable manifest content, downstream use of manifest-produced URIs only, and publication failure if canonical URIs cannot be recorded or verified against the artifact set.  
  **Tradeoff:** This constrains storage layout and migration choices.

## Safe Resources

### Admission And Capacity

- **Invariant:** A request must never be admitted if admission would violate the budget or quota policy in force for that request.  
  **Enforcement:** Admission-control checks before execution, policy references in published routing policy, and fail-closed routing logic; if remaining budget, quota state, or policy validity cannot be proven, admission deterministically rejects or degrades according to policy.  
  **Tradeoff:** This trades off availability and some revenue opportunities in favor of spend control.

- **Invariant:** A model version must never receive routed traffic through an internal inference target before the corresponding serving workload is reconciled, ready, and bound to the intended target and model version.  
  **Enforcement:** Authoritative desired and observed deployment state, reconciliation, readiness-gated routing checks, target-to-version binding validation, and unroutable or fallback-only treatment when readiness or binding cannot be proven.  
  **Tradeoff:** This delays rollout and recovery until control-plane state converges.

## Safe Governance

### Authorization And Capability Boundaries

- **Invariant:** Protected operations must never proceed when authorization or capability binding cannot be proven.  
  **Enforcement:** Fail-closed authorization checks, policy-gated MCP bindings, tenant-scope validation, active-version checks before execution or mutation, and denial when uncertainty, timeout, missing binding, or validation failure occurs.  
  **Tradeoff:** This reduces availability during dependency failures but preserves security boundaries.

- **Invariant:** An MCP capability must never be callable unless its tenant scope, policy binding, and active authorization state are proven.  
  **Enforcement:** `MCPServerBinding`, policy references, active-version checks, execution-gateway validation before invocation, and fail-closed rejection for unknown, unauthorized, disabled, or unprovable bindings.  
  **Tradeoff:** This can reduce tool availability during policy or configuration drift.

### Experiment Integrity

- **Invariant:** The same `user_id + experiment_id` under the same effective bucket version must never resolve to different assignment outcomes while stickiness is expected to hold.  
  **Enforcement:** Server-side GrowthBook SDK evaluation only, captured experiment identity and bucket-version context, immutable exposure logging, and explicit `not_eligible` or failure when assignment inputs or sticky context cannot be proven.  
  **Tradeoff:** This limits freedom to change allocation behavior for already-sticky users.

- **Invariant:** An evaluated experiment assignment, including `not_eligible`, must never produce zero required exposure records or more than one canonical immutable exposure record.  
  **Enforcement:** Exposure schema contract, immutable persistence, assignment-time logging, and reconciliation on logging failure; if the required exposure record cannot be persisted, the failure is explicit and the traffic does not silently disappear from experiment accounting.  
  **Tradeoff:** This adds write amplification on the routing path.

- **Invariant:** Assignment results, exposure records, and analytics inputs must never refer to conflicting experiment identities for the same evaluated experiment event.  
  **Enforcement:** Captured experiment references, exposure schema requirements, analytics contracts that reuse the same identifiers, and incomplete or failed downstream analytics state when identity propagation is inconsistent.  
  **Tradeoff:** This increases routing-to-analytics contract coupling.

### Lifecycle Integrity

- **Invariant:** A repo-owned model version must never become candidate, production-active, deprecated, or retired through inference-time ambiguity or in-place mutation alone.  
  **Enforcement:** Lifecycle APIs, append-only transition records, validation gates, explicit override metadata for exception-based promotion, and fail-closed transition handling when target identity, approval metadata, or transition validity cannot be proven.  
  **Tradeoff:** This slows release velocity relative to implicit latest-version activation.

- **Invariant:** Prompt versions, agent definition versions, and MCP binding versions must never become active through mutable aliases, latest-row ambiguity, or in-place mutation.  
  **Enforcement:** Versioned artifact schemas, rollout APIs, explicit state transitions, routing or execution references by concrete version identifiers, and fail-closed behavior when the referenced version is missing, invalid, or inactive for the tenant.  
  **Tradeoff:** This adds operational overhead compared with mutable aliases.

### Online Feature Contract Integrity

- **Invariant:** When a Redis value is present for a governed customer realtime feature field, the final merged scoring input must never use the offline value for that same field instead.  
  **Enforcement:** Deterministic field-by-field merge logic, online feature definitions, parity inspection over normalized expected versus actual records, and explicit scoring failure or separately documented no-Redis fallback when the contract cannot be applied.  
  **Tradeoff:** This makes scoring behavior sensitive to Redis freshness and contract hygiene.

- **Invariant:** A governed Redis customer feature record must never omit the required identity and freshness fields or use a non-contract key pattern.  
  **Enforcement:** Configured key pattern, feature-definition contract, typed normalization, write-path validation, and parity inspection; records that fail the contract are treated as invalid governed serving records.  
  **Tradeoff:** This imposes schema discipline on a flexible store.

- **Invariant:** Rebuild or recovery must never silently redefine feature meaning or values for the same online feature-version contract without an explicit version change or documented reconciliation action.  
  **Enforcement:** Online feature definitions, version identifiers in records, rebuild controls, parity or reconciliation tracking, and explicit reconciliation state or new-version publication when rebuild cannot preserve contract meaning.  
  **Tradeoff:** This makes rebuild workflows slower and more explicit.

---

## Known Gaps That Must Be Preserved As Gaps

- `config/features/offline_feature_defs.yaml` is missing even though absorbed feature-assembly code depends on it
- `tools/demo_realtime_scoring.py` references `score_entities(...)`, which is absent
- some absorbed runtime defaults still embed data-platform-specific hostnames
