# Data Model

Canonical entity definitions, sourced from `foreignfm-website/supabase/migrations/`. Apps in any language consume this schema — Swift, TS, Liquid all just read the same Postgres tables via the Supabase SDK.

## Tables

### `auth.users` (Supabase-managed)
Standard Supabase Auth row. Don't mutate directly — use the Supabase Auth SDK. Linked to `public.profiles` 1:1 via `user_id`.

### `public.profiles` (migration `0007`, `0008`)
One row per signed-in user. Auto-provisioned via the `handle_new_user` trigger when an `auth.users` row is created.

| Column | Type | Notes |
|---|---|---|
| `user_id` | uuid PK | references `auth.users(id) on delete cascade` |
| `handle` | text unique | 2–32 chars, `^[a-z0-9][a-z0-9_-]*$`. Default `user_<8 hex>` |
| `display_name` | text | 1–64 chars |
| `bio` | text \| null | ≤280 chars |
| `avatar_url` | text \| null | |
| `is_public` | bool | default true; gates RLS read |
| `share_live_listening` | bool | default true |
| `is_staff` | bool | default false |
| `customize_design` | jsonb \| null | Customize Module design prefs (theme, globe color, font scheme, logo, starfield) |
| `customize_music` | jsonb \| null | Customize Module music prefs (regions, hide-heard) |
| `created_at` / `updated_at` | timestamptz | auto-touched |

**RLS**: read if `is_public OR user_id = auth.uid()`; update only own row.

### `public.shows` (migration `0003`)
One row per radio show / station / publisher.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `slug` | text unique | URL slug (e.g. `sasha-marie`) |
| `name` | text | "Sasha Marie Radio" |
| `intro` | text \| null | Short editorial blurb |
| `hero_url` | text \| null | Hero image |
| `lat` / `lng` | numeric \| null | Globe pin coordinates |
| `city` / `country` | text \| null | "San Diego" / "USA" |
| `soundcloud_url` | text \| null | Used by ingestion script |
| `instagram_url`, `website_url` | text \| null | |
| `artist_id` | uuid \| null | references `public.artists(id)` (when single-artist) |

### `public.episodes` (migrations `0001`, `0002`, `0006`)
One row per chapter / mix / episode. **Per-show composite uniqueness** on `(show_id, slug)` and `(show_id, chapter_number)` — global slug/chapter uniqueness was dropped in `0006` when the schema generalized from SMR-only.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `show_id` | uuid | references `public.shows(id) on delete cascade` |
| `slug` | text | unique per show |
| `chapter_number` | int | sequential within a show; unique per show |
| `title` | text | |
| `poster_url` | text \| null | front-cover artwork |
| `poster_url_back` | text \| null | back-cover (tracklist art) — Sasha-only currently |
| `soundcloud_url` | text | playback source |
| `soundcloud_id` | text \| null | for the SC widget |
| `duration_seconds` | int \| null | |

### `public.artists` (migration `0003`)
One row per artist (used when shows are single-artist projects).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `slug` | text unique | |
| `name` | text | |
| `bio` | text \| null | |
| `hero_url` | text \| null | |
| `socials` | jsonb \| null | flexible social links |

## Coming next (gated on the editorial / research pipeline)

The Customize Module's Music tab previews controls (mood, era, language, "more like this") that need richer per-show data. The plan:

- Extend `shows` with `genres[]`, `moods[]`, `era`, `languages[]`, `vibe_tags[]`, `editor_notes`
- Extend `episodes` with `tracklist[]` (artist + title + bpm/key when available), `featured_artists[]`, `recorded_at`
- Add `playback_events(user_id, episode_id, action, ts)` for taste signals
- Add `user_likes(user_id, episode_id)` for explicit favorites

Not yet scheduled. See `roadmap.md`.

## Where this is implemented

- Migrations: [`foreignfm-website/supabase/migrations/`](https://github.com/studio-schema/foreignfm-website/tree/main/supabase/migrations) — apply with `supabase db push --include-all`
- TypeScript types: `foreignfm-website/src/lib/types.ts` (and a generated/checked-in copy from Supabase if/when wired up)
- Swift types (future): generated in `foreignfm-ios` against the same schema
