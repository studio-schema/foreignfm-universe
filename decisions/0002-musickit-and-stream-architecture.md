# ADR 0002 — MusicKit + Stream architecture

**Status**: Accepted · 2026-05-03

## Context

The Stream surface (web `/stream` + iOS Stream tab) needs to:

1. Browse and play a hybrid catalog — FOREIGN.FM-curated content (existing SoundCloud shows + future native uploads) **plus** the Apple Music catalog
2. Run a per-user, customizable recommendation algorithm — region, mood, era, language, similar-to-artist, discovery↔comfort. "This is where the money is."
3. Work on web + iOS with shared state (likes, watch history, customize prefs all sync via Supabase)

The framework question came up: do we need to migrate web off SvelteKit to do this? No — MusicKit JS works on web, MusicKit Swift on iOS, and SvelteKit handles the UI fine. The blockers are content + Apple Developer enrollment + the algorithm engine, not the framework.

## Decisions

### 1. Content sources: hybrid

| Source | Role | When playable |
|---|---|---|
| **SoundCloud** (existing) | The current FOREIGN.FM show catalog (Sasha Marie, etc.) | Always — via the SC widget |
| **Supabase Storage** (future) | FOREIGN.FM-native uploaded tracks (mixes, exclusive sets) | Always — via HTML5 `<audio>` |
| **Apple Music catalog** (via MusicKit JS / Swift) | Full Apple catalog browsing + search | 30s previews for everyone; full playback for users with their own Apple Music subscription |

The user can play any source as a single seamless experience; the algorithm scorer ranks all three sources with the same scoring function.

### 2. MusicKit setup — gated on Apple Developer enrollment

- Web: MusicKit JS v3, configured with a developer-token JWT signed server-side from `/api/musickit-token` (env vars `APPLE_TEAM_ID`, `APPLE_KEY_ID`, `APPLE_AUTH_KEY`). 6-month expiry; mint-per-request OK at small scale.
- iOS: MusicKit Swift, same developer-token + the user's MusicUserToken (captured via Apple's authorize flow).
- Personal Apple Developer account works for testing now; LLC enrollment swaps the env vars later (one rotation, no code change).

### 3. Algorithm engine — `for_you_feed` + scorer

Schema (web migration `0012_algorithm.sql`):

- `tracks` — unified row per playable song/episode (sources: foreignfm | apple | soundcloud) with editorial dimensions (genres[], moods[], era, region, language, bpm, energy, editorial_score)
- `listen_events` — per-user signal source (play/complete/skip/like/dislike), append-only
- `for_you_feed` — cached top-N output per user, regenerated on customize_music change OR nightly cron

Scorer (Supabase Edge Function, planned):

```
score = base_match (genre/mood/era/region match × pref weights from customize_music)
      + history_match (similar artists/genres to recently liked/completed)
      + editorial_bump (staff-tuned boost)
      + region_bonus
      − heard_penalty (if hideHeard and user has completed)
      − excluded_penalty (excludedSeries)
      + discovery_jitter (controlled by discoveryRatio slider)
```

Top N stored in `for_you_feed`; client reads from there for fast response.

### 4. Customize as the user-facing tuning surface

The Music tab of the existing Customize Module is the UI for the algorithm. Web ships extensive controls (region chips, mood weight sliders, era weight sliders, language chips, similar-to-artist tag input, discovery slider, hide-heard toggle, mute-series chips). iOS gets a parallel UI in PR6+.

All tuning persists to `profiles.customize_music` (jsonb column added in `0008_profile_customize.sql`) — syncs across web + iOS automatically.

### 5. Auth — Supabase as identity, MusicKit as optional add-on

- Supabase Auth = canonical identity (display name, profile, customize prefs, library)
- "Link Apple Music" is an optional profile setting that captures the user's MusicUserToken into `profiles.apple_music_user_token` (encrypted at rest)
- Without linking → 30s previews + full FOREIGN.FM-native playback
- iOS uses Sign in with Apple → exchanges identity token for a Supabase session (`signInWithIdToken(provider: .apple)`)

### 6. Stream UX is the same on web + iOS

Both shells: persistent player at the bottom, sidebar/tab nav, search, For-You + curated rails, events strip. Web sidebar is left-rail; iOS uses the bottom TabView. Same data, same vocabulary.

## Consequences

- **One framework per surface** — SvelteKit on web, SwiftUI on iOS, Liquid/Hydrogen on Shopify (later). No cross-platform UI framework (React Native, Flutter) — each surface gets native feel.
- **The algorithm IS the differentiator.** FOREIGN.FM's edge is curatorial + the customizable scorer. Without those, we're just a Spotify reskin.
- **Supabase scales the shared layer.** Profiles, customize prefs, listen events, for-you feeds, events — all live in Postgres. Migrations live in `foreignfm-website/supabase/migrations/` and apply once for both surfaces.
- **Apple Music is opt-in.** Lazy load MusicKit only if the user has linked their account. Avoids developer-token round-trips for users who never use it.

## Reversibility

- Swapping audio engines later (SoundCloud → Mux for video, Supabase Storage → Cloudflare Stream) is mechanical — same data layer, swap the play URL.
- If MusicKit becomes too constrained or expensive, we can pivot to TIDAL / Spotify / ACR Cloud catalog APIs without touching the algorithm or schema.
- The customize_music JSONB shape is versioned only by client code — adding new dimensions doesn't require a migration.

## Related

- Schema migrations on the web side: `0007_profiles.sql` (auth + profile shape), `0008_profile_customize.sql` (customize jsonb), `0011_events.sql`, `0012_algorithm.sql`
- The Customize Module Music tab: `foreignfm-website/src/lib/components/customize/tabs/MusicTab.svelte`
- Stream page: `foreignfm-website/src/routes/stream/+page.svelte`
- iOS scaffold: `foreignfm-ios/FOREIGNFM/`
