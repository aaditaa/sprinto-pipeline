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

## Live pipeline run (2026-11-19)

With real `GEMINI_API_KEY`, `HUNTER_API_KEY` (`HUNTER_MODE=live`), a dedicated Neon
database (`sprinto`, created fresh alongside the account's other, unrelated Postgres
database — see "Known limitations"), and `PIPELINE_ENV=prod`:

```
python main.py stage1   # 24 feeds, 668 items fetched, 373 unique, 197 pooled after gates
python main.py run      # Run 1: pool_size 197, batches_used 1, researched 14, qualified 5,
                         #        stop_reason target_reached, n_drafted 5, n_dropped 2, n_flagged 7
```

This matches the shape of the original build's own demo run almost exactly (their demo
notes record "pool 313 ... 14 researched, 5 of 5 qualified ... target_reached inside the
first batch") — strong evidence the ported logic behaves the same way against live data,
not just in unit tests.

What the gates actually did, end to end:
- **Dropped** Model ML and Yuno, both for `certifications: already holds SOC 2, ISO 27001 and GDPR` — correctly recognized as not needing Sprinto.
- **Flagged** 7 companies: 6 for unresolved domains (VitusCare, Olyv, Hyphenate, FinMaq, Cent, Lissun — all one-word/ambiguous names where none of the 5 TLD guesses matched), 1 (Ant International) for `is_saas` still unknown after the agent loop chased it through 3 refetches.
- **Qualified + drafted** 5: Griffin, Arva AI, Sprive, Pearler, CARPL.ai — each got a real Hunter-found PoC (e.g. Pearler → Nick Nicolaides, Co-Founder) and a Gemini-written, validator-passed draft on the first attempt.
- **Auto-send fired zero times.** All 5 qualified companies had an unknown/undated audit signal, so each was `QUALIFIED (+review)` and `auto_send.skip_needs_review` (in `icp.json`) correctly held every draft for a human instead of sending — exactly the documented behavior, and exactly why the original README calls this out under "Known limitations" (sites rarely date their own signals). **No email was auto-sent** — by design. To close out the case's "a personalised cold email lands in your inbox" requirement, one draft (CARPL.ai → Vidur Mahajan, CEO, chosen for the most specifically grounded hook — it names a real enterprise partner, DeepHealth) was approved and sent with `python main.py send 5`:

```
sent email 5 to <MY_EMAIL> (resend id 01a1f49e-d7cf-7d9e-afb4-cb5b636dbc0b, edit rate 0.000)
subject: [DRAFT for Vidur @ CARPL.ai] Series A and enterprise security reviews
```

The other 4 qualified drafts (Griffin, Arva AI, Sprive, Pearler) remain held in `emails.status = 'drafted'`, available for review/approval through the dashboard once it's running against this database.

## Dashboard UI validation (2026-11-19, local `npm run dev` against the real `sprinto` database)

Every number and status rendered matches the actual database state exactly — nothing
was mocked or faked:

| Page | Checked | Result |
|---|---|---|
| Overview | Stat strip, 7-step tracker, run picker | Pool 197, researched 14, qualified 5/5, flagged 7, dropped 2, Hunter left 45, step counts 8/8/5/5/5/5/1 — all correct |
| Run pipeline | "Web runs need one setting" warning | Correctly flags missing `GITHUB_DISPATCH_TOKEN`, matches the known limitation already documented |
| Review queue | Flagged company cards (VitusCare, Olyv, ...), Qualify/Drop/Research-domain actions | Evidence, quote, source link and the exact attempted TLDs all correct; action controls present |
| Drafts & sends | List + email-client editor | Lists exactly the 4 still-held drafts (CARPL.ai correctly dropped off the list after being sent); editor shows To/From/Subject/body correctly |
| Companies | Filterable/sortable table | CARPL.ai correctly shows status "Sent"; filter chips (Qualified/Drafted/Sent/Flagged/Dropped/Pooled) present |
| Company drill-down | Decision trail, tabs (Research/Contact/Emails/Feedback) | CARPL.ai's trail shows "Pass · Name parse" and "Pass · Funding stage" with verbatim quotes + source links, exactly as designed |
| Feedback loop | 6-stage diagram | Correctly shows `3/6 stages live`: Act 5, Signal 1, Capture 1, Apply/Retrieve/Store 0 ("wired, waiting for data") |
| Command palette (⌘K) | Open, type-ahead | Lists pages + companies with status (`carpl.ai · sent`, `pearler.com · drafted`, ...) |
| Dark mode | Theme toggle | Switches cleanly, contrast holds up, no broken styles |
| Mobile (375×812) | Hamburger nav, stat grid reflow | Drawer opens/closes correctly, 2-column stat grid, no horizontal overflow |
| Console | All of the above | **Zero errors or warnings** on any page |

## What still hasn't been validated, and why

| Not yet tested | Blocked on |
|---|---|
| A production Vercel deployment of the dashboard | A Vercel project connected to `aaditaa/sprinto-pipeline` (Root Directory `web`) with `web/.env.local`'s values set as its env vars — not done since it wasn't asked for; local `npm run dev` already proves the app works |
| Approving/sending the other 4 held drafts | Each is a deliberate send action; one (CARPL.ai) was sent on explicit approval, the other 4 are intentionally left for you to review |
| GitHub Actions-triggered runs from the dashboard | A fine-grained GitHub token (`GITHUB_DISPATCH_TOKEN`) on the new Vercel project, per the original README's "Run on domains from the web" section |
| Production deploy | A Vercel project connected to `aaditaa/sprinto-pipeline`, separate from the original `sprinto-outreach` deployment, which belongs to the original repo and is left untouched |

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
