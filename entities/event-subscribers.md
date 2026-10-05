---
type: Component
title: Event Subscribers
description: com.tidewell.policy.events.Subscribers represents the event ingestion layer in policy-admin.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/entities/event-subscribers.md
tags:
- policy-admin
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/java/com/tidewell/policy/events/Subscribers.java
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/resources/application.yml
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

<!-- anchor: src/main/java/com/tidewell/policy/events/Subscribers.java:L1-L7 -->
<!-- anchor: src/main/resources/application.yml:L1-L17 -->
<!-- anchor: README.md:L1-L28 -->

# Event Subscribers

`com.tidewell.policy.events.Subscribers` represents the event ingestion layer in `policy-admin`. It listens to inbound asynchronous events published by external services over GCP Pub/Sub to trigger updates to policy records and drive policy lifecycle workflows.

## Responsibilities

The `Subscribers` component handles two primary inbound event subscriptions:

- **`customer.profile.updated`**: Refreshes the correspondence address on the associated [[entities/policy-model|policy]].
- **`billing.payment.missed`**: Tracks payment failure events. After a customer records a second missed payment, this subscriber initiates the cancellation letter workflow as part of the [[concepts/policy-lifecycle]].

## Configuration

Inbound Pub/Sub subscriptions are configured in `application.yml` under the `pubsub.subscriptions` namespace:

```yaml
pubsub:
  subscriptions:
    profile-updated: customer.profile.updated
    payment-missed: billing.payment.missed
```

For the complete list of inbound and outbound event contracts, refer to the [[summaries/api-spec]].

## Dependencies

- **Pub/Sub Subscriptions**:
  - `customer.profile.updated`: Consumed from upstream customer profile services.
  - `billing.payment.missed`: Consumed from `billing-service`.
- **Domain & Persistence**:
  - Updates policy state and address data stored in the `policies` table ([[entities/policy-model]]), upholding service encapsulation defined in [[decisions/adr-0003-services-own-their-data]].
- **Downstream Services**:
  - Client endpoint configured at `clients.notifications` (`http://notifications-hub/v1`) for sending notices (such as cancellation letter notifications).
