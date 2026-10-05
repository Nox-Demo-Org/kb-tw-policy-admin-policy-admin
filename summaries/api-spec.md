---
type: Interface Reference
title: API and Integration Specification
description: This page outlines the external interfaces exposed and consumed by policy-admin, including REST HTTP endpoints, gRPC rating dependencies, outbound domain events, and inbound event subscriptions.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/summaries/api-spec.md
tags:
- policy-admin
- summaries
sources:
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD//Users/akash/Documents/Projects/Project-NoX/demo/tidewell/sources/uploads/policy-admin-openapi.yaml
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/java/com/tidewell/policy/api/PolicyController.java
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/java/com/tidewell/policy/events/PolicyEvents.java
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

<!-- anchor: /Users/akash/Documents/Projects/Project-NoX/demo/tidewell/sources/uploads/policy-admin-openapi.yaml:L1-L26 -->
<!-- anchor: src/main/java/com/tidewell/policy/api/PolicyController.java:L1-L26 -->
<!-- anchor: src/main/java/com/tidewell/policy/events/PolicyEvents.java:L1-L15 -->

# API and Integration Specification

This page outlines the external interfaces exposed and consumed by `policy-admin`, including REST HTTP endpoints, gRPC rating dependencies, outbound domain events, and inbound event subscriptions.

In accordance with [[decisions/adr-0003-services-own-their-data]], external consumers must interact with policy data strictly through these REST endpoints and Pub/Sub events rather than directly querying database tables.

---

## REST API Specification

Defined in OpenAPI 3.8.0 and implemented via [[entities/policy-controller]]:

### Base Path
All REST endpoints are rooted at `/v1/policies` on port `8080`.

### Endpoints

| Method | Path | Summary | Request Body | Response (Success) | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/v1/policies/{id}` | A policy with its cover lines | None | `200 OK` (`PolicyView`) | Retrieves policy details and associated cover lines by ID. |
| `POST` | `/v1/policies` | Issue a policy | `IssueRequest` | `201 Created` (`PolicyView`) | Issues a new policy post-purchase and triggers `policy.policy.issued`. See [[concepts/policy-lifecycle]]. |
| `POST` | `/v1/policies/{id}/changes` | Mid-term change | `ChangeRequest` | `200 OK` (`PolicyView`) | Submits a mid-term policy adjustment, re-priced via `RatingService.Price`. See [[concepts/mid-term-changes]]. |

---

### Request & Response Schemas

#### Data Transfer Objects (DTOs)

- **`PolicyView`**
  ```json
  {
    "id": "string",
    "customerId": "string",
    "product": "home | motor",
    "status": "string",
    "startDate": "YYYY-MM-DD",
    "renewalDate": "YYYY-MM-DD",
    "annualPremiumPence": 0,
    "cover": [
      {
        "peril": "string",
        "limitPence": 0,
        "excessPence": 0
      }
    ]
  }
  ```
  *(Note: All monetary fields such as `annualPremiumPence`, `limitPence`, and `excessPence` are represented as 64-bit integer pence).*

- **`CoverLine`**
  - `peril` (`String`): The peril or risk covered (e.g., flood, accidental damage).
  - `limitPence` (`long`): Maximum payout limit in pence.
  - `excessPence` (`long`): Policy excess amount in pence.

- **`IssueRequest`**
  - `customerId` (`String`): Identifier for the customer.
  - `product` (`String`): Product type (`home` or `motor`).
  - `quoteId` (`String`): Reference quote identifier.

- **`ChangeRequest`**
  - `type` (`String`): Adjustment type.
  - `effectiveDate` (`String`): Effective date of the adjustment.
  - `details` (`Map<String, String>`): Key-value map of updated policy attributes.

---

## gRPC Dependencies

`policy-admin` relies on gRPC communication to evaluate policy pricing and adjustments managed via [[entities/rating-client]].

- **Service Target**: `dns:///rating-service:9090` (`clients.rating`)
- **RPC Method**: `RatingService.Price`
- **Execution Deadline**: 300 ms
- **Used By**: Mid-term adjustment endpoint (`POST /v1/policies/{id}/changes`) to re-price adjustments in real time.

---

## External HTTP Clients

- **Notifications Hub**: Configured under `clients.notifications` pointing to `http://notifications-hub/v1`.

---

## Pub/Sub Events

Domain event schemas are defined in `PolicyEvents.java` and wired through `application.yml`.

### Outbound Events (Published Topics)

Managed by the application across policy lifecycle triggers:

| Event Topic | Java Record Schema | Key Fields | Description |
| :--- | :--- | :--- | :--- |
| `policy.policy.issued` | `PolicyEvents.Issued` | `policyId`, `customerId`, `product`, `startDate`, `annualPremiumPence`, `paymentFrequency` | Published when a policy is purchased or issued. Used by billing and identity. |
| `policy.renewal.due` | `PolicyEvents.RenewalDue` | `policyId`, `customerId`, `renewalDate`, `newPremiumPence` | Published nightly by [[entities/renewal-scheduler]] 21 days before renewal (`renewal.notice-days: 21`). See [[concepts/renewal-process]]. |
| `policy.policy.cancelled` | `PolicyEvents.Cancelled` | `policyId`, `cancelledOn`, `reason` | Published when a policy is cancelled. |

### Inbound Events (Subscriptions)

Consumed by [[entities/event-subscribers]] to update internal state:

| Subscription Config | Topic Name | Purpose |
| :--- | :--- | :--- |
| `pubsub.subscriptions.profile-updated` | `customer.profile.updated` | Updates customer profile or correspondence address on active policies. |
| `pubsub.subscriptions.payment-missed` | `billing.payment.missed` | Tracks missed payments to initiate policy cancellation workflows. |
