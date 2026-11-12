# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this project is

Apiary is an internal **software factory**: a pipeline of narrow agents and automated gates that moves software changes from intent to production, with humans setting intent, designing gates and approving outcomes. It is a proof of concept for use at work and a way to automate improvements to the owner's own projects (The Wire, Eyes on Hawkeye, others). Over-engineering is acceptable when it teaches something.

## Current phase

**Planning only. There is no code yet.** Do not scaffold applications, install dependencies or choose a tech stack unless the owner asks. The work right now is philosophy and strategy, tracked in [PLAN.md](PLAN.md).

## Where things are

- [PLAN.md](PLAN.md): the living planning doc and source of truth. Decisions are in §0 (D1-D5), the loop in §2, tooling in §3, admin UI in §4, risks in §5, next steps in §6, references in §7, theme in §8.
- [SOURCE-NOTES.md](SOURCE-NOTES.md): detailed per-source notes behind PLAN.md §1.2.
- [README.md](README.md): short overview.

## Working conventions

- **PLAN.md is shared.** The owner edits it too, so re-read it before editing and treat on-disk changes as deliberate. Don't revert them.
- Mark open questions `Q:` and decisions `DECISION:`. Add new decisions to the §0 table with the next `D#`.
- When a decision changes something, update every section it touches (the loop, tooling, principles, to-dos in §6).
- Keep the doc honest about sourcing: label second-hand or unverified claims as such, and keep detailed per-source evidence in SOURCE-NOTES.md rather than growing PLAN.md §1.2.
- Paraphrase sources; don't reproduce long passages.

## Naming and vocabulary

The project is **Apiary** (D5). Target projects are hives, agents are workers, work items are cells, the human operator is the beekeeper (full map in PLAN.md §8.1). Use the theme in docs, identifiers and the UI where it helps, but use **plain words for anything an on-call human must understand** (logs, runbooks, compliance-facing material).

## Design principles to respect

The full list is PLAN.md §1.6. The ones most likely to matter when proposing designs:

- Gates over trust: verify independently of the agent that did the work, and never let an agent grade its own work.
- Least privilege per role; enforcement lives outside the agent's reach (sandbox, egress control, credentials never inside the sandbox).
- The factory cannot modify its own gates, policies or context files without a human.
- Everything leaves receipts (prompts, plans, transcripts, evidence).
- Prefer small, boring, inspectable components over frameworks.
- Agents get a refuse/escalate path; blocked is a valid outcome.

## Git and sharing

- This repo is **public** on the owner's GitHub account (D16), so everything committed is world-readable. Never commit secrets, private employer details or anything the owner wouldn't want public. Don't add collaborators or publish artifacts of it without asking.
- Commit and push only when the owner asks.
- Never commit secrets, tokens or credentials.
