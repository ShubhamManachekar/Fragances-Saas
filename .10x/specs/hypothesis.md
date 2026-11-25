# Anchored hypothesis — Client Studio production hardening (input for Goldfish review)

"Client Studio" — a multi-tenant SaaS: fragrance client portal + manufacturer dashboard for one perfume manufacturer. Harden the existing CLAUDE.md plan into a production-ready, low-latency SaaS with a UI that non-technical users (both client brands and manufacturer staff) can use without training.

- Stack stays as planned: Next.js 15 App Router + TypeScript strict, Tailwind + shadcn/ui, TanStack Query/Table, zod shared schemas, Supabase (Postgres + RLS keyed on JWT client_id, Auth, Realtime, Storage, pg_cron), Vercel, GitHub Actions, Sentry.
- Hosting: database and server functions in Mumbai (Supabase ap-south-1, Vercel bom1). Users are worldwide; static assets and cacheable pages go through edge caching.
- Budget: $0 (free tiers, sample data only) until first paying client, then Vercel Pro + Supabase Pro on day one of real data.
- Scale one year out: small (<50 client companies, <500 users, uploads <10k rows).
- Speed goal: "feels instant", no hard number — skeleton screens, cached data, optimistic updates.
- Auth: manufacturer staff via company SSO (MFA enforced by their IdP); client users via email + authenticator app (TOTP) as mandatory MFA.
- Map: MapLibre GL + Protomaps (self-hosted OSM-based tiles file, no API key). Two styles: Classic (default) and muted Hologram in brand colours, no glow. Approved deviation from the plan's custom canvas/SVG map.
- Visual: ivory, burgundy, antique gold; Cormorant Garamond headings, Jost body. Banned "AI look": gradients, glows, glass blur, sparkle icons, robot/chat-bubble assistants.
- Home screens (resolving "one main action per screen" vs owner wanting several tasks): client home = one primary "Upload sales report" action, order status shown alongside read-only; staff home = one "Needs attention" list combining open client requests, client account issues, notable sales changes.
- Security rules from CLAUDE.md are non-negotiable.

Constraints: no timelines, plain English, no deviation from CLAUDE.md beyond the approved deviation above. The repo has no application code yet; brief.md and prototype.html do not exist.
