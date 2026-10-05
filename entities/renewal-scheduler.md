---
type: Component
title: RenewalScheduler
description: com.tidewell.policy.renewal.RenewalScheduler is a Spring scheduled component that automates the identification and notification of upcoming policy renewals.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/entities/renewal-scheduler.md
tags:
- policy-admin
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/java/com/tidewell/policy/renewal/RenewalScheduler.java
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/resources/application.yml
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

<!-- anchor: src/main/java/com/tidewell/policy/renewal/RenewalScheduler.java:L1-L16 -->
<!-- anchor: src/main/resources/application.yml:L1-L17 -->
<!-- anchor: README.md:L1-L28 -->

# RenewalScheduler

`com.tidewell.policy.renewal.RenewalScheduler` is a Spring scheduled component that automates the identification and notification of upcoming policy renewals. It executes a nightly batch scan to locate active policies approaching their renewal date and emits the `policy.renewal.due` domain event.

For higher-level workflows, see [[concepts/renewal-process]] and [[concepts/policy-lifecycle]].

---

## Responsibilities

- **Scheduled Execution**: Runs on a nightly cron trigger (`0 0 2 * * *`) via `@Scheduled` to execute `publishDueRenewals()`.
- **Renewal Identification**: Scans the `policies` database table for records meeting active renewal criteria:
  ```sql
  SELECT id FROM policies WHERE renewal_date = current_date + :noticeDays AND status = 'active'
  ```
- **Event Dispatch**: Emits the `policy.renewal.due` Pub/Sub event for each identified policy to trigger downstream billing and customer renewal notices.

---

## Configuration

The scheduler relies on properties defined in `src/main/resources/application.yml`:

| Property | Type | Default / Configured Value | Description |
| :--- | :--- | :--- | :--- |
| `renewal.notice-days` | `int` | `21` | Days before the `renewal_date` that `policy.renewal.due` is published and notices are dispatched. |
| `pubsub.topics.renewal-due` | `String` | `policy.renewal.due` | Outbound Pub/Sub topic name for due renewal notifications. |

---

## Event Payload Contract

When `publishDueRenewals()` executes, it broadcasts a JSON payload to the `policy.renewal.due` topic containing:

- `policy_id`: Identifier of the policy reaching renewal.
- `customer_id`: Identifier of the policyholder.
- `renewal_date`: The policy's scheduled renewal date.
- `new_premium_pence`: Recalculated renewal premium represented as an integer in pence.

Downstream consumers of this event include:
- `notifications-hub`: Sends customer-facing renewal communications (via `http://notifications-hub/v1`).
- `billing-service`: Prepares renewal billing schedules.

For the full event manifest and schemas, see [[summaries/api-spec]].

---

## Dependencies

- **Database**: Queries the `policies` table owned by `policy-admin` (see [[entities/policy-model]] and [[decisions/adr-0003-services-own-their-data]]).
- **Spring Scheduling**: Uses Spring's `@Scheduled` annotation with cron expression `0 0 2 * * *`.
- **Pub/Sub Infrastructure**: Outbound event publisher targeting the `policy.renewal.due` topic.
- **Related Components**:
  - [[concepts/renewal-process]]: Core business rules governing renewal notice windows and execution.
  - [[entities/policy-model]]: Schema definitions for policy status and renewal date fields.
  - [[summaries/api-spec]]: Specifications for the `policy.renewal.due` event payload.
