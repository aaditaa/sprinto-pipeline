# Component inventory

Every component below was carried over as-is from the original build (no feature was
rewritten or dropped). "Reused" means the file is unchanged; this whole project is
reused by design per the case's component map ([components-map.html](components-map.html) /
[components-map.png](components-map.png)).

## Pipeline (`pipeline/`, Python)

| Module | Responsibility |
|---|---|
| `config.py` | Loads `.env` secrets and `icp.json` thresholds into typed config objects; nothing is hard-coded |
| `db.py` | Postgres connection helper + the plain-SQL migration runner (`main.py migrate`) |
| `gnews.py` | Stage 1 news source: decodes Google News RSS redirect links, extracts candidate company sites |
| `stage1.py` | Headline parsing, the 4 pass/fail gates, dedupe/cooldown, and the rank formula |
| `resolve.py` | Turns a company name into a domain (TLD guesses + funding-article link), with a same-name-company check |
| `http.py` | The single outbound `fetch()`: real user-agent, retry/backoff, on-disk dev cache, never-raise failures |
| `scraper.py` | Crawl budget (≤6 fetches/company), page-type routing via `icp.json`'s `page_keywords` |
| `ats.py` | Greenhouse/Lever/Ashby public JSON APIs for dated job postings when a site doesn't surface its careers page |
| `llm.py` | Gemini wrapper: structured JSON, dev-mode caching, retries, token usage logging |
| `extract.py` | Prompts Gemini for structured facts, then verifies every quote actually appears on its cited page |
| `qualify.py` | The 5 ICP gates, the agent loop (refetch on unknowns, ≤3 rounds), outcome decision, fit score |
| `poc.py` | One Hunter Domain Search per company, cached to disk + Neon, stage-based contact ranking |
| `write.py` | Builds the evidence list, picks the hook, makes the single Gemini writer call, runs the validator |
| `mailer.py` | Resend send with no `to` parameter exposed — mail can only reach `MY_EMAIL` |
| `batch.py` | The run loop: next 20 unseen companies, stop at 5 qualified or 3 batches, one bad company never halts the run |
| `feedback.py` | The 6-stage feedback loop: capture events, distil `learnings`, retrieve/apply them in later drafts and ranking |
| `actions.py` | Applies human decisions from the dashboard (qualify/drop/provide-domain/etc.) as audited `decisions` rows |
| `usage.py` | Logs every external call (Gemini/Hunter/News/Resend) to `api_usage`; never raises |

Entry points: `main.py` (CLI dispatcher), `dashboard.py` (Streamlit fallback UI).

## Web dashboard (`web/`, Next.js 16 / React 19 / Tailwind 4)

### Pages (`web/app/`)
| Route | Purpose |
|---|---|
| `/` (`page.tsx`) | Overview: stat strip, 7-step pipeline tracker, run measures, drop/flag reasons |
| `/run` | Start the pipeline on up to 3 domains via GitHub Actions; recent domain-run results |
| `/review` | Human gate 1 — qualify, approve hook, drop with reason, provide domain |
| `/drafts` | Human gate 2 — edit with live word count/edit-rate/grounding check, switch hook, rate, Approve & send |
| `/companies`, `/companies/[id]` | Search/filter the pool; full drill-down per company (gates, research, contact, emails, signals) |
| `/loop` | Feedback-loop diagram with live counts, edit-rate comparison, learnings table |
| `app/api/search/route.ts` | Search endpoint over Neon |
| `app/api/webhooks/resend/route.ts` | Signed Resend webhook (delivered/bounced/opened/clicked/complained/delayed) |

### Components (`web/components/`)
| File | Purpose |
|---|---|
| `shell/sidebar.tsx`, `shell/topbar.tsx`, `shell/command-menu.tsx`, `shell/theme-toggle.tsx` | App chrome: nav, Ctrl/⌘K command palette, light/dark toggle |
| `pipeline-tracker.tsx` | The animated 7-step tracker with human-gate markers |
| `decision-trail.tsx` | Per-company timeline of every gate/decision with its quote, URL and date |
| `draft-editor.tsx` | Email-client-style draft editor with hover evidence on every claim |
| `review-card.tsx` | Review-queue card: qualify/drop/provide-domain actions |
| `run-form.tsx`, `run-select.tsx` | "Run on domains" form and the run picker |
| `outcome-select.tsx` | Records a prospect's reply/meeting/no-response outcome |
| `learning-toggle.tsx` | Retire/re-activate a learned lesson |
| `loop-diagram.tsx` | The 6-stage feedback-loop diagram (Act→Signal→Capture→Store→Retrieve→Apply) |
| `tabs.tsx`, `common.tsx` | Shared layout/tab primitives |
| `brand/logo.tsx` | Original Sprinto-styled wordmark (not Sprinto's real logo) |
| `motion/reveal.tsx` | GSAP-driven reveal-on-scroll wrapper |
| `ui/*` | shadcn primitives (button, card, dialog, input, select, table, tabs, textarea, tooltip, badge, progress, separator, skeleton, sonner) plus Magic UI effects (animated-beam, animated-shiny-text, border-beam, magic-card, number-ticker, shimmer-button, blur-fade) |

### Library (`web/lib/`)
| File | Purpose |
|---|---|
| `db.ts` | Neon serverless client; throws fast if `DATABASE_URL` is unset |
| `queries.ts` | All read queries used by server components |
| `loop.ts` | Feedback-loop aggregate queries for the `/loop` page |
| `feedback.ts` | Writes dashboard-originated feedback events |
| `grounding.ts` | Client-side "every claim is grounded in evidence" check used live in the draft editor |
| `github.ts` | Dispatches the `run-domains.yml` workflow from the "Run on domains" page |
| `run-limits.ts` | Enforces the 3-domains/3-runs-per-day web-run caps |
| `utils.ts` | Misc helpers (`cn()` class merge, etc.) |

## Database (`migrations/`)
7 plain-SQL migrations build the 8-table schema described in
[ARCHITECTURE.md](ARCHITECTURE.md#data-model): `runs`, `companies`, `research`,
`decisions`, `contacts`, `emails`, `api_usage`, plus the feedback-loop tables added in
`005_feedback_loop.sql` and the queued-action/trigger-mode columns added in
`006_pipeline_triggers.sql` / `007_trigger_mode.sql`.
