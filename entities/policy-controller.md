---
type: Component
title: PolicyController
description: com.tidewell.policy.api.PolicyController is the primary Spring REST controller in policy-admin that exposes HTTP endpoints for retrieving policy details, issuing new policies, and processing mid-term adjustments.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/entities/policy-controller.md
tags:
- policy-admin
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/java/com/tidewell/policy/api/PolicyController.java
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD//Users/akash/Documents/Projects/Project-NoX/demo/tidewell/sources/uploads/policy-admin-openapi.yaml
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

<!-- anchor: src/main/java/com/tidewell/policy/api/PolicyController.java:L1-L26 -->
<!-- anchor: /Users/akash/Documents/Projects/Project-NoX/demo/tidewell/sources/uploads/policy-admin-openapi.yaml:L1-L26 -->
<!-- anchor: README.md:L1-L28 -->

# PolicyController

`com.tidewell.policy.api.PolicyController` is the primary Spring REST controller in `policy-admin` that exposes HTTP endpoints for retrieving policy details, issuing new policies, and processing mid-term adjustments.

In accordance with [[decisions/adr-0003-services-own-their-data]], external services must access policy data through this controller rather than querying the underlying [[entities/policy-model]] (`policies`) and [[entities/cover-model]] (`policy_cover`) database tables directly.

---

## Endpoints

All endpoints are mapped under the base path `/v1/policies`.

### 1. Get Policy Details
* **Route**: `GET /v1/policies/{id}`
* **Method**: `get(@PathVariable String id)`
* **Description**: Returns the specified policy along with its associated cover lines.
* **Consumers**: `customer-portal`, `claims-intake`, `billing-service`.
* **Response**: `PolicyView` (HTTP 200)

### 2. Issue Policy
* **Route**: `POST /v1/policies`
* **Method**: `issue(@RequestBody IssueRequest req)`
* **Description**: Issues a new policy post-purchase as part of the [[concepts/policy-lifecycle]]. Persists the record and emits the `policy.policy.issued` domain event.
* **Response**: `PolicyView` (HTTP 201)

### 3. Mid-Term Adjustment
* **Route**: `POST /v1/policies/{id}/changes`
* **Method**: `change(@PathVariable String id, @RequestBody ChangeRequest req)`
* **Description**: Submits a mid-term change (e.g., changes to address, vehicle, or cover) handled via [[concepts/mid-term-changes]]. The adjustment is re-priced by calling `RatingService.Price` via the [[entities/rating-client]].
* **Response**: `PolicyView` (HTTP 200)

---

## Data Transfer Objects (DTOs)

The controller defines the following Java record structures for request and response serialization:

### `PolicyView`
Represents the complete policy record and associated cover lines returned by the endpoints. Monetary amounts are represented as 64-bit integer values in pence.

| Field | Type | Description |
|---|---|---|
| `id` | `String` | Unique policy identifier |
| `customerId` | `String` | Identifier of the customer owning the policy |
| `product` | `String` | Product type (e.g., `home`, `motor`) |
| `status` | `String` | Current status of the policy |
| `startDate` | `String` | Policy start date |
| `renewalDate` | `String` | Scheduled policy renewal date |
| `annualPremiumPence` | `long` | Annual premium represented in pence |
| `cover` | `java.util.List<CoverLine>` | List of cover lines associated with the policy |

### `CoverLine`
Represents an individual cover line attached to a policy.

| Field | Type | Description |
|---|---|---|
| `peril` | `String` | Peril or coverage name |
| `limitPence` | `long` | Coverage limit in pence |
| `excessPence` | `long` | Excess / deductible in pence |

### `IssueRequest`
Payload required to issue a new policy.

| Field | Type | Description |
|---|---|---|
| `customerId` | `String` | Identifier of the purchasing customer |
| `product` | `String` | Product type (`home` or `motor`) |
| `quoteId` | `String` | Source quote identifier |

### `ChangeRequest`
Payload submitted for a mid-term policy adjustment.

| Field | Type | Description |
|---|---|---|
| `type` | `String` | Type of mid-term modification |
| `effectiveDate` | `String` | Date the change takes effect |
| `details` | `java.util.Map<String, String>` | Key-value pairs containing change attributes |

---

## Responsibilities

- **REST API Exposition**: Exposes endpoints according to the OpenAPI specification (`/v1/policies`, `/v1/policies/{id}`, `/v1/policies/{id}/changes`). Refer to [[summaries/api-spec]].
- **Policy Ingestion and Lifecycle Dispatch**: Accepts policy issuance requests (`POST /v1/policies`) and orchestrates outbound event emission (`policy.policy.issued`) to downstream consumers such as billing and customer identity.
- **Mid-Term Modification Entry Point**: Serves as the REST entry point for policy amendments, delegating re-pricing calculations before applying changes.

---

## Dependencies

- **[[entities/rating-client]] (`RatingService.Price`)**: Invoked during mid-term changes (`POST /v1/policies/{id}/changes`) to recalculate premiums with real-time rating data.
- **[[entities/cover-service]]**: Manages and resolves the policy cover lines returned in `PolicyView`.
- **Database Tables**:
  - [[entities/policy-model]] (`policies` table)
  - [[entities/cover-model]] (`policy_cover` table)
- **Pub/Sub Domain Events**: Triggers publication of `policy.policy.issued` upon policy creation.
