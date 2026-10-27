# Territory — Design Spec

Status: approved pending final user sign-off
Date: 2026-10-27
Repo: https://github.com/cburdick28-spec/territory

## 1. Overview

A personal-use geography "draft your nation, conquer the world" game, inspired
by the mechanic observed in sector196.com (reverse-engineered client bundle
only — no code, assets, or derived datasets from that site are reused). The
player drafts 8 randomly-drawn countries into 8 fixed categories, each
scoring against that country's real-world rank in the category, then watches
an animated world map fill in with "conquered" territory proportional to
their overall score.

**Goal:** personal/private use. Not a public launch — no concern about brand
distinction from the original, but all code, data, and assets must be
independently built or sourced under licenses that permit this use.

## 2. Non-goals (explicitly cut from the original)

- Apple sign-in (requires a paid Apple developer account)
- Marketing plumbing: Tally feedback forms, Discord/social links, Buy Me a
  Coffee, Umami analytics
- Public leaderboards / anti-cheat hardening (no server-trusted scoring —
  acceptable for personal use)
- Exact reproduction of the original's tier thresholds, ceiling-band values,
  or `CONQUEST_ORDER` dataset — these are re-derived independently (see §6)

## 3. Architecture

- **Framework:** Next.js (App Router), deployed to Vercel.
- **Backend:** Supabase — Postgres database, Supabase Auth (Google OAuth
  only), Row Level Security for per-user data access. The app talks to
  Supabase directly from the client using the publishable anon key, mirroring
  the original's trust model (RLS enforces access, not the client).
- **No custom API server.** Game logic (draft, scoring, conquest reveal) runs
  client-side, same as the original.
- **State:** guests play fully using `localStorage` for progress/streak;
  signing in persists the same shape of data to Supabase and reconciles on
  login (local history merges into the account's history, same pattern the
  original uses for its device + account daily history).

## 4. Data model (Supabase/Postgres)

```sql
-- Reference data, seeded once from the data-prep script (§5)
countries (
  id text primary key,          -- ISO 3166-1 alpha-2/3 code
  name text not null,
  capital_lat numeric not null,
  capital_lng numeric not null
);

country_stats (
  country_id text references countries(id),
  category text not null,        -- one of the 8 categories
  raw_value numeric,
  rank int not null,              -- 1 (best) .. 196
  tier text not null,             -- derived from rank, see §6
  source text not null,           -- citation, e.g. "World Bank SP.POP.TOTL, 2025"
  primary key (country_id, category)
);

-- User data (RLS: a user can only read/write rows where user_id = auth.uid())
profiles (
  user_id uuid primary key references auth.users(id),
  display_name text,
  xp int not null default 0,
  level int not null default 1
);

runs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id),   -- null for guest runs synced later, or omitted entirely for guests
  played_at timestamptz not null default now(),
  mode text not null,              -- 'practice' | 'daily' | 'daily-draft'
  picks jsonb not null,            -- { category: { country, rank, tier, score } }
  overall_score numeric not null,
  overall_tier text not null,
  territory_pct numeric not null
);

daily_results (
  user_id uuid references auth.users(id),
  day date not null,
  run_id uuid references runs(id),
  primary key (user_id, day)
);

achievements (
  id text primary key,
  name text not null,
  description text not null
);

user_achievements (
  user_id uuid references auth.users(id),
  achievement_id text references achievements(id),
  unlocked_at timestamptz not null default now(),
  primary key (user_id, achievement_id)
);
```

Guest (signed-out) progress lives entirely in `localStorage` under a
`territory-daily` / `territory-runs` key, mirroring `runs`/`daily_results`
shape so merge-on-sign-in is a straight upsert.

## 5. Country data sourcing

196 countries (193 UN member states + Taiwan, Kosovo, Palestine — same set
observed in the original, which is just the standard "near-universal country
list," not proprietary) × 8 categories = 1,568 data points. These are sourced
from public data via a one-time data-prep script (`scripts/build-country-data.ts`),
not hand-typed, and cited in `DATA_SOURCES.md`:

| Category | Proxy metric | Source |
|---|---|---|
| Military | Active military personnel | World Bank `MS.MIL.TOTL.P1` |
| Population | Total population | World Bank `SP.POP.TOTL` |
| Food Production | Food production index | FAO / World Bank `AG.PRD.FOOD.XD` |
| Industry | Industry value added (% GDP) × GDP, i.e. absolute industrial output | World Bank `NV.IND.TOTL.CD` |
| Natural Resources | Total natural resources rents (% GNI) | World Bank `NY.GDP.TOTL.RT.ZS` |
| Economy | Nominal GDP | World Bank `NY.GDP.MKTP.CD` |
| Innovation | Composite: patent applications + R&D expenditure % GDP (z-score average) | World Bank `IP.PAT.RESD` + `GB.XPD.RSDV.GD.ZS` |
| Location | **Computed**, not sourced: `0.5 × (1 − normalized distance to landmass centroid) + 0.3 × normalized border count + 0.2 × has_coastline` | Computed from `countries` lat/lng + a static borders table |

Each metric is ranked 1–196 (1 = best) within its category to produce
`country_stats.rank`. Missing data (small states often lack World Bank
figures) falls back to a documented imputation rule (regional median) logged
in `DATA_SOURCES.md` so gaps are visible, not silently wrong.

## 6. Game mechanics (independently derived)

**Tiers.** Rank 1–196 maps to a percentile, bucketed into 7 tiers:

| Tier | Percentile (best→worst) |
|---|---|
| Legendary | top 3% (rank 1–6) |
| Elite | next 12% (7–29) |
| Strong | next 20% (30–68) |
| Good | next 20% (69–107) |
| Average | next 20% (108–146) |
| Weak | next 17% (147–179) |
| Poor | bottom 8% (180–196) |

Each tier has a fixed score value (Legendary=196 down to Poor=28, evenly
spaced); a pick's `score` is its tier's value. `overall_score` is the mean of
the 8 picks' scores; `overall_tier` is looked up from that mean the same way.
`territory_pct = overall_score / 196 × 100`, clamped 1–100 (home country is
always "owned").

This is a simpler, self-consistent system — not a guess at the original's
exact (unobserved) tuned thresholds or its ceiling/uncap mechanic.

**Conquest order.** For each home country, compute great-circle (haversine)
distance from its capital to every other country's capital, sort ascending.
`conquestOrderFor(home) = [home, ...sorted-by-distance]`. Revealing
`territory_pct%` of this list (rounded, min 1) matches the original's observed
behavior (`conquestOrderFor` + `showTerritoryOnMap` slicing a fixed order by
territory %) without reusing its dataset.

**Draft loop.** 8 fixed categories (Military, Population, Food Production,
Industry, Natural Resources, Economy, Innovation, Location — these category
*names* are generic/functional, not creative expression). Random draw without
replacement from the 196-country pool per pick; player assigns the drawn
country to any open category; no take-backs once assigned.

## 7. Daily mode & streaks

UTC-day string seeds a small deterministic PRNG (mulberry32) that fixes the
draw order for that day, so every player sees the same sequence. `daily_results`
(or the localStorage equivalent for guests) records one result per UTC day;
streak = consecutive days with a result, broken by any gap.

## 8. XP & achievements

- XP: `+10` per daily completed, `+1` per tier rank above Average per category
  (e.g. Strong=+1, Elite=+2, Legendary=+3; Weak/Poor=0), awarded once per
  calendar day's first completion only (matches the original's anti-farming
  intent without copying its exact formula).
- Level = `floor(sqrt(xp / 50)) + 1` (simple, tunable constant).
- Achievements (initial set, extensible via the `achievements` table):
  first daily played, 7-day streak, 30-day streak, all-8-Legendary run,
  100 runs played, first practice-mode "perfect" (all categories Strong+).

## 9. Auth & pages

- Supabase Auth, Google provider. `/account` page hosts sign-in/out.
- Routes: `/` (home + draft entry), `/draft` (the draft flow), `/daily`,
  `/summary/[runId]` (post-run recap + conquest map), `/rankings` (static
  per-category country rank tables), `/credits` (data sources + asset
  licenses), `/account`.

## 10. Assets & licensing

- World map: a public-domain, Natural Earth–derived SVG with one `<path>`
  per country (same interaction shape as the original's `mapSvg.js` —
  per-country paths toggle a "player-territory" class — but sourced and
  built independently).
- Flags: `flag-icons` (MIT license) rather than the original's per-country
  SVG files.
- All licenses and attributions recorded in `/credits`.

## 11. Testing

- Unit tests (Vitest) for: tier/rank lookup, scoring math, haversine conquest
  ordering, daily seed determinism, XP/level formula.
- Manual browser pass for the draft flow, conquest animation, and Google
  sign-in round trip before calling it done, per this project's verification
  standard.

## 12. Open risks / unknowns

- World Bank data has gaps for small/non-UN-recognized entities (Taiwan,
  Kosovo, Palestine) — needs the imputation fallback noted in §5, and those
  three will need the most manual data curation.
- "Innovation" and "Location" are inherently composite/subjective proxies;
  documented clearly as original design choices, not claims of authoritative
  rankings.
- Scope (§2/§8) is intentionally reduced from the original; can be extended
  later without architecture changes since the schema already has room
  (e.g. more achievements, more modes).
