# Apiary

An internal **software factory**: a system where software changes (features, bug fixes, upgrades) flow from intent to running production code through a pipeline of narrow agents and automated gates, with humans moving from reading every line to specifying intent, designing the gates, and approving outcomes.

Themed as an apiary: target projects are **hives**, agents are **workers**, work items are **cells**, and the human is the **beekeeper**. See [PLAN.md §8](PLAN.md) for the vocabulary.

## Status

**Planning phase. No code yet.** The current work is philosophy and strategy:

1. Define the full loop (Detection, Triage, Plan, Build, Verify, Human Approval, Deploy, Refinement, Monitoring).
2. Choose the tools that move and track work through the loop.
3. Design an admin UI on top (entities, system design, tech).

## Goals

- A proof of concept for running this process at work.
- Automating improvements and bug fixes for the author's own projects (The Wire, Eyes on Hawkeye, and others).

## Docs

| File | What it is |
|---|---|
| [PLAN.md](PLAN.md) | The living planning doc: decisions, what a software factory is and isn't, the loop, tooling, admin UI, risks, references. Start here. |
| [SOURCE-NOTES.md](SOURCE-NOTES.md) | Detailed notes on the source articles (Ona, Anthropic, LaunchDarkly, Yegge) that the plan synthesizes. |
| [CLAUDE.md](CLAUDE.md) | Instructions for Claude Code working in this repo. |

## Starting decisions

- A human approves every merge to start (loosen later via risk tiers).
- Claude Code headless / the Agent SDK is the PoC harness.
- GitHub repos and Issues are the PoC substrate.
- Practical parts of Yegge's "model welfare" ideas are adopted as design principles.

See [PLAN.md §0](PLAN.md) for the current list.
