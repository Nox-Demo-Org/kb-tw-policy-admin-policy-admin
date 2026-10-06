---
mission: NOX-6
title: 'Policy cover checks for insurance claims'
role: developer
status: ai_drafted
version: 1
author: NoX
ai_drafted: true
---

# Build spec: Policy cover checks for insurance claims

## Files and services touched

### `claims-management`
- `src/main/resources/application.yml`: Add HTTP client connection configuration (`policy-admin.url`, connect timeout 500 ms, read timeout 1500 ms) and feature flag `claims.cover-check.use-rest-client`.
- `src/main/java/com/tidewell/claims/cover/dto/PolicyView.java`: Add client DTO record matching `policy-admin` payload schema (`id`, `customerId`, `product`, `status`, `startDate`, `renewalDate`, `annualPremiumPence`, `cover`).
- `src/main/java/com/tidewell/claims/cover/dto/CoverLine.java`: Add client DTO record matching `policy-admin` cover item (`peril`, `limitPence`, `excessPence`).
- `src/main/java/com/tidewell/claims/cover/CoverCheckRepository.java`: Refactor repository to execute HTTP `GET /v1/policies/{id}` calls against `policy-admin`, mapping response cover lines to peril coverage and excess evaluations, gated by `claims.cover-check.use-rest-client` [[kb:claims-management/decisions/direct-database-access-for-cover-check]].
- `src/test/java/com/tidewell/claims/cover/CoverCheckRepositoryTest.java`: Add unit tests for cover evaluation logic, peril limit extraction, excess mapping, and error translation.
- `src/test/java/com/tidewell/claims/cover/CoverCheckRepositoryIntegrationTest.java`: Add WireMock integration tests for HTTP communication with `policy-admin`.
- `src/test/java/com/tidewell/claims/contract/PolicyAdminContractTest.java`: Add consumer-driven contract test for `GET /v1/policies/{id}` against `policy-admin` OpenAPI specification.

### `policy-admin`
- Untouched. Consumes existing `GET /v1/policies/{id}` exposed by `com.tidewell.policy.api.PolicyController` [[kb:policy-admin/entities/policy-controller]].

---

## What to reuse

- **REST Endpoint**: `GET /v1/policies/{id}` on `policy-admin` [[kb:policy-admin/entities/policy-controller]].
- **DTO Model Schema**: `PolicyView` and `CoverLine` contract models from `policy-admin` [[kb:policy-admin/entities/cover-model]]:
  - `CoverLine(String peril, long limitPence, long excessPence)`
  - `PolicyView(String id, String customerId, String product, String status, String startDate, String renewalDate, long annualPremiumPence, List<CoverLine> cover)`
- **Monetary Representation**: 64-bit integer (`long`) pence for all financial fields (`limitPence`, `excessPence`, `annualPremiumPence`) [[kb:policy-admin/entities/cover-model]].
- **Lifecycle & Assignment**: Claim assessment orchestration in `com.tidewell.claims.assignment.HandlerAssignmentService` and claim status lifecycle [[kb:claims-management/concepts/claim-lifecycle]].

---

## Tasks

1. **Add HTTP client configuration and rollout toggle**: In `claims-management` `src/main/resources/application.yml`, define `policy-admin.base-url`, connect timeout (`500ms`), read timeout (`1500ms`), connection pool limits, and `claims.cover-check.use-rest-client: true`.
2. **Define REST client DTO records**: In `src/main/java/com/tidewell/claims/cover/dto/`, create `CoverLine` (`peril`, `limitPence`, `excessPence`) and `PolicyView` records using integer pence fields adhering to the platform standard [[kb:policy-admin/entities/cover-model]].
3. **Refactor `CoverCheckRepository`**:
   - Implement HTTP client call `GET /v1/policies/{id}` to fetch `PolicyView`.
   - Implement peril matching against `PolicyView.cover()` list to extract `limitPence` and `excessPence`.
   - Wrap legacy direct SQL access to `policy_cover` behind `if (!useRestClient)` fallback switch.
4. **Implement error mapping and edge case handling in `CoverCheckRepository`**:
   - Map HTTP `404 Not Found` to `PolicyNotFoundException`.
   - Map missing peril in `cover` list to an uncovered peril result (allowing triage routing to proceed unchanged).
   - Map HTTP `5xx`, connection failures, and timeouts to retryable domain exceptions without altering claim state.
5. **Implement Unit Tests**: Add unit tests in `CoverCheckRepositoryTest.java` verifying DTO parsing, peril limit resolution, excess calculation in pence, and exception mapping.
6. **Implement WireMock Integration Tests**: Add integration tests in `CoverCheckRepositoryIntegrationTest.java` validating HTTP request formation, header propagation, timeout enforcement, and response parsing.
7. **Implement Consumer Contract Test**: Add contract test in `PolicyAdminContractTest.java` verifying `claims-management` consumer expectations against `policy-admin` `GET /v1/policies/{id}` OpenAPI contract.
8. **Decommission legacy SQL reads (Post-verification task)**: Once deployment is verified stable for 14 days, remove fallback database query methods and delete direct `policy_cover` datasource beans.

---

## Test plan

### Unit Tests
- **Test File**: `src/test/java/com/tidewell/claims/cover/CoverCheckRepositoryTest.java`
  - `shouldReturnCoveredPerilWithExactLimitAndExcess()`: Verifies AC-1 and AC-2 by parsing `CoverLine` records and asserting exact `limitPence` and `excessPence`.
  - `shouldFlagPerilAsNotCoveredWhenMissingInCoverList()`: Verifies AC-3 when a requested peril (e.g., `flood`) is not present in the policy's `cover` list.
  - `shouldTranslate404ToPolicyNotFoundException()`: Verifies missing policy edge case.
  - `shouldPropagateRetryableExceptionOn5xxOrTimeout()`: Verifies transient error handling on upstream failures.

### Integration Tests
- **Test File**: `src/test/java/com/tidewell/claims/cover/CoverCheckRepositoryIntegrationTest.java`
  - `shouldFetchPolicyAndMapCoverDetailsOverHttp()`: Stubs WireMock `GET /v1/policies/{id}` with 200 OK and JSON payload; verifies HTTP execution, connect/read timeouts, and returned domain models (AC-1, AC-2).
  - `shouldHandleUncoveredPerilOverHttp()`: Stubs WireMock `GET /v1/policies/{id}` without the claim peril; verifies correct negative cover check outcome (AC-3).
  - `shouldHandlePolicyNotFoundOverHttp()`: Stubs WireMock 404 response; verifies `PolicyNotFoundException`.
  - `shouldHandleHttpTimeoutSafely()`: Stubs WireMock with delayed response (> 1500 ms); verifies timeout exception is raised without mutating claim state.

### Contract Tests
- **Test File**: `src/test/java/com/tidewell/claims/contract/PolicyAdminContractTest.java`
  - `validatePolicyAdminGetPolicyContract()`: Verifies `GET /v1/policies/{id}` response structure matches expected `PolicyView` and `CoverLine` schema (`limitPence`, `excessPence`, `peril`).

### Commands to Run
```bash
# Run unit and contract tests
./mvnw clean test -Dtest="CoverCheckRepositoryTest,PolicyAdminContractTest"

# Run integration tests with WireMock
./mvnw test -Dtest="CoverCheckRepositoryIntegrationTest"

# Run full test suite
./mvnw clean verify
```

---

## Rollout

### Configuration & Flags
- Set `claims.cover-check.use-rest-client=true` in `application.yml` (default enabled).
- Configure `policy-admin` client timeouts: connect timeout `500ms`, read timeout `1500ms`.

### Deployment Steps
1. Deploy `claims-management` with `claims.cover-check.use-rest-client=true`.
2. Monitor response times (target p95 ≤ 100 ms) and error rates (< 0.1%) in Datadog/service logs for 14 days.
3. If latency exceeds 100 ms or error rate exceeds 0.1%, toggle `claims.cover-check.use-rest-client=false` via runtime configuration update to fall back to direct DB read without redeployment.
4. After 14 days of stable execution, execute database permission revocation: revoke `SELECT` on `policy_cover` for `claims-management` database role in PostgreSQL.
5. Remove fallback SQL code from `CoverCheckRepository` in a subsequent cleanup release.

---

## Verification checklist

- [ ] Task 1 done: HTTP client configuration and toggle `claims.cover-check.use-rest-client` added.
- [ ] Task 2 done: `PolicyView` and `CoverLine` DTO records defined with `long` pence fields.
- [ ] Task 3 done: `CoverCheckRepository` refactored to consume `GET /v1/policies/{id}`.
- [ ] Task 4 done: Error handling and edge case translation (404, missing peril, 5xx/timeouts) implemented.
- [ ] Task 5 done: Unit test `CoverCheckRepositoryTest` written and passing (`./mvnw test -Dtest=CoverCheckRepositoryTest`).
- [ ] Task 6 done: Integration test `CoverCheckRepositoryIntegrationTest` written and passing (`./mvnw test -Dtest=CoverCheckRepositoryIntegrationTest`).
- [ ] Task 7 done: Contract test `PolicyAdminContractTest` written and passing (`./mvnw test -Dtest=PolicyAdminContractTest`).
- [ ] Reused components and existing models (`PolicyView`, `CoverLine`, integer pence standard) not duplicated or modified in `policy-admin`.
- [ ] Contracts on `policy-admin` (`GET /v1/policies/{id}`) marked unchanged and untouched.
- [ ] Rollout toggle verified in staging and operational prior to production database permission revocation.
