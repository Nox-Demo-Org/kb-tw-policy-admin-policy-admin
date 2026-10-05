---
type: Component
title: Cover Model
description: The cover model defines the perils, maximum coverage limits, deductibles (excess), and validity date intervals associated with policies managed by policy-admin.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/entities/cover-model.md
tags:
- policy-admin
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/resources/db/schema.sql
- resource: https://github.com/Nox-Demo-Org/policy-admin/blob/HEAD/src/main/java/com/tidewell/policy/api/PolicyController.java
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

<!-- anchor: src/main/resources/db/schema.sql:L1-L18 -->
<!-- anchor: src/main/java/com/tidewell/policy/api/PolicyController.java:L1-L26 -->

# Cover Model

The cover model defines the perils, maximum coverage limits, deductibles (excess), and validity date intervals associated with policies managed by `policy-admin`. 

Per [[decisions/adr-0003-services-own-their-data]], `policy-admin` is the sole data owner of the `policy_cover` entity. All external systems access cover details through `policy-admin` APIs rather than direct database queries.

## Database Schema

Cover records are persisted in PostgreSQL in the `policy_cover` table defined in `src/main/resources/db/schema.sql`:

```sql
CREATE TABLE policy_cover (
  policy_id     UUID REFERENCES policies(id),
  peril         TEXT NOT NULL,             -- fire, flood, theft, escape_of_water, accidental_damage, ...
  limit_pence   BIGINT NOT NULL,
  excess_pence  BIGINT NOT NULL,
  active_from   DATE NOT NULL,
  active_to     DATE
);
```

### Column Definitions

| Column | Type | Nullable | Description |
|---|---|---|---|
| `policy_id` | `UUID` | No | Foreign key referencing `policies(id)` in [[entities/policy-model]]. |
| `peril` | `TEXT` | No | Identifier for the insured peril (e.g., `fire`, `flood`, `theft`, `escape_of_water`, `accidental_damage`). |
| `limit_pence` | `BIGINT` | No | Maximum coverage payout limit represented as an integer in pence. |
| `excess_pence` | `BIGINT` | No | Policyholder deductible/excess amount represented as an integer in pence. |
| `active_from` | `DATE` | No | Start date when this peril coverage line takes effect. |
| `active_to` | `DATE` | Yes | End date for this cover line; null if the line is currently active with no set expiry. |

### Monetary Values
All financial values (`limit_pence`, `excess_pence`) adhere to the platform-wide monetary standard and are stored as 64-bit integer values in pence.

---

## Domain and API Representation

Cover lines are projected to REST clients via the [[entities/policy-controller]] DTO records:

```java
public record CoverLine(String peril, long limitPence, long excessPence) {}
```

A list of `CoverLine` objects is embedded inside `PolicyView` responses:

```java
public record PolicyView(
    String id, 
    String customerId, 
    String product, 
    String status, 
    String startDate,
    String renewalDate, 
    long annualPremiumPence, 
    java.util.List<CoverLine> cover
) {}
```

Cover lines are returned when retrieving policy details via `GET /v1/policies/{id}`, issuing a policy via `POST /v1/policies`, or applying mid-term adjustments via `POST /v1/policies/{id}/changes`.

---

## Responsibilities

- **Peril Definition**: Track specific risks insured under a policy (such as `fire`, `flood`, `theft`, `escape_of_water`, and `accidental_damage`).
- **Financial Thresholds**: Maintain the coverage ceiling (`limit_pence`) and customer deductible (`excess_pence`) for each peril.
- **Validity Windows**: Track coverage timelines via `active_from` and `active_to` to support policy inception, adjustments (see [[concepts/mid-term-changes]]), and policy expirations.
- **Data Encapsulation**: Serve as the backing structure for `CoverLine` projections exposed via [[entities/cover-service]] and [[entities/policy-controller]].

---

## Dependencies

- **[[entities/policy-model]]**: `policy_cover.policy_id` has a foreign key constraint linking to `policies(id)`.
- **[[entities/cover-service]]**: Business logic service responsible for resolving and querying coverage limits and perils for policies.
- **[[entities/policy-controller]]**: Exposes `CoverLine` within the `PolicyView` model across REST endpoints.
- **[[decisions/adr-0003-services-own-their-data]]**: Enforces that direct database access to `policy_cover` is restricted to `policy-admin`.
