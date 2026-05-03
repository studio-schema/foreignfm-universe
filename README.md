# FOREIGN.FM — Universe

Meta-docs and coordination repo for the FOREIGN.FM ecosystem. Source code for each product lives in its own sibling repo. This repo holds the architectural map: what each surface does, how they share a backend, and where decisions are recorded.

## Sibling repos

| Repo | Status | Stack | Purpose |
|---|---|---|---|
| [`foreignfm-website`](https://github.com/studio-schema/foreignfm-website) | Live | SvelteKit + Vercel + Supabase | The website — show globe, deck listening experience, profiles |
| `foreignfm-ios` | Planned | Swift / SwiftUI / Supabase Swift SDK | Native iOS companion app |
| `foreignfm-shop` | Planned | TBD (Shopify Liquid theme or Hydrogen storefront) | Tickets + merch + apparel + vinyl |

All three surfaces share **one Supabase project** — auth, profiles, shows, episodes, and (eventually) playback signals are the same data, regardless of which app you opened it from.

## Local layout

The parent folder `~/App Development/foreignfm-universe/` is just a workspace container:

```
foreignfm-universe/         ← parent folder (just disk; not itself a repo until this docs repo is initialized)
├── foreignfm-universe/      ← THIS repo: docs, architecture, decisions
├── foreignfm-website/           ← sibling repo: SvelteKit website
├── foreignfm-ios/           ← sibling repo: iOS app (planned)
└── foreignfm-shop/          ← sibling repo: Shopify (planned)
```

## What lives where

**This repo (`foreignfm-universe`)** — markdown only, no code:
- [`architecture.md`](./architecture.md) — system map: how the surfaces interact
- [`data-model.md`](./data-model.md) — canonical Show/Episode/Profile schemas
- [`auth-flow.md`](./auth-flow.md) — Supabase Auth across surfaces
- [`roadmap.md`](./roadmap.md) — what's shipped, in progress, planned
- [`decisions/`](./decisions/) — Architecture Decision Records (ADRs)

**Each sibling repo** — code + its own README + its own deploy story.

## Getting oriented

1. Read [`architecture.md`](./architecture.md) — 5 minutes, gives you the system map.
2. Skim [`data-model.md`](./data-model.md) — what shows, episodes, profiles look like.
3. Browse [`decisions/`](./decisions/) — why things are the way they are.
4. Then `cd ../foreignfm-website && cat CLAUDE.md` for the website specifics.
