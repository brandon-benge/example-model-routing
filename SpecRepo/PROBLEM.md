# Problem Framing

## One-Sentence Mission

We are building a DigitalOcean-hosted, multi-tenant AI platform that deterministically routes and serves models across tenants, and we now also own the training, model-registry, internal inference deployment, hot-feature-serving, prompt and agent rollout, MCP server hosting, and MCP integration workflow required to operate that platform.

## Users / Actors

Primary:

- internal product teams consuming routed inference
- enterprise customers receiving routed model behavior
- ML/platform engineers training and promoting repo-owned models
- agent and application engineers shipping prompt, tool, and agent workflows

Secondary:

- platform management
- SRE
- finance and chargeback consumers
- upstream data platform teams publishing offline features and events
- security and compliance stakeholders reviewing governed platform behavior

## Scope Boundaries

### In Scope

- routing and policy control plane for inference requests
- request-time model selection based on policy, availability, latency, cost, and experiment state
- budget and authorization enforcement
- versioned routing policy publication and immutable capture of referenced GrowthBook experiment bindings
- immutable routing, audit, and chargeback records
- feedback and data contracts for decisions, outcomes, exposures, labels, and replay
- experiment analytics, governance, and guardrail evaluation for routing-controlled experiments
- GrowthBook as the experimentation control plane for feature flags, assignment, targeting, exposure tracking, experiment analysis, and decision rollout evaluation
- API-managed operator workflows for control-plane administration
- Kubeflow-managed ML pipelines, model training workflows, ML experiment tracking, and model deployment workflows for repo-owned models
- ML-owned model training for the current baseline classifiers
- model artifact publication to MinIO-compatible object storage
- model registry writes and reads through `iceberg.silver.ml_model_registry`
- model lifecycle governance for candidate, production, rollback, deprecation, and retirement state
- inference APIs for repo-owned customer, campaign, and advertiser models
- execution routing across both internally hosted inference services and externally hosted provider endpoints
- deployment and reconciliation of internally hosted inference workloads, including API-driven creation of serving pods in dedicated regular or GPU-backed namespaces when required by promotion or experiment rollout
- PostgreSQL-backed authoritative serving state for internal inference deployments, reconciliation progress, and readiness-gated controls
- DigitalOcean Kubernetes GPU node pool operations for internal inference workloads, including quota-aware scheduling, right-sizing, and capacity reporting when promotion explicitly selects GPU-backed serving
- deployment and reconciliation of internally hosted MCP services used by governed agent paths
- PostgreSQL-backed authoritative serving state for hosted MCP services, including readiness, endpoint publication, and auth-bound activation controls
- Redis-backed online feature serving for the customer realtime path
- serving-owned offline/online parity and reconciliation for that online feature path
- broader online feature platform semantics for feature definitions, freshness, rebuild, and serving contracts
- API-managed rollout semantics for repo-owned models, prompts, and agents, including candidate, canary, shadow, A/B, rollback, deprecation, and retirement controls
- agentic orchestration contracts plus MCP server hosting and integration for governed tool and enterprise data access
- tenant-scoped usage accounting and cost allocation for training and inference workloads
- integration with shared prerequisite platform resources from `../example-data-pipeline-w-ml` where reuse is viable
- optional cloud portability patterns so execution targets may later extend beyond DigitalOcean without changing core control-plane contracts

### Out of Scope

- a first-party browser UI for platform management
- upstream offline feature-table generation and backfills
- ownership of the data platform's generic Iceberg datasets outside ML registry writes
- Kafka topic production
- schema-registry administration
- product-specific quality-threshold definition
- mandatory dependence on AWS, Azure, or other cloud-managed AI services for core operation
- heavyweight identity, secrets, service-mesh, or observability platform components beyond required Keycloak and Kubernetes-native delivery mechanisms in the initial build

## Ownership Boundary Versus Data Platform

This repo now owns:

- model training
- model registry writes and reads
- inference APIs
- experimentation and rollout logic
- Kubeflow-backed ML pipeline orchestration, ML experiment tracking, and model deployment workflows for repo-owned models
- prompt and agent rollout contracts
- MCP integration contracts
- hosted MCP service deployment and lifecycle control
- Redis-backed online feature serving
- offline/online feature parity checks owned by serving
- DigitalOcean-hosted internal inference deployment with explicit regular-vs-GPU serving-class control

The upstream data repo remains an external dependency only. It publishes offline feature tables and upstream events, but those datasets and pipelines are not re-owned here.

Shared prerequisite resources from `../example-data-pipeline-w-ml` may be reused and extended rather than duplicated, especially:

- PostgreSQL
- MinIO-compatible object storage
- Kafka
- Kafka Connect
- Iceberg
- Schema Registry
- dbt

This reuse is mandatory for deployment in DigitalOcean Kubernetes. This repo must consume the existing PostgreSQL, MinIO, Kafka, and Iceberg platform services rather than introducing duplicate services here, while keeping interfaces portable enough that optional future cloud adapters can be layered on later.

## Explicit Upstream Dependencies

The absorbed training workflow depends on:

- `iceberg.silver.customer_purchase_features_v1`
- `iceberg.silver.customer_purchase_realtime_features_v1`
- `iceberg.silver.campaign_success_features_v1`
- `iceberg.silver.advertiser_budget_features_v1`

The current scoring code additionally depends on:

- `iceberg.silver.silver_customer_daily_metrics`
- `iceberg.silver.silver_order_header`
- `iceberg.silver.customer_realtime_features_v1_parity`
- `iceberg.silver.silver_campaign_daily_metrics`
- `iceberg.silver.silver_advertiser_daily_metrics`

## Success Definition

The system is successful when:

- routing decisions remain deterministic, version-bound, and auditable
- admitted requests remain attributable and budget-safe
- decision, exposure, outcome, audit, chargeback, and label data remain joinable under explicit contracts
- experiment governance can distinguish draft, running, paused, and completed states with guardrail-aware analysis inputs
- GrowthBook-driven experiments use deterministic server-side assignment with user-level stickiness and warehouse-SQL metric definitions
- GrowthBook remains the system of record for feature flags, A/B tests, experiment analysis, and decision rollout control
- Kubeflow provides the system workflow for ML pipelines, model training, ML experiment tracking, and model deployment workflows
- routing policy can be reasoned about as an explicit schema rather than ad hoc operator intent
- repo-owned model training can materialize artifacts, publish them, and register them
- repo-owned models can move through an explicit lifecycle rather than implicit latest-row selection only
- inference can resolve the current repo-owned model version from the registry
- internal inference targets can be deployed, become ready, and receive traffic under explicit control-plane and experiment-rollout rules
- hosted MCP services can be deployed, become ready, and be callable only under explicit control-plane, tenant-policy, and auth-bound rollout rules
- regular and GPU-backed inference capacity can be partitioned, scheduled, and reported with tenant-aware usage visibility
- experiment evaluation uses GrowthBook-compatible statistics over Iceberg/Trino-backed metrics, including CUPED, Bayesian inference, and sequential testing
- prompts and agents can move through explicit rollout paths under the same governance discipline as models
- MCP-exposed tools and data sources can be bound to explicit capability schemas, tenant policies, and auditable invocation paths
- internally hosted MCP services can be activated through explicit lifecycle and readiness controls rather than ad hoc endpoint configuration
- customer realtime scoring can merge offline context with Redis-backed hot features
- campaign and advertiser scoring can execute from offline features
- serving parity checks can detect mismatches between expected and actual Redis online records
- online feature definitions can express entity keys, freshness, TTL, rebuild expectations, and parity obligations
- the baseline platform remains lean, API-managed, and deployable on DigitalOcean with required Keycloak, without requiring External Secrets, service mesh, or a centralized observability stack in the initial build
- Keycloak can be populated declaratively through GitOps as the identity bridge until a fuller security platform is available, without weakening authorization or audit requirements

## Explicit Open Questions

- `config/features/offline_feature_defs.yaml` is referenced by the absorbed ML code but missing
- `tools/demo_realtime_scoring.py` references `score_entities(...)`, which is absent
- hard-coded data-platform hostnames still exist in some absorbed runtime defaults
- the current rollout mechanism for absorbed ML models is effectively latest-manifest selection by `trained_at`; no separate model-release state machine exists in code
- the current absorbed training flow does not yet define a stable dataset-version contract separate from feature table names and timestamps
- the intended normal promotion policy is candidate-first after successful validation, with explicit production promotion later; promotion must also explicitly declare the serving class and therefore the target inference namespace; manual override promotion remains possible by exception when validation does not pass
