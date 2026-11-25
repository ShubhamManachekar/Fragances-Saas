# Client Studio — Production-Ready First Build

* **Author:** ShubhamManachekar
* **Date:** 2026-11-25
* **Status:** Brainstorm brief (input for `/egm:egm-draft`). Reviewed by 7 independent Goldfish lenses: design, technical accuracy, simplicity, comprehension, implementation readiness, structure, inventive.
* **Plan source:** `CLAUDE.md` only. `docs/brief.md` and `docs/prototype.html` are not in the repo, so references to "port from prototype" and "reuse CSS variables from the prototype" cannot be followed yet.

## Overview

Client Studio is a web app for one perfume manufacturer. It has two sides:

* **Client portal** — used by the manufacturer's client brands. They check their orders, upload sales reports, see their sales by region and perfume, and send requests for new fragrance ideas.
* **Manufacturer dashboard** — used by the manufacturer's own staff. They review every client's sales, answer client requests and manage client accounts.

This brief takes the existing plan in `CLAUDE.md` and makes it ready for real clients. It must be quick to use for users anywhere in the world, and simple enough that people with no technical background can use it without training. It keeps the planned stack and security rules. The only approved change to the plan is the map engine (see Key Decisions).

### Words used in this brief

| Word | Meaning |
| --- | --- |
| Manufacturer | The one perfume company that runs Client Studio. One install serves one manufacturer. There is no layer above it. |
| Client company (tenant) | A brand that buys from the manufacturer. Each has one `client_id`. All of its data is walled off from other client companies. |
| Client user | A person who works at a client company and signs in to the portal. Belongs to exactly one client company. |
| Staff | A person who works for the manufacturer. Has no `client_id`. Can see across all client companies through separate, explicit rules. |
| Sample data | Made-up data, labelled as sample, used until the first paying client. Never real client data. |

## Analysis

The reviewers agreed that the plan's direction holds, but found that it describes a whole product rather than one feature, and that several pieces it depends on are missing or will not work as written. The findings are grouped below, most serious first.

### 1. Things that will not work as written

* **Uploads larger than about 4.5 MB fail on Vercel.** Vercel server functions reject request bodies over about 4.5 MB, and Next.js Server Actions cap them at 1 MB by default. The plan allows 5 MB. Resolution: the browser uploads the file straight to Supabase Storage using a short-lived signed upload link. The storage bucket itself enforces the 5 MB limit and the allowed file types. A server step then reads the file from storage, checks its real type from its contents (not its name), and parses it with hard limits on rows, columns and time.
* **Company sign-in (SAML SSO) for staff is a paid Supabase feature.** It cannot be built or tested during the $0 phase. If the manufacturer uses Google Workspace or Microsoft 365, Supabase's free Google or Microsoft sign-in, restricted to the manufacturer's email domain, gives the same result: staff sign in with their work account, and their company's system enforces MFA.
* **MFA is not mandatory unless the app enforces it.** Supabase lets each user choose MFA. The app must check, on the server and in database rules, that client users have completed the second step (Supabase calls this level `aal2`). Staff signing in through Google/Microsoft show as level `aal1` in Supabase, so staff need their own rule based on how they signed in. Supabase has no built-in recovery codes.
* **`client_id` is not in the login token by default.** A Supabase "custom access token hook" (free) must add `client_id` and `role` to each token. Never store these in `user_metadata`, because users can edit it. Tokens stay valid until they expire, so removing someone's access needs a short token lifetime plus ending their sessions.
* **The free tier allows two active projects.** Dev and staging use both. Production is a third project and must be on Pro from the start, never seeded with sample data.
* **Supabase's built-in email sender is heavily rate-limited** and, on Free, only sends to the project team. Client invitations and password resets need an outside email service. This is a new dependency not listed in the plan.
* **The `xlsx` package on npm is outdated and has known security flaws.** SheetJS must be installed from its own download site at a pinned version.
* **Next.js 15 has had serious security fixes**, including one where middleware sign-in checks could be skipped. Pin the latest patched 15.x. Never rely on middleware alone for access checks; the plan's rule "server checks role on every request" covers this and must be followed.

### 2. Missing pieces the screens depend on

* **Orders have no data source.** The client home shows order status, but the ProcessWare ERP link is not set up and there is no Order table. A source for orders is needed before that panel can show anything real.
* **The staff "Needs attention" list has no rules.** "Client account issues" and "notable sales changes" are not defined. Each item type needs: what puts it on the list, what takes it off, and who can clear it. Sales changes must use a fixed, explainable rule calculated in the database (for example "more than X% change from last period", X adjustable). No AI.
* **The sales upload flow is not defined.** Missing: required columns, what happens if the same report is uploaded twice, whether a corrected report replaces the old one, how row errors are shown, and whether matched columns are remembered per client.

### 3. Speed for worldwide users (the main trade-off)

A round trip from Mumbai to Europe or the US takes roughly 150–300 ms. That cannot be removed, only hidden. Almost every screen shows one client company's private data, so it cannot be stored in a shared edge cache. Doing so would risk showing one client's data to another.

The approach that hides the delay:

* Only public files go through the edge cache: the app's code, fonts, map files and the sign-in page. Every signed-in response is marked private and never stored in a shared cache.
* One trip to Mumbai per screen. The browser talks to the Next.js server in Mumbai, which talks to the database in the same region.
* The browser keeps recently loaded data (TanStack Query) and shows it at once, then quietly refreshes it, labelled "updated X ago". This cache is keyed by user and wiped on sign-out.
* Data for the next likely screen is loaded on hover or focus.
* Dashboard totals are worked out ahead of time by scheduled database jobs, so a screen needs one small request instead of many.
* Showing a result before the server confirms it ("optimistic update") is allowed only for small, reversible actions such as marking an item done. Uploads and anything written to the audit log show success only after the server confirms.
* Upload preview happens in the browser straight away. Saving happens on the server.

### 4. Non-technical users

* **One main action per screen, chosen by what is due.** The main button keeps the same position and its label changes with the situation. Client home: "Upload sales report" when a report is due, otherwise order status with nothing pressing. Staff home: the "Needs attention" list, sorted by a fixed urgency rule.
* **MFA setup and lost phones.** Staff invite each client user. The first sign-in walks them through authenticator setup on one guided screen. A long "remember this device" period means the second step is rarely asked for again. If a phone is lost, staff reset the user's MFA after checking who they are. The reset is written to the audit log and the user is emailed.
* **Errors in plain language** that say what is wrong and how to fix it, for example: "Column 'Qty' was not recognised. Choose which field it is."
* **A test for "no training":** a non-technical tester completes "upload a sales report" and "clear a Needs attention item" with no help.
* **Accessibility:** WCAG 2.2 AA, which includes the plan's 44 px targets and visible focus.

### 5. Visual design

The plan's fonts (Cormorant Garamond, Jost) and colours (ivory, burgundy, antique gold) are already outside the usual AI-product defaults. Banned: gradients, glows, glass blur, sparkle icons, robot or chat-bubble assistants. Any AI-assisted step (later column matching) is labelled in plain words ("Suggested match"), with no special visual effects. The muted Hologram style must be written as exact colour and opacity values with no blur or shadow, so it can be checked against the ban. Dark-mode values for the palette must be defined.

### 6. Map

Use MapLibre GL with OpenStreetMap-based data (Protomaps), self-hosted, no API key. Reviewers found:

* A full-world, street-level tiles file is about 100 GB. Sales are shown by country or region, so a low-zoom extract (a few tens of MB, made with `pmtiles extract --maxzoom`) is enough. Zoom is capped to match.
* Protomaps borders are lines, not shapes that can be filled with colour. Shading regions by sales needs a separate small shapes file (Natural Earth, under 1 MB) whose IDs match the Region table.
* The tile host must support partial-file (range) requests. Test this on Vercel static hosting and Supabase Storage before choosing.
* Fonts and icons for the map must also be self-hosted. The OpenStreetMap credit must be shown. MapLibre is a large library, so load it only on screens with a map. It needs WebGL, so plan a simple fallback.
* Classic and Hologram are two colour themes over the same map, not two separate builds.

### 7. Security and privacy

* **Cross-client insights can leak one client's numbers.** With fewer than 50 clients, "at least 5 sharing" is not enough on its own. A client can subtract its own figures from a total, or compare results across filters. If the feature is built, it needs: totals worked out ahead of time on a fixed set of results; at least N contributors not counting the client asking; no single contributor above a set share of any figure; no free-form filters; one narrow database function as the only thing that reads across clients. When too few clients share, show the client a comparison against its own history instead of an empty panel. At this scale the feature will mostly have too few sharers to show anything.
* **The service key (which skips all database rules) is never used for user requests.** Only scheduled jobs may use it. A CI check blocks it from browser code. PR previews connect only to dev.
* **Audit log:** remove update and delete rights for every app role and add a trigger that blocks changes. The database owner can still change rows; record who holds that access. Audited events: sign-in, MFA reset, upload, column-mapping change, sharing change, AI switch change, role change.
* **Sentry** must strip request bodies, upload contents and emails, and tag errors with `client_id` and role only.
* **Data location:** all data is stored in India (Mumbai). This must be accepted in writing for client contracts.
* **Signed links** skip database rules once issued, so they need a short, stated expiry.

### 8. Operations

* **Production is a separate Pro project**, created from scripts in the repo, never upgraded from dev. All schema, security rules, functions and scheduled jobs live as migration files in git. Only CI applies them: staging first, then production after manual approval. Changes only go forward; the previous app version must keep working against the new schema.
* **Rebuild staging from scratch regularly** to rehearse the switch to production. Automated tests check tenant isolation on every migration: client A cannot read client B; a missing claim means no access; another client's record returns "not found".
* **Backups:** the nightly `supabase db dump` exports schema only by default and skips stored files. It is fine for sample data but must stop before real data, because dumps in GitHub would hold client data. For production: Pro daily backups, a separate encrypted backup of uploaded files, a stated acceptable data loss and restore time, and a recorded restore drill.
* **Monitoring:** external uptime checks; every scheduled job records its runs and alerts on failure or a missed run; one request ID passed through browser, server, Sentry and the Upload/AuditLog rows; Upload rows carry status (received, parsed, matched, failed), a plain-English failure reason and row count. Free-tier logs are kept only briefly, so anything needed to investigate "my upload failed yesterday" goes into Postgres.
* **A short runbook:** outage, suspected tenant leak, bad migration, restore. One named incident owner.

### Ideal Final Result

Each user opens a screen and the one right next step is already shown, with its data already on their device. The database enforces tenant privacy in one place. The whole environment can be rebuilt from the repo at any time, at $0 until real data arrives, and going live is a script run plus a billing change.

## Goals vs. Non-Goals

* **In-Scope Goals (first build):**
  * Tenancy and data model: core tables plus Order, `client_id` on every tenant table, staff role model, deny-by-default RLS, "not found" rule, immutable audit log.
  * Sign-in: client users (email + mandatory authenticator app, staff-run MFA reset); staff (work account sign-in, restricted to the manufacturer's domain).
  * Environments: dev and staging on Free, production on Pro, all built from repo scripts; caching rules; backups; monitoring; free-to-paid switch checklist.
  * Design system: colour and font tokens (light and dark), components, banned list, accessibility rules.
  * Sales upload: CSV and XLSX parsed and saved; PDF accepted and stored for staff to look at.
  * Order table and staff order entry (dev only); client Orders panel hidden at first launch until the CRM is connected.
  * Olfactory graph, professional style.
  * Staff "Needs attention" list: open requests and account issues.
  * Client requests (ideas) and client management for staff.
  * Map: MapLibre, Classic and muted Hologram themes, regions shaded by sales.
* **Out-of-Scope Non-Goals (later features, data model kept ready):**
  * CRM order sync, ProcessWare ERP sync, and with them the Realtime live order feed (refresh on focus or on a timer until then).
  * Reading tables out of PDFs.
  * AI (Claude) column matching. The per-client AI switch column exists but is off.
  * Cross-client insights. The `share` column exists.
  * "Notable sales changes" in the Needs attention list.
  * SMS codes, SAML SSO, more than one manufacturer, users belonging to more than one client company.

## Alternatives Weighed

* **Hosting in several regions to cut latency.** Rejected by author: cost and complexity at under 50 clients. Delay is hidden in the browser instead.
* **Edge-caching signed-in pages.** Rejected by reviewers: risk of showing one client's data to another.
* **Upload through a Vercel function.** Rejected: Vercel's ~4.5 MB body limit is below the plan's 5 MB.
* **SAML SSO for staff during development.** Rejected: paid feature, cannot be tested at $0. Work-account sign-in through Google/Microsoft used instead.
* **SMS codes for MFA.** Rejected by author: costs money per message and is weaker.
* **Leaflet with public OpenStreetMap tiles.** Rejected by author: OSM's free tile servers forbid heavy commercial use and look off-brand.
* **Full-world street-level Protomaps file.** Rejected by reviewers: about 100 GB, far beyond free storage. Low-zoom extract used instead.
* **Dropping the Hologram style.** Rejected by author: keep both, Hologram rebuilt muted with no glow.
* **Upgrading the dev project to Pro for production.** Rejected by reviewers: carries sample data, test users and loose settings into production.
* **Nightly database dumps to GitHub for production backups.** Rejected by reviewers: puts real client data into GitHub storage, and skips uploaded files.
* **AI scoring for the Needs attention list or sales changes.** Rejected: plan forbids the LLM from calculating insights.

## Key Decisions & Staging for Next Steps

**Decided:**

1. Stack as in `CLAUDE.md`, with Next.js pinned to the latest patched 15.x.
2. Home region Mumbai (Supabase ap-south-1, Vercel bom1). Data stored in India.
3. $0 with sample data only until the first paying client. Production on Pro before any real client data is uploaded.
4. Approved deviation: map engine is MapLibre + Protomaps low-zoom extract + Natural Earth region shapes, replacing the custom canvas/SVG map. Two themes: Classic (default) and muted Hologram.
5. Uploads go straight from browser to Supabase Storage through signed upload links.
6. `client_id` and `role` added to tokens by a custom access token hook. Client users must be at MFA level `aal2`.
7. Only public static files go through the edge cache.
8. Optimistic updates only for small, reversible actions.

9. Orders: a separate CRM (not ProcessWare) will supply orders later. The Order table is designed to match that future feed. During development the panel shows labelled sample data and staff can enter orders by hand. At first launch the Orders panel is hidden, so the client home shows only the "Upload sales report" action, until the CRM is connected.
10. Staff sign in with Google Workspace (free Supabase Google sign-in, restricted to the manufacturer's domain). Their Google account enforces MFA.
11. Email: Resend (new dependency, approved), with the manufacturer's sending domain set up with SPF/DKIM.
12. Olfactory graph is in the first build. Style: professional, not cartoonish — thin lines, muted brand colours, serif labels, no bubbly shapes, bright fills or animation for its own sake.
13. Production data loss target: minutes. Use Supabase point-in-time restore (paid add-on, adds to the day-one cost) plus a separate encrypted backup of uploaded files and a recorded restore drill.

**Open — none blocking `/egm:egm-draft`.** Remaining details (thresholds, signed-link expiry, row caps) are settled inside each sub-project spec.

**Suggested spec order** (one spec per sub-project, matching "one feature per session"):

0. Umbrella brief (this document, once approved).
1. Tenancy, roles and data model.
2. Sign-in (both paths), MFA setup and recovery.
3. Environments, hosting, caching, backups, monitoring (can run alongside 2).
4. Design system (can start alongside 1–3).
5. Sales upload.
6. Orders (Order table, staff entry, hidden client panel until CRM).
7. Staff "Needs attention" list, requests and client management.
8. Map, charts and olfactory graph.

## Update: product focus — oil-usage analytics

*Added after the owner clarified the core purpose. This section takes priority over earlier sections where they differ.*

### What the product is for

We supply fragrance oils to client brands that make finished goods such as deodorants, soaps, body washes and shampoos. Each client sends us a portfolio file in our standard format. It lists their products, which of our oils each product uses, where it sells and how many units sold. Client Studio combines every client's portfolio into one nameless market view. Each client then sees which oils are most used, which are rising, where their range differs from the market, and how scent families compare. Our team also features oils it wants to push: new inspirations and ready stock.

### Standard input file (one row per product, per city, per period)

| Column | Content | Example |
| --- | --- | --- |
| product_name | Client's product name | Fresh Deo |
| category | Deodorant, Soap, Body wash or Shampoo (list kept by staff) | Deodorant |
| oil_code | Our oil code from the catalog | CS-101 |
| market | City where it sells, from our city list (Mumbai, Delhi, Dubai, Riyadh, London, Paris, New York, Los Angeles, São Paulo, Lagos, Johannesburg, Singapore, Tokyo, Sydney; list kept by staff) | Dubai |
| launch_date | YYYY-MM | 2025-03 |
| period | Quarter, e.g. Q4 2026 | Q4 2026 |
| units_sold | Whole number; may be left empty if the client lets us fill it from our oil sales | 45000 |

Template: `docs/templates/client-portfolio-template.csv`. Column names may differ; the upload screen matches them. Rows with an unknown oil code, category, market or period are skipped and listed in plain language. Re-uploading a period replaces that period's figures for the matched products.

### Analytics shown to clients

1. **Most-used oils** — share of units by oil across all categories combined (no per-category filter on screen). Each oil shows how many brands use it and whether this client uses it.
2. **Rising oils** — biggest gains in share since the previous period.
3. **Gaps in your range** — popular oils in the client's own categories that the client does not use, plus featured oils. Each has an "Ask for a sample" action.
4. **Scent families** — the client's family mix next to the market's.

All figures are calculated in the database (SQL), never by an LLM.

### Privacy rule for the market view (replaces "≥ N sharing clients")

* Brand names are never shown to clients. Staff can see names.
* A figure is shown only when at least two brands are behind it, or when it is the client's own data. Oils and scent families used by only one other brand are listed last with no numbers, or left out of the family bar.
* Shares are worked out among visible items only, so a hidden brand's volume cannot be found by subtracting from 100%.
* Every client's products count towards the market view. There is no opt-out in the first build.
* **Residual risk:** with four clients, a client can still guess who is behind a figure from what they know of the market. Owners must accept this in client contracts.
* This replaces the `CLAUDE.md` rule "only when ≥ N share (≥ 5 in production)". **Needs owner confirmation**, and `CLAUDE.md` must be updated once confirmed.

### Our oil sales as a data source

A client may allow us to fill in their volumes from our own oil sales records. This is off by default, the client sets it on their Portfolio page, and every change goes into the audit log. Without it, only the numbers in the client's file are used.

### Featured oils

Staff mark any catalog oil as "New inspiration" or "Ready stock". Featured oils appear on every client's home page and in their gaps list. Staff also see which brands do not use each oil yet.

### Data model additions

`Oil` (code, name, family, notes), `Product` (client_id, name, category, oil_code, market, launch_date), `ProductPeriod` (product_id, period, units, source: client_file or our_sales), `FeaturedOil` (oil_code, tag), `Client.use_our_sales` (boolean). The earlier `Perfume`/`PerfumeNote` tables become `Oil`/`OilNote`.

### Graphs page

One page holds both visuals, each with a Hologram and a Classic style:

1. **City map** — the owner's dotted world map with 14 cities. Each city is a circle sized by units and coloured by its top scent family, with supply routes from our plant. It has a family filter, zoom and pan, and a side panel with the selected city's family mix and products.
2. **Olfactory graph** — products (or brands, for staff) in the centre, linked to their oil's notes on a ring grouped by scent family. Hover or click to highlight; it can be filtered by city.

The Hologram style uses neon colours, gradients and glow at the owner's request. This is a deliberate exception to the earlier "no AI look" rule.

### Palette update

The owner moved the whole app to a light, fresh mint-and-aqua palette, replacing ivory, burgundy and gold. Both Hologram styles are now light (ice and aqua, no glow). The tokens live in `docs/prototype.html` under `:root`.
