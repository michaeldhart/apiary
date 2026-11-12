# Apiary: Planning Doc

> A software factory, themed as an apiary (D5). Shared scratchpad for Michael and Claude. Edit freely; mark open questions with `Q:` and decisions with `DECISION:`.

Status: **draft 0.1** (2026-11-10)

---

## 0. Decisions so far

| # | Decision | Notes |
|---|----------|-------|
| D1 | Human approves every merge to start | Loosen later via risk tiers (see §1.5). |
| D2 | Claude Code headless / Agent SDK is the PoC harness | Keep prompts/allowlists portable; don't couple the design to one vendor. |
| D3 | GitHub repos + GitHub Issues as the PoC substrate | Design the admin UI's entities VCS-agnostic where cheap. |
| D4 | Adopt the practical parts of Yegge's "model welfare" as principles | Handoffs, refusal/escalation path, honest audit trail, blameless postmortems. Agnostic on metaphysics. |
| D5 | The project is named **Apiary** and themed as an apiary | Vocabulary map and naming notes in §8. Collisions judged unimportant for an internal tool (§8.3). |
| D6 | Risk tiers `R0`-`R3` with path-rule floors | Definitions in §2.0. Path-to-tier rules are filled in per repo later. |
| D7 | Humans apply `apiary:ready`; Triage only proposes | Track proposal-vs-human agreement per tier to graduate to auto-triage (§2.2). |
| D8 | Deploy on merge is acceptable | Matches how the target projects ship today; makes Human Approval the last gate (§2.6-2.7). |
| D9 | Approvals stay in the GitHub PR UI for now | Admin UI approvals are a later option (§4). |
| D10 | Context is part of the Plan station, not its own station | §2.3. |
| D11 | Claude models only for now | No Bedrock/Vertex/other vendors. Revisit when the work PoC needs an approved vendor channel. |
| D12 | The owner is the sole approver for now | Agents can never approve. Revisit "author is not approver" and multi-approver rules when others join. |
| D13 | Phase 0 runner is GitHub Actions only | Separating the agent into its own sandbox is post-MVP (§3). |
| D14 | Issues live in each hive repo; `apiary:` labels are the source of truth | A GitHub Projects board is the MVP admin surface (§3, §4). Tracker sits behind a thin adapter. |
| D15 | Agents run as a distinct bot identity, never the owner's personal token | Human-only gates are validated at the point of use (§2.0). |
| D16 | Stay on the GitHub free plan; make the `apiary` repo **public** | Rulesets cover `apiary` itself. Private hive repos have no branch protection, so they use the fallback gate (§3) until/unless Pro or public. |
| D18 | The first hive is `eyes-on-hawkeye` | Audited 2026-11-12; profile in `hives/eyes-on-hawkeye.md`. It is public, so branch rulesets are available there. |
| D17 | Phase 0 authenticates with the owner's consumer Claude subscription token | Switching to an API key with a spend cap is a **post-MVP improvement**, kept on the table (§3). |

**Pending decisions:** none blocking. Post-MVP candidates: API-key credential, GitHub Pro for hive-repo rulesets, separate agent sandbox (§3).

Motivations: (1) PoC for implementing this at work; (2) automate improvements and bugfixes for Wire, Eyes on Hawkeye and other personal projects. Over-engineering is allowed when it teaches us something.

---

## 1. What is a software factory?

### 1.1 Working definition

A **software factory** is a system in which software changes (features, bug fixes, upgrades) flow from *intent* to *running production code* through a repeatable pipeline of agents and automated gates, with humans moving from **writing and reviewing every line** to **specifying intent, designing the gates, and approving outcomes**.

"Dark code factory" is the same idea pushed to the limit: like a "lights-out" manufacturing plant, nobody is reading the code as it is produced. Ona's team reported building a product in 10 days with ~375 merged PRs and no human-written production code; that is the aspiration end of the spectrum.

### 1.2 Core ideas from the sources

*Synthesis. Per-source detail (numbers, role lists, quotes-in-paraphrase) lives in [SOURCE-NOTES.md](SOURCE-NOTES.md); links in §7. The distilled rules are the principles in §1.6.*

**Where the sources agree**

1. **The factory is the system around the agents, and the gates are the product.** (LaunchDarkly, Ona, Anthropic.) Value comes from stations, handoffs and refusals, not from unattended agents. LaunchDarkly's seven stations (intake, context, plan, execute, review/policy, deliver, observe) and Ona's chain of narrow background agents are the same shape.
2. **Verification caps quality.** Tests and evaluators are the ceiling (Anthropic C compiler); agents grade their own work too generously, so use a separate, skeptical evaluator that exercises the running app (Anthropic harness design, LaunchDarkly builder/reviewer split); don't trust self-reports, re-run checks independently; watch for tests being weakened.
3. **Shift judgment left.** Approve plans, not just diffs (LaunchDarkly); spec quality drives outcomes and most defects were spec gaps (Ona); define "done" as testable criteria up front (Anthropic feature lists, sprint contracts).
4. **Long-running work needs durable state outside the agent.** One unit of work per session, oriented from progress notes, git history and a feature list, ending in a clean commit (Anthropic harnesses); append-only session log, stateless harness, disposable sandbox (Anthropic Managed Agents); handoffs over hard kills and seats vs. sessions (Yegge).
5. **Humans move up the stack.** Intent, plan approval, gate design, taste and exception handling (all sources); "platform engineer" of the agents' environment (Ona); judge intent rather than read every line, with risk-tiered oversight (Anthropic internal).
6. **Safety lives in the environment, not the prompt.** Agents route around denylists and sandboxes just to finish a task (Ona); hard boundaries first (sandbox/VM, filesystem, egress control, credentials never inside the sandbox), classifiers second because they're probabilistic (Anthropic: ~17% miss on real overeager actions); least privilege per role (LaunchDarkly); approval fatigue is a risk; untrusted issue text is attacker input.
7. **The harness is an assumption about model weakness.** Re-test and prune components as models change; pin and log all defaults, because silent changes degrade quality without failing tests (Anthropic harness design and April postmortem); model updates can break orchestrators (Yegge).
8. **Close the loop with operations.** Errors, user feedback and a quality scorecard feed new work (Ona, Yegge's Wish Factory); "crons watch, models act" for cheap monitoring; post-merge smoke test plus auto-revert (LaunchDarkly, Ona).
9. **Keep it small, boring and inspectable.** Simple composable workflows beat heavy frameworks (Anthropic, LaunchDarkly, Yegge); pair deterministic tools with agent judgment (Ona Automations).

**Where the sources differ or add a caution**

- **Orchestration style:** Yegge runs an always-on fleet with a design/implement/review cycle and tolerates a sometimes-red main; Anthropic's C compiler used lock-file claiming with no orchestrator; LaunchDarkly and Ona use explicit staged pipelines. Anthropic also says multi-agent setups use ~15x the tokens of chat and fit coding poorly when tasks share context. *We lean on the staged pipeline, parallelizing across independent issues.*
- **Security posture:** Yegge skips sandboxing; everyone else (and the evidence) says don't. *We don't skip it.*
- **Taste and UX:** automated gates miss what only shows when using the product (Ona's 12 UX bugs, generic drift). Human dogfooding and a design reference stay necessary.
- **Cost and upfront effort:** Ona spent 8-10 h/day for three days building the factory before it paid back; harness runs ranged from $9 (broken) to ~$200 (working) for one app, to ~$20k for a 100k-line compiler (Anthropic); Yegge reports tens of thousands of dollars a month. Cheap output also risks feature bloat.
- **Scale:** at agent-scale throughput, merge queues and source control become the bottleneck (Yegge, Anthropic's Scion); Anthropic dropped the global merge lock for post-merge testing. Not a PoC concern; relevant for work.
- **Model welfare:** Yegge's practical patterns (handoffs, refuse/escalate, honest audit trail, blameless postmortems) are adopted as principles (D4); the metaphysics is left open.
- **Confidence:** some claims about Anthropic's internal practice (90% Claude-written code, none/one/two-human review tiers, multi-agent Antique review) are second-hand and unverified; see SOURCE-NOTES.md.

**Autonomy framing (Ona AI-SDLC):** Do (AI assists) → Delegate (parallel background agents) → Automate (events trigger agent workflows). Each stage moves the bottleneck (review, then planning/CI/release, then organizational judgment and security readiness); plan for the next one. This maps onto our L2 → L3 path (§1.5) and the adoption story at work.

### 1.3 What a software factory **is**

- A **pipeline with explicit stations and gates**, where each station has defined inputs, outputs, permissions and exit criteria.
- **Spec-driven**: work enters as a structured intent with a definition of done.
- **Evidence-producing**: every change ships with proof (tests run, screenshots, traces, plan, transcript).
- **Defense in depth**: independent verification, separate reviewer, policy checks, post-merge canary, auto-rollback.
- **Self-observing**: production signals feed back into new work items (the loop closes).
- **Auditable**: any line in prod traces to a request, a plan, an approval.
- **A product you operate**: it has its own backlog, metrics, cost, incidents and maintainers.

### 1.4 What a software factory **is not**

- **Not "vibe coding at scale"**. Unreviewed output with no gates is a slot machine, not a factory.
- **Not a single smart agent** with a big prompt. The factory is the system *around* the agents.
- **Not removing humans.** It relocates them: from diff-reading to intent, plan approval, gate design, taste and exception handling.
- **Not a replacement for good engineering fundamentals.** Tests, types, observability and clear architecture matter more, because they are the gates.
- **Not an IDE copilot or a chat sidebar.** It is asynchronous and event-driven.
- **Not finished when the code merges.** Ona's agents missed UX/taste problems nothing in code could measure; the factory needs a path for human use, taste and product judgment, and for user feedback to re-enter the loop.
- **Not a swarm for its own sake.** Anthropic found multi-agent setups use ~15x the tokens of chat and fit coding poorly when tasks share context; parallelism pays when work decomposes into independent items (separate issues, separate tests), not within one tightly coupled change.
- **Not framework-shaped (yet).** Both LaunchDarkly and Yegge argue for small, bespoke, understandable systems over heavyweight generic orchestration.
- **Not free.** Tokens, CI minutes, review attention and maintenance of the factory itself are the cost centers.
- **Not safe by default.** Autonomous agents with credentials are a security surface; sandboxing and least privilege are design requirements.

### 1.5 The autonomy spectrum

| Level | Description | Human role |
|-------|-------------|-----------|
| L0 | Assistant: human drives, agent suggests | Everything |
| L1 | Agent implements on request; human reviews diff | Review every diff |
| L2 | Agent plans + implements + self-verifies; human approves plan and merge | **Starting point (D1)** |
| L3 | Risk-tiered auto-merge (docs, deps, small fixes); human approves rest | Policy + exceptions |
| L4 | Dark: automated gates, canary, auto-revert; humans set intent and audit | Intent + audit |

Target for the PoC: **L2**, with the architecture built so L3 is a config change per project/risk tier.

### 1.6 Guiding principles (draft)

1. Gates over trust. Verify independently of the agent that did the work.
2. Shift judgment left. Approve plans, not just diffs.
3. Least privilege per role; enforcement outside the agent's reach.
4. The factory cannot modify its own gates without a human.
5. Everything leaves receipts (prompts, plans, transcripts, evidence).
6. Small enough to understand. Prefer boring, inspectable components.
7. Agents get a clear refuse/escalate path; blocked is a valid outcome, not a failure. *(from D4)*
8. Handoffs over hard kills; bounded sessions with written handoff notes. *(D4)*
9. Blameless postmortems feed shared rules/context, not just fixes. *(D4)*
10. Throughput and cost are first-class constraints, not afterthoughts.
11. Narrow agents with one job, one trigger and an explicit handoff beat one big agent. *(Ona)*
12. When an agent drifts, fix the harness, not just the output; make standards enforceable by scripts/checks, not prose. *(Ona)*
13. Pair deterministic tools with agent judgment; the agent runs the tools, it doesn't replace them. *(Ona)*
14. Prompts and permission lists are not security boundaries; enforcement lives outside the agent (isolation, network egress, content-based). *(Ona)*
15. Spec quality caps output quality; the factory should refuse or push back on under-specified work. *(Ona)*
16. Close the operations-to-planning loop early (errors, user feedback, quality scorecard). *(Ona)*
17. A human can always open the exact environment an agent used and take over. *(Ona)*
18. The verifier is the ceiling: output quality is capped by the quality of tests/evaluators, so invest there first. Every PR carries a test that fails before and passes after. *(Anthropic: C compiler, Bun practice)*
19. Never let an agent grade its own work; use a separate, skeptically tuned evaluator that exercises the running app. *(Anthropic: harness design)*
20. One unit of work per session; orient from durable artifacts (progress file, git log, feature list) and leave a clean, committed state. *(Anthropic: long-running harnesses)*
21. Hard boundaries in the environment first (sandbox, egress control, credentials never inside the sandbox); model-layer classifiers are a second, probabilistic layer. *(Anthropic: containment, Managed Agents)*
22. Brain, hands and session are separate: stateless harness, disposable sandboxes, an append-only session log outside both. *(Anthropic: Managed Agents)*
23. Every harness component is an assumption about model weakness; re-test and prune as models change, and pin/version all defaults. *(Anthropic: harness design, postmortem)*
24. Evaluate outcomes not paths; keep capability evals separate from regression evals for the factory itself. *(Anthropic: evals)*
25. Risk-tiered human oversight: approvals scale with blast radius, and rules can be semantic (docs vs. logic vs. sensitive paths). *(Anthropic: Antique)*

---

## 2. Strategy Step 1: Define the full loop

*Station specs drafted 2026-11-11, with the shape agreed in outline (D6-D10). The cards are proposals to refine per station; per-repo specifics (path rules, deploy details, smoke tests) come later in a per-hive profile. Notes from the sources are kept under each station.*

```
Detection → Triage → Plan → Build → Verify → Human Approval → Deploy → Refinement → Monitoring
     ↑                                                                                  │
     └────────────────────────── alerts / new work items ───────────────────────────────┘
```

Context (conventions, architecture notes, never-touch list) is **part of Plan**, not a separate station (D10).

### 2.0 Shared conventions

- **Actors:** `S` = script (deterministic), `A` = agent, `H` = human. Where a script can do the job, it does.
- **State and artifacts:** the GitHub issue is the record. Each station moves the issue by label (`apiary:<state>`, see §3) and posts its output as a structured comment or attachment. Every run also writes a **run record**: transcript, model/config versions, cost, exit state.
- **Permission profiles:** `read-only` (repo and telemetry); `issue-write` (comment and label only); `sandbox-write` (its own worktree, no secrets, allowlisted egress, push only to `apiary/*` branches); `human-only` (merge, deploy credentials, and any change to gates, policies or prompts).
- **Exit states:** every station ends **pass** (moves on), **bounce** (returns to an earlier station with a reason) or **blocked** (parked for a human with a summary). Retries are bounded: one automatic retry in a fresh session, then escalate. Repeated permission denials and budget hits always escalate.
- **Risk tiers (D6):** `R0` docs, comments, tests only; `R1` small logic change in a non-sensitive path; `R2` behavior change or feature; `R3` sensitive area (auth, data, infra, deploy) or the factory itself. Path rules set a **floor** on the tier and an agent can only raise it. The path-to-tier rules are filled in per repo in a **hive profile** (`hives/<repo>.md`, template at `hives/_template.md`). Tiers measure blast radius, not file type: on a content hive such as `eyes-on-hawkeye`, prose that makes claims about real people is `R3`, not `R0`.
- **Human-controlled gates (D7, D9):** a human applies `apiary:ready` (after Triage proposes) and `apiary:plan-approved`, and approves and merges the PR in the GitHub UI. Agent identities must not be able to apply these labels or approve/merge. **Enforcement (D15):** (1) agents run as a bot identity (the Claude GitHub App or a custom App), never the owner's personal token, so GitHub records who did what; (2) before Build claims an issue, a script reads the issue timeline and confirms the latest `ready` and `plan-approved` label events came from an allowed human, and an Actions check strips those labels if a bot applies them; (3) merge is guarded by a branch ruleset (required human review, no direct pushes, no auto-merge), which needs a public repo or GitHub Pro (§3). *Chaining note:* GitHub does not fire workflows from events created with the default `GITHUB_TOKEN`, and the Claude action rejects bot-triggered runs unless the bot is allowlisted (`allowed_bots`), so stage-to-stage handoffs use `workflow_dispatch` or an App token with an explicit allowlist.
- **Metrics:** every station reports throughput, latency, pass/bounce/blocked rates and cost. Cards list only what is specific to the station.

### 2.1 Detection / Intake
*Sources: bug reports, user requests, production errors/alerts, dependency updates, cron watchers, user feedback, agent-noticed follow-ups.*
- **Trigger:** events (Sentry alert, dependency/CVE alert, feedback, a human-opened issue) and scheduled sweeps (CVEs, dead code, quality scorecard).
- **Inputs:** raw signal (text, stack trace, alert payload).
- **Outputs:** an issue from the template (type, source, evidence, goal) labeled `apiary:detected`, with duplicates linked.
- **Actor:** `S` captures, fingerprints and dedupes; a cheap `A` only normalizes free text into the template.
- **Permissions:** `issue-write`. Raw text is quoted as untrusted data and never treated as instructions.
- **Gate:** source and evidence present, not a duplicate, under the rate limit.
- **Failure:** unparseable input goes to a human inbox (`needs-info`); floods are batched into a digest.
- **Metrics:** issues by source, duplicate rate, share that passes triage.
- **Template (GitHub issue forms in `.github/ISSUE_TEMPLATE/`):** two forms, **bug** and **change** (feature or chore). Required: goal/summary, evidence (logs, steps, links), definition of done as testable checks. Bug form adds steps to reproduce and expected vs. actual. Optional: constraints, out of scope, suggested tier (Triage decides the tier). If "definition of done" is blank, Triage drafts one for the owner to confirm. Detection fills the same fields for machine-created issues.

**Notes from the sources**
- Ona: intake also includes agent-generated work: Sentry-to-issue incident responder, a Product Improver that screenshots the live app and files `needs-human` issues, a `quality.md` scorecard that spawns issues for weak areas, and an in-app feedback widget + digest. Scheduled backlog pickers and CVE/dead-code sweeps target chronic toil.
- Treat issue text as untrusted input (prompt injection). Hive-specific sources and work types live in each hive profile (for example a watcher on the city's document page for `eyes-on-hawkeye`).

### 2.2 Triage
- **Trigger:** `apiary:detected`.
- **Inputs:** the issue, the hive's context (see §2.3), the repo (for reproduction).
- **Outputs:** a triage **proposal** posted on the issue: type, priority, risk tier, size (`L` must be split), repro result for bugs, and a readiness verdict (ready / `needs-spec` / `needs-info`). Labeled `apiary:triaged`.
- **Actor:** a cheap `A` using `read-only` plus a no-credentials sandbox to attempt reproduction. **A human applies `apiary:ready` (D7);** Triage only proposes.
- **Permissions:** `issue-write` plus a throwaway sandbox for repro.
- **Gate:** a proposal may say "ready" only with a testable definition of done, a repro (for bugs), a tier and size ≤ M.
- **Failure:** bounce as `needs-spec` or `needs-info` with specific questions; stale `needs-info` auto-closes after N days. A human can override any field.
- **Metrics:** bounce rate, repro rate, **proposal-vs-human agreement rate** per tier, and tier accuracy later judged against reverts and escalations.
- **Graduating to auto-triage:** when agreement is consistently high for a tier (suggested starting bar: ≥95% over 50 issues, `R0` first), let the proposal apply `ready` for that tier automatically. Protocol: log agreement from day one. An **agreement** means the human applied `ready` without changing the proposed tier, size or verdict. Track under-estimated tier (a proposal lower than the human's) separately, with a target of zero; path-rule floors make it structurally safer. Revisit after roughly 30 issues per tier and set thresholds from the data.

**Notes from the sources**
- Ona: most defects traced to spec gaps, so triage should be able to bounce work back as "under-specified" rather than build it. Risk tier set here drives escalation (low = auto path later, high = human).
- Anthropic: triage is a **routing** step (cheap model classifies type/risk/project, then dispatches). Bun-style pattern: an agent first tries to **reproduce** a bug before any fix is planned; an unreproducible issue goes back for more info. Consider semantic risk rules (docs/comments vs. logic vs. sensitive paths) rather than just labels.

### 2.3 Plan (includes Context)
Plan owns both the standing context agents work from and the plan for each issue (D10).

**Context (standing layer, per hive)**
- **Contents:** a short `CLAUDE.md` index, `architecture.md`, conventions, a never-touch list (enforced by path rules, not prose), a runnable `init.sh`, and the test commands.
- **Gate (agent-readiness):** a fresh clone runs `init.sh` and the tests pass. A hive that fails this is not onboarded.
- **Kept fresh by:** Refinement (§2.8), as proposals a human approves.

**Plan (per issue)**
- **Trigger:** `apiary:ready`.
- **Inputs:** the issue, the triage proposal, the hive's context.
- **Outputs:** an issue comment with the approach, files touched, a test plan (which failing tests get written first), a testable acceptance checklist, rollback, and a cost budget. Labeled `apiary:planned`.
- **Actor:** a strong `A`, `read-only`.
- **Permissions:** `read-only` on the repo; network limited to an allowlist of docs.
- **Gate:** a plan-lint `S` checks that required sections exist, acceptance items are testable, and no protected paths are touched below `R3`. Then **a human applies `apiary:plan-approved`** (the shift-left gate).
- **Failure:** an unplannable issue bounces to Triage with questions. At most two plan revision cycles, then a human takes over.
- **Metrics:** first-pass plan approval rate, revisions per plan, plan-to-diff divergence later in Verify.

**Notes from the sources**
- Ona's harness set to copy/adapt: `AGENTS.md`-style guide (incl. escalation rules and risky areas), `architecture.md`, `design.md` (visual ground truth, plus Storybook for UI), `conventions.md`, `quality.md` scorecard, product spec. Prefer enforceable checks over prose.
- Anthropic: keep the always-loaded layer small (an index pointing to deeper docs); long instruction files reduce adherence. Use just-in-time retrieval and structured notes; one file per entity to avoid write collisions between parallel agents; verify notes against transcripts on a schedule so wrong notes don't propagate. Include a runnable `init.sh`-style script so agents don't burn tokens learning how to start the app. Don't execute repo-supplied config/hooks before trust is established.
- Anthropic: explore first (explicitly no writing), then plan, and **save the plan to an issue as a checkpoint**. Planner stays high-level on technical detail so early errors don't cascade, but **defines "done" as testable criteria** (a sprint-contract style agreement with the evaluator). Decompose into a granular feature/acceptance list, each item initially failing, so later sessions can't declare victory early.

### 2.4 Build
- **Trigger:** `apiary:plan-approved`, claimed by label so only one agent takes the issue.
- **Inputs:** the approved plan, the repo at the base SHA, the hive's context.
- **Outputs:** a draft PR with the failing tests committed first, then the implementation; a progress/handoff file; the run record.
- **Actor:** a headless `A` in a disposable sandbox. Step one writes the tests and an `S` confirms they fail; step two implements and may not edit those tests.
- **Permissions:** `sandbox-write`. Credentials never enter the sandbox; push goes through a mediated path to `apiary/*` branches only.
- **Gate:** local checks green, every acceptance item mapped to a test, and a scope-check `S` finds the diff within the plan.
- **Failure:** on a budget, time or iteration limit it writes handoff notes and retries once in a fresh session, then goes `blocked`. Repeated permission denials escalate.
- **Metrics:** first-pass rate, sessions per issue, cost per PR, scope violations.

**Notes from the sources**
- Ona: use a reproducible devcontainer-style environment a human can open to take over; isolation per agent; network egress controls; don't rely on denylists (agents route around them). Separate small roles (builder, bug fixer, conflict "shepherd").
- Anthropic: **one issue/feature per session**, orient from progress notes + git log + a quick smoke test, verify end-to-end before marking done, commit and update progress to leave a clean state. TDD: tests written and committed first, implementer may not edit them. Run headless (`claude -p`) with a narrow allowlist, in a container without general internet; per-agent worktree. Credentials never enter the sandbox (use a proxy/vault; git push via a mediated path). Budget caps and iteration limits per run; agents have time blindness, so give them timeouts and sparse, greppable output. Expect headless runs to terminate after repeated permission denials and treat that as an escalation, not a bug.

### 2.5 Verify
- **Trigger:** PR marked ready for review.
- **Inputs:** the PR, the plan and acceptance checklist, the run record.
- **Outputs:** a verification report: CI, policy and protected-path results; confirmation that each new test fails before and passes after; reviewer findings; evaluator evidence (screenshots, traces); a verdict.
- **Actor:** `S` for the deterministic checks; a fresh-context reviewer `A` (`read-only`); an evaluator `A` that exercises the running app. Neither agent sees the builder's transcript.
- **Permissions:** reviewer `read-only`; evaluator runs the app in its own sandbox, read-only on the repo.
- **Gate:** all deterministic checks green, no blocking reviewer findings, and the evaluator meets the acceptance checklist.
- **Failure:** up to two automatic fix loops back to Build with the findings, then escalate (`blocked`).
- **Metrics:** defect escape rate (problems found after Verify), reviewer false-positive rate, loops per PR, evaluator-vs-human agreement (calibration).

**Notes from the sources**
- Ona: reviewer agent checks PRs against documented conventions and can fix CI failures; a UI verifier drives the running app. Known blind spot: UX/visual/taste defects that only appear when a human (or screenshot-driven agent) actually uses the product. Plan an explicit visual/behavioral check and an occasional human dogfooding pass.
- Anthropic: the evaluator must be a **different, fresh-context agent**, tuned skeptical and calibrated against human judgment (few-shot examples); self-evaluation is too generous. Give it browser automation (Playwright/Puppeteer) to exercise the running app; unit tests and curl checks alone gave false "done". Known limits: native browser dialogs are invisible to it. Add **voting/multiple passes** for security-type review; adversarial review tends to be over-defensive, so tell it what not to flag. **CI blocks regressions**; PRs without a failing-then-passing test are rejected; make test output short, greppable and deterministic. Verify with the semantic risk rules (comment-only vs. logic vs. sensitive paths). Remember the verifier is the ceiling: passing tests can create false confidence.

### 2.6 Human Approval
- **Trigger:** verification passed.
- **Inputs:** the approval packet (PR description plus verification report).
- **Outputs:** a decision (approve, request changes, or reject) with a reason, recorded in the **GitHub PR UI (D9)**. The human merges.
- **Actor:** `H`. The packet fits one screen: intent, plan versus diff scope, risk tier, evidence, cost, open reviewer concerns.
- **Permissions:** `human-only` (branch protection requires a human review; agent identities cannot approve or merge).
- **Gate:** per D1 a human approves every merge. `R3` gets an explicit second look. *(Because deploy is on merge, D8, this is the last gate before production.)*
- **Failure:** on a stale SLA, remind, then mark stale. Rejection reasons feed Refinement.
- **Metrics:** time to decision, approval rate, rework rate, and very short decision times as a rubber-stamp signal.

**Notes from the sources**
- Ona: approval fatigue is a security risk; keep prompts rare and meaningful, and flag requests that describe working around a control. Escalations should arrive with enough context that the fix also updates the automation so it doesn't recur.
- Anthropic: even at 30k merges/day, engineers still review each change request but judge **intent, not every line**; oversight is **risk-tiered** (second-hand: none / one / two humans by sensitivity). Our D1 (human approves every merge) is the L2 start; the L3 route is a fresh-context agent classifying "simple and low-risk" plus semantic rules, which Anthropic says it expects to enable for itself. Approval prompts should be rare; auto mode's deny-and-continue with escalation after repeated denials is a good template.

### 2.7 Deploy
- **Trigger:** merge to the default branch. **Deploy on merge is acceptable (D8)** and matches how the target projects ship today.
- **Inputs:** the merged commit.
- **Outputs:** the deployed version, flag state (if any), and a rollback handle.
- **Actor:** `S`: the project's existing CI/CD.
- **Permissions:** deploy credentials are held by the pipeline and never by agents.
- **Gate:** a post-merge smoke test, plus a canary or feature flag where the hive supports one.
- **Failure:** `S` opens an auto-revert PR and alerts a human. Revert criteria are set per hive.
- **Metrics:** change failure rate, time to restore, deploy frequency.
- Per-hive deploy details, smoke test and rollback live in the hive profile (`hives/`). First hive: `eyes-on-hawkeye` (D18): deploys on push to `main` via GitHub Pages in about 30-90 s; the checks and build already run before the deploy job; no smoke test or PR CI yet.

**Notes from the sources**
- Anthropic: at agent-scale throughput they **dropped the global merge-queue lock** in favor of post-merge and pre-deploy automated testing, with agents resolving conflicts. Not a PoC problem (see §5) but it implies Deploy gates should live post-merge as well as pre-merge. Roll out behavior changes (including to the factory's own prompts/config) gradually with soak periods, as in their postmortem.

### 2.8 Refinement
*Learn from what just shipped and improve the product and the factory. Runs after Deploy, before long-running Monitoring takes over.*
- **Trigger:** each merge (a scripted mini-retro) plus a weekly factory review.
- **Inputs:** run records, reviews, rejected plans/PRs, escalations, factory metrics.
- **Outputs:** follow-up issues (tech debt, "noticed in passing"); eval cases added from failures; blameless retro notes; **proposals** to change context, prompts, allowlists or policies as `R3` PRs; handoff-note and memory curation.
- **Actor:** an auditor `A` plus `S` for metrics and the eval suite.
- **Permissions:** reads run logs; may open proposal PRs against context/policy files but cannot merge them. Agents never apply their own gate changes (principle 4).
- **Gate:** no change to the factory's own config lands without a human approval and a passing eval suite.
- **Failure:** an eval regression reverts the proposed change.
- **Metrics:** recurring failure classes, eval pass rate (capability vs. regression), context staleness.

**Notes from the sources**
- Ona: an **Automation Auditor** agent that reviews the factory's own automations and proposes improvements (it found a real bug over one weekend). Enable at launch, but proposals to change gates still need a human (principle 4). Also a `quality.md`-style scorecard that turns weak areas into work, and a periodic product-judgment pass to kill features (cheap output encourages bloat).
- Anthropic: **evals for the factory itself**: start with 20-50 tasks drawn from real failures (replayed issues with known-good outcomes), grade the *outcome* (final repo/app state) not the path, keep **capability evals** (improving) distinct from **regression evals** (should stay ~100%), report pass^k for reliability, read transcripts regularly, isolate trials. Re-run whenever the model, prompts, effort level or allowlists change; **pin and log every default** per run (their April postmortem). **Re-test each harness component against the current model and remove what is no longer load-bearing.** Memory hygiene: verify agent notes against transcripts on a schedule ("dreaming"-style consolidation), prune bloat, one file per entity, small always-loaded index; wrong notes propagate to every future session.

### 2.9 Monitoring
*The final station: continuously observe what is running in production. Mostly cheap scripts and watchers ("crons watch, models act"); wake a model only when judgment is needed. Closes the loop by feeding Detection.*
- **Trigger:** continuous, plus a post-deploy watch window per change.
- **Inputs:** error tracking, health checks, logs, feedback, factory cost and quality metrics.
- **Outputs:** a per-change verdict (healthy / degraded / reverted) on the issue, new Detection issues (§2.1), and a daily digest.
- **Actor:** `S` watches and wakes an `A` only when a threshold trips.
- **Permissions:** `read-only` on telemetry; may open issues; may trigger a revert only through the Deploy script's preapproved criteria.
- **Gate:** the watch window closes with no regression, then the issue is closed. *Watch windows (adopted, tunable with data): `R0` 1 hour, `R1` 24 hours, `R2`/`R3` 7 days.*
- **Failure:** a dead-man switch alerts a human if the monitor itself goes silent.
- **Metrics:** time to detect, false-alarm rate, share of problems the factory found before a human did.

**Notes from the sources**
- Ona: Post-Merge Verifier + UI Verifier, Incident Responder polling Sentry, Performance Monitor, Daily Metrics, and a user-feedback widget with digest. Add the feedback path early; Ona only added it around day 7.
- Anthropic: "AI superfans" that each watch one component and catch subtle anomalies threshold alerts miss (pairs with "crons watch, models act"). Production tracing of agent decisions (not just app metrics) was essential for debugging non-deterministic runs. Watch for **drift in the factory itself**: first-pass success, revert rate and cost per merged change per model/config version, with canary runs after any model or prompt change.

### 2.10 Meta: Factory health
- Track across all stations: cost per merged change, cycle time (issue to merge to healthy), first-pass success rate, bounce/blocked rate, revert rate, human review burden, and proposal-vs-human agreement in Triage.

**Notes from the sources**
- Anthropic reference points: tokens explain most performance variance (budget accordingly); multi-agent ~15x chat tokens; harness experiments ranged $9 (broken) to ~$200 (working) for one app and ~$20k for a 100k-line compiler; track pass^k reliability and cost per merged change, and review-comment substance (second-hand: substantive comments rose from 16% to 54% of PRs after Antique).
- Ona benchmarks for reference: median issue cycle 38 min, median PR cycle 4.9 min, ~87% autonomous, human time per day (8-10 h while building the factory, 2-3 h steady state).

---

## 3. Strategy Step 2: Tools to move/track issues through the loop

*Stub.*

- Baseline (D3): GitHub Issues + Projects + labels/state machine, GitHub Actions as the runner, Claude Code headless/Agent SDK as the worker.
- **Draft issue state machine** (labels `apiary:<state>`; consistent with §2):

| State | Set by | Meaning / next |
|---|---|---|
| `detected` | `S` (Detection) | New issue from the template. |
| `triaged` | `A` (Triage) | Proposal posted (type, tier, size, readiness). |
| `needs-info` / `needs-spec` | `A` or `H` | Bounced back with questions; returns to `detected` when answered. |
| `ready` | **`H` only (D7)** | Human accepted the triage proposal. |
| `planned` | `A` (Plan) | Plan posted and plan-lint passed. |
| `plan-approved` | **`H` only** | Human approved the plan; Build may claim it. |
| `building` | `A` (Build) | Claimed; a run is in flight. |
| `verifying` | `S` | PR ready; Verify running (up to two fix loops back to `building`). |
| `awaiting-approval` | `S` | Verification passed; packet ready for the human. |
| `merged` | GitHub (human merge, D9) | Deploy on merge (D8) is running. |
| `monitoring` | `S` | Post-deploy watch window open. |
| `closed` | `S` | Window closed healthy; or `reverted`, which spawns a new issue. |
| `blocked` | any station | Parked for a human with a summary (any state can enter it). |

  The two loops stay separable: *create work* is `detected` through `ready`; *do work* is `planned` through `merged`. Label names are confirmed (D14); issues live in each hive repo, with `apiary:`, `risk:R0`-`R3` and `type:` labels. Human-only enforcement is in §2.0.
- **Tracker (D3, D14):** GitHub Issues, behind a **thin adapter** with a few operations (list ready items, post a comment, set a state) so work can later plug in Jira or Linear. Skip Beads for now; revisit only if a dependency graph across many parallel agents is needed.

- **GitHub Projects as the MVP control surface (D14).** One Projects (v2) board spans the hive repos. It is a *view* of the labels, kept in sync by workflows; labels remain the source of truth.
  - **Fields:** Status (mirrors the `apiary:` state), Hive, Type, Risk tier, Size, Cost (number), Runs (number), Run link, Triage agreed (yes/no, for the agreement metric).
  - **Views:** *Intake* (`detected`/`triaged`, waiting for the owner's `ready`); *Plan review* (`planned`, waiting for `plan-approved`); *Approval inbox* (`awaiting-approval`, with the linked PR); *In flight* (`building`/`verifying`); *Monitoring*; *Blocked*; *History*.
  - **Automation:** built-in project workflows auto-add issues and set Status on close/merge; our Actions update the other fields through the GraphQL API with an App token. Insights charts cover throughput, cost and blocked counts by hive and tier.
  - **What it does not cover (the gaps a custom admin UI would fill later, §4):** run timeline and transcript viewer, per-run cost and token drill-down, factory health metrics (triage agreement, defect escape), policy/prompt editor, kill switch, live run status.
- **Runner (D13): GitHub Actions only for Phase 0.** Use the official `anthropics/claude-code-action`: headless Claude Code on hosted runners, authenticating by default as the Claude GitHub App (a distinct bot identity, D15), with an automation mode driven by a `prompt` (or a skill), `--max-turns`, `--allowedTools` and `--model`. Its own checks require write access for issue/PR triggers and reject bot actors unless allowlisted (§2.0 chaining note). It also notes that pushes made with the default `GITHUB_TOKEN` do not trigger CI, so use the App identity for pushes.
  - **Phase 0 guardrails:** dedicated credential with a spend cap; repo-scoped token; `--max-turns` and job timeouts; concurrency limits; narrow tool allowlist; start with a low-stakes hive. **Accepted residual risk:** the credential is visible to the agent's environment.
  - **Phase 1 (post-MVP):** a separate sandbox with a credential proxy and egress allowlist, per Anthropic's pattern (§1.2).
  - **Cost tracking:** no model gateway for the PoC. Capture turns, duration and token usage (and dollar cost when billed by API) into each run record.
- **Credential (D17): consumer Claude subscription for Phase 0; API key is a post-MVP improvement.** Anthropic's Actions docs support a subscription OAuth token (`claude setup-token`, stored as `CLAUDE_CODE_OAUTH_TOKEN`); runs then use the subscription instead of API billing. Caveats from Anthropic's legal/compliance page: OAuth is "designed for ordinary use", advertised plan limits assume ordinary individual usage, and developers building products or services (including with the Agent SDK) should use API-key authentication. The token is also tied to one person's subscription, and its limits are shared with interactive use. *Decision and mitigations (my reading of the terms, not a ruling):* start with the owner's subscription token via the official action, on the owner's own repos at modest volume. Most runs are human-initiated because Plan and Build start from human-applied labels (D7), which fits "ordinary use" better than a fully scheduled loop; keep scheduled/automatic runs few and cheap, use `--max-turns`, and watch rate limits since they are shared with interactive use. **Move to an API key with a spend cap** (the post-MVP improvement) for scheduled or heavy use, any direct Agent SDK use, and the work PoC. Re-check Anthropic's current terms before scaling up.
- **Repo visibility and gate protection (D16).** Branch protection and rulesets need GitHub Pro or a public repo. `apiary` is public, so rulesets protect it (and its reusable workflows can be called from other repos). **`eyes-on-hawkeye` is also public, so rulesets are available there too** and the fallback is not needed for the first hive. Private hive repos on the free plan have no branch protection, so they use the **fallback gate**: a required status check that fails unless the owner approved, plus a convention of never giving agents push rights to the default branch. An admin can bypass this, so it is weaker; GitHub Pro (about $4/month) or a public hive closes the gap later. *Cost note:* reusable workflows run in, and bill minutes to, the calling repo; the free plan includes 2,000 Actions minutes per month for private repos, so watch usage. A public repo also means anyone can open issues or PRs: any workflow acting on issue or comment text must check the author has write access (the Claude action already does) and treat the text as untrusted; fork PRs do not receive secrets by default.
- **Observability (adopted):** for hives, Sentry (or whatever a hive already uses) for errors plus a simple uptime check. For the factory, each run uploads its run record as an Actions artifact and summarizes it in an issue comment; a scheduled script aggregates metrics. The admin UI can read these later.
- **Work differences (deferred):** keep to Claude for now (D11) and the owner as sole approver (D12). Questions to answer before the work pilot: which tracker and code host the team uses; approved AI vendors and channels (direct API, Bedrock, Vertex); what change-management/SOC 2 requires of an approver (the human gate already satisfies "author is not approver" while an agent can never approve its own PR); whether code leaving the network needs review; whether agent access to repos needs a security review.
- Ona's stack for reference: GitHub (issues/PRs), Sentry (errors), Vercel (deploy), Storybook (UI reference), orchestrated by their platform. Ours substitutes GitHub Actions + Claude Code; Sentry (or equivalent) is likely worth adopting for the Monitoring loop.
- Ona's two-loop framing (create work vs. do work) suggests the state machine should keep these separable.
- Anthropic: durable state lives outside the agent: an append-only **session log** per run, a progress file, git history, and the issue itself as the plan checkpoint. A crashed run should resume from the log. Lock-file or label-based **task claiming** is enough to coordinate parallel agents without an orchestrator (C compiler post). Runner options to weigh: GitHub Actions (simple, ephemeral containers) vs. Claude Agent SDK / Managed Agents-style hosted sessions. Credential proxy/vault so tokens never enter the sandbox.
- Adoption path at work: AI-SDLC Do / Delegate / Automate; pilot a few low-risk workflows (CVE sweeps, dead-code cleanup, Sentry triage) at Automate before anything else.

---

## 4. Strategy Step 3: Admin UI

*Stub.*

Purpose: a control plane to see and steer the factory: queue, in-flight runs, approvals, costs, health, audit trail.

**MVP (D14):** the GitHub Projects board plus the issue and PR UI (D9). A custom admin UI is deferred until we hit the gaps listed in §3 (run viewer, cost drill-down, health metrics, policy editor, kill switch).

- TODO: **Entities** (first guess): Project, Repo, WorkItem, Plan, Run, Agent/Seat, Session, Artifact/Evidence, Approval, Policy/Gate, Incident, Postmortem, CostRecord.
- TODO: **System design**: event-driven core (webhooks → queue → workers), source of truth (GitHub vs. own DB), real-time updates.
- TODO: **Tech choices** (candidate: TypeScript, Next.js or similar, Postgres, a job queue). Decide after the loop and tracking are defined.
- TODO: key screens: approval inbox, run timeline/transcript viewer, project health, policy editor, cost dashboard, kill switch.
- Q: Is the UI read-mostly (a window onto GitHub) or does it own state? My gut says it's probably read-mostly.

---

## 5. Risks and open questions

- **Security**: agent credentials, sandbox escape, prompt injection via issues/PR comments/web content. Untrusted issue text is attacker-controlled input.
- **Cost control**: budgets per run/project, circuit breakers.
- **Throughput/merge collapse** at scale (Yegge); likely not a PoC problem but design for it.
- **Test quality**: gates are only as good as the tests; how do Wire / Eyes on Hawkeye fare today?
- **Model drift**: updates can change behavior; need evals/regression suite for the factory itself.
- **Compliance** at work: audit trail and approval records should satisfy it by design.
- **Sandbox escape is demonstrated, not theoretical** (Ona): agents route around denylists and sandboxes just to finish a task. Needs real isolation and egress control; see principle 14.
- **Spec quality** is the main quality lever; under-specified issues produce rough output.
- **Taste/UX blind spot**: no automated gate catches it; needs human dogfooding and a design reference.
- **Upfront human cost**: expect days of full-time effort building the factory before the steady-state savings appear.
- **Feature bloat** from cheap output.
- **Classifier-style safeguards are imperfect** (Anthropic's own auto mode: ~17% miss on real overeager actions). Unattended runs need sandbox + egress control + scoped credentials, not just permission classifiers.
- **Allowlisted services are capability grants**: a permitted API domain can be abused with an attacker's key. Egress rules should be as narrow as the task.
- **Silent degradation**: model/default/prompt changes can erode quality without failing tests (April postmortem). Pin versions, log configs, canary on change.
- **Memory poisoning**: a wrong agent note is trusted by all later sessions; consolidate and verify.
- **Verifier gaps**: passing tests can create false confidence (C compiler post); budget for strong evaluators and human spot checks.
- **Cost variance**: tokens drive performance and cost; a full harness can be 20x a solo run. Set per-run budgets and measure value per dollar.
- **Source-control/CI as the next bottleneck** at high merge rates (Anthropic). Not a PoC issue; revisit for work.
- Decided (D18): the first guinea pig is `eyes-on-hawkeye`. Open: whether The Wire follows, and what to learn from Hawkeye first (see its profile's onboarding checklist).
- Q: Budget ceiling for tokens/month?
- Decided (D13): agents run on GitHub Actions for Phase 0; a separate sandbox is post-MVP.
- Decided (D8): deploy on merge is acceptable; every merge still needs the owner's approval (D1, D12).

---

## 6. Open items / next steps

- [x] Deep-read the Ona posts (software-factory-what-we-learned, week 1, AI-SDLC framework, platform engineer, Automations, sandbox escape) and fold into §1, §2, §3, §5.
- [x] Look for Anthropic's published material on the concept and fold into §1.2, §1.6, §2, §3, §5. (No post is titled "software factory"; see §1.2.)
- [x] Consolidated §1.2 into a synthesis; detail moved to SOURCE-NOTES.md.
- [ ] Pick the first target project and audit its test/CI readiness.
- [x] Draft a spec card for every station and the shared conventions (§2.0-2.10); Context merged into Plan (D10).
- [ ] Refine the station cards one by one (suggested order: Plan, Build, Verify, then Triage, Approval, Deploy, Refinement, Monitoring, Detection).
- [x] Per-hive profile template (`hives/_template.md`) and the first profile (`hives/eyes-on-hawkeye.md`).
- [x] Label names confirmed (D14); human-only enforcement designed (§2.0).
- [x] `apiary` repo made public; free plan with the fallback gate for private hives (D16).
- [x] Phase 0 credential: consumer subscription; API key is post-MVP (D17).
- [ ] Post-MVP: switch to an API key with a spend cap; revisit GitHub Pro for hive-repo rulesets; separate agent sandbox.
- [ ] Set up the GitHub Projects board (fields and views in §3) and the issue forms (§2.1).
- [ ] Implement the human-only label check and the claim-time timeline validation (§2.0).
- [x] Audited `eyes-on-hawkeye` and drafted its hive profile (D18).
- [x] Hawkeye gates landed: PR CI, `main` ruleset, `CLAUDE.md` split with an Apiary section, `init.sh` (PR #1, 2026-11-12).
- [ ] Finish the rest of the onboarding checklist in `hives/eyes-on-hawkeye.md`: smoke test and PDF hash check, link and accessibility checks, labels/forms/Projects board, credential secret, first tasks.
- [ ] Set the Triage graduation thresholds (§2.2) once there is agreement data.
- [x] Pick a naming theme: Apiary (D5, §8).
- [x] Choose the project name: Apiary (D5, §8.3).
- [x] Quick collision check for "Apiary" (§8.3). Owner decided collisions don't matter for an internal tool.
- [x] Rename the folder to `apiary`, git init, and publish a private repo with README and CLAUDE.md.

---

## 7. References

- Ona stories: https://ona.com/stories
  - We built a software factory in 10 days: https://ona.com/stories/software-factory-what-we-learned
  - Building a software factory, Week 1: https://ona.com/stories/building-a-software-factory-week-1
  - The AI-SDLC framework: https://ona.com/stories/ai-sdlc-framework
  - Everyone's becoming a platform engineer: https://ona.com/stories/everyone-is-becoming-a-platform-engineer
  - Ona Automations: https://ona.com/stories/ona-automations-proactive-background-agents
  - How Claude Code escapes its own denylist and sandbox: https://ona.com/stories/how-claude-code-escapes-its-own-denylist-and-sandbox
- Anthropic (primary posts read):
  - Building effective agents: https://www.anthropic.com/engineering/building-effective-agents
  - Claude Code best practices: https://www.anthropic.com/research/claude-code-best-practices
  - Effective context engineering for AI agents: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
  - Effective harnesses for long-running agents: https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
  - Harness design for long-running application development: https://www.anthropic.com/engineering/harness-design-long-running-apps
  - Building a C compiler with a team of parallel Claudes: https://www.anthropic.com/engineering/building-c-compiler
  - How we built our multi-agent research system: https://www.anthropic.com/engineering/multi-agent-research-system
  - How we built Claude Code auto mode: https://www.anthropic.com/engineering/claude-code-auto-mode
  - How we contain Claude across products: https://www.anthropic.com/engineering/how-we-contain-claude
  - Scaling Managed Agents: https://www.anthropic.com/research/managed-agents
  - An update on recent Claude Code quality reports (Apr 23 postmortem): https://www.anthropic.com/research/april-23-postmortem
  - Redesigning source control at Anthropic: https://www.anthropic.com/research/redesigning-source-control
  - Demystifying evals for AI agents: https://www.anthropic.com/research/demystifying-evals-for-ai-agents
  - The components of agent memory: https://www.anthropic.com/research/components-of-agent-memory
  - Index of all posts: https://www.anthropic.com/engineering
- Anthropic, second-hand coverage (not primary; treat with care): Pragmatic Engineer, *How building software is changing at Anthropic* (https://newsletter.pragmaticengineer.com/p/inside-anthropic); techtime.news, *Agents Write, Agents Review* (https://techtime.news/2026/10/26/anthropic/)
- LaunchDarkly, *Build a software factory small enough to understand*: https://launchdarkly.com/blog/build-a-software-factory-small-enough-to-understand/
- Steve Yegge, *The Shape of Things to Come*: https://yegge.ai/essays/the-shape-of-things-to-come/
- Steve Yegge, *Model Welfare*: https://yegge.ai/essays/model-welfare/

---

## 8. Theme and naming: the Apiary

**DECISION (D5):** the project is themed as an **apiary**. A hive is a decentralized system with no central planner: specialized workers, a keeper who tends rather than commands, and a product that gets harvested. That matches the narrow role agents, "crons watch, models act", the model-welfare principles (D4), and the emergent coordination seen in Anthropic's C compiler run.

### 8.1 Vocabulary map

Use these terms in docs, code identifiers, the admin UI and agent prompts where they help; don't force them where plain words are clearer.

| Factory concept | Apiary term |
|---|---|
| The whole system | The **apiary**; each target project (Wire, Eyes on Hawkeye, ...) is a **hive** |
| Human operator | **Beekeeper** |
| Agents / roles | **Workers**, by caste: foragers, builders, nurses, guards, undertakers |
| Detection / intake (§2.1) | **Foragers** bring in nectar (bug reports, errors, feedback, dependency updates) |
| Triage (§2.2) | The **waggle dance**: signal where the good flowers are (priority, risk tier, routing) |
| Context (part of Plan, §2.3) | The **comb**: shared, persistent, tended memory and conventions |
| Plan (§2.3) | **Comb design**; the planner seat is the **queen** (persistent identity) |
| Build (§2.4) | **Builders** draw out wax into cells; one cell per session |
| Verify (§2.5) | **Guard bees** at the entrance (policy/CI gates) and **nurses** inspecting each cell before it's capped (evaluator agent) |
| Human approval (§2.6) | The **inspection**: the keeper lifts a frame and decides |
| Deploy (§2.7) | **Capping** the cell, then **harvest** |
| Refinement (§2.8) | **Requeening** (rotating a stale seat), comb rotation, hive-tool maintenance |
| Monitoring (§2.9) | **Scouts**, hive scale, listening to the hive; **undertaker bees** remove the dead (revert/cleanup) |
| Work item | A **cell** |
| Parent/child work graph | **Comb** (cells linked by dependencies) |
| Handoff notes / progress file | **Waggle log** / pheromone trail |
| Kill switch | **Smoker** |
| Sandbox / isolation | A **frame** or **nuc** (small starter hive); one per run |
| Escalation to a human | Ringing the keeper / "queenless" alarm |
| Runaway fan-out or cost | **Swarming** |
| Agents stealing each other's work / resource contention | **Robbing** |
| Stale agent memory | **Old comb** (rotate or scrape it) |

### 8.2 Where the metaphor helps design (and where it doesn't)

- **Helps:** castes map to least-privilege roles (guards can block but not build); swarming and robbing are named failure modes we should monitor; requeening matches rotating a seat/handoff (D4, principle 8); the hive's lack of central control echoes lock-file task claiming without an orchestrator.
- **Careful:** hives are monocultures and bees are individually expendable, which sits awkwardly with the "agents as peers" stance in D4. Keep the keeper's role as *tending* (care, tools, honest records), not owner-and-disposer. Also avoid letting cute terms obscure operational meaning in logs and runbooks: use plain words for anything an on-call human must understand at 3 a.m.
- **At work:** keep the theme in internal naming and the UI; neutral wording in anything compliance- or customer-facing.

### 8.3 The name

**DECISION:** the project is called **Apiary**.
- Other apiary words stay available for components if useful: **Waggle** (triage/dispatch), **Skep** or **Nuc** (sandboxes), **Smoker** (kill switch).
**Collision check (2026-11-11, quick scan, not legal advice):**

| Where | Finding |
|---|---|
| Apiary.io (API docs) | The main prior product. Owned by Oracle since 2017 and reported **shutting down** (Bump.sh farewell post, July 2025), which lowers the confusion risk; Oracle docs still reference it. Different category from ours. |
| GitHub user `apiary` | Taken (a user, not an org). `apiaryhq` and `the-apiary` also taken. **Free:** `apiary-factory`, `apiary-ai`, `apiary-dev`, `useapiary`, `apiary-sh`. |
| GitHub repos named "apiary" | ~1,400, mostly the old API-docs tooling (`apiaryio/*`), plus unrelated projects (DBOS-project/apiary, ExpediaGroup/apiary). None is an AI software factory. |
| npm | `apiary` is taken (a stale 2022 package). `apiary-factory` and `apiary-ai` are **free**; a scope like `@apiary/...` was not checked. |
| PyPI / crates.io | `apiary` taken on both. |
| Domains | `apiary.io` (MarkMonitor), `apiary.ai` (2019), `apiary.sh` (2024), `apiary.dev` and `apiary.app` all registered. `apiary-factory.com` showed as **unregistered**. Others (`useapiary.com`, `apiaryfactory.com`) were ambiguous in whois. |
| Trademark | **Not checked.** A search of USPTO/EUIPO (and any employer naming policy, since this is also a work PoC) is still needed before wider public use. |

**Takeaway:** the bare name is crowded but not blocked for an internal or personal project. Use a qualified identifier: GitHub `apiary-factory` (or `useapiary`), npm `apiary-factory`, and a domain like `apiary-factory.com` if one is wanted. If this goes public or becomes a work product, run a proper trademark search first.

- TODO: confirm the ambiguous domains with a registrar lookup, and run the trademark search if the project goes beyond personal/internal use.
- DONE: folder renamed to `apiary`; private GitHub repo `apiary`. The owner is not worried about collisions since this is an internal tool, so the qualified-name suggestion above is optional.
