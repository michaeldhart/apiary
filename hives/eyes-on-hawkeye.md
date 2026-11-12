# Hive profile: eyes-on-hawkeye

Status: **draft (audited, not yet onboarded)** · Last audited: 2026-11-12 · Owner: Michael Hart

## 1. Identity
- **Repo:** [michaeldhart/eyes-on-hawkeye](https://github.com/michaeldhart/eyes-on-hawkeye), **public**, default branch `main`, 38 commits at audit time, 0 open issues or PRs.
- **What it is:** a community-run, bilingual (English/Spanish) archive of public records about a proposed data center, at eyesonhawkeye.com. Documents (PDFs with OCR text), a client-side timeline, people/organization pages, and AI-written summaries that the site discloses. It is a **content and accountability site more than an app**; accuracy and editorial stance matter more than features.
- **Stack:** Jekyll 4 static site (Ruby 3.4 in CI), Python 3 helper scripts with no dependencies, client-side JS (timeline, Lunr search). GitHub Pages deploys from Actions.
- **How it ships today:** push to `main` triggers the Pages workflow and goes live in roughly 30-90 seconds. No staging.

## 2. Agent-readiness (PLAN.md §2.3 gate)
Measured on a fresh clone in a throwaway directory (2026-11-12):
- **Init:** `bundle install` (about 28 s cold, gems into a local path).
- **Checks:** `python3 scripts/check_people.py` and `python3 scripts/check_i18n.py` (each under 0.1 s, no dependencies).
- **Build:** `JEKYLL_ENV=production bundle exec jekyll build --baseurl ""` (about 1 s, 124 pages, **zero warnings**).
- **Time to green:** about 30 seconds. All six hosted PDFs match the `sha256` in their front matter (verified by a one-off script).
- **Extra tools, only for ingesting new city documents:** poppler (`pdftotext`, `pdfinfo`, `pdftoppm`), `ocrmypdf` and `tesseract`; not on a stock runner (installable with apt).

| Readiness item | Status | Detail |
|---|---|---|
| Fresh clone goes green | ✅ | About 30 s, no warnings. |
| CI on pull requests | ❌ | `pages.yml` runs only on push to `main` and manual dispatch, so nothing checks a PR before merge. |
| Branch ruleset on `main` | ❌ | Not protected. **Available because the repo is public** (the free-plan limit in D16 does not apply here). |
| Automated tests | ⚠ | Two registry/translation check scripts gate the deploy; there are no link, accessibility or PDF-hash checks in CI. The CLAUDE.md "zero axe violations" standard is audited by hand. |
| Behavioral/visual checks | ⚠ | Manual preview and an axe-core procedure in CLAUDE.md. |
| Smoke test after deploy | ❌ | None. |
| Init script | ⚠ | None; trivial to add (see checklist). |
| Context files | ⚠ | Strong content, but see §3 (size, conflicts). |
| Issue forms, labels, Projects board | ❌ | None yet. |
| Secrets | ✅ | None in the repo. |
| Deploy blocked by failing checks | ✅ | The people/i18n checks and the build run before the deploy job, so a failing check stops a bad deploy. |

## 3. Context
- **Files agents load:** `CLAUDE.md` (about 18.7 KB), `README.md` (8 KB), and one project skill, `.claude/skills/add-city-document` (a full ingest procedure, including OCR, privacy omissions, Spanish writing rules and a verification step).
- **Overrides needed to run under Apiary:** `CLAUDE.md` says "commit and push when asked" and that pushing to `main` deploys. Under Apiary, **agents never push to `main`**: they work on `apiary/*` branches and open PRs, and the owner merges. Add a short "Running under Apiary" section saying so.
- **Size:** `CLAUDE.md` mixes always-needed conventions with a long content-state table, per-document history and ingest narratives. Recommend splitting it into a short index (a few KB: voice rules, never-touch list, commands) plus deeper docs loaded on demand (editorial, Spanish style, accessibility, content state), following the small always-loaded-layer principle (§1.2).
- **Human decisions agents must not reverse:** names from the public records stay visible; personal emails, phone numbers, home addresses and IP addresses are omitted from OCR text (the PDF is untouched); the home-page overview wording was approved line by line; headline copy the owner edited stays as edited; summaries are AI-written and must keep the AI disclosure everywhere they appear; never invent a date (`undated: true`); a PDF is the record and must never be altered.

## 4. Risk rules (D6)
Tiers measure **blast radius, not file type**. Here prose is the sensitive part: the site makes factual statements about identifiable people, so "just content" is not `R0`.

| Path / pattern | Tier floor | Notes |
|---|---|---|
| `README.md` | R0 | Docs only. |
| `Gemfile`, `Gemfile.lock`, `robots.txt`, `sitemap.xml`, favicons, social cards | R1 | Build must pass. |
| `_layouts/**`, `_includes/**`, `assets/css/**`, `assets/js/**` (styling, markup) | R1 | Must keep zero accessibility violations and the AI-disclosure notes. |
| `_plugins/i18n.rb`, `assets/js/timeline.js`, search (`search.*`, `search.js`), `timeline.*`, `_data/types.yml`, `_data/i18n/*.yml` UI strings, `scripts/add_document.py`, `scripts/make_og_card.py` | R2 | Behavior changes; renaming a type slug means updating every record that uses it. |
| `_documents/**`, `_entries/**`, `_people/**`, `es/**` records, tags, roles, dates, quotes | **R3** | Claims about real people. A change that alters no claim, name, date, role or quote (for example a broken-link or formatting fix) may be proposed as R2 and confirmed by the owner. |
| `index.md`, `es/index.md` ("What the Records Show So Far") | **R3** | Wording was approved line by line; agents may only **propose** edits, listed for approval. |
| `about.md`, `es/about.md`, `contact.md`, `es/contact.md` | **R3** | Carries the AI, analytics and privacy disclosures. |
| `_config.yml` | **R3** | Analytics code, contact form, overview-date markers. |
| `.github/workflows/**`, `scripts/check_people.py`, `scripts/check_i18n.py`, `CLAUDE.md`, `.claude/**` | **R3** | These are the factory's gates and context (principle 4). |
| New files in `assets/documents/**` | **R3** | Only through the ingest skill/script; the hash is computed by the script. |

- **Human-only fields:** `es_reviewed: true` (a person's attestation that a Spanish text was checked), `sha256` (script-computed only), `overview_through` / `overview_reviewed` (set only together with an approved overview edit).
- **Never touch:** existing PDFs in `assets/documents/` (no edits, replacements or deletions), `_site/`, `.jekyll-cache/`, DNS and Pages settings.

## 5. Work types and Detection sources
| Work type | Source / trigger | Typical tier |
|---|---|---|
| **Ingest a new city document** (flagship) | A scheduled watcher compares the PDF links on the city's data-center information page with the `file:` entries in `_documents/` and opens an issue for each new one. Build follows the existing ingest skill. | R3 |
| Overview may be stale | `check_people.py` already warns when a record is newer than `overview_through`; turn that into an issue. The result is a proposal for the owner to approve. | R3 |
| Spanish gaps | `check_i18n.py` failures | follows the path rules |
| Bugs and accessibility regressions | Owner-filed issues; failed checks | R1-R2 |
| Features | RSS feed, corrections log, a switch to Pagefind if the search index grows (owner's open work list) | R2 |
| Dependency and Ruby bumps | Scheduled sweep | R1 |
| Link rot and PDF integrity | Scheduled checks (see §8) | R1 |

## 6. Verify (PLAN.md §2.5)
- **Required checks on every PR (to add as a PR workflow):** the two check scripts; production build with zero warnings; a **PDF hash check** (each hosted PDF matches its `sha256`); an **internal link check** (for example html-proofer; it would likely have caught the recent bug where timeline PDF links were cut off at the first hyphen); an **axe-core accessibility pass** over the main page types in both color schemes.
- **Evaluator (behavioral):** serve the built site and exercise the timeline (keyboard and click), search in both languages, a document page, a people page, the language switch, in light and dark.
- **Reviewer focus:** every factual statement attributed to who made it; quotes and dates exact and checked against the source page image; no new personal contact details in text; AI disclosure present in any new view that shows a summary; Spanish twin present and matching the Spanish style rules; no generated claim of fact beyond what the records show.

## 7. Deploy and rollback (PLAN.md §2.7)
- **Mechanism:** merge to `main` triggers `pages.yml` (checks, build, upload, deploy) in roughly 30-90 s. Deploy-on-merge is accepted (D8).
- **Blocking gates already in place:** the people and Spanish checks and the build.
- **Smoke test (to add, after deploy):** request `/`, `/es/`, `/timeline.json`, `/search.json`, one document page and its PDF; expect HTTP 200, valid JSON, the PDF's hash equal to the front matter, and the `hreflang` and AI-disclosure markers present.
- **Rollback:** a revert PR (about 1.5 minutes to redeploy). For a factual error about a person, treat the revert as urgent and notify the owner immediately; it still needs the owner's approval (D1).
- **Note:** the workflow cancels an in-progress deploy when a newer push arrives; this was seen once in recent runs, which is fine.

## 8. Monitoring (PLAN.md §2.9)
A static site has no server-side errors to catch, so signals are about availability, integrity and drift:
- **Deploy status:** a failed deploy run opens an issue.
- **Daily smoke and uptime check** (scheduled workflow) using the smoke test in §7.
- **Weekly link check** of external source links and the city page.
- **PDF integrity:** scheduled hash check of the live files against front matter.
- **Overview staleness** (existing warning) and the **count of unreviewed Spanish records** as drift signals.
- **City-page watcher** (Detection, §5).
- GoatCounter (cookieless) shows traffic but is not an error signal.
- **Watch windows:** R0-R1 1 hour (smoke test), R2 24 hours, R3 7 days. For R3 content the automated signals are weak, so the real check is the owner's re-read and corrections arriving through the site's contact form.
- **Human feedback channel:** the Contact page's form (corrections and tips).

## 9. Gates and permissions
- **Branch ruleset on `main` (to set; allowed because the repo is public):** require a pull request; require the PR checks to pass; require one approving review from someone other than the PR author (agent PRs come from a bot identity, so the owner can approve); block force pushes and deletions; keep admin bypass for the owner only, for hotfixes.
- **Repo settings:** enable "automatically delete head branches"; keep auto-merge **off** (D1).
- **Labels, issue forms, board:** none today. Add the `apiary:`, `risk:` and `type:` labels, the bug and change issue forms, and add the repo to the Apiary Projects board.
- **Secrets the factory needs:** the Claude credential for Phase 0 (D17) as a repo secret; none exists today. Workflows from forks do not receive it.
- **Who may open or label issues that trigger runs:** the repo is public and has issues enabled, so **strangers can open issues**. Detection and Triage must act only on issues authored by the owner, collaborators or the Apiary bot, and must treat all issue text as untrusted. Only the owner can apply `ready` and `plan-approved` (§2.0).

## 10. Run limits (initial guesses, calibrate with run data)
- Standard task: about 40 turns, 60-minute job timeout. Ingest task: about 150 turns, 180-minute job timeout (OCR plus many entries). One concurrent run per hive.
- Actions minutes are free here (public repo). Costs under D17 show up as subscription usage rather than dollars.

## 11. Onboarding checklist
Items marked **(R3)** touch gates or context, so the owner (with Claude, interactively) lands them first; the factory's own runs start only after the gates exist.
- [ ] Add a PR workflow that runs the checks and build without deploying **(R3)**.
- [ ] Set the branch ruleset on `main` and the repo settings above.
- [ ] Add the Apiary section to `CLAUDE.md` and split it into a short index plus on-demand docs **(R3)**.
- [ ] Add `init.sh` (bundle install, both checks, build).
- [ ] Add the smoke test and PDF hash check scripts, then the link check and axe-core pass **(R3, workflows)**.
- [ ] Create labels, issue forms and add the repo to the Projects board.
- [ ] Add the Claude credential secret (D17).
- [ ] Choose the first factory tasks. Suggested: a small R1 or R2 item (for example the RSS feed, R2) before any R3 content ingest; then the first ingest, with the owner approving every claim.
