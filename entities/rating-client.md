---
type: Component
title: RatingClient
description: RatingClient (src/main/java/com/tidewell/policy/rating/RatingClient.java) is the gRPC client component responsible for integrating policy-admin with rating-service to perform pricing evaluations.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/entities/rating-client.md
tags:
- policy-admin
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/java/com/tidewell/policy/rating/RatingClient.java
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/resources/application.yml
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

<!-- anchor: src/main/java/com/tidewell/policy/rating/RatingClient.java:L1-L4 -->
<!-- anchor: src/main/resources/application.yml:L1-L17 -->
<!-- anchor: README.md:L1-L28 -->

# RatingClient

`RatingClient` (`src/main/java/com/tidewell/policy/rating/RatingClient.java`) is the gRPC client component responsible for integrating `policy-admin` with `rating-service` to perform pricing evaluations.

## Responsibilities

- **gRPC Pricing Invocations**: Communicates with `rating-service` by executing the `RatingService.Price` RPC method.
- **Deadline Management**: Applies a strict 300 ms deadline on pricing requests to ensure low-latency response times during repricing operations.
- **Mid-Term Adjustment Repricing**: Supports dynamic premium recalculation required when processing policy adjustments submitted to `POST /v1/policies/{id}/changes` (see [[concepts/mid-term-changes]]).

## Configuration

The client connection is configured in `src/main/resources/application.yml`:

```yaml
clients:
  rating: dns:///rating-service:9090 # gRPC RatingService.Price, 300 ms deadline
```

- **Target Address**: `dns:///rating-service:9090`
- **Call Method**: `RatingService.Price`
- **Call Deadline**: 300 ms

## Dependencies

- **Downstream gRPC Service**:
  - `rating-service` (`RatingService.Price`) – external pricing and rating engine.
- **Used By**:
  - [[entities/policy-controller]]: Invoked during policy adjustment flows when recalculating premiums on `POST /v1/policies/{id}/changes`.
  - [[concepts/mid-term-changes]]: Underpins the re-rating step in the policy amendment workflow.
  - [[concepts/policy-lifecycle]]: Part of the broader policy pricing and modification lifecycle.
  - [[summaries/api-spec]]: Documented as an external consumed gRPC contract.
