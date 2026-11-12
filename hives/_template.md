# Hive profile: <name>

*Copy this file to `hives/<repo-name>.md` and fill it in. A hive profile is the per-repo specifics that PLAN.md §2 deliberately leaves open: path-to-tier rules, readiness commands, verification, deploy, monitoring and gate settings. It is public, so keep secrets and sensitive details out; point to the hive's own docs instead.*

Status: **draft | onboarded | paused** · Last audited: YYYY-MM-DD · Owner: <who>

## 1. Identity
- **Repo / visibility / default branch:**
- **What it is** (one or two sentences) and **stack:**
- **How it ships today:**

## 2. Agent-readiness (PLAN.md §2.3 gate)
A fresh clone must go green with no tribal knowledge.
- **Init command:** (install dependencies, start the app if relevant)
- **Check / test commands:** (everything CI runs, plus anything else that must pass)
- **Build command:**
- **Time to green (measured):**
- **Extra tools needed for some tasks:** (e.g. OCR, browsers)
- **Scorecard:** ✅ ready / ⚠ partial / ❌ missing, for: fresh clone green, PR CI, branch ruleset, automated tests, behavioral/visual checks, smoke test, context files, issue forms/labels/board, secrets.

## 3. Context
- **Files agents load:** (CLAUDE.md, skills, docs) and their size.
- **Overrides needed to run under Apiary:** (e.g. "agents never push to the default branch")
- **Human decisions agents must not reverse:**

## 4. Risk rules (D6)
Path rules set a **floor**; an agent can only raise a tier.

| Path / pattern | Tier floor | Notes |
|---|---|---|
| | | |

- **Human-only fields and files:** (things an agent must never set or edit)
- **Never touch:**

## 5. Work types and Detection sources
| Work type | Source / trigger | Typical tier |
|---|---|---|
| | | |

## 6. Verify (PLAN.md §2.5)
- **Required checks on every PR:**
- **Behavioral / visual evaluation:** (what the evaluator agent exercises)
- **Reviewer focus:**

## 7. Deploy and rollback (PLAN.md §2.7)
- **Mechanism and duration:**
- **Gates that block a bad deploy:**
- **Smoke test:**
- **Rollback:**

## 8. Monitoring (PLAN.md §2.9)
- **Signals:**
- **Watch windows by tier:**
- **Human feedback channels:**

## 9. Gates and permissions
- **Branch ruleset on the default branch:**
- **Labels / issue forms / Projects board:**
- **Secrets the factory needs:**
- **Who may open or label issues that trigger runs:**

## 10. Run limits (initial guesses, calibrate with run data)
- **Max turns / job timeout / concurrency:**

## 11. Onboarding checklist
- [ ] ...
