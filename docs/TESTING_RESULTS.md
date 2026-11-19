# Testing & validation results

Honest status as of this independent copy's first setup. Nothing below is claimed as
working unless it was actually run.

## What was run, and passed

| Check | Command | Result |
|---|---|---|
| Python unit tests | `python -m pytest -q` | **117 passed**, 0 failed (stage1 regex/gates, quote verification, qualifier gates + fit score, PoC picking rules, email validator, feedback loop, auto-send/domain-list behavior) |
| Python CLI loads | `python main.py --help` | Works; all 11 subcommands registered |
| Python deps install | `pip install -r requirements.txt` | Clean install, no conflicts |
| Node deps install | `npm install` (in `web/`) | Clean install (550 packages; 10 moderate audit advisories in transitive deps, none in direct dependencies — see "Known limitations") |
| ESLint | `npm run lint` | **0 errors, 0 warnings** |
| TypeScript | compiled as part of `next build` | **0 type errors** |
| Production build | `next build` | **Fails with no env vars set** (by design — `lib/db.ts` throws immediately if `DATABASE_URL` is missing) and **succeeds** once `DATABASE_URL`/`RESEND_API_KEY`/`RESEND_FROM`/`MY_EMAIL` are set to any syntactically valid value — confirming this is a configuration requirement, not a code defect |

## What could not be validated yet, and why

These all require a credential or resource that only the account owner can supply —
nothing here was skipped for convenience:

| Not yet tested | Blocked on |
|---|---|
| A real pipeline run (`main.py stage1`, `main.py run`) | A `GEMINI_API_KEY` — extraction and writing are Gemini calls, there is no offline substitute for them |
| Dashboard pages that read real data (Overview, Companies, Drafts, Loop) | A real `DATABASE_URL`. This account's Neon is Vercel-managed, so a project can't be created through the Neon API directly (`create_project` returned `action restricted; reason: organization is managed by Vercel`) — it has to be created once from the Vercel dashboard's Storage tab |
| Every button/interaction, loading/empty/error states, responsive layout | Same — most interactions are server actions that touch Postgres; a handful of pure-UI states (command palette open/close, theme toggle, static layout at mobile/tablet/desktop widths) could still be spot-checked against a running `npm run dev` once a `DATABASE_URL` exists, even with an otherwise-empty database |
| Sending a real email | `RESEND_API_KEY` + `MY_EMAIL` (recipient is hardcoded to this address, never the prospect, by design) |
| Hunter live lookups | Optional — `HUNTER_MODE=mock` (the default) reads `fixtures/hunter_mock.json` and costs nothing; a `HUNTER_API_KEY` is only needed for `HUNTER_MODE=live` |
| GitHub Actions-triggered runs from the dashboard | A fine-grained GitHub token (`GITHUB_DISPATCH_TOKEN`) on the new Vercel project, per the original README's "Run on domains from the web" section |
| Production deploy | A Vercel project connected to `aaditaa/sprinto-pipeline` with Root Directory `web`, plus its own env vars (separate from the original `sprinto-outreach` deployment, which belongs to the original repo and is left untouched) |

## How to finish validation once credentials are available

```bash
# 1. Fill in the real values
nano .env               # GEMINI_API_KEY at minimum
nano web/.env.local      # DATABASE_URL, RESEND_API_KEY, RESEND_FROM, MY_EMAIL

# 2. Apply the schema to the real database
.venv/bin/python main.py migrate
.venv/bin/python main.py check      # sanity-checks all configured services

# 3. Exercise the pipeline without spending Hunter/Resend credits
.venv/bin/python main.py stage1               # news -> ranked pool (Gemini not yet called)
.venv/bin/python main.py run --target-qualified 1 --max-batches 1   # one small research run

# 4. Look at the dashboard against real data
cd web && npm run dev     # http://localhost:3000
```

## Known limitations (carried over + newly observed)

From the original build (still true here, unchanged):
- Domain resolution relies on the funding article and `name + TLD` guesses; anything else needs "Provide domain" in the dashboard.
- Company sites rarely date their own signals; most dated signals come from job boards, so many qualified companies land in "needs review".
- Funding evidence links point to Google News redirect URLs, not the publisher's page.
- The agent loop's last spare fetch often chases headcount, which sites rarely state.

Newly observed while setting up this independent copy:
- **Neon provisioning is account-gated.** This Vercel-managed account can't create a Neon project through the API; it must be created once via Vercel's Storage UI, then the connection string copied into `.env`/`web/.env.local`.
- **`npm audit` reports 10 moderate advisories**, all in transitive dependencies of the toolchain (not runtime code reachable from the app). Worth a `npm audit fix` pass before a real production deploy, but outside this case's scope.
- **The original README's "Live dashboard" link** (`sprinto-outreach-delta.vercel.app`) points at the original author's Vercel deployment, not this copy. This copy needs its own Vercel project before that line in the README is accurate here.
