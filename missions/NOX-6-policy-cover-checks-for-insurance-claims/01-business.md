---
mission: NOX-6
title: 'Policy cover checks for insurance claims'
role: business
status: ai_drafted
version: 1
author: NoX
ai_drafted: true
---

# Business requirement: Policy cover checks for insurance claims

## The request
"Refactor CoverCheckRepository in claims-management to consume policy-admin REST endpoints instead of direct database reads against policy_cover, establishing integration and contract tests."

## Problem
Claims handling and policy management share internal data connections that make system maintenance riskier and slow down future improvements across both areas.

## Who is affected
Claims handlers and policyholders. Several hundred claims are assessed and settled across home and motor policies each week.

## What should change
Nothing should look or behave differently for claims handlers or policyholders. The team is updating internal connections between systems so that cover checks remain dependable and easier to maintain.

## What "done" looks like
Claims handlers continue checking policy cover details, limits, and excess amounts on incoming claims as usual, with no changes to their everyday workflow.

## Examples
None: nothing changes for customers or staff.

## Verification checklist
- [ ] Claims handlers can still view policy cover details and excess amounts when assessing a claim.
- [ ] New claims can still be processed and settled without interruptions.
