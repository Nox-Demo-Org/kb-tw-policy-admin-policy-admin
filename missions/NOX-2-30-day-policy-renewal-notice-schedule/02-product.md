---
mission: NOX-2
title: '30-Day Policy Renewal Notice Schedule'
role: product
status: ai_drafted
version: 1
author: NoX
ai_drafted: true
---

# Product spec: 30-Day Policy Renewal Notice Schedule

## Goal
Policyholders receive their annual renewal notice, pricing, and policy schedule 30 days prior to policy expiration rather than 21 days, giving them more time to review their cover, contact customer support, and renew on time [[kb:policy-admin/concepts/renewal-process]].

## User stories
- As a policyholder with an active home or motor policy, I want to receive my renewal notice 30 days before my current policy expires, so that I have sufficient time to evaluate cover options, make adjustments, and renew before coverage lapses.
- As a customer service representative, I want renewal notifications to go out with 30 days of lead time, so that renewal enquiries and support volumes are distributed over a wider, more manageable window.

## Acceptance criteria
- **AC-1**: Given an active insurance policy whose expiration date is exactly 30 days away, when the daily renewal schedule runs, then a renewal notice is generated and dispatched to the policyholder [[kb:policy-admin/entities/renewal-scheduler]].
- **AC-2**: Given an active policy with an expiration date more than 30 days away, when the daily renewal schedule runs, then no renewal notice is generated for that policy on that day.
- **AC-3**: Given an active policy with an expiration date 30 days away, when the policyholder logs into the customer portal, then their renewal quote, updated policy schedule, and renewal confirmation options are visible and actionable.
- **AC-4**: Given a renewal notice has been triggered at the 30-day mark, when customer support staff view the policyholder's communication and activity history, then the renewal notice dispatch is recorded with a timestamp indicating it occurred 30 days before the expiration date.

## Edge cases
- **Transition / Cutover Window**: Policies currently between 22 and 30 days from expiration on the day this change goes live will trigger renewal notices on the next scheduled run so that no policyholder is skipped during the transition *(suggestion)*.
- **Cancelled or Inactive Policies**: Policies marked as cancelled or lapsed prior to reaching the 30-day window must not generate renewal notices *(from the map)* [[kb:policy-admin/concepts/renewal-process]].
- **Pending Mid-Term Adjustments**: Policies undergoing an in-progress mid-term change when the 30-day threshold is reached must have their renewal notice calculated against the active cover state once the adjustment completes *(from the map)* [[kb:policy-admin/concepts/mid-term-changes]].

## Out of scope
- Modifying the renewal document template, statutory FCA wording, or email copy.
- Changing the renewal pricing calculation, rating algorithms, or cover underwriting rules.
- Altering the direct debit or payment collection execution schedule (payment collection remains scheduled around the policy anniversary date).

## Success metric
- **Baseline**: 73% policy renewal conversion rate [[kb:policy-admin/concepts/renewal-process]].
- **Target**: 82% policy renewal conversion rate [[kb:policy-admin/concepts/renewal-process]].
- **Measurement**: Tracked via the monthly operations dashboard across home and motor policy portfolios, evaluated 60 days following deployment.

## Priority
**P2 (High)**: Directly improves customer retention and conversion from 73% to 82% without introducing architectural complexity. The cost of delay is continued policy lapse from policyholders feeling rushed within the historical 21-day window.

## Verification checklist
- [ ] AC-1: Active policies expiring in 30 days trigger and dispatch renewal notices on the daily schedule run.
- [ ] AC-2: Active policies expiring in more than 30 days do not trigger renewal notices.
- [ ] AC-3: Policyholders whose policies are within 30 days of expiration see their renewal quote and options in the customer portal.
- [ ] AC-4: Customer support staff can confirm 30-day renewal notice events in the policyholder's communication log.
- [ ] Edge case: Policies falling within the 22-to-30 day window at release time trigger notices on the subsequent run without being skipped.
- [ ] Edge case: Cancelled or lapsed policies do not receive renewal notices.
- [ ] Edge case: Policies with active mid-term changes correctly handle renewal notice generation.
- [ ] Success metric: Review monthly operations dashboard at 60 days post-launch to verify renewal conversion rate tracks toward 82%.
