---
type: Component
title: CoverService
description: CoverService (com.tidewell.policy.cover.CoverService) is the domain service responsible for querying and aggregating policy coverage lines, perils, limits, and excesses across policies owned by a given customer.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/entities/cover-service.md
tags:
- policy-admin
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/java/com/tidewell/policy/cover/CoverService.java
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/resources/db/schema.sql
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

<!-- anchor: src/main/java/com/tidewell/policy/cover/CoverService.java:L1-L30 -->
<!-- anchor: src/main/resources/db/schema.sql:L1-L18 -->

# CoverService

`CoverService` (`com.tidewell.policy.cover.CoverService`) is the domain service responsible for querying and aggregating policy coverage lines, perils, limits, and excesses across policies owned by a given customer.

## Responsibilities

- **Customer Cover Aggregation**: Provides `coverForCustomer(String customerId)` to collect all cover lines across all policies belonging to a customer. This method is utilized during policy lookup queries (such as `GET /v1/policies?customerId=`).
- **Policy and Coverage Traversal**: Orchestrates two-step retrieval across repositories:
  1. Queries `PolicyRepository` to retrieve all policy IDs for the given `customerId`.
  2. Iterates over each returned `PolicyRow` and queries `CoverRepository` for its associated coverage lines (`findByPolicyId`).

## Dependencies

- **`CoverRepository`**: Internal interface providing `findByPolicyId(String policyId)`. Queries the underlying `policy_cover` database table (see [[entities/cover-model]]).
- **`PolicyRepository`**: Internal interface providing `findByCustomerId(String customerId)` returning `PolicyRow(String id)`. Queries the underlying `policies` database table (see [[entities/policy-model]]).
- **[[entities/policy-controller]]**: Consumes coverage data when serving policy inspection and retrieval endpoints.
- **[[decisions/adr-0003-services-own-their-data]]**: Ensures external services interact through `policy-admin` interfaces rather than querying `policy_cover` and `policies` directly.

## Data Schema Interaction

`CoverService` operates over data backed by the following database tables:

- **`policies`**:
  - `id` (`UUID PRIMARY KEY`)
  - `customer_id` (`UUID NOT NULL`)
  - `product` (`TEXT` - `home` | `motor`)
  - `status` (`TEXT` - `active` | `lapsed` | `cancelled`)
  - `start_date` (`DATE`)
  - `renewal_date` (`DATE`)
  - `annual_premium_pence` (`BIGINT`)

- **`policy_cover`**:
  - `policy_id` (`UUID REFERENCES policies(id)`)
  - `peril` (`TEXT`, e.g., `fire`, `flood`, `theft`, `escape_of_water`, `accidental_damage`)
  - `limit_pence` (`BIGINT`)
  - `excess_pence` (`BIGINT`)
  - `active_from` (`DATE`)
  - `active_to` (`DATE`)
