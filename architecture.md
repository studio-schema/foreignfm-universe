# Architecture

## System map

```
                       ┌──────────────────────┐
                       │  Supabase (cloud)    │
                       │  — auth              │
                       │  — profiles          │
                       │  — shows / episodes  │
                       │  — (future) signals  │
                       └──────────┬───────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
   ┌────────▼────────┐   ┌────────▼────────┐   ┌────────▼────────┐
   │  foreignfm-website  │   │  foreignfm-ios  │   │ foreignfm-shop  │
   │   (SvelteKit)   │   │     (Swift)     │   │  (Shopify TBD)  │
   │   Vercel        │   │   App Store     │   │   Shopify       │
   └─────────────────┘   └─────────────────┘   └─────────────────┘
```

One Supabase project. Three apps. Each app is independent in code, deploy, and release cadence — but they all read from and write to the same backend.

## Why polyrepo, not monorepo

See [`decisions/0001-polyrepo-with-meta-docs.md`](./decisions/0001-polyrepo-with-meta-docs.md) for the full reasoning. Short version: iOS needs Xcode at repo root, Shopify CLI assumes a specific layout, and there's no shared *code* yet — only shared *backend*. A monorepo would add tooling overhead with no current payoff.

## What each surface owns

### `foreignfm-website` — the discovery surface
- 3D hex-polygon globe of all show stations (one HTML pin per show; click → show page)
- Deck listening experience: 4 view modes (poster / vinyl / disc / iPod) per show
- Floating CornerPlayer with custom UI over a hidden SoundCloud iframe
- Customize Module: per-user design + (future) algorithm settings
- Auth: email + magic link via Supabase Auth
- SoundCloud ingestion script (`scripts/ingest-shows.mjs`) populates Supabase from public SC profiles

### `foreignfm-ios` — the companion / on-the-go listening (planned)
- Native iOS app, AirPlay, lock-screen controls, CarPlay
- Reads same shows + episodes from Supabase
- Reads/writes same profile (display name, customize prefs, likes)
- Push notifications for new chapters from followed shows

### `foreignfm-shop` — tickets + physical (planned)
- Event tickets (in-person listening rooms, tour stops)
- Vinyl pressings of mixes
- Apparel + prints
- Either a standalone Shopify Liquid theme or a Hydrogen storefront — TBD based on whether we want React parity with web

## Backend ownership

Supabase is the single source of truth for:
- **Identity** (`auth.users`)
- **Profile** (`public.profiles` — display name, handle, customize_design, customize_music, …)
- **Catalog** (`public.shows`, `public.episodes`)
- **Future signals** (likes, playback completions, follows — to be added)

Migration files currently live in [`foreignfm-website/supabase/migrations/`](https://github.com/studio-schema/foreignfm-website/tree/main/supabase/migrations) because web is the only consumer today. When iOS ships, we may extract migrations to a dedicated `foreignfm-supabase` repo — or keep them in web with iOS reading the deployed schema. Decision deferred.

## Shared assets — current strategy

Right now there is no shared code (only one app exists). When duplication starts hurting:
- **Brand tokens** (colors, fonts, type scale) → `foreignfm-design` npm package
- **Supabase types** (TS) → web-local for now; iOS does its own Swift codegen against the same schema
- **Logo + brand assets** → either in this docs repo or in `foreignfm-design`

Resist the urge to extract before there are two real consumers.

## Deploy boundaries

| Surface | Deploys via | Triggered by |
|---|---|---|
| Web | Vercel | git push to `foreignfm-website` `main` branch |
| iOS | App Store Connect / TestFlight | Xcode archive + upload (manual or fastlane) |
| Shop | Shopify CLI / Vercel (if Hydrogen) | TBD |

Different cadences are a feature, not a bug.
