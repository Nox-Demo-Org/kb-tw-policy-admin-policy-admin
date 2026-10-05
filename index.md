---
okf_version: '0.2'
title: policy-admin
description: policy-admin is the central policy management service for Tidewell Mutual, responsible for the lifecycle of home and motor insurance policies including policy issuance, mid-term adjustments, annual renewals, and cancellations.
generated:
  at: '2026-10-05T12:44:46Z'
---

# policy-admin

`policy-admin` is the central policy management service for Tidewell Mutual, responsible for the lifecycle of home and motor insurance policies including policy issuance, mid-term adjustments, annual renewals, and cancellations.

### Main Components
- **PolicyController**: Spring REST controller exposing endpoints to retrieve policy data (`GET /v1/policies/{id}`), issue new policies (`POST /v1/policies`), and submit mid-term adjustments (`POST /v1/policies/{id}/changes`).
- **CoverService**: Service resolving coverage limits and perils associated with policies.
- **RenewalScheduler**: Nightly cron-scheduled job scanning for active policies approaching renewal (`renewal.notice-days`) and publishing renewal due notifications.
- **RatingClient**: gRPC client connecting to `rating-service` (`RatingService.Price`) for real-time policy and mid-term adjustment repricing.
- **PolicyEvents & Subscribers**: Pub/Sub event infrastructure managing outbound domain events (`issued`, `renewal.due`, `cancelled`) and inbound event triggers (`customer.profile.updated`, `billing.payment.missed`).

### Data Flows
1. **Policy Issuance**: New policies are posted to `/v1/policies`, persisted into database tables `policies` and `policy_cover`, and emitted over GCP Pub/Sub via `policy.policy.issued` to billing and identity services.
2. **Mid-term Changes**: Changes submitted to `/v1/policies/{id}/changes` invoke `rating-service` via gRPC for recalculation and update the policy record.
3. **Renewals**: Nightly execution by `RenewalScheduler` queries policies reaching the notice window and broadcasts `policy.renewal.due` to billing and notifications.
4. **Event Ingestion**: Ingests `customer.profile.updated` to update policyholder records and `billing.payment.missed` to trigger cancellation workflows.

### Key Design Decisions
- **Service Data Ownership (ADR-0003)**: Direct database queries by external systems into `policies` and `policy_cover` tables are deprecated; consumers must interact exclusively via REST endpoints and domain events.
- **Pence Monetary Standard**: All financial values (premiums, limits, excesses) are stored and transmitted as 64-bit integer values in pence.

<!-- okf:contents -->

## Contents

- [Concepts and flows](/concepts/index.md) — 3 pages. Flows, lifecycles and cross-cutting mechanisms.
- [Architecture decisions](/decisions/index.md) — 1 page. One ADR per architecture decision the code or documents make evident.
- [Components and data models](/entities/index.md) — 7 pages. One page per significant component and core data model.
- [Interfaces and references](/summaries/index.md) — 1 page. API, event and module references for the application.
- [Change log](/log.md) — every generation and sync, newest first.

<!-- /okf:contents -->
