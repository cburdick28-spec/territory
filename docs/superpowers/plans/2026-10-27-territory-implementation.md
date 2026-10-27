# Territory Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Territory, a personal-use "draft a nation, conquer the world" geography game — Next.js + Supabase, with an independently-sourced 196×8 country dataset and independently-derived scoring/conquest mechanics.

**Architecture:** Next.js App Router deployed to Vercel; Supabase Postgres + Auth (Google only) accessed directly from the client with RLS enforcing per-user access, no custom API server. Pure game-logic functions (tiers, scoring, conquest order, daily seeding, XP) live in `lib/game/` and are unit-tested independently of the UI; a one-time data-prep script builds the country dataset from public sources before the game logic ever touches it.

**Tech Stack:** Next.js (App Router, TypeScript), Vitest, Supabase (Postgres, Auth, JS client), `flag-icons` (MIT), a Natural Earth–derived public-domain world map SVG.

**Spec:** `docs/superpowers/specs/2026-10-27-territory-design.md`

## Global Constraints

- No custom API server; all game logic runs client-side against Supabase directly with RLS (spec §3).
- Exactly 8 categories, in this order: Military, Population, Food Production, Industry, Natural Resources, Economy, Innovation, Location (spec §6).
- Exactly 196 countries: 193 UN member states + Taiwan, Kosovo, Palestine (spec §5).
- Tier table (rank range → tier → score), used everywhere a tier is computed (spec §6):
  | Tier | Rank range | Score |
  |---|---|---|
  | Legendary | 1–6 | 196 |
  | Elite | 7–29 | 168 |
  | Strong | 30–68 | 140 |
  | Good | 69–107 | 112 |
  | Average | 108–146 | 84 |
  | Weak | 147–179 | 56 |
  | Poor | 180–196 | 28 |
- `territory_pct = clamp(overall_score / 196 * 100, 1, 100)` (spec §6).
- Conquest order = haversine distance from home country's capital to every other country's capital, ascending (spec §6).
- Daily mode seeds a mulberry32 PRNG from the UTC day string (spec §7).
- XP: +10 for a calendar day's first daily completion; +1/+2/+3 per category scoring Strong/Elite/Legendary respectively (Weak/Poor/Average = +0), summed once per day (spec §8). Level = `floor(sqrt(xp / 50)) + 1` (spec §8).
- No Apple sign-in, no Tally/Discord/Buy-Me-a-Coffee/Umami integrations (spec §2).
- Flags from `flag-icons` (MIT); world map from a public-domain Natural Earth–derived SVG; both credited on `/credits` (spec §10).
- Country data sourced per the table in spec §5, every value cited in `DATA_SOURCES.md`; missing values use a documented regional-median imputation, never silently dropped (spec §5, §12).

## Review Focus

- Drafting the same country twice in one run must be impossible — the draw pool must shrink on every pick (spec §6 "random draw without replacement").
- `territory_pct` and the derived owned-country count must stay inside [1, 100] / [1, 196] at both extremes (all-Poor and all-Legendary runs), never 0 or over 100 (spec §6).
- A daily streak must break on a missed UTC day and must not be inflated by playing the same day's daily more than once (spec §7).
- Merging a guest's `localStorage` history into a Supabase account on sign-in must not duplicate or drop a day that already has a `daily_results` row on the server (spec §3, §9).
- Country-stat ranking must not crash or produce `NaN`/duplicate ranks when a metric is missing for a country (e.g. Kosovo/Palestine/Taiwan often lack World Bank figures) — the documented imputation fallback must kick in instead (spec §5, §12).

---

## Phase A: Project Foundation

### Task 1: Scaffold the Next.js + Vitest project

**Files:**
- Create: `package.json`, `tsconfig.json`, `next.config.ts`, `.gitignore`, `vitest.config.ts`
- Create: `app/layout.tsx`, `app/page.tsx` (placeholder home page)

**Interfaces:**
- Produces: a runnable `npm run dev` Next.js app and a runnable `npm test` Vitest suite, which every later task's tests rely on.

- [ ] **Step 1: Scaffold with `npx create-next-app@latest . --typescript --app --no-tailwind --eslint --src-dir=false --import-alias "@/*"`**, answering prompts to match this repo (already has `README.md`, allow overwrite).
- [ ] **Step 2: Add Vitest** — `npm install -D vitest`, add `"test": "vitest run"` to `package.json` scripts, create `vitest.config.ts` with `test.environment = "node"`.
- [ ] **Step 3: Write a trivial smoke test** `lib/__smoke__.test.ts` asserting `1 + 1 === 2`.
- [ ] **Step 4: Run `npm test` — expect it to pass.**
- [ ] **Step 5: Run `npm run build` — expect it to succeed.**
- [ ] **Step 6: Delete the smoke test file** (it was only to prove the harness works).
- [ ] **Step 7: Commit and push**

```bash
git add -A && git commit -m "chore: scaffold Next.js + Vitest project" && git push
```

---

### Task 2: Supabase schema and client wiring

**Files:**
- Create: `supabase/migrations/0001_init.sql`
- Create: `lib/supabase/client.ts`
- Create: `lib/supabase/types.ts`
- Create: `.env.example`
- Modify: `.gitignore` (ensure `.env.local` is ignored)

**Interfaces:**
- Produces: `createBrowserClient(): SupabaseClient<Database>` from `lib/supabase/client.ts`, and the `Database` row types from `lib/supabase/types.ts` (`Countries`, `CountryStats`, `Profiles`, `Runs`, `DailyResults`, `Achievements`, `UserAchievements`), which every data-access task after this one imports.

- [ ] **Step 1: Write `supabase/migrations/0001_init.sql`** containing exactly the schema from spec §4 (`countries`, `country_stats`, `profiles`, `runs`, `daily_results`, `achievements`, `user_achievements`), plus RLS policies: enable RLS on `profiles`, `runs`, `daily_results`, `user_achievements`; policy `using (auth.uid() = user_id)` for select/insert/update on each. `countries`, `country_stats`, `achievements` are public-read, no-write-from-client (service role only).
- [ ] **Step 2: Write `lib/supabase/types.ts`** — hand-written TypeScript interfaces mirroring the migration's columns exactly (types, nullability).
- [ ] **Step 3: Write `lib/supabase/client.ts`** exporting `createBrowserClient(): SupabaseClient` using `@supabase/supabase-js`, reading `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` from env.
- [ ] **Step 4: Write `.env.example`** listing `NEXT_PUBLIC_SUPABASE_URL=` and `NEXT_PUBLIC_SUPABASE_ANON_KEY=` with a comment pointing at a Supabase project's Settings → API page.
- [ ] **Step 5: `npm install @supabase/supabase-js`.**
- [ ] **Step 6: Run `npm run build` — expect it to succeed** (client file compiles even without a live project; it only fails at runtime without env vars).
- [ ] **Step 7: Commit and push**

```bash
git add -A && git commit -m "feat: add Supabase schema, RLS policies, and client" && git push
```

> **Manual, execution-time step (not automatable by an agent):** create a Supabase project, run `supabase/migrations/0001_init.sql` against it (via the Supabase SQL editor or CLI), enable the Google provider under Authentication → Providers, and fill in `.env.local` from `.env.example`. Later tasks that touch Supabase at runtime (seeding, auth, account sync) depend on this being done first.

---

## Phase B: Core Game Logic (pure functions, TDD)

### Task 3: Tier lookup

**Files:**
- Create: `lib/game/tiers.ts`
- Test: `lib/game/tiers.test.ts`

**Interfaces:**
- Produces: `type Tier = "Legendary" | "Elite" | "Strong" | "Good" | "Average" | "Weak" | "Poor"`, `tierForRank(rank: number): Tier`, `scoreForTier(tier: Tier): number`, used by Tasks 4, 6, and 8.

- [ ] **Step 1: Write failing tests** in `lib/game/tiers.test.ts` covering every boundary from the Global Constraints tier table: `tierForRank(1) === "Legendary"`, `tierForRank(6) === "Legendary"`, `tierForRank(7) === "Elite"`, `tierForRank(29) === "Elite"`, `tierForRank(30) === "Strong"`, `tierForRank(68) === "Strong"`, `tierForRank(69) === "Good"`, `tierForRank(107) === "Good"`, `tierForRank(108) === "Average"`, `tierForRank(146) === "Average"`, `tierForRank(147) === "Weak"`, `tierForRank(179) === "Weak"`, `tierForRank(180) === "Poor"`, `tierForRank(196) === "Poor"`; and `scoreForTier("Legendary") === 196`, `scoreForTier("Poor") === 28`.
- [ ] **Step 2: Run `npm test lib/game/tiers.test.ts` — expect FAIL** ("tierForRank is not defined").
- [ ] **Step 3: Implement `tierForRank` and `scoreForTier` in `lib/game/tiers.ts`** using the exact rank ranges and scores from the Global Constraints table.
- [ ] **Step 4: Run `npm test lib/game/tiers.test.ts` — expect PASS.**
- [ ] **Step 5: Commit and push**

```bash
git add lib/game/tiers.ts lib/game/tiers.test.ts && git commit -m "feat: add tier lookup" && git push
```

---

### Task 4: Scoring and territory percentage

**Files:**
- Create: `lib/game/scoring.ts`
- Test: `lib/game/scoring.test.ts`

**Interfaces:**
- Consumes: `tierForRank`, `scoreForTier` from Task 3.
- Produces: `type Pick = { category: string; country: string; rank: number; tier: Tier; score: number }`, `scoreRun(picks: Pick[]): { overallScore: number; overallTier: Tier; territoryPct: number }`, used by Tasks 8, 11, 12.

- [ ] **Step 1: Write failing tests**: `scoreRun` with 8 picks all rank 1 (all Legendary) → `overallScore === 196`, `overallTier === "Legendary"`, `territoryPct === 100`; all rank 196 (all Poor) → `overallScore === 28`, `territoryPct` clamped to `1` (not `14.3`, per Global Constraints clamp-to-minimum-1 rule — note: compute the raw value first in a comment in the test to show why clamping to the spec's literal formula still yields `>= 1` here, and add a second case with a deliberately tiny raw percentage, e.g. a hypothetical `overallScore` of `1`, asserting `territoryPct === 1` via the clamp); a mixed set of 8 picks spanning several tiers → assert `overallScore` is their arithmetic mean and `overallTier === tierForRank` of whatever rank maps to that mean score via `scoreForTier`'s inverse (assert against a hand-computed expected value, not a recomputation of the function under test).
- [ ] **Step 2: Run `npm test lib/game/scoring.test.ts` — expect FAIL.**
- [ ] **Step 3: Implement `scoreRun` in `lib/game/scoring.ts`**: `overallScore = mean(picks.map(p => p.score))`; `overallTier` = the tier whose score is closest to `overallScore` (since `overallScore` is a mean, it may fall between two tiers' fixed scores — resolve by nearest, ties round down to the lower tier); `territoryPct = Math.min(100, Math.max(1, overallScore / 196 * 100))`.
- [ ] **Step 4: Run `npm test lib/game/scoring.test.ts` — expect PASS.**
- [ ] **Step 5: Commit and push**

```bash
git add lib/game/scoring.ts lib/game/scoring.test.ts && git commit -m "feat: add run scoring and territory percentage" && git push
```

---

### Task 5: Conquest order (haversine distance)

**Files:**
- Create: `lib/game/conquest.ts`
- Test: `lib/game/conquest.test.ts`

**Interfaces:**
- Consumes: `{ id: string; name: string; capitalLat: number; capitalLng: number }[]` (a `Country` shape, defined in this file and re-used by Task 9's data loader).
- Produces: `haversineKm(a: {lat:number,lng:number}, b: {lat:number,lng:number}): number`, `conquestOrderFor(home: string, countries: Country[]): string[]` (returns country names, home first), `ownedCount(territoryPct: number, totalCountries: number): number`, used by Tasks 8 and 13.

- [ ] **Step 1: Write failing tests**: `haversineKm` for two known coordinate pairs with a pre-computed expected distance (e.g. London 51.5072,-0.1276 to Paris 48.8566,2.3522 ≈ 343–344 km, assert within a 2 km tolerance); `conquestOrderFor` on a small fixture of 4 countries asserts the home country is first and the rest are ascending by distance; `ownedCount(50, 196) === 98`, `ownedCount(1, 196) === 1` (never 0), `ownedCount(100, 196) === 196`.
- [ ] **Step 2: Run `npm test lib/game/conquest.test.ts` — expect FAIL.**
- [ ] **Step 3: Implement** `haversineKm` (standard great-circle formula, Earth radius 6371 km), `conquestOrderFor` (map + sort by `haversineKm` from home's coords), `ownedCount` (`Math.min(total, Math.max(1, Math.round(territoryPct / 100 * total)))`).
- [ ] **Step 4: Run `npm test lib/game/conquest.test.ts` — expect PASS.**
- [ ] **Step 5: Commit and push**

```bash
git add lib/game/conquest.ts lib/game/conquest.test.ts && git commit -m "feat: add haversine-based conquest ordering" && git push
```

---

### Task 6: Daily seed (deterministic PRNG)

**Files:**
- Create: `lib/game/dailySeed.ts`
- Test: `lib/game/dailySeed.test.ts`

**Interfaces:**
- Produces: `mulberry32(seed: number): () => number` (returns a `0..1` generator), `seedFromUtcDay(day: string): number` (hashes a `YYYY-MM-DD` string to a 32-bit int), `shuffleWithSeed<T>(items: T[], seed: number): T[]`, used by Task 14.

- [ ] **Step 1: Write failing tests**: `mulberry32(42)` called twice in sequence returns two specific, pre-recorded floats in `[0,1)` that are deterministic across runs (run the implementation once during development to capture the exact expected values into the test — do not assert a *range*, assert the *exact* float, since determinism is the entire point); `seedFromUtcDay("2026-10-27")` is deterministic (same input twice gives the same output) and differs from `seedFromUtcDay("2026-10-28")`; `shuffleWithSeed([1,2,3,4,5], seedFromUtcDay("2026-10-27"))` called twice with the same day produces identical output arrays, and differs (in general — assert inequality for this specific fixture) from the same list shuffled with a different day's seed.
- [ ] **Step 2: Run `npm test lib/game/dailySeed.test.ts` — expect FAIL.**
- [ ] **Step 3: Implement `mulberry32`** (standard public-domain mulberry32 algorithm), `seedFromUtcDay` (e.g. a simple string hash such as FNV-1a over the `YYYY-MM-DD` string), `shuffleWithSeed` (Fisher–Yates using the seeded generator).
- [ ] **Step 4: Run `npm test lib/game/dailySeed.test.ts` — expect PASS**, updating the exact expected floats in the test from the real implementation's output if the first guess was wrong.
- [ ] **Step 5: Commit and push**

```bash
git add lib/game/dailySeed.ts lib/game/dailySeed.test.ts && git commit -m "feat: add deterministic daily seeding" && git push
```

---

### Task 7: XP and leveling

**Files:**
- Create: `lib/game/xp.ts`
- Test: `lib/game/xp.test.ts`

**Interfaces:**
- Consumes: `Pick[]`, `Tier` from Tasks 3–4.
- Produces: `xpForRun(picks: Pick[], isFirstDailyToday: boolean): number`, `levelForXp(xp: number): number`, used by Task 16.

- [ ] **Step 1: Write failing tests**: `xpForRun(picks, true)` with 8 Average picks and `isFirstDailyToday=true` → `10` (just the daily bonus, no per-category bonus); with one Strong, one Elite, one Legendary and five Average, `true` → `10 + 1 + 2 + 3 = 16`; same picks with `isFirstDailyToday=false` → `1 + 2 + 3 = 6` (no daily bonus); `levelForXp(0) === 1`, `levelForXp(50) === 2`, `levelForXp(200) === 3` (per `floor(sqrt(xp/50))+1`).
- [ ] **Step 2: Run `npm test lib/game/xp.test.ts` — expect FAIL.**
- [ ] **Step 3: Implement `xpForRun`** (sum per-category bonus by tier: Strong=1, Elite=2, Legendary=3, else 0; plus 10 if `isFirstDailyToday`) **and `levelForXp`** (`Math.floor(Math.sqrt(xp / 50)) + 1`).
- [ ] **Step 4: Run `npm test lib/game/xp.test.ts` — expect PASS.**
- [ ] **Step 5: Commit and push**

```bash
git add lib/game/xp.ts lib/game/xp.test.ts && git commit -m "feat: add XP and leveling" && git push
```

---

### Task 8: Draft state machine

**Files:**
- Create: `lib/game/draft.ts`
- Test: `lib/game/draft.test.ts`

**Interfaces:**
- Consumes: `tierForRank`, `scoreForTier` (Task 3), `scoreRun` (Task 4), a `CountryStats` lookup shape `{ country: string; category: string; rank: number }[]` (matches Task 9's loader output).
- Produces: `const CATEGORIES: readonly string[]` (the 8 names, in spec order), `createDraft(countryPool: string[]): DraftState`, `drawNext(state: DraftState): { state: DraftState; drawn: string }`, `assign(state: DraftState, category: string, country: string, stats: CountryStats[]): DraftState`, `isComplete(state: DraftState): boolean`, `finalize(state: DraftState): Pick[]`, used by Tasks 12 and 14.

- [ ] **Step 1: Write failing tests**: `createDraft` on an 196-item pool has all 8 `CATEGORIES` open and an unmodified pool; `drawNext` removes the drawn country from the *returned* state's pool (confirming no mutation of the input) but does **not** remove it from `openCategories`; `assign` moves the given category from open to filled, looks up the right `CountryStats` row, and computes `tier`/`score` via Task 3; **assigning the same country to a second category after assignment must still be possible only if it wasn't consumed from the pool at draw time** — explicitly test that drawing 8 times via `drawNext` and assigning each draw never produces a duplicate country across the 8 `Pick`s (the Review Focus "no repeats" case); `isComplete` is true only once all 8 categories are filled; `finalize` on a complete state returns exactly 8 `Pick`s in `CATEGORIES` order.
- [ ] **Step 2: Run `npm test lib/game/draft.test.ts` — expect FAIL.**
- [ ] **Step 3: Implement** `CATEGORIES`, `createDraft`, `drawNext`, `assign`, `isComplete`, `finalize` in `lib/game/draft.ts` as an immutable state machine (each function returns a new `DraftState`, never mutates its argument).
- [ ] **Step 4: Run `npm test lib/game/draft.test.ts` — expect PASS.**
- [ ] **Step 5: Commit and push**

```bash
git add lib/game/draft.ts lib/game/draft.test.ts && git commit -m "feat: add draft state machine" && git push
```

---

## Phase C: Country Data Pipeline

### Task 9: Country list, borders table, and ranking/imputation logic

**Files:**
- Create: `data/countries-meta.json` (the 196 countries: id, name, capital, lat/lng — hand-curated, small, no API needed)
- Create: `data/borders.json` (country id → array of bordering country ids, for the Location metric)
- Create: `lib/data/rank.ts`
- Test: `lib/data/rank.test.ts`

**Interfaces:**
- Produces: `rankWithImputation(values: (number | null)[], higherIsBetter: boolean, regionOf: (index: number) => string): { rank: number; imputed: boolean }[]`, used by Task 10's data-prep script.

- [ ] **Step 1: Write `data/countries-meta.json`** with exactly 196 entries (193 UN members + Taiwan, Kosovo, Palestine), each `{ id, name, capitalLat, capitalLng, region }` (region used for imputation grouping, e.g. UN M49 sub-region names).
- [ ] **Step 2: Write `data/borders.json`** mapping each country id to its land-bordering neighbors' ids (island nations map to `[]`).
- [ ] **Step 3: Write failing tests** for `rankWithImputation`: an array with no nulls ranks 1 (best) for the highest value when `higherIsBetter=true`, and 1 for the lowest value when `higherIsBetter=false`; a `null` entry gets imputed to the median of its region's non-null values *before* ranking, and its result has `imputed: true`; ties resolve by stable input order (first-seen gets the better rank); the output length always equals the input length.
- [ ] **Step 4: Run `npm test lib/data/rank.test.ts` — expect FAIL.**
- [ ] **Step 5: Implement `rankWithImputation`** in `lib/data/rank.ts`.
- [ ] **Step 6: Run `npm test lib/data/rank.test.ts` — expect PASS.**
- [ ] **Step 7: Commit and push**

```bash
git add data/countries-meta.json data/borders.json lib/data/rank.ts lib/data/rank.test.ts && git commit -m "feat: add country metadata, borders table, and ranking/imputation logic" && git push
```

---

### Task 10: Data-prep script — build the 196×8 country dataset

**Files:**
- Create: `scripts/build-country-data.ts`
- Create: `DATA_SOURCES.md`
- Generated output: `data/countries.generated.json` (committed)

**Interfaces:**
- Consumes: `data/countries-meta.json`, `data/borders.json` (Task 9), `rankWithImputation` (Task 9).
- Produces: `data/countries.generated.json` shaped as `{ countryId: string; category: string; rawValue: number | null; rank: number; tier: Tier; source: string; imputed: boolean }[]` (196 × 8 = 1568 rows), consumed by Task 11's seed script and by `lib/data/countries.ts` (Task 13).

- [ ] **Step 1: Write `scripts/build-country-data.ts`** as a standalone Node script (`tsx scripts/build-country-data.ts`) that: fetches World Bank API indicators `MS.MIL.TOTL.P1`, `SP.POP.TOTL`, `AG.PRD.FOOD.XD`, `NV.IND.TOTL.CD`, `NY.GDP.TOTL.RT.ZS`, `NY.GDP.MKTP.CD`, `IP.PAT.RESD`, `GB.XPD.RSDV.GD.ZS` (most recent available year per country, via `https://api.worldbank.org/v2/country/all/indicator/{code}?format=json&per_page=300`); computes Innovation as the z-score average of the last two indicators; computes Location per spec §5's formula using `data/countries-meta.json` + `data/borders.json`; maps World Bank country codes to this project's 196-country list (countries with no World Bank entry — Taiwan, Kosovo, and any others — get `rawValue: null` and go through imputation); calls `rankForRankWithImputation` per category; calls `tierForRank` (Task 3) on each rank; writes the full 1568-row array to `data/countries.generated.json`.
- [ ] **Step 2: Write `DATA_SOURCES.md`** documenting, per category, the exact World Bank indicator code (or "computed" for Location), the fetch date, and a one-paragraph note on the imputation rule from spec §5/§12.
- [ ] **Step 3: Install the script's runtime dep**: `npm install -D tsx`.
- [ ] **Step 4: Run `npx tsx scripts/build-country-data.ts`.**
- [ ] **Step 5: Verify the output**: write a one-off check (can be a throwaway `node -e` or a real test in `lib/data/countries.generated.test.ts`) asserting `data/countries.generated.json` has exactly 1568 rows, exactly 196 distinct `countryId`s, exactly 8 distinct `category`s, and every row has a `rank` between 1 and 196 with no two rows in the same category sharing a rank.
- [ ] **Step 6: Spot-check 2–3 known values by hand** against the source (e.g. China or India should rank near the top of Population) and note any surprises in `DATA_SOURCES.md`.
- [ ] **Step 7: Commit and push**

```bash
git add scripts/build-country-data.ts DATA_SOURCES.md data/countries.generated.json lib/data/countries.generated.test.ts && git commit -m "feat: build 196x8 country dataset from public sources" && git push
```

---

### Task 11: Supabase seed script

**Files:**
- Create: `scripts/seed-supabase.ts`

**Interfaces:**
- Consumes: `data/countries-meta.json`, `data/countries.generated.json` (Tasks 9–10), a Supabase **service-role** client (not the browser anon client from Task 2 — this script runs locally with elevated access to bypass RLS for the one-time seed).

- [ ] **Step 1: Write `scripts/seed-supabase.ts`** (`tsx scripts/seed-supabase.ts`): reads `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` from env (document both in `.env.example`, service key never committed); upserts `data/countries-meta.json` into `countries`; upserts `data/countries.generated.json` into `country_stats`; upserts a starter `achievements` list (the 6 from spec §8: first daily, 7-day streak, 30-day streak, all-8-Legendary, 100 runs, first practice-mode perfect).
- [ ] **Step 2: Add `SUPABASE_SERVICE_ROLE_KEY=` to `.env.example`** with a comment that it's a local-only secret, never exposed to the client bundle.
- [ ] **Step 3: Commit and push**

```bash
git add scripts/seed-supabase.ts .env.example && git commit -m "feat: add Supabase seed script" && git push
```

> **Manual, execution-time step:** with a live Supabase project configured (Task 2's manual step done), run `npx tsx scripts/seed-supabase.ts` once to populate `countries`, `country_stats`, and `achievements`.

---

## Phase D: UI — Practice Draft Flow

### Task 12: Country data loader + draft screen

**Files:**
- Create: `lib/data/countries.ts`
- Create: `components/DraftScreen.tsx`, `components/CategoryTile.tsx`, `components/FlagImage.tsx`
- Modify: `app/draft/page.tsx`

**Interfaces:**
- Consumes: `createBrowserClient` (Task 2), `createDraft`/`drawNext`/`assign`/`isComplete`/`finalize`/`CATEGORIES` (Task 8).
- Produces: `loadCountryPool(): Promise<{ names: string[]; stats: CountryStats[] }>` from `lib/data/countries.ts` (queries `countries` + `country_stats`), used by Task 14's daily page too.

- [ ] **Step 1: Install `flag-icons`**: `npm install flag-icons`; import its CSS once in `app/layout.tsx`.
- [ ] **Step 2: Write `lib/data/countries.ts`** exporting `loadCountryPool()`, a thin Supabase query wrapper.
- [ ] **Step 3: Write `components/FlagImage.tsx`** — `FlagImage({ countryId }: { countryId: string })`, renders a `flag-icons` `<span className={`fi fi-${countryId.toLowerCase()}`} />`.
- [ ] **Step 4: Write `components/CategoryTile.tsx`** — `CategoryTile({ category, pick }: { category: string; pick?: Pick })`, shows category name and (once filled) the flag, country, and tier.
- [ ] **Step 5: Write `components/DraftScreen.tsx`** — owns the `DraftState` via `useState` seeded from `createDraft`, renders the currently-drawn country's flag and the 8 `CategoryTile`s as click targets for assignment, calls `assign` on click and `drawNext` after (unless `isComplete`).
- [ ] **Step 6: Wire `app/draft/page.tsx`** to call `loadCountryPool()` on mount and render `DraftScreen` once loaded, redirecting to a new `app/summary/[runId]/page.tsx` placeholder (`return <div>TODO</div>` for now — Task 13 fills it in) once `isComplete`.
- [ ] **Step 7: Manual browser verification**: run `npm run dev`, play a full practice draft (8 picks, no repeats visible), confirm it reaches the summary placeholder without errors.
- [ ] **Step 8: Commit and push**

```bash
git add -A && git commit -m "feat: add practice draft flow UI" && git push
```

---

### Task 13: World map, summary screen, and run persistence trigger

**Files:**
- Create: `public/map/world.svg`
- Create: `components/WorldMap.tsx`
- Modify: `app/summary/[runId]/page.tsx`

**Interfaces:**
- Consumes: `scoreRun` (Task 4), `conquestOrderFor`, `ownedCount` (Task 5), `Pick[]` (passed from the draft flow, e.g. via a client-side store or route state — implementer's choice, document which in the component).
- Produces: `WorldMap({ ownedCountryIds }: { ownedCountryIds: string[] })`, reusable by Task 14's daily summary.

- [ ] **Step 1: Source `public/map/world.svg`** — a public-domain, Natural Earth–derived world map SVG with one `<path id="{countryId}">` per country (an existing open-license SVG world map keyed by ISO codes satisfies this; if the chosen source uses different id casing/codes than `data/countries-meta.json`, normalize one to match the other in this step).
- [ ] **Step 2: Write `components/WorldMap.tsx`** — renders the SVG inline (so paths are addressable by `id`), applies a `player-territory` CSS class to each path whose id is in `ownedCountryIds`.
- [ ] **Step 3: Wire `app/summary/[runId]/page.tsx`** — given the finalized `Pick[]`, calls `scoreRun`, then `conquestOrderFor(homeCountry, countries)` and `ownedCount(territoryPct, 196)`, slices the order to that count, renders `WorldMap` with those ids plus the overall score/tier.
- [ ] **Step 4: Manual browser verification**: complete a practice draft end-to-end and confirm the summary page shows a map with the home country plus a plausible number of neighbors highlighted.
- [ ] **Step 5: Commit and push**

```bash
git add -A && git commit -m "feat: add world map and run summary screen" && git push
```

---

## Phase E: Persistence, Daily Mode, Auth

### Task 14: Guest persistence + daily mode + streaks

**Files:**
- Create: `lib/storage.ts`
- Modify: `app/daily/page.tsx`

**Interfaces:**
- Consumes: `seedFromUtcDay`, `shuffleWithSeed` (Task 6), `loadCountryPool` (Task 12), `createDraft` family (Task 8).
- Produces: `saveLocalRun(run: StoredRun): void`, `loadLocalRuns(): StoredRun[]`, `computeStreak(days: string[]): number`, used by Task 15's account-merge.

- [ ] **Step 1: Write failing tests** in `lib/storage.test.ts` (using a jsdom or manual `localStorage` mock) for `computeStreak`: `["2026-10-25","2026-10-26","2026-10-27"]` (consecutive, most-recent-last) → `3`; the same list with a gap (`["2026-10-24","2026-10-26","2026-10-27"]`) → `2` (streak only counts the unbroken run ending at the latest day); an empty list → `0`; playing two runs on the same day must appear as one entry in the day list passed to `computeStreak` (document this as the caller's responsibility, not `computeStreak`'s — the Review Focus "no double-count" case belongs to how `saveLocalRun` de-dupes by day, tested next).
- [ ] **Step 2: Add a failing test for `saveLocalRun`**: saving two runs with the same `day` keeps only one `daily` entry for that day (second save overwrites, doesn't append) — assert `loadLocalRuns().filter(r => r.day === "2026-10-27").length === 1` after two saves for that day.
- [ ] **Step 3: Run `npm test lib/storage.test.ts` — expect FAIL.**
- [ ] **Step 4: Implement `lib/storage.ts`**: `saveLocalRun` reads/writes a `territory-runs` localStorage key (array of `StoredRun = { day: string; picks: Pick[]; overallScore: number; overallTier: Tier; territoryPct: number }`), de-duping by `day`; `loadLocalRuns` parses it (empty array if absent/corrupt — wrap `JSON.parse` in try/catch); `computeStreak` walks the sorted unique day list backward from the latest day, counting consecutive calendar days.
- [ ] **Step 5: Run `npm test lib/storage.test.ts` — expect PASS.**
- [ ] **Step 6: Wire `app/daily/page.tsx`**: compute today's UTC day string, seed via `seedFromUtcDay`, use `shuffleWithSeed` on the loaded country pool before handing it to the same `DraftScreen` component from Task 12 (reused, not duplicated), call `saveLocalRun` on completion, display the streak from `computeStreak`.
- [ ] **Step 7: Manual browser verification**: play the daily twice in the same session, confirm it reflects one streak day, not two.
- [ ] **Step 8: Commit and push**

```bash
git add -A && git commit -m "feat: add guest persistence, daily mode, and streaks" && git push
```

---

### Task 15: Google sign-in and account sync

**Files:**
- Create: `components/AuthButton.tsx`
- Modify: `app/account/page.tsx`
- Create: `lib/sync.ts`
- Test: `lib/sync.test.ts`

**Interfaces:**
- Consumes: `createBrowserClient` (Task 2), `loadLocalRuns` (Task 14).
- Produces: `mergeLocalRunsToAccount(client: SupabaseClient, userId: string, localRuns: StoredRun[]): Promise<void>`, exercised by this task's tests via a mocked Supabase client.

- [ ] **Step 1: Write a failing test** for `mergeLocalRunsToAccount` using a hand-rolled fake Supabase client (an object with the same `.from().select()/.upsert()` shape, enough to drive the function): given local runs for days `["2026-10-26","2026-10-27"]` and a server that already has a `daily_results` row for `2026-10-27`, the merge **must upsert the server's `2026-10-27` run unchanged (server wins on conflict) and insert the missing `2026-10-26` run** — assert the fake client's recorded calls reflect exactly one insert (for `10-26`) and zero writes for `10-27` (the Review Focus "merge must not duplicate or drop" case).
- [ ] **Step 2: Run `npm test lib/sync.test.ts` — expect FAIL.**
- [ ] **Step 3: Implement `mergeLocalRunsToAccount`** in `lib/sync.ts`: query the user's existing `daily_results` days, diff against `localRuns`' days, insert `runs` + `daily_results` rows only for days absent on the server.
- [ ] **Step 4: Run `npm test lib/sync.test.ts` — expect PASS.**
- [ ] **Step 5: Write `components/AuthButton.tsx`** — calls `client.auth.signInWithOAuth({ provider: "google" })` / `client.auth.signOut()`, shows "Sign in" or the user's display name.
- [ ] **Step 6: Wire `app/account/page.tsx`**: on auth state change to signed-in, call `mergeLocalRunsToAccount`, then display profile (XP, level, streak computed from server `daily_results`).
- [ ] **Step 7: Manual verification** (requires the live Supabase project from Task 2's manual step, Google provider enabled): sign in with a real Google account, confirm a prior guest run appears in the account.
- [ ] **Step 8: Commit and push**

```bash
git add -A && git commit -m "feat: add Google sign-in and guest-to-account run sync" && git push
```

---

## Phase F: XP, Achievements, and Polish

### Task 16: Achievement unlock logic

**Files:**
- Create: `lib/game/achievements.ts`
- Test: `lib/game/achievements.test.ts`
- Modify: `app/summary/[runId]/page.tsx` (call unlock check on run completion), `app/account/page.tsx` (list unlocked achievements)

**Interfaces:**
- Consumes: `Pick[]`, `Tier` (Task 3–4), `StoredRun[]`/streak (Task 14).
- Produces: `checkAchievements(context: { picks: Pick[]; streak: number; totalRunsPlayed: number; mode: "practice" | "daily" }): string[]` (returns newly-unlocked achievement ids from the 6 in spec §8).

- [ ] **Step 1: Write failing tests**, one per achievement from spec §8: first daily (`mode: "daily"`, `totalRunsPlayed: 1`) → includes `"first-daily"`; `streak: 7` → includes `"streak-7"` (and `streak: 6` does not); `streak: 30` → includes `"streak-30"`; all 8 picks `tier: "Legendary"` → includes `"all-legendary"`; `totalRunsPlayed: 100` → includes `"runs-100"`; `mode: "practice"` with all 8 picks tier Strong-or-better → includes `"practice-perfect"` (and one Average pick among the 8 does not).
- [ ] **Step 2: Run `npm test lib/game/achievements.test.ts` — expect FAIL.**
- [ ] **Step 3: Implement `checkAchievements`** checking each of the 6 conditions independently, returning all that match (not just the first).
- [ ] **Step 4: Run `npm test lib/game/achievements.test.ts` — expect PASS.**
- [ ] **Step 5: Wire into `app/summary/[runId]/page.tsx`**: after `scoreRun`, call `checkAchievements`, and for signed-in users, upsert any new ids into `user_achievements` (skip silently for guests — achievements are account-only, matching spec §8's table design).
- [ ] **Step 6: Wire into `app/account/page.tsx`**: list the signed-in user's unlocked achievements (name + description from the `achievements` table) alongside XP/level.
- [ ] **Step 7: Commit and push**

```bash
git add -A && git commit -m "feat: add achievement unlock logic" && git push
```

---

### Task 17: Rankings and credits pages, final pass

**Files:**
- Modify: `app/rankings/page.tsx`
- Modify: `app/credits/page.tsx`
- Modify: `app/page.tsx` (home page: links to Draft / Daily / Rankings / Credits / Account)

**Interfaces:**
- Consumes: `loadCountryPool`/direct `country_stats` query (Task 12), `DATA_SOURCES.md` content (Task 10).

- [ ] **Step 1: Wire `app/rankings/page.tsx`**: for each of the 8 `CATEGORIES`, query `country_stats` filtered by category ordered by `rank`, render a table (country name, flag, rank, raw value).
- [ ] **Step 2: Wire `app/credits/page.tsx`**: render `DATA_SOURCES.md`'s content (read at build time via `fs.readFileSync`, or restate the table manually) plus explicit credit lines for `flag-icons` (MIT) and the world map SVG's source/license (Task 13).
- [ ] **Step 3: Wire `app/page.tsx`** as the home screen with navigation to every route built so far.
- [ ] **Step 4: Full manual browser pass** (per spec §11): fresh guest plays a practice draft and the daily, signs in with Google, confirms streak/XP/achievements persist across a page reload, checks rankings and credits render correctly.
- [ ] **Step 5: Commit and push**

```bash
git add -A && git commit -m "feat: add rankings and credits pages, wire home navigation" && git push
```

> **Manual, execution-time step:** deploy to Vercel (`vercel --prod` or the Vercel dashboard's Git integration), set the three env vars from `.env.example` (anon key public, service-role key stays local-only and is never set on Vercel since no server code uses it) in the Vercel project settings, and add the deployed URL to the Supabase Auth provider's allowed redirect URLs.

---

## Self-Review Notes

- **Spec coverage:** §3 architecture → Tasks 1–2; §4 data model → Task 2; §5 country data → Tasks 9–11; §6 mechanics → Tasks 3–5, 8; §7 daily/streaks → Tasks 6, 14; §8 XP/achievements → Tasks 7, 16; §9 auth/pages → Tasks 15, 17; §10 assets → Tasks 12–13; §11 testing → woven through every task plus Task 17's final pass; §12 risks (imputation, small-state data) → Task 9's `rankWithImputation`.
- **Type consistency checked:** `Pick`, `Tier`, `DraftState`, `CountryStats`, `StoredRun` are each defined once (Tasks 3, 4, 8, 14) and referenced by name (not redefined) in every later task that consumes them.
- **Review Focus coverage:** no-repeat draws → Task 8 step 1; territory clamp at extremes → Task 4 step 1; streak break/no-double-count → Task 14 steps 1–2; guest/account merge conflict → Task 15 step 1; missing-data imputation → Task 9 steps 3–6.
