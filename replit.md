# Workspace

## Overview

PalpiteStats — a premium dark-themed football analytics and betting insights platform. Provides global football statistics, match analysis, player stats, bookmaker odds comparison, AI-powered predictions, value bets, user authentication, 5-day free trial, and subscription management.

## Recent Changes (Session 15 — API Loop Detection and Protection System)

### New: `lib/request-guard.ts`
Self-contained guard module (zero external dependencies) integrated into every `apiFetch()` call:

**Loop detection:** Sliding-window counter per endpoint. If ≥5 calls in 10 s → `LOOP DETECTED` logged → circuit-breaker opens for 60 s.

**Circuit-breaker:** `isBlocked(endpoint)` + `recordCall(endpoint)`. Blocked calls serve stale in-memory cache or return soft-error — no API call made. Auto-clears after 60 s.

**Inflight coalescing:** `coalescedFetch(key, fetcher)` — if N concurrent requests for the same endpoint arrive simultaneously, only ONE HTTP call is made; all waiters receive the same promise result. Prevents duplicate calls from parallel component mounts.

**Request log:** Circular buffer (500 entries). Every API call, cache hit, inflight join, and block is logged with endpoint, timestamp, outcome, and note. Filterable by outcome type.

**Aggregate counters:** `totalApiCalls`, `totalCacheHits`, `totalInflight`, `totalBlocked`, `totalLoopEvents`, `cacheHitRate` — all live in-process.

### Integration into `apiFetch()` — 5-step flow
①  Cache hit → `recordCacheHit()` → return (no API call)
②  Suspended state → return stale/error
③  API key check → return stale/error
④  `coalescedFetch(key, ...)` — concurrent callers join existing promise
⑤  Inside coalesced fetcher: `recordCall()` → if blocked return stale, else `trackApiCall()` + HTTP fetch

### New endpoints
- `GET /api/request-log?limit=N&filter=TYPE` — paginated guard log (loop_detected, blocked, inflight, cache_hit, api_call)
- `GET /api/worker/status` now includes `guard` field
- `GET /api/api-status` now includes `guard` field (cacheHitRate, blockedEndpoints, inflightNow, etc.)

### Frontend interval protection (Cache-Control headers)
- `GET /api/liveMatches` → `Cache-Control: public, max-age=60, stale-while-revalidate=120`
- `GET /api/upcomingMatches` → `Cache-Control: public, max-age=300, stale-while-revalidate=600`
- `GET /api/matchDetails/:id` → live: `max-age=30`, DB fallback: `max-age=180`
  HTTP clients (browsers, CDNs, proxies) cannot re-request before the stated interval.

## Recent Changes (Session 14 — Central matchDataWorker + DB Statistics Persistence)

### New: `lib/match-data-worker.ts`
Central background coordinator. The single named worker that orchestrates ALL batch fixture fetches:
- **Upcoming matches** (`fixtures?next=20`) — refreshed every **5 min** (was 10 min in `FIXTURE_LIST_TTL`)
- **Finished matches** (`fixtures?last=20`) — refreshed every **15 min** (was NOT fetched at all)
- **Live matches** — handled separately by `live-engine` (60 s adaptive); worker just tracks the timing
- **Emergency guard** — skips all fetches when `isEmergency()` returns true
- Exposes `getStatus()` → `WorkerStatus` (running, totalCycles, lastCycleAt, upcomingNextInSec, finishedNextInSec)
- Callbacks-based design to avoid circular imports with the routes layer

### New fetch functions in `football-api.ts`
- `fetchUpcomingFixtures()` — calls `?next=20`, saves to DB. Uses `UPCOMING_TTL = 5 min` for apiFetch cache
- `fetchFinishedFixtures()` — calls `?last=20`, saves to DB. Uses `FINISHED_TTL = 15 min` for apiFetch cache
- Both injected into `matchDataWorker.configure()` on startup

### New canonical frontend endpoints
- `GET /api/liveMatches` — alias for `/api/live/matches`; serves from live-engine-cache
- `GET /api/upcomingMatches` — serves NS/TBD fixtures from DB; includes `nextRefreshInSec`
- `GET /api/matchDetails/:id` — serves from live-cache first, then DB fallback; lazy-loads stats
- `GET /api/worker/status` — worker telemetry (cycles, usage %, next fetch countdowns)

### `fixture-db.ts` — `team_stats_cache` table (statistics persistence)
- New `team_stats_cache` table auto-created on module load
- `saveTeamStats(teamId, leagueId, season, data)` — upsert after successful API fetch
- `getTeamStats(teamId, leagueId, season, ttlMs)` — DB-first read with TTL check
- `/fixture/:id/team-stats` now checks DB before calling API; saves on successful fetch
- Team stats now survive server restarts — no re-fetching daily quota on cold start

### API call reduction impact
- `fixtures?last=20` — zero calls before, now one call per 15 min (was missing entirely)
- `fixtures?next=20` — same call count but now explicit with 5 min TTL
- Team stats (season) — DB-first means zero API calls for all cached teams on restart
- Total theoretical max (all workers combined): ~36 calls/day at current usage

## Recent Changes (Session 13 — Advanced API Reduction System)

- **football-api.ts** — Emergency mode architecture (two-tier quota management):
  - `isThrottled()` at 80% → monitoring/logging only; `isEmergency()` at 85% → full restriction
  - **Emergency guards added to all background jobs**: `scheduleBackgroundRefresh`, `fetchTopLeaguePrelive`, `warmupFeaturedCache` — all skip their API cycles when `isEmergency()` returns true, serving from in-memory/DB cache instead
  - Live polling scheduler (`scheduleLiveRefresh`) already backs off to 90s at 80% (spec-compliant); emergency log messages differentiated
  - `/api-status` endpoint now includes `emergencyMode`, `bgJobsActive` fields
- **LiveMatchesSection.tsx** — Three new frontend systems:
  1. **Auto-pause** (`useInactivityPause`): tracks scroll/click/keydown/mousemove/touchstart; after 60 s inactivity, `isPaused = true` → incoming WS payloads buffered in `pendingDataRef` instead of applied; on resume, flushes buffer immediately. Shows "Pausado" badge with `PauseCircle` icon in header.
  2. **Visible match tracking** (`VisibleMatchCard`): each match card wrapped with `IntersectionObserver` (300px preload margin); cards outside viewport render a lightweight placeholder (teams, score, minute only); cards inside render the full `LiveMatchCard` with GPI/stats/momentum. First 4 cards assumed visible on load.
  3. **Progressive stale warning**: stale data warning (`AlertTriangle`) suppressed when paused (since data intentionally not updating)

## Recent Changes (Session 12 — API Optimization: Quota Management)

- **football-api.ts** — Three targeted optimizations to extend the free-tier 100-call/day quota:
  1. **`SEASON_STATS_TTL = 12h`** (new constant) — all `/teams/statistics?`, `/standings?`, `/fixtures?team=...&last=5`, and `/players?id=...` calls now cached 12h instead of 10 min. Season stats don't change during the day; this cuts ~99% of repeated team-stat API calls
  2. **Daily request counter** — `dailyCallCount` incremented inside `apiFetch()` on every real outbound request; resets at midnight; configurable via `API_DAILY_LIMIT` env var (default 100 for free tier)
  3. **Adaptive live polling** — changed from `setInterval(30s)` to recursive `setTimeout` (`scheduleLiveRefresh`): 60s normal, auto-bumps to 90s when `dailyCallCount ≥ 80% × DAILY_LIMIT`; outer watchdog updated from 60s → 120s to match
- **`/api-status` endpoint** now exposes: `dailyCallCount`, `dailyCallLimit`, `usagePercent`, `throttleActive`, `pollInterval`
- **TTL constants clarified**: `STATS_TTL` = per-fixture in-match data (10 min); `SEASON_STATS_TTL` = team/player season stats (12h); `TEAMS_TTL` = standings (6h)

## Recent Changes (Session 11 — Live Match System Overhaul)

- **live-engine.ts** (full rewrite):
  - Added `LiveEvent` + `LiveMatchEvents` interfaces; `liveEventsStore` Map; `fetchEventsForFixture()` — calls `fixtures/events?fixture={id}`, parses goals/cards/VAR/substitutions/assists
  - Added `homeTeamId` + `awayTeamId` to `LiveFixture` so timeline can determine home vs away alignment
  - Fixed `fetchStatsForFixture()` smart retry: fresh data + has stats → skip 55s; empty stats → retry after 20s; stamps empty entry to avoid hammering the API
  - `EMPTY_TEAM_STATS()` helper fixes missing `dangerousAttacks` in fallback away-team object
  - `PRIORITY_LEAGUE_IDS` set (top-6 + UCL/UEL/UECL + CONMEBOL cups + tier-2) used to sort matches for per-cycle processing
  - `MAX_LIVE_FIXTURES = 20` hard cap per worker cycle (API budget protection)
  - Worker 3 renamed to `runLiveDataWorker` — fetches stats + events for all priority matches every **60 s** (previously 90 s, stats only, capped at 8)
  - `updateFromApiResponse` evicts events store when match ends; triggers background stats+events fetch on every fixture refresh cycle
  - New export: `getLiveEvents(fixtureId)`
- **football-api.ts** `/live/matches`: imports `getLiveEvents`; embeds `events` array + `eventsStale` flag + `statsStale` flag + `ts` timestamp in each match response; ghost stats (all-zero) filtered from response so frontend never shows incorrect zeros
- **LiveMatchesSection.tsx** (full rewrite):
  - `LiveEvent` type + `homeTeamId`/`awayTeamId` on `LiveMatch` interface
  - `LiveTimeline` component: chronological timeline of goals/cards/VAR; home vs away alignment based on `teamId`; shows player name + assist; substitutions filtered out
  - Loading state: spinner + "Carregando estatísticas ao vivo..." when stats not yet available
  - Stale warning: `AlertTriangle` + "Dados ao vivo atualizando..." shown on section header when query data is > 90s old; per-match stale warnings for stats + events
  - `retry: 2` on react-query to recover from transient API failures
  - All previous features preserved: GPI engine, Momentum Bars, Goal Alert, Hot Match Scanner, Goal Probabilities

## Recent Changes (Session 10 — Live Intelligence Engine Upgrade)

- **live-engine.ts**: Added `dangerousAttacks` field to `TeamStats` interface; `mapTeamStats()` now parses "Dangerous Attacks" stat from API-Football `/fixtures/statistics` response
- **LiveMatchesSection.tsx** (complete rewrite): Added Goal Pressure Index (GPI 0–100) engine; Poisson-based real-time goal probabilities; `enrichMatch()` computes homeGPI/awayGPI/matchGPI/goalAlert per match client-side — zero extra API calls
  - **GPI formula**: SoT×40% + Shots×20% + Corners×15% + Possession dominance×15% + Dangerous Attacks×10%
  - **🚨 Goal Alert**: Banner shown when matchGPI > 75 AND (maxShotsOnTarget ≥ 5 OR maxCorners ≥ 6)
  - **Attacking Momentum Bars**: Per-team GPI bars with color coding (green→amber→red scale)
  - **🔥 Hot Match Scanner**: Top 5 live matches ranked by highest GPI in collapsible orange panel
  - **Real-time Goal Probabilities**: Poisson model computes "Next 10'" / "O1.5" / "O2.5" / "O3.5" live using elapsed time + matchGPI + current score; updates every 60s
  - **GPI color bar** at top of each match card (green/amber/red based on intensity)
  - All stats expandable per match via tap

## Recent Changes (Session 9 — League Expansion + AI Prediction Card)

- **football-api.ts**: `TOP_LEAGUES` expanded to include all tier-2 domestic leagues (40 Championship, 141 La Liga 2, 79 2.Bundesliga, 136 Serie B Italy, 62 Ligue 2, 72 Série B Brazil) + Copa America (9), Copa Libertadores (13), Copa Sudamericana (11); `TOP_SIX_LEAGUES` expanded to 15 leagues covering tier-1, tier-2, and European cups; `SCANNER_LEAGUES` expanded to add all second-tier + South American cups (13, 11, 9, 73) + WC Qualifiers (31, 35); Copa do Brasil (73) added to SCANNER_LEAGUES
- **football-api.ts** `/fixture/:id/analysis`: Added `marketRating` field (Excelente Oportunidade ≥75%, Boa Oportunidade 65-75%, Oportunidade Razoável 55-65%, Alto Risco <55%) and `insight` field (natural language summary from stats data)
- **leagues.ts**: Internacional section reordered (UCL/UEL/UECL first), Copa America (9) added, International Friendlies/Amistosos (667) added; Brazil section now includes Copa do Brasil (73)
- **leaguePriority.ts**: Fixed Copa do Brasil ID (66→73); added Copa Libertadores (13, pri 10), Copa Sudamericana (11, pri 11), Copa America (9, pri 12); added all tier-2 leagues with proper priorities (40→15, 141→16, 79→17, 136→18, 62→19, 72→20); Copa do Brasil 73→21
- **MatchInsights.tsx**: "Best Bet" section redesigned to "Previsão IA" — structured 4-row card: Previsão (market name) / Probabilidade (% in large font) / Avaliação (color-coded market rating badge) / Insight (natural language insight text); uses `marketRating` from API or falls back to confidence-level mapping

## Recent Changes (Session 8 — Performance & Live Engine)

- **lib/live-engine.ts** (NEW): Live Match Engine with two background workers — Worker 2 polls `fixtures?live=all` every 60s; Worker 3 re-fetches per-match statistics every 90s. Stores live fixture data + full team stats (shots, shotsOnTarget, possession, corners, fouls, yellowCards, redCards). Exported: `startLiveEngine()`, `getLiveMatches()`, `getLiveStats()`, `getLiveCount()`
- **lib/cache-manager.ts** (NEW): Centralized `CacheManager` class with named TTL tiers (LIVE 60s, FIXTURES 10min, ODDS 5min, TEAMS 6h, LEAGUES 6h, PLAYERS 12h, SQUAD 24h). Singleton `cacheManager` exported.
- **football-api.ts**: Updated TTLs to spec (FIXTURE_LIST_TTL 10min, FORM_TTL 12h, added LEAGUES_TTL 6h + TEAMS_TTL 6h); imports + starts live engine; added `GET /api/live/matches` and `GET /api/live/stats/:fixtureId` endpoints; `liveCount` field in api-status response
- **LiveMatchesSection.tsx** (NEW): Real-time live match display — pulsing "Ao Vivo" badge, match cards with score + elapsed minute, tap-to-expand stats panel (possession bar, 7 stat types). Polls every 60s. Returns null when no live matches (doesn't clutter UI).
- **Home.tsx**: LiveMatchesSection added as Step 1 (topmost priority); scanner sections (CornerScanner, CardScanner, OpportunityScanner) converted to `React.lazy()` code-split with `Suspense` fallback skeleton; loading order: Live → Favorites → Picks → Stats → Scanners

## Recent Changes (Session 7 — AI Opportunity Scanner)

- **OpportunityScannerSection.tsx** (NEW): Purple/violet scanner — composite confidence score combining Over 2.5 Goals (40%), BTTS (40%), Attack Index (20%); circular confidence ring; three factor badges per match; top 5 picks with ≥70% threshold, graceful fallback to best 5 if none qualify
- **football-api.ts**: Added `teamGoalsCache` + `fetchTeamGoalAvg()` (24h cache); added `/api/scanner/opportunities` endpoint with Poisson math; 20-min result cache; 60-call budget
- **Home.tsx**: Added `OpportunityScannerSection` after CardScannerSection

## Recent Changes (Session 6)

- **CornerScannerSection.tsx**: New homepage section "Alta Probabilidade de Escanteios" — sky/blue theme; 3-column grid; shows home avg + away avg + total, Over 8.5/9.5 progress bars; refresh button; hides when no data
- **CardScannerSection.tsx**: New homepage section "Alta Probabilidade de Cartões" — red theme; same card layout; shows avg cards per team + combined total, Over 3.5/4.5 probability bars
- **football-api.ts**: Added `/scanner/corners` and `/scanner/cards` endpoints; 20 min result cache; corners uses 3-step data strategy: (1) per-team 24h in-memory cache, (2) scan existing in-memory fixture stats (zero-cost), (3) fetch last 5 fixtures + their stats as fallback; cards uses `/teams/statistics` card data; `poissonCDF` helper for probability computation; `cornersTeamCache` Map persists between requests for 24h
- **Home.tsx**: Added `CornerScannerSection` and `CardScannerSection` after `HotMatchesSection`

## Recent Changes (Session 5)

- **AccumulatorSection.tsx**: New component on homepage — "Acumulador do Dia" amber card; auto-selects 3 matches with ≥65% confidence from top leagues; markets: Mais de 1.5 Gols, Mais de 2.5 Gols, Ambas Marcam, Vitória Mandante, Dupla Chance 1X/X2; shows combined odds, fair odds per pick, league + kickoff time; Betano + Betfair affiliate buttons
- **football-api.ts**: Added `/accumulator-of-the-day` endpoint — Poisson analysis on up to 25 prelive fixtures from TOP_SIX_LEAGUES; new `over15Prob(λh, λa)` helper; 15-minute cache; picks sorted by confidence, top 3 unique matches returned; endpoint validates ≥2 picks before returning
- **Home.tsx**: Imported and placed `<AccumulatorSection />` between TopBetsSection and GoalProbabilitySection

## Recent Changes (Session 4)

- **leagues.ts**: Added "Internacional" section at top with UEFA Nations League (5), Euro Championship (4), FIFA World Cup (1), World Cup Quals (31/35), Copa Libertadores (13), Copa Sudamericana (11), UCL (2), UEL (3), Conference League (848)
- **BetSimulator.tsx**: New page `/bet-simulator` — stake input + odds input → retorno/lucro; quick stake buttons, quick odds buttons, scale table, affiliate CTA
- **AiPerformance.tsx**: New page `/ai-performance` — tracks predictions in localStorage; green/red outcome marking; win rate progress bar; stats grid; filter tabs
- **App.tsx**: Added routes for `/bet-simulator` and `/ai-performance`
- **Layout.tsx**: Added "Performance AI" (Brain icon) and "Simulador" (Calculator icon) nav items; added Brain/Calculator imports
- **ValueBets.tsx**: Upgraded card with ⚡ VALUE BET corner badge, gold branding, EV positive section with explicit "POSITIVO" label, 3-column stats grid (Odd Bookmaker / Odd Justa / Prob AI)
- **football-api.ts**: HotMatch interface extended with `bttsPct` and `over25Pct` fields; computed from Poisson probs in warmupFeaturedCache
- **Home.tsx**: HotMatchItem interface updated with bttsPct/over25Pct; HotMatchesSection cards now show xG + BTTS% + O2.5% columns; GoalProbabilitySection header renamed to "🔥 Goal Radar" with red pill badge

## Recent Changes (Session 3)

- **Layout.tsx**: Added "Ligas" trophy-icon toggle button next to logo; fixed header div structure
- **LeagueNav.tsx**: Collapsible country/league sidebar with star favorites + overflow scroll
- **useFavoriteLeagues.ts**: localStorage hook for persisting league favorites
- **leagues.ts**: 12 countries × 2 divisions, all league IDs mapped
- **AffiliateButtons.tsx**: Reusable Betfair + Betano buttons (compact/full variants)
- **Home.tsx**: FavoriteLeaguesSection (shows starred leagues' matches), GoalProbabilitySection (top 5 by xG), URL `?league=X` param reading
- **TopPlayers.tsx**: Fully rewritten to use real `/api/top-players-stats` endpoint; 4 tabs (scorers/assists/shots/keyPasses), player photos + team logo overlays, position badges, horizontal bars
- **FixtureDetail.tsx**: Stats tab reorganized into 3 labeled sections (Forma / Ataque / Defesa); AffiliateButtons added to AI Analysis tab
- **football-api.ts**: `/api/top-players-stats` endpoint added — fetches Premier League (39), Brasileirão (71), La Liga (140); returns 4 sorted lists; 6h cache

**Target domain:** `www.palpitestats.com.br`

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle)
- **Frontend**: React + Vite, Tailwind CSS v4, Recharts, Framer Motion, React Query, Wouter
- **AI**: OpenAI via Replit AI Integrations (gpt-5.2)
- **Auth**: bcryptjs (password hashing) + session tokens in DB
- **Payments**: Stripe (set `STRIPE_SECRET_KEY` secret)

## Structure

```text
artifacts-monorepo/
├── artifacts/              # Deployable applications
│   ├── api-server/         # Express API server (port 8080)
│   └── palpite-stats/      # React + Vite frontend (root path /)
├── lib/                    # Shared libraries
│   ├── api-spec/           # OpenAPI spec + Orval codegen config
│   ├── api-client-react/   # Generated React Query hooks
│   ├── api-zod/            # Generated Zod schemas from OpenAPI
│   ├── db/                 # Drizzle ORM schema + DB connection
│   └── integrations-openai-ai-server/  # OpenAI integration server package
├── scripts/                # Utility scripts
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── tsconfig.json
└── package.json
```

## Database Schema

### Sports Data
- **leagues** — Football leagues (id, name, country, logoUrl)
- **teams** — Football clubs (id, name, shortName, logoUrl, leagueId)
- **matches** — Match data with probabilities and odds
- **players** — Player profiles
- **player_stats** — Per-match player performance
- **odds** — Bookmaker odds per match

### Auth & Subscriptions
- **users** — User accounts (id, email, passwordHash, emailVerified, emailVerifyToken)
- **subscriptions** — User plans (id, userId, plan, status, trialStartAt, trialEndAt, currentPeriodEnd, stripeCustomerId)
- **sessions** — Auth session tokens (id, userId, token, expiresAt)

## Pages

### Core Platform Pages
1. **Home** (`/`) — Today's matches (LIVE/Upcoming/Finished), clickable match cards, demo banner
2. **Fixture Detail** (`/fixture/:id`) — Full match analysis with 6 tabs: Overview, Team Stats, H2H, Players, Odds, AI Analysis
3. **Match Detail** (`/matches/:id`) — Internal DB match detail
4. **Matches** (`/matches`) — Searchable match list from DB
5. **Daily Analysis** (`/analysis`) — AI probability analysis for all today's matches
6. **Value Bets** (`/value-bets`) — AI-detected value bets where bookmaker odds > fair odds (EV filtering)
7. **Top Players** (`/top-players`) — Per-90 player rankings, filterable by position/league/metric
8. **Players** (`/players`) — Player search + stats table
9. **Player Detail** (`/players/:id`) — Player profile + per-90 charts
10. **Odds** (`/odds`) — Odds comparison table
11. **AI Predictions** (`/ai`) — Chat assistant powered by OpenAI

### Auth & Commerce
12. **Login** (`/login`) — User login with email/password
13. **Register** (`/register`) — User registration with 5-day trial, LGPD consent
14. **Pricing** (`/pricing`) — Plans (R$20/mo, R$50/3mo, R$95/6mo, R$180/yr)
15. **Privacy** (`/privacy`) — LGPD-compliant privacy policy
16. **Subscription Success** (`/subscription/success`) — Post-payment confirmation

### SEO Pages (redirect-to-canonical)
- `/jogos-ao-vivo` → Home (live matches)
- `/melhores-apostas-hoje` → /value-bets
- `/palpites-hoje` → /analysis
- `/palpites/:match` → match prediction page
- `/time/:team` → team page
- `/jogador/:player` → player page
- `/h2h/:match` → head-to-head page

## API Endpoints

### Internal (PostgreSQL / Drizzle)
- `GET /api/matches?date=YYYY-MM-DD` — Date matches from DB
- `GET /api/matches/:id` — Match detail + analysis from DB
- `GET /api/players?search=&page=&limit=` — Player search
- `GET /api/players/:id` — Player detail + stats
- `GET /api/odds?date=YYYY-MM-DD&matchId=` — Bookmaker odds from DB
- `GET /api/leagues` — All leagues
- `POST /api/ai/predict` — AI prediction via OpenAI

### Auth
- `POST /api/auth/register` — Register new user (creates trial subscription)
- `POST /api/auth/login` — Login, returns session token
- `POST /api/auth/logout` — Invalidate session
- `GET /api/auth/me` — Current user + subscription + accessLevel
- `POST /api/auth/verify-email` — Verify email token
- `POST /api/auth/stripe/checkout` — Create Stripe checkout session
- `POST /api/auth/stripe/webhook` — Handle Stripe webhook

### Live Data (API-Football with Demo Fallback)
All endpoints automatically fall back to rich demo data when API-Football is unavailable/suspended:
- `GET /api/matches-today` — All fixtures for today (+ `demo: true` flag when using fallback)
- `GET /api/fixture/:id` — Fixture details (supports demo IDs 99001–99012)
- `GET /api/fixture/:id/analysis` — AI probability analysis (Poisson model)
- `GET /api/fixture/:id/odds` — Bookmaker odds (Bet365, Betano)
- `GET /api/fixture/:id/stats` — Team match statistics (in-match, real-time)
- `GET /api/fixture/:id/team-stats` — Season statistics for both teams (season=2025, fallback to last-5)
- `GET /api/fixture/:id/last5` — Last 5 matches for each team
- `GET /api/fixture/:id/h2h` — Head-to-head history (last 3, cached 6h)
- `GET /api/prelive-matches` — Upcoming NS fixtures from 6 main leagues (from DB, 60s cache)
- `GET /api/top-bets` — Top 3 best betting opportunities of the day (probability ≥ 65%, cached 30min)
- `GET /api/hot-matches` — Top 5 hottest matches by goal trend + league priority (cached 30min)
- `GET /api/fixture/:id/standings` — League standings for home + away teams (rank, points, form)
- `GET /api/player-stats?id=&season=` — Player season statistics
- `GET /api/team-stats?team=&league=&season=` — Team statistics
- `GET /api/live-odds?fixture=` — Legacy alias for fixture odds
- `GET /api/fixture-analysis?homeTeam=&awayTeam=&league=` — Legacy alias
- `GET /api/api-status` — Current API status (suspended flag, cache size)

## Demo Data System

**CRITICAL:** The API-Football account is currently suspended. The backend has a complete demo fallback:

- **12 demo fixtures** (IDs 99001–99012) across UCL, Premier League, Brasileirão, LaLiga, Bundesliga, Serie A
- 3 LIVE matches (Real Madrid 2-1 Arsenal 67', Man City 1-0 Tottenham 34', Flamengo 1-1 Palmeiras 52')
- 5 Upcoming matches (Barcelona vs Atletico, PSG vs Bayern, São Paulo vs Corinthians, Napoli vs Inter, Bayer vs Dortmund)
- 4 Finished matches (Man United 0-2 Chelsea, Sevilla 3-1 Betis, Bayern 3-1 Leipzig, Atletico Mineiro 2-2 Grêmio)

**Suspension detection:** After first API-Football call fails, all subsequent calls for 5 minutes skip the HTTP request and return demo data immediately (~40ms response time).

## Probability Engine

Implemented in `football-api.ts` → `calcMatchProbabilities()`:
- **Model**: Poisson distribution for goal scoring
- **Parameters**: homeAttack, homeDefend, awayAttack, awayDefend, leagueAvg (1.35)
- **Outputs**: homeWin, draw, awayWin, over25, under25, btts, playerGoal, cornerOver9, over35cards
- **Value bet detection**: `fairOdds = 1 / probability`; `valueBet = bookmakerOdds > fairOdds`

## Subscription & Access Levels

- **`trial`** — 5-day full access after registration
- **`full`** — Paid subscriber (monthly/quarterly/semiannual/annual)
- **`limited`** — Trial expired or not logged in (basic odds visible, AI analysis blurred)

## Required Secrets

- `API_FOOTBALL_KEY` — API-Football.com API key (required; account currently suspended — reactivate at dashboard.api-football.com)
- `STRIPE_SECRET_KEY` — Stripe secret key (required for payments)
- `STRIPE_WEBHOOK_SECRET` — Stripe webhook secret (optional, for webhook verification)
- `JWT_SECRET` — JWT signing secret (optional, defaults to insecure placeholder)

## Social Media & SEO

- OG image: `/og-image.png` (16:9, auto-generated)
- Favicon: `/logo.png`
- Meta tags: Open Graph + Twitter Card configured in `index.html`
- Configured for domain: `www.palpitestats.com.br`
- Language: `pt-BR`

## League Priority (Flashscore-style ordering)

Top leagues shown first: UEFA Champions League, UEFA Europa League, UEFA Conference League,
Ligue 1, Premier League, La Liga, Serie A, Bundesliga, Brasileirão, Primeira Liga, etc.
Implemented in `src/lib/leaguePriority.ts`.

## TypeScript & Composite Projects

- `lib/*` packages are composite and emit declarations via `tsc --build`
- `artifacts/*` are leaf packages checked with `tsc --noEmit`

## Production Deployment

- **Deployment target**: `autoscale` (configured in `.replit`)
- **Build**: `bash scripts/build-prod.sh` — builds frontend (BASE_PATH=/) + API server bundle
  - Frontend output: `artifacts/palpite-stats/dist/public/`
  - Server output: `artifacts/api-server/dist/index.cjs`
- **Run**: `PORT=<assigned> node artifacts/api-server/dist/index.cjs`
- **Serves**: Express serves API at `/api/*` + React SPA static files at all other routes
- **Health check**: `GET /health` → `{ status: "ok", timestamp: "..." }`
- **Crash protection**: `uncaughtException` + `unhandledRejection` handlers prevent server exit on unexpected errors
- **Custom domain**: `www.palpitestats.com.br` — configure in Replit deployment settings after publishing
- **Port**: Defaults to 3000 if `PORT` env var is not set; Replit always provides `PORT` in production

## Key Commands

- `pnpm --filter @workspace/api-spec run codegen` — regenerate API client/zod from openapi.yaml
- `pnpm --filter @workspace/db run push` — push schema changes to DB
- `bash scripts/build-prod.sh` — production build (frontend + API)
