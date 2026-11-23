# UTFR Business Suite

A sponsorship CRM and business-operations tool built for **University of Toronto Formula Racing**, a student Formula SAE team. It replaced a sprawl of disconnected Google Sheets that the business team used to run sponsorship, finances, marketing and procurement.

**[▶ Open the live demo](https://sadekarroh.github.io/UTFR-Business-Suite/)**

> **This repository is a sanitized demo build.** There is no backend, no sign-in, and no real data — every company, person, figure and transaction is invented. See [What's different in this build](#whats-different-in-this-build).

---

## The problem

Sponsorship and business operations lived across many spreadsheets with no shared state: who owns which sponsor relationship, what each sponsor was promised and whether it was delivered, which parts still need buying, when expected revenue actually lands, and whether the team can afford a given purchase right now. Nothing was linked, nothing was auditable, and reconciling it was manual. This puts all of it behind one interface with a single source of truth and role-based access.

## What it does

**Sponsorship CRM** — Sponsor records across a 10-tier taxonomy with pipeline status, plus per-sponsor deliverables tracking: every item promised to a sponsor, with owner, due date and completion state, so the team can prove it honoured its contracts.

**Contacts & follow-ups** — Individual contacts linked to sponsor organisations, and dated follow-up actions so nothing goes stale silently.

**Deliverables dashboard** — A cross-sponsor rollup of every outstanding commitment.

**Needs list** — Procurement items moving through a real lifecycle (Not Started → Outreach → Sponsor Provided → Shipping → On Hold → Purchased → Received). Those seven domain statuses are mapped onto a four-state task board so procurement appears alongside all other work without engineering having to learn a second vocabulary.

**Cash-flow decision tool** — Not a budget breakdown. It answers *"can we buy this, and if not, when can we?"* Revenue lines carry a confirmation status and an expected arrival date, so the model knows how much is coming, *when*, and *how certain*. Three scenario modes (Confirmed only / + Promised / + Everything) stress-test a purchase against different levels of optimism. Enter a prospective purchase and it reports whether the team can cover it in a given month, and if not, finds the earliest month that works. It distinguishes "in the budget" from "on top of the budget" spend so planned purchases aren't double-counted, and separates *can we pay this month* from *does the season balance* — so a season-level shortfall doesn't veto every individual purchase.

**Social planner** — Calendar, list and board views of planned posts, with a request→claim workflow, and a subscribable iCalendar feed.

**Design request queue** — Intake and claiming for graphics work, unclaimed sorted to the top then by urgency, cross-linked with the social planner.

**Unified task board** — Pulls work from every module into one board with My Tasks, overdue and time-window filters. Cards open in place rather than navigating away.

**Email campaign tool** — Guide → Compose → Recipients → Preview → Send, designed so a non-technical team member can run a campaign with no terminal and no setup. Anything in `{curly braces}` is replaced with the matching spreadsheet column, with fuzzy column matching, `{token|fallback}` defaults, and an explicit warning listing any merge field that matches no column — so it can't silently ship a literal `{First name}` to 300 people. Recipients load from the CRM or an uploaded spreadsheet. Addresses are validated, unusable ones flagged and skipped.

Sends are **paced to avoid looking like bulk mail**: gaps between messages are drawn from a normal distribution (mean 5 min, σ 1.6 min, clamped to 45s–15min) using a Box–Muller transform, confined to weekday working hours. The distribution was verified empirically over 200,000 samples — measured mean 5.004 min, σ 1.598, 68.2% within 1σ, 95.3% within 2σ.

**Sponsor insights** — Charted portfolio analytics with a configurable sponsor set, exportable to PNG.

**Team access** — Three roles (Associate / Lead / Exec) with enforced permissions.

**Excel export** — Multi-sheet `.xlsx` with selectable scope.

---

## How it's built

**Front end.** One self-contained HTML file: ~7,800 lines, all markup, CSS and JavaScript. Vanilla JS — no framework, no build step, no bundler, no transpiler. ~420 functions. Hand-written CSS with a design-token system, light/dark themes, and a translucent "glass" surface treatment behind an `@supports` feature query so browsers without `backdrop-filter` get solid surfaces rather than broken ones. Accessibility gates honour `prefers-reduced-transparency` and `prefers-contrast`.

Two external libraries, both from a CDN: **SheetJS** for Excel export and **Chart.js** for analytics. Everything else — the calendar, kanban boards, modals, mail-merge engine, cash-flow model, iCalendar generator — is written from scratch.

**Back end (production, not in this repo).** Supabase / PostgreSQL: 13 tables, Supabase Auth, and Row Level Security on every table with `SECURITY DEFINER` helper functions implementing the three-role permission model *in the database* rather than in client code — the client is untrusted by design. Some deliberate choices there: the credentials table has INSERT and UPDATE policies but no SELECT policy, making it write-only from the browser; the secrets table has RLS enabled with zero policies, making it unreadable by any signed-in user and reachable only server-side. Two Deno edge functions (one for immediate campaign sends, one draining the scheduled queue) and two `pg_cron` jobs. A localStorage mirror keeps the app usable offline, with a background refresh syncing other people's edits.

## Engineering notes

**Testing.** 13 headless-browser harnesses driven by Playwright with the database layer mocked, covering modal behaviour, status mapping, the finance affordability model, the mail-merge engine, send pacing and scroll behaviour. A structural validation pass runs after every edit: it parses all four script blocks to catch syntax errors, then walks the DOM asserting tag balance and that all 14 panels sit at the correct nesting depth — which caught a stray closing tag that had silently pushed six panels out of the scroll container. Because the whole app is one large file, edits are applied by size-guarded scripts that assert the exact span being replaced and the exact occurrence count before writing anything, so a bad edit aborts instead of corrupting the file.

**A performance bug worth describing.** Scrolling had degraded to ~270 ms per frame — about 3.5 fps. I instrumented frame times under realistic data volumes and bisected by selectively disabling CSS properties. Three plausible hypotheses were ruled out with measurements first: `background-attachment: fixed` on the page gradient (273 ms vs 281 ms — no effect), the blur on the sticky topbar and sidebar (269.8 vs 269.6 — no effect), and scroll observers (there are none).

The actual cause: 13 modal overlays sit in the document permanently, hidden when closed with `opacity: 0`. Invisible — but not removed from painting, and each carried a full-viewport `backdrop-filter: blur(12px)`. Every frame, the browser was snapshotting the entire viewport and blurring it thirteen times over. Two CSS changes fixed it — `visibility: hidden` on closed overlays to take them out of paint, and scoping the blur to `.modal-overlay.open` so a closed modal can never request one.

| | before | after |
|---|---|---|
| avg frame | 270 ms (≈3.5 fps) | 16.5 ms (60 fps) |
| worst frame | 607 ms | 40 ms |

**Cost profiling.** Audited database size, per-sync payload and request volume against the hosting tier's limits. Found one table being read 8,211×/day versus ~3,100 for every other table — a status heartbeat fetching a whole row every 15 seconds just to light up a connection indicator. Cut to an id-only request.

---

## What's different in this build

This repository is the public demo. It differs from production in four ways:

1. **No backend.** The transport layer (`supa()`) implements the same interface as the Supabase REST layer, backed by `localStorage`. Every caller is unchanged — rows just come from your browser instead of PostgreSQL.
2. **No sign-in.** You land as an Exec so every capability is visible.
3. **No real data.** Every company, person, figure and transaction is invented.
4. **No email sending.** The merge fields, preview, validation and the normally-distributed scheduler all run; the SMTP transport and credential storage are removed.

Nothing here talks to a network except Google Fonts and the two CDN libraries. Changes you make save to your browser only; **Reset demo** in the user menu clears them.

## Running it

Download `index.html` and open it in a browser. That's the whole thing.

---

## Credits

Built and maintained by **Rohan Sadekar**. I specified the features, designed the workflows and data model, made the product decisions, supplied the data and iterated on it with the team.

Implementation was done with **AI-assisted development** (Claude) — code, refactoring, debugging, performance profiling and test authoring, written to my direction and specification.
