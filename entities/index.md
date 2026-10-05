# Components and data models

One page per significant component and core data model.

## Pages

- [Cover Model](/entities/cover-model.md) — The cover model defines the perils, maximum coverage limits, deductibles (excess), and validity date intervals associated with policies managed by policy-admin.
- [CoverService](/entities/cover-service.md) — CoverService (com.tidewell.policy.cover.CoverService) is the domain service responsible for querying and aggregating policy coverage lines, perils, limits, and excesses across policies owned by a given customer.
- [Event Subscribers](/entities/event-subscribers.md) — com.tidewell.policy.events.Subscribers represents the event ingestion layer in policy-admin.
- [PolicyController](/entities/policy-controller.md) — com.tidewell.policy.api.PolicyController is the primary Spring REST controller in policy-admin that exposes HTTP endpoints for retrieving policy details, issuing new policies, and processing mid-term adjustments.
- [Policy Model](/entities/policy-model.md) — The policy model represents the core domain entity and underlying database schema for insurance policies within policy-admin.
- [RatingClient](/entities/rating-client.md) — RatingClient (src/main/java/com/tidewell/policy/rating/RatingClient.java) is the gRPC client component responsible for integrating policy-admin with rating-service to perform pricing evaluations.
- [RenewalScheduler](/entities/renewal-scheduler.md) — com.tidewell.policy.renewal.RenewalScheduler is a Spring scheduled component that automates the identification and notification of upcoming policy renewals.
