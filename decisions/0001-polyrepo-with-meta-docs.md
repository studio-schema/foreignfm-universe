# ADR 0001 — Polyrepo with a meta-docs repo

**Status**: Accepted · 2026-05-03

## Context

FOREIGN.FM started as a single SvelteKit web app (`sashamarie-radio`, later renamed `foreignfm-web`). Plans now include an iOS companion (`foreignfm-ios`) and a Shopify-backed marketplace (`foreignfm-shop`). All three will share one Supabase project — auth, profiles, shows, episodes, future taste signals — so the question came up: should everything live in one monorepo?

## Options considered

### A. Mega-monorepo
One repo with `apps/web`, `apps/ios`, `apps/shop`, `packages/shared`, `supabase/` at root. Single git history.

**Pros**: visible co-location, shared types/tokens easy to extract, single PR can touch multiple surfaces.

**Cons**:
- iOS/Xcode tooling expects to sit at repo root — fastlane, App Store Connect, TestFlight, certificate management all assume it
- Shopify CLI assumes a specific repo layout at root
- Vercel deploys would need `--cwd` flags or a custom build path
- Different release cadences (App Store review vs. Vercel push vs. Shopify drop) become tangled in one git history
- Monorepo tooling (Turborepo / Nx / pnpm workspaces) adds cognitive + maintenance overhead with no current payoff (no shared code yet)

### B. Polyrepo
Each surface is its own repo: `foreignfm-web`, `foreignfm-ios`, `foreignfm-shop`. Each gets native tooling at root.

**Pros**: cleanest per-surface ergonomics; independent release cadence; native CI/CD per platform; easier to onboard collaborators per-surface.

**Cons**: no obvious "home" for cross-cutting docs, architecture, brand bible, decision history. The "universe" only exists conceptually.

### C. Polyrepo + meta-docs repo (chosen)
Same as B, plus a small `foreignfm-universe` repo containing only markdown — architecture, data model, auth flow, roadmap, ADRs.

**Pros**:
- Best of B (clean per-surface ergonomics)
- The "universe" exists as a real, browsable place — not a folder structure, but a docs repo
- New collaborators clone the docs repo first to understand the system
- Decisions are versioned and durable (this file is an example)

**Cons**:
- Extra repo to maintain — but it's markdown only, near-zero overhead

## Decision

**Option C.** Polyrepo for code, plus a meta-docs repo named `foreignfm-universe`.

Local layout (parent folder `~/App Development/foreignfm-universe/` is just disk, not a repo):

```
foreignfm-universe/
├── foreignfm-universe/   ← this docs repo
├── foreignfm-web/         ← SvelteKit website (live)
├── foreignfm-ios/         ← future iOS app
└── foreignfm-shop/        ← future Shopify
```

## Consequences

- **Shared backend doesn't require shared code.** Supabase is one cloud project; each app uses its native SDK. Schema lives in `foreignfm-web/supabase/migrations/` until iOS ships and we decide whether to extract to a `foreignfm-supabase` repo.
- **Shared code is deferred.** When duplication becomes painful (likely when iOS lands and brand tokens or a `Show` type need to live in two places), we'll extract a `foreignfm-design` (or similar) npm package. We're not pre-building infrastructure.
- **Each surface ships on its own cadence.** App Store releases, Vercel deploys, Shopify drops — independent.
- **The "universe" is documented, not collocated.** This repo is the source of truth for how the system works as a whole.

## Reversibility

If this turns out wrong (e.g., we discover web + Hydrogen storefront share 80% of components and the cross-repo shuffle is killing us), we can collapse `foreignfm-web` + `foreignfm-shop` into a pnpm workspace monorepo later. iOS would stay separate either way.

The decision is one-way for iOS (no value in collapsing it in), reversible for web + shop.
