# Feature Spec: [FEATURE_NAME]

**Status:** Draft | In Review | Approved | In Progress | Implemented
**Branch:** [NNN-feature-name]
**Created:** [DATE]

---

## Problem / Goal

What user problem does this solve?
What is the business outcome when this is done?
Focus on WHY — no implementation details in this file.

## Scope

### In scope

- Item 1

### Out of scope

- Item 1 — reason: defer to Epic X / post-MVP / separate story

## User stories

As a [Buyer / Seller / Admin / Platform],
I want [capability],
so that [benefit].

**Acceptance criteria:**

Given [context]
When [action]
Then [expected outcome]
And [additional outcome]

---

## Services affected

| Service              | Change type | Summary                         |
| -------------------- | ----------- | ------------------------------- |
| order-service        | Modify      | Add Dispute aggregate           |
| notification-service | Consume     | New DisputeOpenedEvent consumer |

## Apps affected

| Apps   | Change type | Summary                                    |
| ------ | ----------- | ------------------------------------------ |
| seller | Add         | Add new Dispute Page                       |
| buyer  | Modify      | Modify current Page to add Dispute section |

## Domain model changes

Describe new aggregates, value objects, or changes to existing ones.
Use the existing naming from `services/<name>/README.md` when extending.
