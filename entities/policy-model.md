---
type: Component
title: Policy Model
description: The policy model represents the core domain entity and underlying database schema for insurance policies within policy-admin.
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/entities/policy-model.md
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

# Policy Model

The policy model represents the core domain entity and underlying database schema for insurance policies within `policy-admin`. It encapsulates header-level policy attributes, ownership links to customer records, coverage schedules, and financial terms.

## Database Schema

The `policies` table is defined in `src/main/resources/db/schema.sql`:

```sql
CREATE TABLE policies (
  id            UUID PRIMARY KEY,
  customer_id   UUID NOT NULL,
  product       TEXT NOT NULL,             -- home | motor
  status        TEXT NOT NULL,             -- active | lapsed | cancelled
  start_date    DATE NOT NULL,
  renewal_date  DATE NOT NULL,
  annual_premium_pence BIGINT NOT NULL
);
```

### Column Definitions

- **`id`** (`UUID`, Primary Key): Unique identifier for the policy.
- **`customer_id`** (`UUID`, Not Null): Foreign identifier referencing the customer identity record owned by `customer-identity`.
- **`product`** (`TEXT`, Not Null): The line of business for the policy. Supported values:
  - `home`
  - `motor`
- **`status`** (`TEXT`, Not Null): Current state of the policy. Permitted values:
  - `active`
  - `lapsed`
  - `cancelled`
- **`start_date`** (`DATE`, Not Null): The calendar date when policy coverage commences.
- **`renewal_date`** (`DATE`, Not Null): The calendar date on which the policy is due for annual renewal.
- **`annual_premium_pence`** (`BIGINT`, Not Null): Total annual cost of the policy stored as an integer value in pence (1/100th of GBP).

## Domain Representations and DTOs

In the API boundary (`com.tidewell.policy.api.PolicyController`), the policy entity is projected to clients via the `PolicyView` record:

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

Associated coverage terms are defined through [[entities/cover-model]] lines.

## Responsibilities

- **Entity Persistence**: Serve as the authoritative relational table (`policies`) for policy lifecycle details and customer associations.
- **Financial Value Precision**: Maintain all premium amounts in integer pence (`annual_premium_pence`) to prevent floating-point rounding errors across services.
- **Lifecycle Tracking**: Reflect the current operational status of the policy (`active`, `lapsed`, `cancelled`) as managed by [[concepts/policy-lifecycle]].
- **Data Encapsulation**: Enforce service boundaries where `policy-admin` acts as the exclusive owner of policy records in accordance with [[decisions/adr-0003-services-own-their-data]].

## Dependencies

- **[[entities/cover-model]]**: Child coverage lines (`policy_cover` table) link to `policies.id` via foreign key reference `policy_id`.
- **[[entities/policy-controller]]**: Exposes REST operations to create, read, and mutate policy instances (`GET /v1/policies/{id}`, `POST /v1/policies`, `POST /v1/policies/{id}/changes`).
- **Customer Identity Service**: Policies maintain a reference to `customer_id` representing the customer identity profile.
