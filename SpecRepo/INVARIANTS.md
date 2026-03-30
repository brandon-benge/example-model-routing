# Invariants

1. Given the same routing inputs, policy version, referenced GrowthBook experiment state, and availability snapshot, the platform must reproduce the same experiment assignment and routing decision for audit and replay purposes. Model output determinism is out of scope.
2. A request must never be admitted if it would violate its allocated budget; under quota exhaustion, the system must deterministically degrade or reject at admission time.
3. Routing decisions and all derived records must be tenant-isolated.
4. Every admitted request must have exactly one immutable authoritative decision record identified by a stable `decision_id`.
5. Published routing policy versions and captured GrowthBook experiment references are immutable and monotonically versioned or timestamped for replay.
6. Authorization boundaries are fail-closed.
7. Audit records for routing decisions and configuration publication are immutable and retained for the required compliance window.
8. The absorbed ML workflow must not claim ownership of upstream offline feature generation. All upstream feature tables remain external published dependencies.
9. A training run that proceeds past validation must emit a self-consistent artifact set for one model version: dataset, model, metrics, and manifest.
10. When artifact publication is enabled, canonical artifact URIs must be recorded and must point to MinIO-compatible `s3://` locations.
11. When model registration is enabled, the registry write must target `iceberg.silver.ml_model_registry`.
12. Inference for repo-owned models must resolve the active manifest from the registry rather than depending on a prewarmed local cache.
13. The customer realtime inference path must merge offline context with Redis online features, and Redis values must override offline values field-by-field when present.
14. Customer online feature records in Redis must use the configured key pattern and carry `feature_version`, `last_event_ts`, `updated_at`, and `ttl_seconds`.
15. Serving-owned parity checks must compare normalized Redis customer records against expected records derived from offline and parity inputs.
16. Platform management in this repo is API-managed; the governed system does not require or assume a first-party browser UI.
17. Every admitted request must produce data that can be joined across decision, experiment exposure, execution outcome, audit, and chargeback records by stable identifiers.
18. Every execution outcome must map back to the exact model identity that produced it; internal outcomes must identify the exact model version and manifest, and external outcomes must identify the exact provider/model identifier returned or configured at execution time.
19. GrowthBook is the authoritative experimentation control plane for experiment definition, assignment, targeting, exposure semantics, traffic allocation, and evaluation configuration in the governed system.
20. Experiment assignment must be computed through the server-side GrowthBook SDK and must be deterministic and idempotent for the same `user_id + experiment_id`, with user-level stickiness preserved across repeated evaluations of the same effective bucket version.
21. Every experiment assignment, including `not_eligible`, must emit exactly one immutable exposure record containing at least `experiment_id`, `user_id`, `model_version`, assignment timestamp, and `variant_id` when assigned.
22. All governed experiment metrics must be defined as SQL over warehouse tables queryable through Iceberg/Trino; ad hoc in-memory or UI-only metric definitions are not sufficient.
23. Experiment evaluation must use the GrowthBook statistical engine, including CUPED variance reduction, Bayesian inference, and sequential testing where supported by the configured analysis.
24. Experiment targeting and segmentation rules must support user attributes, geography, device type, and explicit eligibility filters.
25. Experiment lifecycle management must support `draft`, `running`, `paused`, and `completed` states through GrowthBook-managed experiment state and must support an immediate kill switch that stops new assignment without this repo mutating a duplicate experiment-definition payload.
26. Traffic allocation may change dynamically for a running GrowthBook experiment, but those changes must not break deterministic assignment consistency for already-sticky users within the same effective bucket version.
27. The experimentation control plane must support backend and ML experimentation, including model routing, prompt experimentation, and feature-pipeline variants, under the same assignment and exposure contract.
28. Experiment assignment, exposure logging, and experiment analytics inputs must bind to the same GrowthBook experiment identity, GrowthBook rule or phase identity when applicable, and decision identifiers used by routing.
29. A published routing policy version must be schema-valid, explicitly versioned, and replayable; implicit operator intent is not an acceptable routing contract.
30. Repo-owned model lifecycle transitions must be explicit and monotonic; a model cannot become production-active, deprecated, or retired through inference-time ambiguity alone.
31. Online feature definitions must declare authoritative entity key, version identity, freshness/TTL expectations, and parity obligations before they are treated as serving contracts.
32. Online feature rebuild or recovery paths must not silently redefine feature values for the same feature version without an explicit version change or documented reconciliation action.
33. If a dedicated GPU node pool exists, only inference-serving pods that execute model inference may schedule onto it; control-plane APIs, routing APIs, workers, storage, and supporting services must run on default non-GPU nodes.
34. The normal model-release path is: validation passes, artifacts and registry state are published as a candidate version, and production activation requires an explicit promotion step.
35. Validation failure must block the normal candidate-first release flow, but the governed system must still support an explicit exception-based promotion path with auditable override metadata.
36. This repo must run on Kubernetes in DigitalOcean and must reuse the existing shared platform resources already provisioned for `../example-data-pipeline-w-ml` instead of introducing duplicate PostgreSQL, MinIO, Kafka, Kafka Connect, Iceberg, Schema Registry, or dbt services.
37. Reuse of shared PostgreSQL, MinIO, Kafka, Kafka Connect, Iceberg, Schema Registry, and dbt resources must preserve ownership boundaries, tenant isolation, and naming clarity; reuse does not transfer ownership of the upstream platform into this repo.
38. A model version must not receive routed traffic through an internal inference target until the corresponding serving workload has been reconciled, is ready, and is bound to the intended model version and target identity.
39. API-driven promotion or experiment rollout may create or update internal inference pods, but those actions must mutate explicit desired state rather than issuing ad hoc unmanaged pod creation.
40. CockroachDB is the authoritative store for desired and observed state of internal inference deployments, including reconciliation progress and readiness-gated serving controls; routing eligibility for internal targets must derive from that authoritative state.
41. DigitalOcean-hosted GPU capacity is a governed platform resource; tenant or workload allocation, scheduling, and scaling decisions must be explicit, auditable, and bounded by quota or policy rather than ad hoc operator action.
42. Prompt versions, agent definitions, and MCP tool bindings must move through explicit versioned rollout states; they must not become active through implicit latest-row or in-place mutation behavior.
43. MCP-exposed tools and enterprise data connections must be tenant-scoped, policy-gated, and fail closed when capability binding or authorization cannot be proven.
44. The initial platform build must remain deployable without requiring Keycloak, External Secrets, service mesh, OpenTelemetry, Prometheus, Loki, or Tempo/Jaeger; authentication, secret delivery, and operational visibility may initially be satisfied by Kubernetes-native and application-native mechanisms.
45. Core control-plane, routing, model lifecycle, prompt lifecycle, agent lifecycle, and MCP contracts must remain portable enough that optional future cloud-provider adapters can be added without changing authoritative API semantics or stored decision records.

## Known Gaps That Must Be Preserved As Gaps

- `config/features/offline_feature_defs.yaml` is missing even though absorbed feature-assembly code depends on it
- `tools/demo_realtime_scoring.py` references `score_entities(...)`, which is absent
- some absorbed runtime defaults still embed data-platform-specific hostnames
