# Auth Flow

One Supabase Auth instance, three platforms — each uses the appropriate SDK but the underlying user record (`auth.users`) and profile (`public.profiles`) are shared. A user signed in on the website sees the same profile, customize prefs, and (future) likes when they sign in on iOS.

## Identity provider

- **Supabase Auth** (default email + magic link). OAuth providers can be added per surface as needed.
- Session storage:
  - Web: `localStorage` key `foreignfm-auth` (PKCE flow), persisted across reloads
  - iOS: Supabase Swift SDK keychain storage
  - Shop: TBD (depends on Shopify integration model)

## Sign-up flow (web today)

1. User submits email at `/auth/sign-up`
2. Supabase sends magic link
3. User clicks link → redirects to `/auth/callback` → session is set
4. Trigger `handle_new_user` (`migration 0007`) auto-creates `public.profiles` row with default `handle = user_<8 hex>` and `display_name = email`
5. App routes to `/onboarding` for handle + display name selection if profile is fresh
6. After onboarding → `/account` or back to where they came from

## Profile sync

- `bootAuth()` (`foreignfm-website/src/lib/auth/session.ts`) loads session on mount and exposes:
  - `session` — current Supabase session
  - `user` — derived from session
  - `signedIn` — boolean derived
  - `profile` — full row from `public.profiles`
- On sign-in / auth change, `refreshProfile()` re-fetches the row.

## Customize prefs hand-off (web)

Designed so signed-out users still get persistence (localStorage), and signing in later doesn't lose their settings:

1. **Signed out** → all customize state lives in `localStorage` under `foreignfm-customize` (one JSON blob)
2. **Sign in** → `bootCustomize()` (`foreignfm-website/src/lib/customize/persist.ts`) waits for the profile row, then:
   - If profile has `customize_design` / `customize_music`, those win (server is source of truth across devices)
   - If profile fields are null but localStorage has data, push localStorage up to the profile (one-time migration)
3. **On every change** → debounced upsert to `profiles` + immediate localStorage mirror

## Cross-platform considerations

Same flow will apply on iOS:
- Sign-in via the Supabase Swift SDK
- Read `customize_design` / `customize_music` from the profile row
- Apply locally (iOS-native equivalents — e.g. dark mode, accent color, type size)
- Write back debounced

Open questions for when iOS ships:
- Do we need per-platform overrides? (e.g., "starfield: shader" doesn't make sense on iOS)
- Do we split the JSON into `web_design`, `ios_design`? Or keep one shared payload with platform-specific keys ignored by the other?
- Likely answer: shared shape, each platform reads only the keys it understands. Extensible.

## Row-Level Security (current)

- `profiles` — readable if `is_public` or own row; writable only by owner (no inserts from clients — trigger handles it; no deletes from clients — cascade from `auth.users`)
- `shows` / `episodes` — public read for now; writes require service role (used by ingestion script with `SUPABASE_SERVICE_ROLE_KEY`)
- `customize_design` / `customize_music` columns inherit `profiles` RLS — only the owner can see/write their own

## Service role usage

`SUPABASE_SERVICE_ROLE_KEY` (in `foreignfm-website/.env`, never committed) is required for:
- The ingestion script (`scripts/ingest-shows.mjs`) — bulk inserts into `episodes` bypass RLS
- (Future) admin tools

Never expose this key to the client. Only used in server-side scripts and (eventually) edge functions.
