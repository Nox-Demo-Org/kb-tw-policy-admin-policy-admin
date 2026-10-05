---
type: Concept
title: Policy Lifecycle
description: The policy-admin service manages the end-to-end lifecycle of insurance policies (both home and motor) for Tidewell Mutual.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/concepts/policy-lifecycle.md
tags:
- policy-admin
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

# Policy Lifecycle

The `policy-admin` service manages the end-to-end lifecycle of insurance policies (both home and motor) for Tidewell Mutual. The core lifecycle transitions through stages of policy creation, active maintenance and adjustments, annual renewals, and eventual cancellation.

---

## Lifecycle Stages and Workflows

```
  [ Purchase / Quotation ]
             │
             ▼
      POST /v1/policies
             │
             ▼
     ┌───────────────┐
     │    Issued     │ ──▶ emits policy.policy.issued
     └───────┬───────┘
             │
             │◀──────────────────────────────────────┐
             ▼                                       │
     ┌───────────────┐                               │
     │    Active     │ ──▶ POST /v1/policies/{id}/changes (Mid-term changes)
     └───────┬───────┘
             │
             ├──▶ Nightly Renewal Window (renewal.notice-days)
             │      │
             │      ▼
             │   ┌───────────────┐
             │   │  Renewal Due  │ ──▶ emits policy.renewal.due
             │   └───────────────┘
             │
             ▼
     [ Inbound Triggers: billing.payment.missed / Manual ]
             │
             ▼
     ┌───────────────┐
     │   Cancelled   │ ──▶ emits policy.policy.cancelled
     └───────────────┘
```

---

## 1. Policy Issuance

A policy lifecycle begins when a policy is purchased. 

- **Trigger**: An API consumer submits an issuance request via `POST /v1/policies` containing `customerId`, `product`, and `quoteId` (see [[entities/policy-controller]]).
- **Persistence**: Persisted to the `policies` table and coverage details to `policy_cover` (see [[entities/policy-model]] and [[entities/cover-model]]).
- **Outbound Event**: Emits `policy.policy.issued` (`PolicyEvents.ISSUED`).
  - **Payload**: `policyId`, `customerId`, `product`, `startDate`, `annualPremiumPence`, `paymentFrequency`.
  - **Consumers**: `billing-service` (to set up payment schedules) and `customer-identity` (to record customer tenure).

---

## 2. Active Policy & Mid-Term Changes (MTA)

While active, a policy's details can be queried via `GET /v1/policies/{id}`. Active policies may undergo mid-term adjustments (such as change of address, vehicle changes, or coverage adjustments):

- **Trigger**: Submitted via `POST /v1/policies/{id}/changes` with a `ChangeRequest` (`type`, `effectiveDate`, `details`).
- **Repricing**: Adjustments are re-priced in real time by invoking the gRPC endpoint `RatingService.Price` on `rating-service` via [[entities/rating-client]].
- For comprehensive details, see [[concepts/mid-term-changes]].

---

## 3. Annual Renewal

Policies approaching their renewal date undergo the renewal workflow:

- **Trigger**: The [[entities/renewal-scheduler]] evaluates active policies against the configured notice period (`renewal.notice-days`, default `21` days in `application.yml`).
- **Outbound Event**: Emits `policy.renewal.due` (`PolicyEvents.RENEWAL_DUE`).
  - **Payload**: `policyId`, `customerId`, `renewalDate`, `newPremiumPence`.
  - **Consumers**: `notifications-hub` (to dispatch customer renewal notices) and `billing-service`.
- Detailed renewal timing, operational metrics, and proposed adjustments are documented in [[concepts/renewal-process]].

---

## 4. Policy Cancellation

A policy moves to a terminal cancelled state due to external business triggers or missed payments:

- **Inbound Triggers**: Handled by [[entities/event-subscribers]], including the inbound `billing.payment.missed` event.
- **Outbound Event**: Emits `policy.policy.cancelled` (`PolicyEvents.CANCELLED`).
  - **Payload**: `policyId`, `cancelledOn`, `reason`.
  - **Consumers**: `billing-service` and `claims-management`.

---

## Architectural & Data Boundaries

In accordance with [[decisions/adr-0003-services-own-their-data]], all external systems interacting with the policy lifecycle must do so strictly via the public REST APIs (`/v1/policies/*`) or GCP Pub/Sub domain events defined in `PolicyEvents`. Direct database queries against `policies` and `policy_cover` tables are restricted.

All monetary values across the lifecycle (annual premiums, limits, excesses) are represented in standard integer pence (`long`).
