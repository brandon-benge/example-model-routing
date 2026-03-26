# Invariants

1. Given the same routing inputs, policy version, experiment configuration, and availability snapshot, the platform must reproduce the same experiment assignment and routing decision for audit and replay purposes. Model output determinism is out of scope.
2. A request must never be admitted if it would violate its allocated budget; under quota exhaustion, the system must deterministically degrade or reject at admission time.
3. Routing decisions and all derived records must be tenant-isolated.
4. Every admitted request must have exactly one immutable authoritative decision record identified by a stable `decision_id`.
5. Published routing policy and experiment configuration versions are immutable and monotonically versioned.
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
18. Experiment assignment, exposure logging, and experiment analytics inputs must bind to the same immutable experiment version and decision identifiers used by routing.
19. A published routing policy version must be schema-valid, explicitly versioned, and replayable; implicit operator intent is not an acceptable routing contract.
20. Repo-owned model lifecycle transitions must be explicit and monotonic; a model cannot become production-active, deprecated, or retired through inference-time ambiguity alone.
21. Online feature definitions must declare authoritative entity key, version identity, freshness/TTL expectations, and parity obligations before they are treated as serving contracts.
22. Online feature rebuild or recovery paths must not silently redefine feature values for the same feature version without an explicit version change or documented reconciliation action.
23. If a dedicated GPU node pool exists, only inference-serving pods that execute model inference may schedule onto it; control-plane APIs, routing APIs, workers, storage, and supporting services must run on default non-GPU nodes.
24. The normal model-release path is: validation passes, artifacts and registry state are published as a candidate version, and production activation requires an explicit promotion step.
25. Validation failure must block the normal candidate-first release flow, but the governed system must still support an explicit exception-based promotion path with auditable override metadata.
26. When suitable shared platform resources already exist in `../example-data-pipeline-w-ml`, this repo must prefer extending or namespacing those resources over creating duplicate platform services.
27. Reuse of shared PostgreSQL, MinIO, Kafka, Kafka Connect, Iceberg, Schema Registry, and dbt resources must preserve ownership boundaries, tenant isolation, and naming clarity; reuse does not transfer ownership of the upstream platform into this repo.
28. A model version must not receive routed traffic through an internal inference target until the corresponding serving workload has been reconciled, is ready, and is bound to the intended model version and target identity.
29. API-driven promotion or experiment rollout may create or update internal inference pods, but those actions must mutate explicit desired state rather than issuing ad hoc unmanaged pod creation.
30. CockroachDB is the authoritative store for desired and observed state of internal inference deployments, including reconciliation progress and readiness-gated serving controls; routing eligibility for internal targets must derive from that authoritative state.

## Known Gaps That Must Be Preserved As Gaps

- `config/features/offline_feature_defs.yaml` is missing even though absorbed feature-assembly code depends on it
- `tools/demo_realtime_scoring.py` references `score_entities(...)`, which is absent
- some absorbed runtime defaults still embed data-platform-specific hostnames
