# Status

**Phase:** 0 — Brainstorming (Goldfish review done; concepts_brief.md written; all decisions resolved)
**Feature:** client-studio-production

## Answers from owner
- Plan source: CLAUDE.md only (brief/prototype not provided)
- Users: worldwide; home region Mumbai (Supabase ap-south-1, Vercel bom1); edge cache for static
- Non-technical: both client brands and manufacturer staff
- Banned "AI look": gradients & glow, sparkle/bot icons. Hologram kept but rebuilt muted, no glow; Classic default
- Map: MapLibre GL + Protomaps (self-hosted OSM tiles file, no API key)
- Scale +1 yr: small (<50 clients, <500 users, uploads <10k rows)
- Budget: $0 until first paying client, then paid tiers per plan (Vercel Pro + Supabase Pro)
- Speed: "feels instant" — no hard number; skeletons, cache, optimistic UI
- Client home: orders + upload. Staff home: sales, requests, client management (conflicts with "one main action" — to resolve)
- MFA: staff via company SSO (MFA enforced by IdP); clients via authenticator app (TOTP)

## Deviations from plan (approved)
- Map engine: custom canvas/SVG → MapLibre + Protomaps (both styles kept)
