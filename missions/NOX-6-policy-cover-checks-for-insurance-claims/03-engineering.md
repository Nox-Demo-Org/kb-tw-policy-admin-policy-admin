---
mission: NOX-6
title: 'Policy cover checks for insurance claims'
role: engineering
status: draft
version: 1
author: dev
ai_drafted: false
---

# Engineering design: Policy cover checks for insurance claims

## Applications changing
- **`claims-management` (Claims Handling Team)**: Must change. `CoverCheckRepository` currently executes direct SQL read queries against the `policy_cover` table in the `policy-admin` database [[kb:claims-management/decisions/direct-database-access-for-cover-check]]. It must be refactored to consume the `policy-admin` REST API (`GET /v1/policies/{id}`) over HTTP, removing cross-database coupling and establishing integration and contract test suites.
- **`policy-admin` (Policy Admin Team)**: Must **not** change. The existing endpoint `GET /v1/policies/{id}` already exposes the complete `PolicyView` model, including the embedded collection of `CoverLine` records (`peril`, `limitPence`, `excessPence`) [[kb:policy-admin/entities/cover-model]].
- **`claims-intake`, `billing-service`, `fraud-scoring`, `payments-gateway`, `customer-portal`**: Must **not** change. These downstream and upstream services have no direct coupling to the internal cover check data access layer of `claims-management`.

## Approach
`claims-management` will retire direct database access to `policy-admin`'s PostgreSQL database and implement an HTTP client adapter within the cover verification subsystem [[kb:claims-management/entities/cover-check]]:

1. **HTTP Client Integration**: Refactor `CoverCheckRepository` to query `policy-admin` via `GET /v1/policies/{id}` [[kb:policy-admin/entities/policy-controller]]. The repository will extract peril eligibility, maximum coverage limits (`limitPence`), and applicable deductible amounts (`excessPence`) directly from the returned `PolicyView.cover` list [[kb:policy-admin/entities/cover-model]].
2. **Resilience & Connection Management**: Configure dedicated HTTP client timeouts (connect timeout 500 ms, read timeout 1500 ms) and connection pooling to keep p95 latency under the target threshold of ≤ 100 ms [[kb:claims-management/decisions/direct-database-access-for-cover-check]].
3. **Error & Edge Case Mapping**:
   - `404 Not Found`: Translate to a policy-not-found domain exception.
   - Peril missing in `cover` list: Evaluate claim as non-covered peril, preserving standard triage routing [[kb:claims-management/concepts/claim-lifecycle]].
   - `5xx` / Network timeouts: Raise a transient, retryable cover-check error without mutating claim state.
4. **Contract Testing**: Implement Consumer-Driven Contract (CDC) tests against `policy-admin`'s OpenAPI specification to guarantee contract alignment on `GET /v1/policies/{id}`.

## Contracts affected

| Contract | Owner App | Consumers | Status |
| :--- | :--- | :--- | :--- |
| `GET /v1/policies/{id}` | `policy-admin` | `customer-portal`, `claims-intake`, `billing-service`, `claims-management` (new consumer) | **Unchanged** |
| `GET /v1/claims/{id}` | `claims-management` | `customer-portal`, internal claims UI | **Unchanged** |
| `claims.handler.assigned` | `claims-management` | Internal event stream | **Unchanged** |
| `claims.claim.settled` | `claims-management` | `payments-gateway`, internal event stream | **Unchanged** |
| `policy_cover` (Direct DB table read) | `policy-admin` | `claims-management` (retiring access) | **Breaking (Access Revoked)** |

## Must not break
- **Claim Assessment & Triage Lifecycle**: Claim state transitions (`reported` → `in_review`) and triage flows must remain identical [[kb:claims-management/concepts/claim-lifecycle]].
- **Peril and Excess Calculations**: Cover checks must return exact matching peril limits and deductibles identical to previous SQL query results [[kb:claims-management/entities/cover-check]].
- **Pence Representation**: Deductibles and limits must remain in integer pence (`long`) without precision loss [[kb:policy-admin/entities/cover-model]].
- **Existing Consumers of `policy-admin`**: The payload structure of `GET /v1/policies/{id}` must not be modified or disrupted for existing clients (`customer-portal`, `claims-intake`, `billing-service`) [[kb:policy-admin/summaries/api-spec]].

## Architecture, guardrails and standards
- **ADR-0003 (Each Service Owns Its Data)**: Enforces service encapsulation by prohibiting direct database queries across application boundaries [[kb:policy-admin/decisions/adr-0003-services-own-their-data]]. This design eliminates the expired March 2024 temporary exception [[kb:claims-management/decisions/direct-database-access-for-cover-check]] and unblocks `policy-admin` schema refactoring.
- **Pence Monetary Standard**: All financial values (`annualPremiumPence`, `limitPence`, `excessPence`) are strictly 64-bit integers (`long`/`int64`) in pence [[kb:policy-admin/entities/cover-model]].
- **Service Layering**: Keep HTTP communication encapsulated inside the data access/repository layer (`CoverCheckRepository`), exposing clean domain models to `HandlerAssignmentService` and claim evaluation coordinators.
- **HTTP Client Guardrails**: HTTP calls must specify explicit connect/read timeouts and propagate retryable failures cleanly to prevent thread pool exhaustion.

## Test strategy

| Level | What it proves | AC or Contract Covered |
| :--- | :--- | :--- |
| **Unit** | `CoverCheckRepository` correctly maps `PolicyView` and `CoverLine` DTOs into domain cover objects, accurately extracting peril limits and calculating excess in pence. | AC-1, AC-2, AC-3 |
| **Integration** | `CoverCheckRepository` HTTP client handles WireMock stubs for `GET /v1/policies/{id}`: HTTP 200 (valid cover lines), HTTP 200 (peril absent), HTTP 404 (missing policy), and HTTP 500/timeout (retryable error propagation). | AC-1, AC-2, AC-3, Edge cases |
| **Contract** | Consumer contract test verifies `claims-management` expectations for `GET /v1/policies/{id}` response schema (`PolicyView` containing `cover: List<CoverLine>`) against `policy-admin` contract. | `GET /v1/policies/{id}` |
| **End-to-End** | End-to-end claim intake and review flow verifies that a reported claim correctly transitions to `in_review` and computes excess and limits via HTTP cover check. | AC-1, AC-2, AC-3 |

## Rollout and rollback
1. **Configuration Toggle**: Introduce a configuration property in `claims-management` (`claims.cover-check.use-rest-client`, default `true`). When set to `false`, it falls back to direct database reads during initial deployment validation.
2. **Deployment Sequence**:
   - Deploy `claims-management` with REST client enabled.
   - Run synthetic cover checks in staging and production to verify response latency (p95 ≤ 100 ms) and error rates (< 0.1%).
   - Monitor for 14 days under production claim load.
3. **Database Decommissioning**:
   - Once verified stable, revoke `claims-management` read permissions on `policy_cover` in the `policy-admin` PostgreSQL database instance.
   - Remove legacy SQL queries and fallback database driver configuration from `claims-management`.
4. **Rollback Plan**:
   - If p95 latency exceeds 100 ms or error rates exceed 0.1% prior to database credential revocation, toggle `claims.cover-check.use-rest-client=false` via configuration update without requiring full application rollback.

## Risks

| Risk | Likelihood | Mitigation |
| :--- | :--- | :--- |
| **Increased Cover Check Latency**: HTTP REST latency higher than local database query, impacting handler review times. | Medium | Target p95 ≤ 100 ms supported by `policy-admin`'s baseline ~80 ms p95 [[kb:claims-management/decisions/direct-database-access-for-cover-check]]. Configured HTTP connection pooling and strict timeouts mitigate thread starvation. |
| **`policy-admin` Availability Bottleneck**: Transient outages or network blips in `policy-admin` causing cover check failures. | Low | Classify HTTP `5xx` and timeouts as transient errors with standard backoff retries, avoiding claim state corruption. |
| **Field Mapping Discrepancies**: Subtle schema differences between raw SQL rows and `CoverLine` JSON serialization. | Low | Consumer contract tests and unit tests asserting exact integer pence values across perils (`fire`, `flood`, `theft`, `escape_of_water`, `accidental_damage`). |

## Verification checklist
- [ ] Scope matches the design: only `claims-management` is modified, and no unexpected applications are changed.
- [ ] The `GET /v1/policies/{id}` contract on `policy-admin` is marked unchanged and untouched.
- [ ] Direct database access to `policy_cover` is removed from `claims-management`.
- [ ] Architecture compliance: ADR-0003 is honoured and pence monetary representation is preserved across all cover models.
- [ ] Test strategy delivered at every level: unit, integration (WireMock), contract (CDC), and end-to-end.
- [ ] Rollout toggle verified and rollback mechanism operational prior to database permission revocation.
