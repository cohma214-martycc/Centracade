# CLAUDE.md — CENTRARCADE

Single-file HTML5 arcade hub of quick Centrapay-branded games built on a shared 16-bit engine. One-thumb, portrait, mobile-first. The entire app is `index.html`.

**Phases 1–5 are SHIPPED.** The roster is settled at **11 games** (STACK · SCAN · HOWLER · SWINGBALL · GUNGE · CHATTER · TAZO · SCRAMBLE · DAIRY · KNUCKLEBONES · WEAVER) plus the once-a-day seeded **TUCK SHOP RUN** daily. **The base-game era is over — no new games.** Work now proceeds in this order:

- **Phase 6 — REFINEMENT ⬅ IN PROGRESS.** Telemetry first, then per-game polish (HOWLER difficulty + rocket redraw, TAZO difficulty + colour separation, GUNGE contestant-in-tank), housekeeping, the roster-wide par retune, and one shipped **determinism bug fix (B1)**.
- **Phase 7 — SKINS.** Cosmetic era packs (default: NZ 90s/00s) over frozen mechanics. Requires a string-extraction reshaping first.
- **Phase 8 — MUTATORS** and **Phase 9 — CREATURE**: **placeholder idea logs only. Do not build.** Append owner ideas as they arrive.

Pruning a game or two over time remains on the table (D26) — decided from telemetry, not vibes.

> **Branch policy** (owner instruction, 2026-07-18): all development and final pushes go to the repository's **default branch**. The former feature-branch-per-unit workflow is retired — commit units directly to the default branch.

This file has four jobs: (1) hard invariants you must never break, (2) an accurate map of the current code — **§4 is now the single home for every game's as-shipped truth**, (3) the settled roadmap, (4) the bug/tech-debt ledger (§8). Decisions in the **Decision log** are final — do not relitigate them; implement them.

> **Source of truth**: `index.html`. Where this doc and the code disagree, the code wins and this doc is wrong — fix it in the same commit.

---

## 1. Non-negotiable rules

1. **Single file.** All markup, CSS, and JS in one HTML file. No libraries, no build step. Only external dependency is the `Press Start 2P` Google font import.
2. **Logical resolution 360×640.** All game code works in this coordinate space. DPR capped at 2. Physical scaling handled once via `sc` + `ctx.setTransform` — never per-frame math against window dimensions.
3. **The Centrapay mark is settled geometry.** `drawMark()` uses six blocks on a 100×71 grid:
   `[[54,0,29,18],[82,17,18,18],[17,18,33,18],[0,35,18,18],[50,35,33,18],[17,53,29,18]]`, corner radius `0.22 × block height`, skipped below 4px. Never adjust these numbers. The mark appears in **every** skin/era (Phase 7) — it is the brand, not the theme.
4. **Icons are drawn, not glyphs.** Unicode symbols render blank on too many mobile fonts (this bit us with the house icon). Any new icon — skin selector, stats readout, fortune teller, creature — is drawn with canvas paths. (The mute speaker glyph is the one grandfathered exception; do not add more.)
5. **Audio only after a gesture.** `AudioContext` created lazily in `audio()`, resumed on tap (iOS requirement). All synthesis routes through `beep()`.
6. **Legacy best-score keys keep migrating.** `loadBest(key, legacy)` folds in old standalone keys: STACK ← `cps2_best`. Never remove this. **Pruned games keep their keys** — see rule 10.
7. **All localStorage access wrapped in try/catch.** Private browsing and sandboxes throw.
8. **`'use strict'`, no implicit globals.**
9. **Games reuse the shared systems** — `drawBG`, `drawReadyScreen`, `drawOverScreen`, FX (`burst`/`pop`), rank helpers, chrome. Cohesion is the point of the hub. A bespoke over-screen is a code smell.
10. **Pruning removes a game from the roster, never its data.** Drop it from `GAMES`/`PALMAP`; leave its `bestKey` and any legacy key untouched and documented as dormant (§2). A returning player's history must survive a prune and a re-add.
11. **iOS standalone viewport is hard-locked.** Fixed body, `100dvh`, `touch-action:none`, and the `touchmove`/`touchstart`/`gesturestart` preventDefaults exist because home-screen web-app mode rubber-bands and zoom-shifts without them. Game input runs on pointer events, which dispatch before touch defaults — that is why swallowing the touch defaults is safe. Do not "tidy" these away.
12. **Skins are display-only (D20).** Game `id`s, `bestKey`s, `par`s, rank *thresholds*, seeds, scoring, physics, and the daily sequence are **theme-invariant**. A skin may change names, taglines, ready/over copy, rank *titles*, banner text, palettes, and (later) drawn sprites — nothing that affects a number a player earns or a board a seed produces. Two players on different skins playing the same daily must get byte-identical boards and comparable totals (D22).

---

## 2. localStorage keys

| Key | Purpose |
| --- | --- |
| `arc_mute` | Mute toggle ('1'/'0') |
| `arc_stack_best` | STACK best (legacy: `cps2_best`) |
| `arc_scan_best` | SCAN best |
| `arc_howler_best` | HOWLER best |
| `arc_swing_best` | SWINGBALL best (fresh — never inherited CHAIN's history, D1) |
| `arc_gunge_best` | GUNGE best |
| `arc_chatter_best` | CHATTER best |
| `arc_tazo_best` | TAZO TYCOON best |
| `arc_scramble_best` | SCRAMBLE best |
| `arc_dairy_best` | DAIRY WARMER best |
| `arc_bones_best` | KNUCKLEBONES best |
| `arc_weaver_best` | WEAVER best |
| `arc_daily_state` | JSON: `{dayKey, played, total, result, emojiGrid}` — enforces one attempt/day (consumed at start) |
| `arc_daily_streak` | Consecutive daily completions |
| `arc_daily_lastdone` | dayKey of the last completed daily (for streak continuity) |
| **Phase 6** | |
| `arc_stats` | Telemetry (Unit 6.0 ✅) — JSON `{games:{<id>:{n,totalSecs,recent:[≤20],daily:{n,totalSecs,recent:[≤20]}}}, run:{n,totalSecs,recent:[≤20]}}`. Recorded on every game-over (free play + daily) and daily finish; dumped to console on menu entry. |
| **Phase 7 (planned)** | |
| `arc_theme` | Selected skin/era id (default `nz90`). Display-only (rule 12). |
| **Phase 8–9 (idea-log placeholders — not built)** | |
| `arc_mut_best_<gameid>_<mutid>` | Per-game per-mutator bests, separate from clean bests |
| `arc_creature` | JSON creature state: `{xp, stage, hatchedAt}` |
| **Dormant** | |
| `arc_chain_best` / `chain_best` | CHAIN best. **CHAIN was pruned in Phase 5 — keys retained, unread, never deleted** (rule 10). If CHAIN returns, `loadBest('arc_chain_best','chain_best')` restores history intact. |
| `arc_gauntlet_best` | Free-play gauntlet best. **Dead since 4.5 R3.** Not written, not read, not deleted. |

---

## 3. Architecture (current code)

### Shared engine (top of file)

- `PALS` — **ten** palettes `{top, bot, ring, acc, glow, sky}`: `0` ember/orange, `1` teal, `2` purple (**hub palette** — menu, interstitials, end card, pause, and the `gamePal` fallback), `3` backyard green, `4` party pink, `5` amber, `6` GUNGE slime lime, `7` TAZO holo blue, `8` DAIRY caramel, `9` WEAVER navy/red-sock. The four Phase-5 hue sets are still `// PROVISIONAL`.
- Helpers: `wrap(a)` (angle → −π..π), `rr(x,y,w,h,r)` (rounded-rect path; caller fills/strokes), `segDist(px,py,ax,ay,bx,by)` (point→segment — SCAN's swept beam, WEAVER's hazard-vs-thread), `segsCross(...)` (strict proper-crossing seg-seg test, shared endpoint = no cross — WEAVER's self-crossing test; `segDist` is the wrong primitive for that).
- Audio: `beep`, semantic wrappers `sHit/sPerfect/sGold/sBad/sStage/sTick`; `buzz(ms)` haptics (no-op on iOS Safari — never rely on it).
- FX: module-level `parts`/`pops` with `burst()`, `pop()`, `updateFX()`, `drawFX()`. Games clear both arrays in `init()`.
- Background: `drawBG(pal, skylineY)`; chrome: `drawChrome`, `homeRect`, `muteRect`, `inR`; screens: `drawReadyScreen`, `drawOverScreen`; ranks: `rankFor`/`nextRank` over `{s, t}` tables.
- Daily RNG: `mulberry32(a)`, `hashStr(s)`, `dayKeyOf(date)`, and `srnd(g)` = `g.srng ? g.srng() : Math.random()`.

### Game interface contract

Every game is an object literal implementing:

```
id        string, unique, lowercase
name      display name                     ← Phase 7: reads from skin pack
tag       one-line menu subtitle           ← Phase 7: reads from skin pack
bestKey   localStorage key                 (theme-invariant, rule 12)
par       number — gauntlet normalisation  (theme-invariant)
ranks     [{s, t}] ascending               (thresholds invariant; titles skinnable)
init()    full state reset (incl. parts=[]; pops=[]), sets st='ready'
start()   st: 'ready' → 'play'
update(dt) dt seconds, clamped to 0.05 by the loop
render()  draws everything incl. HUD and ready/over overlays
tap(x,y)  ready-start, over-restart (450ms debounce via overAt), gameplay input
seed(rng) accept a seeded PRNG (daily). srng=null ⇒ free play
onOver    optional — gauntlet controller hooks game-over (one-shot, cleared on fire)
spaceTap  optional bool; spacebar routes to tap() (positionless games only)
press/drag/release  optional pointer stream (D12) — see below
```

State machine: `'ready' → 'play' → 'over'`. Restart requires `performance.now() - overAt > 450`.

**Pointer stream (D12, wired):** a game declares the stream by defining `press`. The router (`gameGesture` + `gameDown()`) then forwards down/move/up and never calls `tap()` during play; `ready`/`over` interactions still resolve through `tap()` **on release**. Works identically in free play and the gauntlet. `pointercancel` routes to `release(lastCoords)` — an abandoned drag can never hang a gesture. Consumers: HOWLER (press+drag+release, drag = aim ghost only), TAZO (press+release, no drag by design), WEAVER (full consumer — all three carry live state). GUNGE and DAIRY are tap-only (D13). Games without `press` keep zero-latency tap-on-pointerdown, untouched. Keep `gameGesture` and `menuDrag` strictly separate.

### Determinism model for the daily (read before touching any `srnd` call)

The daily's promise is: **same day ⇒ same content for everyone**. The mechanism is Nth-draw determinism — the seeded stream's call *order* is what's shared, so every seeded draw site must consume a number of draws that does **not depend on player behaviour**, or the divergence must be inherent and acceptable:

- **Fixed-count-per-board draws** (GUNGE genBoard, WEAVER genBoard/rollBoats, DAIRY dealTray-per-placement, BONES per-toss shuffles, SCRAMBLE init, SWINGBALL per-hit zone, HOWLER per-resolve wind/target): board/piece/throw N is identical for all players. ✅
- **Inherently divergent** (TAZO spawns — the board state, and thus empties, depends on player choices; SCAN's endless spawn stream): acceptable — the *sequence* of draws is shared, consumption naturally differs. ✅
- **Split-stream fix — WEAVER (B1, fixed 6.0)**: `spawnFlare`'s draw count varies with player pace, so it must never share a stream with board generation. WEAVER now derives a stable integer base (`dailySeed`, one fixed draw at seed time) and splits: a **per-stage board stream** (`brnd()` = `mulberry32(dailySeed ^ 'weaver-board' ^ (stage+1)·C)`, fixed draw count ⇒ boards identical for everyone) and a **separate hazard stream** (`hrnd()`, persistent ⇒ Nth flare route shared). Free play ⇒ `Math.random` throughout. Any future seeded feature must be checked against this model.

### Router

- `mode`: `'menu' | 'game' | 'gauntlet'`; `cur` = active game. Menu card tap → `cur=g; cur.onOver=null; cur.srng=null; cur.init(); mode='game'`. Global `tick` drives pulses. Frame loop clamps dt to 0.05.
- **Menu**: page 0 = full-page TUCK SHOP RUN tile; game cards 2×2 from page 1 (`PER_PAGE=4`; 11 games ⇒ 4 pages). Card selection resolves on `pointerup` (drag = page swipe); game taps fire on `pointerdown`. Swipe, dots/arrows, ←/→ keys all page. Menu subtitle count derives from `GAMES.length`.
- **`PALMAP`/`gamePal(id)`** — the **one** module-level id→palette source of truth, with a `PALS[2]` fallback so a missing entry degrades instead of freezing the frame loop (the GUNGE-launch crash). Adding/removing a game = one line here. `drawMenuIcon` branches hardcode their own colours (its old lookup was dead code and was deleted).
- **`paused`**: set on `visibilitychange` mid-play (game or gauntlet); loop skips `update`, draws `drawPauseOverlay()`; tap/space resumes.
- **`GAUNTLET`** — daily-only controller outside the game contract. `beginDaily()` is the sole entry; consumes the attempt at start (bailing forfeits); sequences `GAMES` to first game-over via `onOver`; always seeds `mulberry32(seedBase + idx*101)` before `init()`; normalises `contribOf = round(score/par × 250)` cap 625; renders `playing → interstitial → end → sharecard`; reads only public fields.
- Bests cached via `refreshMenuBests()`; daily tile via `refreshDaily()`; both refresh on home-exit. `resize()` only on resize/orientation events (rule 2). Frame loop starts after the font loads (1.5s timeout fallback).

### Palettes and CHATTER's cycle

CHATTER is the only palette-cycling game; its cycle is the **explicit list** `CHATTER_PALS=[2,3,4,5,0,1]` (decoupled from `PALS.length`, so appending palettes can't recolour it). Known quirk: stage 0 opens on the hub purple and mismatches its green menu card — pre-existing, owner call pending (§7). Drawing a surface directly (BONES asphalt precedent, HOWLER rocket livery) remains fine where a full palette isn't warranted.

---

## 4. Current games — as-shipped truth + tuning constants

*One entry per game, in `GAMES` (daily) order. All constants provisional unless stated — retune from telemetry (Unit 6.0), not feel-in-a-vacuum. Owner play observations so far are folded in bold.*

- **STACK** (`par:20`, PALS[0], `spaceTap`) — block height `BH 42`, start width `SW 264`, perfect window `PFCT 5 + spd*0.4`px, speed 2.4 → cap 9.5 (+0.13/block). Perfect preserves width; ≤6px landing = topple. Deterministic — no seeded content.
- **SCAN** (`par:35`, PALS[1]) — 3 lives (escape = life lost); $1/$2/$3 chips + ~15% QR (+5 gold). Speed mult `1 + score*0.02`. Beam collision tests the segment swept by the tip per frame (`segDist`).
- **HOWLER** (`par:500`, PALMAP 0, press/drag/release) — swipe launcher; **the analog break**. `release` computes `vel` in **logical px/ms**; `power = clamp(vel/VREF 2.0, 0.35, 1.7)` → depth; `dx·0.72` (cap ±150) → lateral. Whistle band `1.85–2.15` px/ms → immunity to wind + ×2 (**synthetic guess — `CALIBRATE` flag logs real throws; on-device pass still owed**). Wind = seeded per-throw lateral accel (`90 + stage·45`, cap 340). Centre 100 / rim 50 / miss = life; 3 misses = over. Distance ladder `30 M → 60 M → 90 M → ULTIMATE DISTANCE` = named milestones over the continuous ramp (`TIER_HITS 3`); fanfare on tier crossings only. Rocket drawn in literal hexes: purple body `#4a1e8a→#8a4ae0`, teal fins `#2ad8b0` (owner-locked livery). **Owner: plays VERY HARD — ease in Unit 6.1. Sprite is wrong: the real Vortex/Aero Howler is a foam football-shaped head with a LONG finned tail — redraw in 6.1.**
- **SWINGBALL** (`par:40`, PALS[3] + gold, `spaceTap`) — 45° ellipse orbit `RX 126`/`RY 54`, pole 120→470, depth-ordered ball (far half behind pole), swaying cap/coil. Speed 1.7 (+0.05/hit, cap 4.2), zone 0.42 → floor 0.14 (−0.012/hit), perfect = inner 33%, slop `1.7×` zone (mistime = unwind + survive). Wind meter 100 (+20/+30/−25) banks a LOOP (×(1+loop)), steps baseSpeed +0.25 / baseZone −0.03, 300ms grace. Owns the `wasInside`/sign-crossing overshoot core as the reference implementation (ex-CHAIN) — edit with care. Break only on wild tap or untapped overshoot.
- **GUNGE** (`par:300` — **known too low, revised guess ~1200, §7**; PALS[6]) — N×N pipe rotation puzzle (5, 6 from stage 3); tap rotates 90°; openings are direction bitmasks (N1 E2 S4 W8). **D16 solved-then-scrambled**: carve one src→dest path (randomised DFS), path cells get the exact 2-opening shape their connections require (never branches into a decoy → cannot leak), decoys fill the rest, rotations-only scramble ⇒ provably solvable (verified 4000 boards). Source-connected pipes are lit. **D17 RELEASE valve** outcome matrix: manual+solved → `+remaining×segments`, advance · manual+mis-solved → leak, −1 life · auto@0+solved → 0 pts, no life lost · auto@0+mis-solved → leak, −1 life. Timer `48 − stage·3.6`, floor 21 (3× original, post-buff). 3 fails = over; the clock is a puzzle clock, never a score cap (D3). **Owner: plays well. Wants the CONTESTANT visible in the tank below, getting gunged — Unit 6.3.**
- **CHATTER** (`par:600` owner median, `CHATTER_PALS` cycle) — 5 slots at 72° from −54°, 2 discs live at start, +1 disc every 2nd stage, stage every 14s, decay 4.0 +2.1/stage. Restrike: ≤20 = PERFECT SAVE (+45, +8pts); ≥80 = +8 energy; else +30. Score `3.6 × avgEnergy/100`/s, ×2 during FULL CHATTER (all ≥85). `fumble()` (−4 all) only on the 2nd consecutive miss. **Harder than its scores suggest — do not nerf it, do not add a time cap. Its difficulty is its cap.**
- **TAZO TYCOON** (`par:4500` sim-derived, PALS[7], press+release, no drag) — 4×4 directional-merge; tiers 0 plastic → 4 mega slammer (`TAZO_COL` ramp + centred `drawMark`). Merge once per tile per move; slide tween + double-strike bounce + `clack()`. **Mega slammer detonates orthogonal CLUTTER only (tier ≤ `SLAM_MAXTIER 1`) and is consumed — full-radius clearing makes the board a perpetual-motion machine (sim-proven); do not "restore" it.** Escalating spawn pressure is first-class: count `min(12, 1+floor(moves/SPAWN_K 6))` (moves-based, skill-proof) + floor-tier rises with score (`TIER_BANDS`, tier-0 gone by 360). Loss = board-lock, ONE terminal state (owner-approved D14 exception). Time caps forbidden (D3) — tune `SPAWN_K`/`TIER_BANDS`. **Owner: plays EASY — tighten in Unit 6.2. Some tier colours are too close — separate in 6.2 (likely pairs: tier-1 cardboard vs tier-3 gold; tier-0 plastic vs the blue board/holo).**
- **SCRAMBLE** (`par:120` — likely too high post-R1; PALS[4]) — 5 kid slots, start 4 active, stage-up wakes any inactive kid; lolly arcs ~0.45s, +34 haul; decay `6.5 × decayMul(0.8–1.4)` +1.9/stage, 10s stages. Score = evenness: `5.0 × (1 − stddev/50)`, FAIR SHARE ×2 (all ≥60 within band 20). Empty meter = kid cries 1.4s (returns at 42) −1 life; 3 = over. Deferred from spec: spatial greedy-kid drift.
- **DAIRY WARMER** (`par:2200` sim-informed — greedy bot *over*-states a human, may need to come DOWN; PALS[8], tap-only D13) — 6×4 shelf, 50px cells; mince-&-cheese 2×2 / sausage roll 1×3 / potato-top L; orientations derived+deduped in `init()`. Tray of 3, **place in any order (D18)**; loss check = "no tray piece fits anywhere" (`trayDead`); STRIKE wipes the shelf + fresh tray (can't chain in one frame); 3 strikes = over. Full horizontal row sells: `+60×(2^rows−1)`, FRESH! banner. Invalid taps are free no-ops with a red misfit ghost (§9).
- **KNUCKLEBONES** (`par:90`, PALS[5] on drawn asphalt) — parabolas to `CATCH_LINE 480` from 5 slots; `BASE_LV 560`/`BASE_G 700`; RUNG speeds flat 1.00, speed ramps with time only (`×1.04`/rung-up, extra `×1.04` on loop). Gold = catch, grey = skip; missed catch or grey tap = drop; 3 = over. Ladder ONESIES→OVER THE FENCE, `CLEARS_PER_RUNG 2`, loops. `+2×(1+loop)`/catch, `+5×(1+loop)`/clean toss. Frame-flip sells the spin.
- **WEAVER** (`par:1000` sim-informed, PALS[9], FULL pointer-stream) — **D16 by inversion**: carve a random simple path of exactly `L = min(14, 7+stage)` cells on a virtual 4×4 grid; carved cells BECOME the knots (+ jitter ≤12; cell 75 ≥ 2·SNAP_R 24 + 2·JITTER ⇒ snap circles never overlap) ⇒ carve order is Hamiltonian by definition; decoys (grid-adjacent, non-consecutive, `min(6, 2+stage)`) added after. Verified 36k boards. **D19**: rope-constrained moves; thread starts only on the two carved-endpoint bollards; lives are hazard-only (boats + flares vs live thread via `segDist`; release/blocked-rope/off-bollard all free); clock is a bonus meter only (`max(14, 30−stage·1.5)`, banks `remaining×1` on completion, expiry costs nothing). Self-crossing via `segsCross`. Unwind by dragging back onto the previous knot. **B1 fixed (6.0): board draws (`brnd`) and flare draws (`hrnd`) run on split seeded streams — daily boards are now identical for all players regardless of weave pace.**

---

## 5. Roadmap

### Phases 1–5 ✅ SHIPPED (compressed record)

1. **Fixes** — spacebar guard, resize handling, font preload, SCAN swept beam, visibility pause, menu caching, feel tweaks.
2. **New games + paginated menu** — SWINGBALL, SCRAMBLE, KNUCKLEBONES.
3. **GAUNTLET** — par normalisation, interstitials, end card. *Machinery lives on as the daily.*
4. **THE DAILY — TUCK SHOP RUN** — one seeded attempt/local day (consumed at start), `#N` from `DAILY_EPOCH 2026-07-16`, streak, emoji grid + share card.
4.5. **Revision pass** — SCRAMBLE dial-up (R1), daily-only gauntlet + full-page tile (R3), SWINGBALL 45° view (R4), KNUCKLEBONES pacing (R5).
5. **Prune CHAIN + five new verbs** — CHAIN pruned (D10, keys dormant); pointer-stream contract (D12); `CHATTER_PALS` + four new palettes; HOWLER (swipe, + distance-ladder revision), GUNGE (rotate puzzle), TAZO (directional merge), DAIRY (packing), WEAVER (path-drawing). Roster 7 → 6 → 11. `PALMAP` consolidated to one map after the GUNGE-launch freeze. Historical detail for each game now lives in §4; the per-unit build narratives were retired from this doc in the Phase-6 restructure — `git log` and code comments hold the archaeology.

---

### Phase 6 — REFINEMENT ⬅ IN PROGRESS

**Why**: the roster is complete but almost every number in it is a guess. The owner has direction-of-change observations (HOWLER too hard, TAZO too easy, GUNGE par too low) but **no measured data**. Rule for this phase (D24): **telemetry lands first; magnitude retunes wait for data; direction-known changes (HOWLER easier, TAZO harder) may proceed on observation but stay `// PROVISIONAL` until measured.** Visual refinements need no data.

#### 6.0 — Telemetry + housekeeping (build first)

- **Telemetry (`arc_stats`) ✅ SHIPPED**: a central play→over transition detector in `frame()` records once per run (free play AND daily) via `recordGameStat`; `recordDailyRun` logs whole-run seconds on daily finish; `dumpStats()` prints the blob on menu entry. On every game-over (free play AND daily), append `{score, secs}` to a per-game record `{n, totalSecs, recent:[≤20 scores], daily:{n, totalSecs, recent:[≤20]}}`; on daily finish, record the whole-run seconds. Cap the JSON small (recent-score ring buffers, no per-event history). One `console.log` per run (the CALIBRATE pattern, always on — it's silent in normal use). Minimum viable read-out: dump `arc_stats` JSON to console on menu entry; a drawn stats overlay is optional later. This is the data source for the par retune (6.5), the daily-length decision (§7), and prune calls (D26). Fold the existing HOWLER/TAZO/WEAVER `CALIBRATE` logging paths into it where cheap (keep `HOWLER.CALIBRATE`'s velocity histogram — that one measures input, not outcomes).
- **BUG B1 fix — WEAVER daily determinism ✅ SHIPPED**: seeded streams split — a per-stage board RNG (`brnd`, `mulberry32(dailySeed ^ 'weaver-board' ^ (stage+1)·C)`, consumed only by `carve`/`genBoard`/`rollBoats`, fixed counts) and a separate persistent hazard RNG (`hrnd`) for flares. The Nth flare's route stays shared; boards 1..k are now identical for every player regardless of pace. Free play unchanged (`Math.random`). Flare site carries a §3 pointer so this class of bug doesn't ship again.
- **Dead-code sweep** (§8 B2) ✅ SHIPPED: removed `dailyPlayedToday()`, `HOWLER.lastOutcome`, `CHATTER.nextStageAt`, `CHATTER.perfects`, and the dead `GAUNTLET.tap` 'playing' branch.
- **Source restructure** ✅ SHIPPED: game object blocks reordered to match `GAMES`/daily order (STACK · SCAN · HOWLER · SWINGBALL · GUNGE · CHATTER · TAZO · SCRAMBLE · DAIRY · KNUCKLEBONES · WEAVER); `GAME N` numbers dropped from the banners (names kept); the now-false "labels track FILE order" caveats corrected. Verified move-only: the line multiset changed by exactly the seven banner edits.
- **Copy fix** ✅ SHIPPED: the interstitial header now reads `DAILY_NAME` (was the stale `GAUNTLET`).

#### 6.1 — HOWLER refinement (owner: "very hard" + sprite wrong)

- **Rocket redraw (no data needed).** The real Vortex/Aero Howler is a **foam football-shaped head with a long tail shaft and three swept fins** — the tail is roughly as long as the head, whistle holes sit in the head. The current sprite is a stubby rocket with base-corner fins. Redraw `drawRocket()`: ovoid purple head (`#4a1e8a→#8a4ae0` gradient, owner-locked), slim shaft, three large swept **teal** fins (`#2ad8b0`) at the rear, whistle-hole dots, cream nose highlight optional. Lives icons update free (they reuse `drawRocket`); update the `drawMenuIcon` howler branch to match. Livery stays purple/teal — that's settled.
- **Difficulty easing (direction known, magnitude provisional).** Candidate levers, roughly in order of expected effect — pick conservatively, mark everything `// PROVISIONAL`, and let `arc_stats` + `CALIBRATE` confirm:
  1. Raise the target floors and starts: `centerR 22→11` and `rimR 46→26` shrink fast — try `centerR()` floor ~14 and slower shrink (−1.0/stage), `rimR()` floor ~32 (−1.6/stage).
  2. Weaken wind: `WIND_BASE 90→~55`, `WIND_STEP 45→~30`.
  3. Consider scoring landing distance with the **vertical (power) error weighted lighter than the lateral error** — power is the hard axis of a thumb-swipe, and currently a small `vel` error moves the landing the full pole length.
  4. Widen the whistle band only *after* the on-device `CALIBRATE` pass — a wrong-centred band widened is still a wrong band.
- **Feedback improvement**: draw a brief landing splash/marker where the rocket actually came down relative to the target — right now a miss teaches nothing about *why*.
- **The on-device whistle calibration pass is still owed** (`VREF`/band are synthetic).

#### 6.2 — TAZO refinement (owner: "easy" + colours too close)

- **Difficulty (direction known)**: tighten the pressure curves, never add a clock (D3). Levers: `SPAWN_K 6→4or5` (count escalates sooner), lower `TIER_BANDS` thresholds (e.g. 60/180/360 → ~40/120/240) so clutter tiers arrive earlier, and only if still soft, `SPAWN_CAP`. Re-run the headless sim (scratchpad pattern from Unit 5) against candidate values before shipping — target median run ~2–2.5 min.
- **Colour separation**: owner reports confusable tiers. Likely pairs: **tier-1 cardboard vs tier-3 gold** (both amber) and **tier-0 plastic vs the holo-blue board/tier-2**. Proposal: tier-0 → neutral slate grey, tier-1 → deeper kraft brown (kill the amber overlap with gold), tier-2 holo keeps cyan, tier-3 gold unchanged, tier-4 white flash unchanged. **Owner to eyeball the exact hexes on-device** — mark `// PROVISIONAL`. Check every tier against the PALS[7] background, not just against each other.

#### 6.3 — GUNGE contestant (owner request)

- Draw a chibi **contestant seated in a clear gunge tank below the grid** (the whole point of the show). On a solved release, the flow animation ends with the gunge pouring onto them — green splash, drips, maybe a resigned blink; on a leak they stay clean. Reuse the SCAN/SCRAMBLE chibi construction (outline + fill rects), all drawn (rule 4).
- **Layout constraint**: the gap between grid bottom (`GY 150 + 300 = 450`) and the valve (`y 466`) is 16px — no room. Options, build session's choice: nudge `GY` up ~10–14px and the valve down a few; or place the tank+contestant beside the valve (valve is 148 wide centred; ~106px clear each side); or shrink the contestant to fit a 40–50px tank tucked under the dest column. **Do not shrink the valve or tile tap targets** to make room.
- Nice-to-have while in the file: surface the current path's `segments` count near the clock so the bank-early gamble (`remaining × segments`) is legible.

#### 6.4 — Remaining per-game polish (owner ideas land here)

Running list — append as observations arrive; each item stays independently shippable:
- SCRAMBLE: par likely down (post-R1); watch for the deferred greedy-kid-drift idea.
- CHATTER: resolve the stage-0 hub-purple question (§7) whenever CHATTER is next touched.
- *(empty slots — this is the refinement idea log)*

#### 6.5 — Roster-wide par retune (needs 6.0 data)

Retune **all eleven pars together** from `arc_stats`, not piecemeal — they are the exchange rate between games in the daily total. Current provisional set and known skew: STACK 20, SCAN 35, HOWLER 500 (likely **down** — game is hard), SWINGBALL 40, GUNGE 300 (**up**, ~1200 revised guess — `remaining×segments` is multiplicative and the 3× timer buff raised `remaining`), CHATTER 600, TAZO 4500 (revisit after 6.2 tightening), SCRAMBLE 120 (likely down), DAIRY 2200 (may need **down** — the greedy sim over-states a clock-less human game), BONES 90, WEAVER 1000. Also decide the **daily length** question here (11 games is past the 5–10 min target): accept, prune (D26), or the seeded-subset option (draw N of 11 from the day's seed, same lineup for everyone) — flagged, not built.

---

### Phase 7 — SKINS (cosmetic era packs)

**Model (D20–D23, settled):** a skin is a **content pack over frozen mechanics**. Every game keeps its verb, par, seed behaviour, scoring, `id`, and `bestKey`; the skin swaps what things are *called* and *look like*. The default pack is the current NZ 90s/00s theme (`nz90`). Future packs re-reference the same games into another era (e.g. an 80s or 10s NZ pack — HOWLER becomes that era's throwing toy, TAZO its collectible craze, and so on). The daily is identical across skins (D22) — share grids and totals stay comparable. v1 packs are **strings + palettes only** (D21); per-theme sprite overrides are a later unit (the drawn objects — rocket, pie warmer, red socks — are the expensive part).

#### 7.0 — String extraction (the reshaping — zero-visual-diff refactor)

The prerequisite. Today every display string is inline in its game object. Extract into a default pack so games *read* their display text instead of owning it:

- Pack shape (suggestion — build session may refine): `const SKINS = { nz90: { label:'NZ 90s/00s', games: { howler: { name, tag, ready:[...], overTitle, tiers:[...], rankTitles:[...], ... }, ... }, hub: { dailyName:'TUCK SHOP RUN', ... } } }` with a tiny accessor (`skin()`/`S(gameId,key)`); active pack from `arc_theme`, default `nz90`.
- **What moves**: `name`, `tag`, ready-screen lines, over-screen titles, rank **titles** (thresholds stay in the game), thematic banner strings (`GUNGED!`, `FRESH!`, `KNITTED!`, tier names), `DAILY_NAME`. **What stays inline**: mechanical/universal labels (`PERFECT!`, `+N`, `MISS`, `BEST`, `STAGE`), anything fed by numbers.
- **Acceptance test**: with only `nz90` present, rendered text is byte-identical to pre-refactor. Do this unit alone, no other changes in the commit.
- Palettes join the pack as index indirection later (a pack may remap `PALMAP` targets or supply new `PALS` rows) — not needed for 7.0.

#### 7.1 — Selector + unlock (D23)

- NZ-90s default. A **drawn** skin selector (rule 4) appears on menu page 0 once `arc_daily_streak ≥ 3`; freely switchable thereafter; choice in `arc_theme`. Before unlock, no UI hint beyond (optionally) a locked chip — keep it quiet.
- Switching is instant and touches display only. The daily share text may carry the pack's daily-name — totals/grid stay comparable regardless (D22).

#### 7.2 — First alternate era pack

- Strings + palettes for one more era (**owner to pick: 80s or 10s**, and the reference for each game — this is a naming/writing exercise, logged per game as ideas arrive). Nostalgia references stay literal per D15's spirit — owner clears each.
- Sprite overrides (per-theme draw functions) are **v2**, explicitly out of scope here.

*Skin idea log (append; don't build until 7.2):*
- *(empty — era candidates and per-game reference ideas go here)*

---

### Phase 8 — MUTATORS 🔒 PLACEHOLDER — DO NOT BUILD

Idea log only. Settled shape when it eventually builds (D5): free-play only, opt-in, never in the daily; feature-flagged (`MUTATORS_ENABLED`); scores to `arc_mut_best_<gameid>_<mutid>`, never clean bests; entry via a drawn paper-fortune-teller icon; passed as an optional `modifiers` object on `init()` that games may ignore.

*Idea log (append as they arrive):*
- GOLD RUSH — gold spawn ×2 (SCAN/BONES/CHAIN-era original)
- MIRROR — SWINGBALL direction flips each loop
- TREMOR — STACK slider sinusoidal wobble
- GREASY FINGERS — DAIRY rotate disabled
- CROSSWIND — HOWLER wind ×2
- FLOOD — GUNGE valve opens early
- *(empty slots)*

### Phase 9 — CREATURE 🔒 PLACEHOLDER — DO NOT BUILD

Idea log only. Settled shape when it eventually builds (D6/D7): pixel tamagotchi on the menu near the brand footer; drawn, palette-aware, idle-animated on `tick`. **Growth-only — never decays, never guilts; sleeps when you're away.** XP from play (~+1/100 normalised, +25/daily); stages egg → hatchling → kid → teen → legend; purely cosmetic; state in `arc_creature` (device-bound accepted). Reacts to NEW BEST / daily completion / 30s idle.

*Idea log (append as they arrive):*
- *(empty slots)*

---

## 6. Decision log (settled — do not reopen)

| # | Decision |
| --- | --- |
| D1 | SWINGBALL is endless (loop-banking multiplier), separate from CHAIN. Fresh best key, no legacy inheritance — survives CHAIN's prune. |
| D2 | ~~Seven-game trial roster~~ — superseded by D9/D10. |
| D3 | Global par constants per game; par normalisation (÷par ×250, cap 625); **no time caps anywhere**; pars from real play data. |
| D4 | Daily = one gauntlet attempt/day (consumed at start; bailing forfeits); individual games unlimited. Content-seeding only. Name: **TUCK SHOP RUN** (skinnable label, Phase 7). `DAILY_EPOCH = 2026-07-16`. |
| D5 | Mutators: opt-in, free-play only, feature-flagged, separate best tables, fortune-teller UI, excluded from the daily. **Phase 8 placeholder — not built.** |
| D6 | Creature: growth-only, device-bound localStorage, cosmetic. **Phase 9 placeholder — not built.** |
| D7 | Café hub metaphor is dead. Creature only. |
| D8 | Social = share card + daily emoji grid. No backend, no accounts. |
| D9 | Free-play gauntlet removed (4.5 R3). TUCK SHOP RUN is the only gauntlet. `arc_gauntlet_best` retired, not deleted. |
| D10 | CHAIN is pruned. First prune; sets the rule-10 pattern — roster entry removed, data keys dormant forever. |
| D11 | Phase 5 = five new games, all in the daily. Roster 7 → 6 → 11. Further pruning later, from play data. |
| D12 | Input contract extends once: optional `press`/`drag`/`release`. Games without `press` keep tap-on-pointerdown. No per-game input hacks. |
| D13 | DAIRY WARMER is tap-select / tap-rotate-button / tap-place. No drag. |
| D14 | Phase-5 games are endless + 3-lives — required for par normalisation. **TAZO is the one owner-approved exception** (single board-lock terminal state). |
| D15 | Nostalgia names stay literal (TAZO TYCOON, HOWLER, GUNGE). Owner clears each reference — applies to future skin packs too. |
| D16 | Seeded content is generated **correct-by-construction**, never generate-and-hope (GUNGE solved-then-scrambled, WEAVER Hamiltonian-first). A shared unsolvable daily board is unacceptable. |
| D17 | GUNGE releases on a drawn RELEASE valve; manual release banks `remaining × segments`; auto-release at 0 banks nothing; clean connection always lands + advances; mis-solved release always leaks −1 life. |
| D18 | DAIRY is a tray-of-3, place-in-any-order packer; loss = "no tray piece fits anywhere"; strike wipes the shelf (can't chain); 3 strikes = over. |
| D19 | WEAVER: (a) rope-constrained moves; (b) thread starts only on the carved-endpoint bollards; (c) lives are hazard-only — release = free retry; (d) the per-board clock is a bonus meter, never a terminator. Self-crossing via `segsCross`, hazards via `segDist`. |
| D20 | **The base-game era is over; skins are cosmetic content packs over frozen mechanics.** Verbs, pars, rank thresholds, seeds, scoring, ids, bestKeys, and the daily sequence are theme-invariant (rule 12). Skins never swap games in or out. |
| D21 | **Skin v1 = strings + palettes only.** Per-theme sprite/draw overrides are a later unit — the drawn objects are the expensive half of theming. |
| D22 | **The daily is identical across skins.** Same seed, same boards, same pars, same normalisation — share grids and totals comparable between players on different skins. Skin may relabel only. |
| D23 | **NZ-90s (`nz90`) is the default skin.** A drawn selector appears once `arc_daily_streak ≥ 3`; freely switchable; stored in `arc_theme`. No first-run choice screen. |
| D24 | **Phase 6 is telemetry-first.** Unit 6.0 ships before magnitude retunes; direction-known changes (HOWLER easier, TAZO harder) may proceed on owner observation but stay `// PROVISIONAL` until measured. |
| D25 | **Phases 8 (mutators) and 9 (creature) are placeholder idea logs.** Append ideas; do not build until the owner re-opens them. |
| D26 | **Pruning 1–2 games over time remains on the table**, decided from `arc_stats` telemetry (play counts, run lengths, daily-length pressure) — not from vibes. CHAIN's rule-10 pattern applies to any future prune. |

## 7. Open decisions (owner to resolve — flag, don't guess)

- **Daily length (11 games).** Over the 5–10 min target; TAZO and DAIRY are the run-length watch items (TAZO sim median ≈3.3 min pre-6.2; DAIRY is near-unlosable for a careful player). Decide at 6.5 with data: accept / prune (D26) / seeded subset (N of 11, same for everyone — flagged, not built).
- **Final pars for all eleven** (6.5) — see the skew table there. All `// PROVISIONAL`.
- **HOWLER whistle window** — on-device `CALIBRATE` pass still owed; `VREF 2.0` / band `1.85–2.15` are synthetic. Same for the 6.1 easing magnitudes.
- **TAZO tier hexes** — 6.2 proposes slate/kraft recolours for tiers 0–1; owner to eyeball on-device. Owner should also confirm *which* pairs actually confused them (the doc's guess: cardboard↔gold, plastic↔board-blue).
- **GUNGE contestant layout** — 6.3 lists three placements; build session picks, owner approves the look.
- **CHATTER stage-0 colour** — its in-play stage 0 uses hub purple (PALS[2]) and mismatches its green menu card. Pre-existing (the old modulo did the same). If unwanted: drop `2` from `CHATTER_PALS` — deliberate one-line recolour. Decide when CHATTER is next touched.
- **First alternate skin era** — 80s or 10s, and the per-game references (Phase 7.2 idea log).
- **Which games get pruned**, if any, and the target roster size (D26; after 6.0/6.5 data).
- **SCRAMBLE/CHATTER attention-mechanic overlap** — tolerated; revisit at prune time.
- **Emoji-grid comparability** — the grid length tracks the roster (11 tiles now); shares aren't comparable across a roster change. Accepted; noted in the share code.

## 8. Known bugs & tech debt (ledger — fix in the unit named)

- **B1 — WEAVER daily determinism bug. ✅ FIXED (6.0).** `spawnFlare` consumed seeded draws mid-board; flare count depends on player pace, so the shared stream drifted and players got **different boards from board 2 onward in the same daily**. Fixed by splitting the seeded stream: per-stage board stream (`brnd`, fixed draw count) + separate persistent hazard stream (`hrnd`) for flares. Verified headless — boards byte-identical across pace, Nth flare route shared. GUNGE/DAIRY/BONES/HOWLER were always safe (fixed draw counts per board/piece/throw); TAZO/SCAN divergence is inherent and acceptable.
- **B2 — Dead code. ✅ SWEPT (6.0):** removed `dailyPlayedToday()` (tile reads `dailyCache`), `HOWLER.lastOutcome` (written, never read), `CHATTER.nextStageAt` (init-only relic), `CHATTER.perfects` (counted, never shown), and `GAUNTLET.tap`'s `state==='playing'` branch (play input routes through `gameDown`). `CHATTER.restrikes`/`drops` kept — restrikes shows on the over-screen; drops was out of B2's named scope.
- **B3 — Interstitial header. ✅ FIXED (6.0)** — now renders `DAILY_NAME` instead of the stale `GAUNTLET`.
- **B4 — `GAME N` source labels. ✅ RESOLVED (6.0).** Game blocks reordered to match `GAMES`/daily order; the `GAME N` numbers were dropped from the banners (names kept) and the stale "labels track FILE order" caveats corrected. Source order and daily order now agree.
- **B5 — `frame()` try/catch — still deferred, deliberately.** One thrown frame kills the rAF loop (the GUNGE-launch freeze); the single-`PALMAP`-with-fallback fix removed the known trigger. A catch-and-drop-frame wrapper could mask real bugs — revisit only if another freeze class appears. Record kept so future sessions don't re-litigate blind.
- **B6 — Press-game ready/over latency (accepted, documented).** For games declaring `press`, ready-start and over-restart fire on *release*, not pointerdown — a one-frame-feel delay vs tap games. Cost of the D12 contract; do not special-case.
- **B7 — `refreshMenuBests` inlines STACK's legacy key.** Cleaner: an optional `legacyKey` field on the game object read generically. Cosmetic; fold into any 6.0 pass touching that function.
- **B8 — HOWLER `pointercancel` velocity quirk (accepted).** A cancelled gesture routes through `release(lastCoords)`; a stale/slow cancel no-ops (dt > `MAX_DT`), a fast one may legitimately fire. Accepted per D12 — no per-game input hacks.

## 9. Style conventions

- Compact canvas code; short locals (`p` palette, `g` gradient/game, `s` size/slot).
- All text in `Press Start 2P` at 5–30px; set `textAlign` explicitly before every text block.
- Glow via `shadowColor`/`shadowBlur`; always reset `shadowBlur=0` after.
- Keep each game's constants at the top of its object as UPPERCASE members so tuning never requires reading gameplay code.
- Comment hard-won constraints inline (home-icon, iOS-viewport, TAZO clutter-only detonation, WEAVER snap-geometry inequality) — future sessions read comments, not commit history.
- Mark every unproven number `// PROVISIONAL`. An unmarked constant claims a confidence we haven't earned.
- Empty/stray inputs are free (no-ops), never penalties — every game follows this; new refinements must too.
- **Idea logs** (Phase 6.4, 7.2, 8, 9) are append-only lists in this doc — log the owner's idea with a date, don't build it, don't reorganise the log.
- Every seeded draw site added or moved must be checked against the **§3 determinism model** (the B1 lesson).
