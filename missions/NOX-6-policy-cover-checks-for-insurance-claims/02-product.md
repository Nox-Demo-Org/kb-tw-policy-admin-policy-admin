---
mission: NOX-6
title: 'Policy cover checks for insurance claims'
role: product
status: ai_drafted
version: 1
author: NoX
ai_drafted: true
---

# Product spec: Policy cover checks for insurance claims

## Goal
Keep existing claims cover check behaviour and claims handler assessment workflows exactly as they are today while `claims-management` switches from direct database reads to `policy-admin` service interfaces [[kb:claims-management/decisions/direct-database-access-for-cover-check]].

## User stories
There are no direct user stories for this technical refactor.

## Acceptance criteria
- **AC-1 (Cover eligibility verification):** Given a claim in review with an assigned handler, When the handler assesses whether the incident peril is covered under the customer's policy, Then the cover check returns the same peril coverage eligibility and policy limits as before [[kb:claims-management/concepts/claim-lifecycle]].
- **AC-2 (Excess determination):** Given a claim undergoing assessment for a covered peril, When the handler reviews the applicable policy excess, Then the calculated excess amount matches the active policy terms exactly as before [[kb:claims-management/entities/cover-check]].
- **AC-3 (Non-covered peril handling):** Given a claim submitted for a peril not included in the customer's policy, When the cover verification check runs, Then the system flags the peril as not covered with no changes to triage outcomes or handler notifications [[kb:claims-management/entities/cover-check]].

## Edge cases
- **Policy not found:** If a claim references a non-existent or invalid policy identifier, the cover check fails with a clear missing-policy error, identical to current behavior *(from the map)*.
- **Service disruption or timeout:** If the policy service is temporarily unreachable or slow, the cover check safely raises a retryable service error without corrupting claim state *(from the map)*.
- **Cancelled policy:** If a claim is reviewed against a policy that has been cancelled, the system evaluates cover validity as of the incident date in line with existing lifecycle rules [[kb:policy-admin/concepts/policy-lifecycle]] *(from the map)*.

## Out of scope
- Any changes to claims handler screens, workflows, or permissions.
- Changes to policy cover rules, peril definitions, or excess calculation logic.
- Modifications to fraud detection, handler assignment, or payout settlement processes [[kb:claims-management/concepts/claim-lifecycle]].

## Success metric
- **Cover check error rate:** Baseline < 0.1% failures → Target < 0.1% failures (0% regression), measured via claims service logs and error monitoring 1 week post-deployment *(suggestion)*.
- **Cover check response time:** Baseline p95 ~80 ms [[kb:claims-management/decisions/direct-database-access-for-cover-check]] → Target p95 ≤ 100 ms *(suggestion)*, measured over 14 days post-release.

## Priority
**P3 (Medium)**. This is a technical debt remediation tracked under backlog issue `TWCLM-7`. Direct database access was an expired temporary exception that violates service boundaries and currently blocks planned database schema refactoring in `policy-admin` [[kb:policy-admin/decisions/adr-0003-services-own-their-data]].

## Verification checklist
- [ ] AC-1: Cover eligibility verification behaves exactly as before during claim review.
- [ ] AC-2: Policy excess retrieval and display behaves exactly as before.
- [ ] AC-3: Non-covered peril checks behave exactly as before.
