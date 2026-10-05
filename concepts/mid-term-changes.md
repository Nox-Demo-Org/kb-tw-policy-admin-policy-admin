---
type: Concept
title: Mid-Term Changes (MTC)
description: Mid-term changes (also referred to as mid-term adjustments or amendments) represent modifications made to an active insurance policy prior to its renewal date.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/concepts/mid-term-changes.md
tags:
- policy-admin
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

# Mid-Term Changes (MTC)

Mid-term changes (also referred to as mid-term adjustments or amendments) represent modifications made to an active insurance policy prior to its renewal date. These changes allow policyholders to alter policy parameters—such as updating an address, changing a vehicle, or modifying coverage options.

Mid-term adjustments form a key operational phase within the [[concepts/policy-lifecycle]].

---

## Overview and Endpoints

Mid-term changes are executed via the REST endpoint exposed by [[entities/policy-controller]]:

- **Endpoint**: `POST /v1/policies/{id}/changes`
- **Controller**: `com.tidewell.policy.api.PolicyController`
- **Request Payload**: `PolicyController.ChangeRequest`
  - `type` (`String`): The category or type of change being performed (e.g., address, car, cover).
  - `effectiveDate` (`String`): The date from which the amendment takes effect.
  - `details` (`Map<String, String>`): Key-value pairs representing specific adjustment attributes.
- **Response**: `PolicyController.PolicyView` containing the updated policy header and its associated `CoverLine` items.

> **Note**: Mid-term changes submitted via `POST /v1/policies/{id}/changes` are not yet offered directly in `customer-portal`.

---

## Re-Rating Integration

When an adjustment modifies the risk profile or coverage structure of a policy, the policy must be repriced:

1. **Rating Integration**: `policy-admin` invokes `rating-service` using the gRPC client [[entities/rating-client]].
2. **RPC Call**: `RatingService.Price`
3. **Execution Deadline**: Configured with a 300 ms deadline.
4. **Outcome**: The revised pricing calculates the updated `annualPremiumPence` and any adjusted cover limits or excesses on the policy.

All monetary values resulting from the re-rating process (such as `annualPremiumPence`, `limitPence`, and `excessPence`) follow the standard 64-bit integer format in pence.

---

## Data Updates and Boundaries

Following successful re-rating, the updated policy attributes and any modified cover lines are persisted across the primary database tables:
- [[entities/policy-model]] (`policies` table)
- [[entities/cover-model]] (`policy_cover` table)

In accordance with [[decisions/adr-0003-services-own-their-data]], external systems cannot directly modify or query the `policies` and `policy_cover` tables during a change. All modifications must flow through the `POST /v1/policies/{id}/changes` REST interface or supported domain event triggers (such as `customer.profile.updated` processed by [[entities/event-subscribers]]).

---

## Related Concepts & Integrations

- [[summaries/api-spec]]: Specification for all REST and gRPC interfaces including `/v1/policies/{id}/changes`.
- [[entities/rating-client]]: Details on the gRPC connection to `rating-service`.
- [[entities/policy-controller]]: REST controller implementation.
- [[concepts/policy-lifecycle]]: End-to-end lifecycle flow from issuance to renewal or cancellation.
- [[decisions/adr-0003-services-own-their-data]]: Architecture decision governing service data boundaries.
