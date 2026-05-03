# Roadmap

Living doc — updated as direction evolves. The active design plan for short-term work lives at `~/.claude/plans/zazzy-imagining-glade.md` (web-side, multi-phase).

## Shipped

### Web (`foreignfm-website`)
- Multi-show platform foundation (generalized from SMR-only)
- Show globe with `globe.gl` hex polygons + per-show HTML pins
- Deck listening experience: 4 view modes (poster / vinyl / disc / iPod)
- CornerPlayer with custom transport over hidden SoundCloud iframe
- Mobile chrome stack (EpisodeHeading, MobileBottomBar, DeckScrollIndicator)
- Auth foundation (Supabase Auth + profiles + handle/display_name onboarding)
- TunedRail Phase 0 (auth + frosted right-edge rail; static content)
- **Customize Module v1** — Design tab fully wired (theme, globe color, font scheme, logo variant, starfield variant), Music tab as honest preview frame, profile sync + localStorage fallback

### Backend
- 8 migrations through `0008_profile_customize.sql`
- SoundCloud ingestion pipeline (`scripts/ingest-shows.mjs`)
- Per-show episode catalog with composite uniqueness

## In progress / next

### Customize Module PR 2 — Music Tab v1 (web)
Wireable controls only (no editorial data needed):
- Region filter (uses existing `shows.country`)
- Likes table (`user_likes`) + button on CornerPlayer
- Hide-heard table (`playback_completions`) + filter on TunedRail
- Migration `0009_user_signals.sql`

### Editorial / Research Pipeline (cross-cutting)
The "real algorithm needs real data" workstream:
- Schema extension: `shows` gets `genres[]`, `moods[]`, `era`, `languages[]`, `vibe_tags[]`, `editor_notes`; `episodes` gets `tracklist[]`, `featured_artists[]`, `recorded_at`
- Long-form per-show editorial pages at `/[show]/about`
- Semi-automated research script: SoundCloud + LLM extraction → human edit → Supabase
- Once data exists: weighted scorer over (region, mood, era, language, similar-to-liked) → personalized TunedRail re-ranking

This is the prerequisite for the gated controls (mood, era, language, "more like this") in the Customize Module's Music tab.

## Planned (no committed timeline)

### iOS app (`foreignfm-ios`)
- Native SwiftUI app
- AirPlay + lock-screen + CarPlay controls
- Push for new chapters from followed shows
- Reads same Supabase project; same profile; same customize prefs
- TestFlight first

### Shopify (`foreignfm-shop`)
- Event tickets (in-person listening rooms)
- Vinyl pressings, apparel, prints
- Either Liquid theme or Hydrogen — decide based on whether we want React parity with web

### Cross-cutting
- Per-user follows (`user_follows`) — follow shows / artists
- Email digest of new chapters from followed shows (Supabase Edge Functions + Resend)
- Public profile pages at `/u/<handle>` showing taste, recent listens, follows

## Deferred / parking lot

- White-label show portals (each show gets its own subdomain that mounts ShowDeckPage)
- Live broadcast schedule integration
- Embedded player widget for shows to embed on their own sites
