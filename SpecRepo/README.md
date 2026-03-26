# SpecRepo

This repository remains the governed source of truth for the model routing and policy control plane.

It is now extended to also govern the ML-owned workflow absorbed from `../example-ml-platform`, specifically:

- model training
- model registry writes and reads
- inference APIs for ML-owned models
- experimentation and rollout logic for those models
- Redis-backed online feature serving
- serving-owned offline/online parity checks

The intent is additive, not substitutive. Existing routing, policy, audit, chargeback, and snapshot disciplines remain part of the governed system.

## How to Read This Repo

Recommended order:

1. `PROBLEM.md`
2. `INVARIANTS.md`
3. `REQUIREMENTS.md`
4. `DATA_MODEL.md`
5. `CONSISTENCY.md`
6. `ARCHITECTURE.md`

## Boundary

This repo owns:

- routing and policy control-plane behavior
- ML training and model artifact lifecycle for repo-owned models
- model registry reads and writes
- inference APIs
- Redis-backed online feature serving
- serving-owned parity and reconciliation behavior

This repo does not absorb ownership of the upstream data platform. The data platform remains an external dependency that publishes offline feature tables and upstream events consumed here.

## External Dependencies Called Out Explicitly

The absorbed ML workflow depends on published offline feature tables from the data platform, especially:

- `iceberg.silver.customer_purchase_features_v1`
- `iceberg.silver.customer_purchase_realtime_features_v1`
- `iceberg.silver.campaign_success_features_v1`
- `iceberg.silver.advertiser_budget_features_v1`

Current code also directly reads:

- `iceberg.silver.silver_customer_daily_metrics`
- `iceberg.silver.silver_order_header`
- `iceberg.silver.customer_realtime_features_v1_parity`
- `iceberg.silver.silver_campaign_daily_metrics`
- `iceberg.silver.silver_advertiser_daily_metrics`

These remain external published interfaces.

## Working Rule

When existing control-plane behavior and absorbed ML workflow behavior both apply, the governed system is the union of those responsibilities. Additive ML ownership must not silently weaken prior guarantees around determinism, authorization, auditability, versioning, or replay.
