---
type: Concept
title: Policy Renewal Process
description: The renewal process in policy-admin identifies active insurance policies approaching the end of their term and dispatches notification events to downstream billing and communication systems.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/concepts/renewal-process.md
tags:
- policy-admin
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

# Policy Renewal Process

The renewal process in `policy-admin` identifies active insurance policies approaching the end of their term and dispatches notification events to downstream billing and communication systems. The renewal cycle runs on an automated daily schedule managed by [[entities/renewal-scheduler]].

## Renewal Identification Workflow

Policy renewal identification executes nightly via a scheduled cron job in [[entities/renewal-scheduler]]:

1. **Schedule**: The scheduler triggers every night at 02:00 (`cron = "0 0 2 * * *"`) via `RenewalScheduler.publishDueRenewals()`.
2. **Policy Selection**: The job queries the [[entities/policy-model|policies]] database table for active policies whose `renewal_date` is exactly `noticeDays` in the future:
   ```sql
   SELECT id FROM policies 
   WHERE renewal_date = current_date + :noticeDays 
     AND status = 'active'
   ```
3. **Event Dispatch**: For each matching policy record, the service publishes a `policy.renewal.due` event over Google Cloud Pub/Sub.

For overall lifecycle transitions including initial issuance and cancellation, see [[concepts/policy-lifecycle]].

## Configuration & Timing

The notice lead time is configured in `application.yml`:

```yaml
renewal:
  notice-days: 21
```

* **Property**: `renewal.notice-days`
* **Default Value**: `21` (days)
* **Background & Regulatory Context**: The 21-day window was originally established in 2021 to accommodate a physical printing and mailing contract (which concluded in 2024). While the Financial Conduct Authority (FCA) requires a minimum notice period of 21 days, internal analysis (FY26 Renewals Review) demonstrated that providing notice 30 days prior increases renewal conversion from 73% to 82%.

## Pub/Sub Event Dispatch

When a policy enters the renewal window, the scheduler emits the event configured under `pubsub.topics.renewal-due` (`policy.renewal.due`).

### Event Payload Schema

| Field Name | Type / Format | Description |
| :--- | :--- | :--- |
| `policy_id` | String / UUID | Unique identifier of the policy being renewed |
| `customer_id` | String / UUID | Policyholder customer identifier |
| `renewal_date` | Date / String | The date on which the renewed term begins |
| `new_premium_pence` | Integer (64-bit) | Renewal premium quoted in integer pence |

Full endpoint and messaging contracts are documented in [[summaries/api-spec]].

## Downstream Consumers

Events published to `policy.renewal.due` are consumed by multiple platform services:

* **`billing-service`**: Prepares payment schedules and directs debit collections for the upcoming policy term.
* **`notifications-hub`** (configured via `clients.notifications: http://notifications-hub/v1`): Formats and delivers the statutory renewal notice to the customer across digital or print channels.

Direct database access to the `policies` table by these downstream services is prohibited under [[decisions/adr-0003-services-own-their-data]]; all renewal data must be consumed through Pub/Sub events or the REST API.
