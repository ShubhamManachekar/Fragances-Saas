# cto decisions

## Stack (decided by project owner, see CLAUDE.md)
- Next.js 15 App Router + React + TypeScript strict; Tailwind + shadcn/ui; Recharts + custom canvas/SVG (map, olfactory graph)
- TanStack Query/Table, react-hook-form + zod shared client/server
- Supabase: Postgres, Auth (SSO + mandatory MFA), Realtime, Storage, pg_cron
- Vercel (Hobby for dev with sample data only; Pro before real clients); GitHub Actions; Sentry free
- ERP: ProcessWare read-only scheduled fetch (not set up)

## Principles
- Don't deviate from the owner's plan (`docs/brief.md`, `CLAUDE.md`) without explicit approval.
- No invented client data; all sample data labelled as sample.

## Active features
- client-studio-production (Phase 0)
