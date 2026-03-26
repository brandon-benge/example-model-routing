# Data Model

## Core Control-Plane Entities

### Entity: `AuthoritativeDecisionRecord`

- Purpose: single source of truth for every admitted inference request
- Source: inference routing layer
- Mutability: immutable
- Primary key:
  - `decision_id`
- Natural key:
  - `tenant_id + idempotency_key`
- Schema:
  - `decision_id`: string
  - `tenant_id`: string
  - `idempotency_key`: string
  - `request_identity`: object
    - `request_hash`: string
    - `subject_key`: string | null
    - `assignment_key`: string | null
    - `surface`: string
    - `request_attributes`: object
  - `route_id`: string
  - `policy_id`: string
  - `policy_version`: integer
  - `snapshot_version`: string
  - `region`: string
  - `experiment_context`: list[object]
    - `experiment_id`: string
    - `experiment_version`: integer
    - `variant_key`: string
  - `selected_target_id`: string | null
  - `admissible_target_ids`: list[string]
  - `decision_state`: enum `selected` | `rejected`
  - `decision_reason_code`: string
  - `budget_policy_reference`: string | null
  - `endpoint_policy_reference`: string | null
  - `decided_at`: timestamp
  - `metadata`: object
- Validation rules:
  - exactly one authoritative record may exist for a given `tenant_id + idempotency_key`
  - `policy_version`, `snapshot_version`, and `region` are required for every persisted decision
  - `selected_target_id` must be null only when `decision_state=rejected`

### Entity: `ExperimentVersion`

- Purpose: immutable published experiment configuration used during routing
- Source: control plane publish workflow
- Mutability: immutable
- Primary key:
  - `experiment_id + experiment_version`
- Schema:
  - `experiment_id`: string
  - `experiment_version`: integer
  - `experiment_name`: string
  - `owner`: string
  - `tenant_scope`: string or `global`
  - `route_scope`: list[string]
  - `status`: enum
    - `draft`
    - `validated`
    - `active`
    - `paused`
    - `completed`
    - `archived`
  - `assignment_key`: string
  - `assignment_unit`: enum
    - `tenant`
    - `user`
    - `session`
    - `request`
    - `entity`
  - `eligibility_rules`: object
  - `variants`: list[object]
    - `variant_key`: string
    - `allocation_percent`: number
    - `target_overrides`: list[string]
    - `is_control`: boolean
  - `success_metrics`: list[object]
    - `metric_name`: string
    - `direction`: enum `increase` | `decrease`
    - `aggregation`: enum `mean` | `rate` | `sum` | `p95` | `p99`
  - `guardrail_metrics`: list[object]
    - `metric_name`: string
    - `direction`: enum `increase` | `decrease`
    - `threshold_type`: enum `absolute` | `relative`
    - `threshold_value`: number
  - `start_criteria`: object
  - `stop_criteria`: object
  - `published_at`: timestamp
  - `published_by`: string
  - `description`: string
- Validation rules:
  - `experiment_version` must increase monotonically within `experiment_id`
  - variant allocation must sum to `100`
  - exactly zero or one variant may set `is_control=true`
  - `assignment_key`, `variants`, `success_metrics`, and `guardrail_metrics` are required before `active`

### Entity: `RoutingPolicyVersion`

- Purpose: immutable published routing-policy schema instance used by snapshot construction and routing replay
- Source: control plane publish workflow
- Mutability: immutable
- Primary key:
  - `policy_id + policy_version`
- Schema:
  - `policy_id`: string
  - `policy_version`: integer
  - `policy_name`: string
  - `tenant_scope`: string or `global`
  - `route_id`: string
  - `surface`: string
  - `regions`: list[string]
  - `target_set`: list[object]
    - `target_id`: string
    - `priority`: integer
    - `enabled`: boolean
    - `eligibility_predicates`: object
    - `capacity_reference`: string | null
  - `experiment_bindings`: list[object]
    - `experiment_id`: string
    - `experiment_version`: integer
    - `variant_target_map`: object
  - `default_target_order`: list[string]
  - `fallback_order`: list[string]
  - `timeout_policy`: object
    - `request_timeout_ms`: integer
    - `connect_timeout_ms`: integer | null
  - `retry_policy`: object
    - `max_attempts`: integer
    - `retryable_error_codes`: list[string]
  - `budget_policy_reference`: string | null
  - `quota_policy_reference`: string | null
  - `degradation_policy`: object
    - `on_budget_exhausted`: enum `reject` | `fallback` | `downgrade`
    - `on_target_unavailable`: enum `reject` | `fallback`
    - `on_snapshot_stale`: enum `reject` | `allow_with_flag`
  - `endpoint_policy_reference`: string | null
  - `publication_metadata`: object
    - `published_at`: timestamp
    - `published_by`: string
    - `change_reason`: string
- Validation rules:
  - `policy_version` must increase monotonically within `policy_id`
  - every `target_id` in `fallback_order` must exist in `target_set`
  - every bound experiment version must exist and be valid for the route scope
  - `default_target_order` and `fallback_order` must not reference disabled or unknown targets at publish time

### Entity: `ModelTarget`

- Purpose: versioned registration of a model/provider endpoint or model artifact target routable by the platform
- Source: control plane
- Mutability: mutable registration with immutable version references

### Entity: `Snapshot`

- Purpose: versioned regional decision snapshot combining routing policy, experiment, provider availability, budget, and endpoint policy state
- Source: snapshot distribution layer
- Mutability: immutable per version

### Entity: `ChargebackRecord` and `AuditRecord`

- Purpose: immutable derived records keyed by `decision_id`
- Source: projection workers
- Mutability: immutable

### Entity: `AuditRecord`

- Purpose: immutable audit and evaluation record derived from a routed decision
- Source: audit projection worker
- Mutability: immutable
- Primary key:
  - `decision_id`
- Schema:
  - `decision_id`: string
  - `tenant_id`: string
  - `route_id`: string
  - `policy_id`: string
  - `policy_version`: integer
  - `snapshot_version`: string
  - `region`: string
  - `selected_target_id`: string | null
  - `request_hash`: string
  - `normalized_input_ref`: string | null
  - `normalized_output_ref`: string | null
  - `subject_key`: string | null
  - `experiment_context`: list[object]
  - `execution_terminal_state`: string
  - `decision_timestamp`: timestamp
  - `completion_timestamp`: timestamp | null
  - `audit_status`: enum `ready` | `audit_failed`
  - `failure_reason`: string | null
  - `metadata`: object
- Validation rules:
  - one audit record may exist per `decision_id`
  - audit failure must be represented through `audit_status` rather than absence

### Entity: `ChargebackRecord`

- Purpose: immutable cost-attribution record derived from a routed decision
- Source: chargeback projection worker
- Mutability: immutable
- Primary key:
  - `decision_id`
- Schema:
  - `decision_id`: string
  - `tenant_id`: string
  - `selected_target_id`: string | null
  - `provider_id`: string | null
  - `provider_model_id`: string | null
  - `billable_state`: enum `billable` | `non_billable` | `chargeback_failed`
  - `input_tokens`: integer | null
  - `output_tokens`: integer | null
  - `metered_cpu_ms`: integer | null
  - `metered_memory_mb_ms`: integer | null
  - `currency`: string | null
  - `billed_amount`: number | null
  - `cost_dimensions`: object
  - `usage_timestamp`: timestamp | null
  - `failure_reason`: string | null
  - `metadata`: object
- Validation rules:
  - one chargeback record may exist per `decision_id`
  - chargeback failure must be represented through `billable_state=chargeback_failed` rather than absence

### Entity: `ExperimentExposureRecord`

- Purpose: immutable record of experiment assignment and exposure outcome for a routed request
- Source: inference routing layer
- Mutability: immutable
- Primary key:
  - `exposure_id`
- Natural key:
  - `decision_id + experiment_id + experiment_version`
- Schema:
  - `exposure_id`: string
  - `decision_id`: string
  - `tenant_id`: string
  - `experiment_id`: string
  - `experiment_version`: integer
  - `variant_key`: string
  - `assignment_key`: string
  - `assignment_unit`: string
  - `exposure_state`: enum `assigned` | `not_eligible`
  - `route_id`: string
  - `policy_id`: string
  - `policy_version`: integer
  - `snapshot_version`: string
  - `region`: string
  - `target_id`: string | null
  - `exposed_at`: timestamp
  - `metadata`: object
- Validation rules:
  - there may be at most one exposure record for a given `decision_id + experiment_id + experiment_version`
  - `variant_key` may be `not_eligible` only when `exposure_state=not_eligible`
  - `policy_version`, `snapshot_version`, and `region` must match the authoritative routing context

### Entity: `ExecutionOutcomeRecord`

- Purpose: immutable terminal execution record bound to a prior authoritative decision
- Source: execution gateway or execution projection layer
- Mutability: immutable
- Primary key:
  - `decision_id`
- Schema:
  - `decision_id`: string
  - `tenant_id`: string
  - `selected_target_id`: string | null
  - `provider_request_id`: string | null
  - `provider_id`: string | null
  - `provider_model_id`: string | null
  - `terminal_state`: enum `success` | `timeout` | `cancelled` | `provider_error` | `platform_error` | `rejected`
  - `finish_reason`: string | null
  - `error_code`: string | null
  - `latency_ms`: integer | null
  - `input_tokens`: integer | null
  - `output_tokens`: integer | null
  - `started_at`: timestamp | null
  - `completed_at`: timestamp
  - `metadata`: object
- Validation rules:
  - one execution outcome record may exist per `decision_id`
  - `terminal_state=success` requires `completed_at`
  - `error_code` is required for non-success states except `cancelled` when cancellation is self-describing in metadata

### Entity: `LabelRecord`

- Purpose: immutable downstream truth or observed-outcome record joinable to prior routed decisions and experiment exposures
- Source: downstream feedback ingestion
- Mutability: immutable
- Primary key:
  - `label_id`
- Natural key:
  - `label_namespace + label_key + observed_at`
- Schema:
  - `label_id`: string
  - `label_namespace`: string
  - `label_key`: string
  - `tenant_id`: string
  - `decision_id`: string | null
  - `subject_key`: string | null
  - `assignment_key`: string | null
  - `label_name`: string
  - `label_value`: string | number | boolean
  - `label_type`: enum `binary` | `multiclass` | `regression` | `event`
  - `observed_at`: timestamp
  - `event_timestamp`: timestamp | null
  - `source_system`: string
  - `confidence`: number | null
  - `metadata`: object
- Validation rules:
  - at least one of `decision_id`, `subject_key`, or `assignment_key` must be present
  - `label_type` must be compatible with the stored `label_value`

### Entity: `ReplayRecord`

- Purpose: immutable record describing a replay request and its resulting evaluated outcome
- Source: replay API/workflow
- Mutability: immutable
- Primary key:
  - `replay_id`
- Schema:
  - `replay_id`: string
  - `requested_by`: string
  - `requested_at`: timestamp
  - `tenant_id`: string
  - `source_mode`: enum `decision_id` | `request_identity`
  - `source_decision_id`: string | null
  - `request_identity`: object | null
  - `replay_mode`: enum `historical_bound` | `current_published`
  - `region`: string | null
  - `input_policy_id`: string | null
  - `input_policy_version`: integer | null
  - `input_experiment_versions`: list[object]
    - `experiment_id`: string
    - `experiment_version`: integer
  - `result_assignment`: object
    - `variant_key`: string
    - `experiment_bindings`: list[object]
  - `result_routing`: object
    - `selected_target_id`: string | null
    - `admissible_targets`: list[string]
    - `terminal_state`: enum `selected` | `rejected` | `error`
    - `error_code`: string | null
  - `divergence`: object
    - `diverged_from_original`: boolean
    - `divergence_reasons`: list[string]
  - `status`: enum `requested` | `completed` | `failed`
  - `failure_reason`: string | null
- Validation rules:
  - exactly one of `source_decision_id` or `request_identity` must be present
  - `historical_bound` replay must preserve the original bound version set when the source decision exists
  - `divergence` must be populated when `source_decision_id` is present and replay completes

## Added ML Workflow Entities

### Entity: `LogisticRegressionModel`

- Purpose: serialized trained model used by repo-owned model inference
- Source: `ml/train.py`
- Stored fields:
  - `feature_names`
  - `means`
  - `stds`
  - `weights`
  - `bias`
  - `label_name`
  - `feature_definition_version`

### Entity: `ModelManifest`

- Purpose: bind one trained model version to its artifact locations and metadata
- Source: `train_from_rows(...)`
- Fields:
  - `feature_group`
  - `label_name`
  - `feature_definition_version`
  - `artifact_paths`
  - `trained_at`
  - `artifact_uris`

### Entity: `ModelRegistryRow`

- Purpose: durable registry row used for latest-model lookup and audit of trained versions
- Source: `iceberg.silver.ml_model_registry`
- Written fields:
  - `feature_group`
  - `label_name`
  - `feature_definition_version`
  - `trained_at`
  - artifact URIs
  - local paths
  - `train_rows`
  - `test_rows`
  - `accuracy`
  - `precision`
  - `recall`
  - `roc_auc`
  - `status`

### Entity: `ModelLifecycleRecord`

- Purpose: immutable record of model lifecycle transitions for repo-owned model versions
- Source: model lifecycle governance workflow
- States represented:
  - `candidate`
  - `validated`
  - `production`
  - `rollback_candidate`
  - `deprecated`
  - `retired`
- Primary key:
  - `lifecycle_event_id`
- Natural key:
  - `feature_group + model_version + transitioned_at`
- Schema:
  - `lifecycle_event_id`: string
  - `feature_group`: string
  - `model_version`: string
  - `manifest_uri`: string
  - `from_state`: string | null
  - `to_state`: enum
    - `candidate`
    - `validated`
    - `production`
    - `rollback_candidate`
    - `deprecated`
    - `retired`
  - `transition_reason`: string
  - `transitioned_by`: string
  - `transitioned_at`: timestamp
  - `replaces_model_version`: string | null
  - `metadata`: object
- Validation rules:
  - lifecycle records are append-only
  - a model version may not transition out of `retired`
  - a `production` transition must identify either the replaced production version or explicitly state none exists
  - `manifest_uri` must match the registered version being transitioned

### Entity: `CustomerOnlineFeatureRecord`

- Purpose: Redis-served hot feature record used by customer realtime scoring
- Source: Flink online-feature job
- Redis key:
  - `features:customer:{customer_id}:v1`

### Entity: `OnlineFeatureDefinition`

- Purpose: authoritative serving contract for one online feature set
- Source: control-plane or configuration-managed feature-definition store
- Primary key:
  - `feature_name + feature_version`
- Contract fields:
  - `feature_name`: string
  - `feature_version`: string
  - `owner`: string
  - `entity_type`: string
  - `entity_key`: string
  - `redis_key_pattern`: string
  - `sources`: list[object]
    - `source_type`: enum `kafka` | `table` | `api`
    - `source_name`: string
    - `filters`: object
  - `aggregations`: list[object]
    - `field`: string
    - `function`: string
    - `as`: string
    - `window_seconds`: integer | null
    - `when`: object | null
  - `transformation_semantics`: object
  - `freshness_expectation_seconds`: integer
  - `ttl_seconds`: integer
  - `parity_strategy`: object
    - `parity_source`: string
    - `comparison_mode`: enum `exact` | `tolerance`
    - `tolerance`: number | null
  - `rebuild_strategy`: object
    - `rebuild_mode`: enum `stream_replay` | `batch_recompute` | `hybrid`
    - `rebuild_inputs`: list[string]
  - `status`: enum `draft` | `active` | `deprecated` | `retired`
  - `published_at`: timestamp
  - `published_by`: string
- Validation rules:
  - `feature_name + feature_version` must be immutable
  - `ttl_seconds` must be greater than or equal to `freshness_expectation_seconds`
  - active definitions must define at least one source and one output field
  - rebuild strategy and parity strategy are required before `active`

### Entity: `OnlineFeatureParityResult`

- Purpose: immutable record of one parity comparison for one online feature version and entity
- Source: serving-owned reconciliation workflow
- Mutability: immutable
- Primary key:
  - `parity_result_id`
- Natural key:
  - `feature_name + feature_version + entity_type + entity_id + compared_at`
- Schema:
  - `parity_result_id`: string
  - `feature_name`: string
  - `feature_version`: string
  - `entity_type`: string
  - `entity_id`: string
  - `expected_payload_ref`: string | null
  - `actual_payload_ref`: string | null
  - `mismatches`: list[object]
    - `field_name`: string
    - `expected_value`: string | number | boolean | null
    - `actual_value`: string | number | boolean | null
  - `result_state`: enum `passed` | `failed` | `parity_failed`
  - `compared_at`: timestamp
  - `compared_by`: string
  - `serving_context`: object
  - `metadata`: object
- Validation rules:
  - `result_state=passed` requires `mismatches=[]`
  - `result_state=failed` requires at least one mismatch entry

### Entity: `ScoreOutputRecord`

- Purpose: optional Redis-persisted score result for downstream use
- Source: scoring functions
- Redis keys:
  - `scores:customer:{customer_id}:purchase_propensity:v1`
  - `scores:campaign:{campaign_id}:success_propensity:v1`
  - `scores:advertiser:{advertiser_id}:budget_expansion_propensity:v1`

## External Entities Consumed But Not Owned

- upstream offline feature tables:
  - `iceberg.silver.customer_purchase_features_v1`
  - `iceberg.silver.customer_purchase_realtime_features_v1`
  - `iceberg.silver.campaign_success_features_v1`
  - `iceberg.silver.advertiser_budget_features_v1`
- scoring-context tables:
  - `iceberg.silver.silver_customer_daily_metrics`
  - `iceberg.silver.silver_order_header`
  - `iceberg.silver.customer_realtime_features_v1_parity`
  - `iceberg.silver.silver_campaign_daily_metrics`
  - `iceberg.silver.silver_advertiser_daily_metrics`
- upstream Kafka topic:
  - `events.session_event`

## Key Relationships

- `AuthoritativeDecisionRecord -> ExperimentVersion`: many-to-one
- `AuthoritativeDecisionRecord -> RoutingPolicyVersion`: many-to-one
- `AuthoritativeDecisionRecord -> Snapshot`: many-to-one
- `AuthoritativeDecisionRecord -> ChargebackRecord`: one-to-one derived
- `AuthoritativeDecisionRecord -> AuditRecord`: one-to-one derived
- `AuthoritativeDecisionRecord -> ExecutionOutcomeRecord`: one-to-one derived
- `AuthoritativeDecisionRecord -> ExperimentExposureRecord`: one-to-many where multiple experiments may be evaluated
- `ExperimentExposureRecord -> LabelRecord`: many-to-many by downstream join key or decision linkage
- `ReplayRecord -> AuthoritativeDecisionRecord`: optional many-to-one when replay originates from a prior decision
- `ModelRegistryRow -> ModelManifest`: manifest URI points to the canonical artifact manifest for the trained version
- `ModelRegistryRow -> ModelLifecycleRecord`: one-to-many immutable lifecycle transitions
- `CustomerOnlineFeatureRecord -> customer_id`: one current hot-feature record per versioned Redis key
- `OnlineFeatureParityResult -> CustomerOnlineFeatureRecord`: many-to-one by feature identity and entity key
- `OnlineFeatureParityResult -> OnlineFeatureDefinition`: many-to-one

## Open Data-Model Gaps

- the registry table has no repository-level stable model-version identifier beyond row content and `trained_at`
- `offline_feature_defs.yaml` is missing, so the source-controlled offline feature-definition entity is incomplete
- the current specs still need a single canonical join-key rule between routed decisions, experiment exposures, labels, and replay records when `decision_id` is unavailable downstream
