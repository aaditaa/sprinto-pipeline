# Architecture overview

This is an independent copy of the Sprinto cold-outreach pipeline, built for MESA PGP
Term 2 — AI Case 2 ("Cold Outreach, at Scale"). It keeps the original two-part design:
a Python pipeline that does the research and decision-making, and a Next.js dashboard
that gives a human the two approval gates the case requires.

```
Google News RSS ──► Stage 1: regex gates + rank (no LLM, no scraping) ──► ranked pool in Neon
                                                                              │
             ┌──────────────── batch runner: next 20 unseen by rank ◄─────────┘
             ▼
  resolve domain ─► crawl (homepage + 2 pages) ─► Gemini extract (quotes verified in code)
             ─► Python gates ─► agent loop: fetch the page most likely to answer what's unknown (≤3)
             ─► dropped │ flagged │ qualified(+review)
                                      │
                    Hunter PoC (1 search) ─► Gemini writer ─► code validator ─► draft in Neon
                                                                                     │
  Web dashboard (Next.js on Vercel): clear flags, switch hook, edit, rate, Approve & send ─► Resend ─► MY_EMAIL only
                                                                                     │
  Feedback loop: every edit / rating / override / outcome ─► feedback_events ─► learn ─► learnings
                 ─► retrieved for the next drafts, hook choice and Stage 1 rank
```

**Governing principle:** Gemini only *extracts* facts (with verbatim quotes that Python
verifies against the source page) and *writes* copy. Every decision — qualify, drop,
pick this contact, pick this hook, send or hold — is made by plain Python rules against
thresholds in [`icp.json`](../icp.json), so it is auditable and unit-testable.

## Tech stack

| Layer | Technology | Why |
|---|---|---|
| Pipeline | Python 3.11+, `google-genai`, `psycopg`, `beautifulsoup4`, `feedparser`, `rapidfuzz` | Scripting + LLM calls + HTML parsing; no web framework needed server-side |
| LLM | Gemini (`gemini-3.8-flash` by default) | Structured JSON extraction and email drafting only — never the decision-maker |
| Contact discovery | Hunter.io Domain Search (mock fixture by default) | Case-study option A; free tier, cached per domain, never re-bought |
| Database | Postgres (Neon serverless driver on the web side) | Single source of truth; every gate, fetch and send is a row, not a log line |
| Email | Resend | Chosen over raw Gmail API for simplicity; recipient is hardcoded to `MY_EMAIL`, matching the case's "send to yourself" requirement |
| Web dashboard | Next.js 16 (App Router, Turbopack), React 19, Tailwind 4, shadcn/ui, Magic UI, GSAP | Server actions read/write Neon directly; Vercel can't run Python, so research-heavy actions are queued for `main.py sync` |
| Automation | GitHub Actions (`run-domains.yml`, `weekly-run.yml`) | Lets the hosted dashboard kick off pipeline runs it cannot execute itself |
| Hosting | Vercel (web), Neon (db) | Matches the original deployment; this independent copy needs its own Vercel/Neon project (see [TESTING_RESULTS.md](TESTING_RESULTS.md)) |

## The 7 case steps, mapped to code

| # | Case requirement | Implementation |
|---|---|---|
| 1 | Scrape the company site | `pipeline/scraper.py`, `pipeline/http.py` (budgeted crawl: homepage + up to 5 more fetches) |
| 2 | Extract structured data | `pipeline/extract.py` + `pipeline/llm.py` (Gemini JSON, quote-verified) |
| 3 | Qualify against ICP, with an agent loop | `pipeline/qualify.py` (gates, fit score, ≤3 refetches chasing unknowns via `refetch_map`) |
| 4 | Find the PoC | `pipeline/poc.py` (Hunter Domain Search, stage-based title ranking) |
| 5 | Two pain points | `pipeline/write.py` (part of the single Gemini writer call) |
| 6 | Personalised draft with a hook | `pipeline/write.py` (evidence list + hook selection in Python, copy from Gemini, validated) |
| 7 | Send automatically | `pipeline/mailer.py` via Resend, recipient hardcoded to `MY_EMAIL` |

## Data model

Eight Postgres tables carry the full audit trail end to end:
`runs → companies → research → decisions → contacts → emails`, plus `api_usage` (every
external call) and the feedback-loop tables (`feedback_events`, `learnings`,
`learning_applications`). Schema in [`migrations/`](../migrations), applied by
`main.py migrate`. See the original README's "Database (Neon)" table for the full column
-level description — it did not need to change for this copy.

## Deployment topology

- **Pipeline**: runs locally (or in GitHub Actions) against Postgres; no server process.
- **Dashboard**: Next.js on Vercel, Root Directory `web`, reading Postgres directly via
  `@neondatabase/serverless` in server actions/routes. It cannot run the Python pipeline,
  so anything that needs crawling or an LLM call is written to `companies.queued_action`
  and picked up by the next `python main.py sync`.
- **Scheduled work**: `.github/workflows/weekly-run.yml` runs `migrate → stage1 → run`
  every Monday; `.github/workflows/run-domains.yml` is dispatched by the dashboard's
  "Run on domains" page.
