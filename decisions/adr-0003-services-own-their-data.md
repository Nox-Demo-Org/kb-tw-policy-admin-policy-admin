---
type: Architecture Decision
title: 'ADR-0003: Each Service Owns Its Data'
description: Prior to 2022, several external services read directly from the policy-admin database tables (policies and policycover).
resource: https://github.com/Nox-Demo-Org/kb-tw-policy-admin-policy-admin/blob/main/decisions/adr-0003-services-own-their-data.md
tags:
- policy-admin
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:44:46Z'
---

# ADR-0003: Each Service Owns Its Data

## Status

**Accepted** (September 2022)  
**Author:** Daniel Okafor (Architecture)

## Context

Prior to 2022, several external services read directly from the `policy-admin` database tables (`policies` and `policy_cover`). As a result, every internal schema migration or database change in `policy-admin` risked breaking dependent downstream services, often discovered only after deploying to production.

## Decision

1. **Table Ownership:** The [[entities/policy-model|policies]] and [[entities/cover-model|policy_cover]] database tables are owned exclusively by `policy-admin`.
2. **Access Abstraction:** External services must consume policy data exclusively through the REST API (`GET /v1/policies/{id}` exposed via [[entities/policy-controller|PolicyController]]) or through asynchronous domain events such as `policy.policy.issued` and `policy.policy.cancelled` (see [[summaries/api-spec|API Specification]]).
3. **Revocation of Direct Access:** Direct read access for other services' database user accounts was removed from `policy-admin`'s database in 2022.

## Consequences

- **Schema Evolution:** Internal database migrations in `policy-admin` can be performed safely without breaking other services, provided the exposed REST endpoints and published domain event contracts remain backward-compatible.
- **Exception Policy:** Any temporary exception requiring direct database access requires formal architecture review and an explicit sunset/end date.
- **Operational Reality:** Some legacy exceptions (such as `claims-management` querying `policy_cover` directly following performance issues during the 2023 storm surge) have delayed planned database refactors, such as cover table schema splits, until consumers fully migrate back to `GET /v1/policies/{id}`.
