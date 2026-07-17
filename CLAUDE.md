# CLAUDE.md — CENTRARCADE

Single-file HTML5 arcade hub of quick Centrapay-branded games built on a shared 16-bit engine. One-thumb, portrait, mobile-first. The entire app is `index.html`. Roster is the full **seven** games (STACK, SCAN, CHAIN, SWINGBALL, CHATTER, SCRAMBLE, KNUCKLEBONES). **Phases 1–4 shipped**: fixes, the seven-game roster + paginated menu, the free-play GAUNTLET, and the once-a-day seeded **TUCK SHOP RUN** daily.

**NEXT UP: Phase 4.5 — the owner's post-playtest revision pass (§5).** Five revisions, all approved: R1 SCRAMBLE difficulty, R2 iOS standalone viewport lock, R3 merge the free-play gauntlet into the daily (full-page tile), R4 SWINGBALL 45° redesign, R5 KNUCKLEBONES pacing. Implement Phase 4.5 before touching Phases 5–6 (mutators, creature).

This file has three jobs: (1) hard invariants you must never break, (2) an accurate map of the current code, (3) the settled roadmap of revisions to implement. Decisions in the **Decision log** are final — do not relitigate them; implement them. Where §3/§4 describe current code that Phase 4.5 changes, the Phase 4.5 spec wins.

---

## 1. Non-negotiable rules

1. **Single file.** All markup, CSS, and JS in one HTML file. No libraries, no build step. Only external dependency is the `Press Start 2P` Google font import.
2. **Logical resolution 360×640.** All game code works in this coordinate space. DPR capped at 2. Physical scaling handled once via `sc` + `ctx.setTransform` — never per-frame math against window dimensions.
3. **The Centrapay mark is settled geometry.** `drawMark()` uses six blocks on a 100×71 grid:
`[[54,0,29,18],[82,17,18,18],[17,18,33,18],[0,35,18,18],[50,35,33,18],[17,53,29,18]]`, corner radius `0.22 × block height`, skipped below 4px. Never adjust these numbers.
4. **Icons are drawn, not glyphs.** Unicode symbols render blank on too many mobile fonts (this bit us with the house icon). Any new icon — creature, fortune teller, share, coins — is drawn with canvas paths. (`►` ► and `★` ★ are proven safe in the shipped code; don't introduce new glyphs beyond those.)
5. **Audio only after a gesture.** `AudioContext` created lazily in `audio()`, resumed on tap (iOS requirement). All synthesis routes through `beep()`.
6. **Legacy best-score keys keep migrating.** `loadBest(key, legacy)` folds in old standalone keys: STACK ← `cps2_best`, CHAIN ← `chain_best`. Never remove this.
7. **All localStorage access wrapped in try/catch.** Private browsing and sandboxes throw.
8. **`'use strict'`, no implicit globals.**
9. **New games must reuse the shared systems** — `drawBG`, `drawReadyScreen`, `drawOverScreen`, FX (`burst`/`pop`), rank helpers, chrome. Cohesion is the point of the hub. A bespoke over-screen is a code smell.
10. **The viewport must be locked in iOS standalone mode** (added in Phase 4.5, R2): the page is `position:fixed`, touch defaults are swallowed, and nothing may reintroduce page scroll, rubber-banding, or double-tap zoom. Any future DOM element (there shouldn't be any) must carry `touch-action:none`.
11. **Never add a 7th `PALS` entry.** CHAIN and CHATTER cycle `stage % PALS.length`; changing the length silently reshuffles their stage colours. New looks are drawn directly (see the BONES asphalt court).

---

## 2. localStorage keys

| Key                             | Purpose                                                                                           |
| ------------------------------- | ------------------------------------------------------------------------------------------------- |
| `arc_mute`                      | Mute toggle ('1'/'0')                                                                             |
| `arc_stack_best`                | STACK best (legacy: `cps2_best`)                                                                  |
| `arc_scan_best`                 | SCAN best                                                                                         |
| `arc_chain_best`                | CHAIN best (legacy: `chain_best`)                                                                 |
| `arc_chatter_best`              | CHATTER best                                                                                      |
| `arc_swing_best`                | SWINGBALL best (starts fresh, does NOT inherit CHAIN's history)                                   |
| `arc_scramble_best`             | SCRAMBLE best                                                                                     |
| `arc_bones_best`                | KNUCKLEBONES best                                                                                 |
| `arc_gauntlet_best`             | **RETIRED (Phase 4.5, R3).** Was the free-play gauntlet best. After R3 it is never read or written again. Leave any stored value in place — do not delete, migrate, or repurpose it. |
| `arc_daily_state`               | JSON: `{dayKey, played, total, result, emojiGrid}` — enforces one attempt/day (consumed at start) |
| `arc_daily_streak`              | Consecutive daily completions                                                                     |
| `arc_daily_lastdone`            | dayKey of the last completed daily (for streak continuity)                                        |
| `arc_mut_best_<gameid>_<mutid>` | Per-game per-mutator bests, separate from clean bests (Phase 5)                                   |
| `arc_creature`                  | JSON creature state: `{xp, stage, hatchedAt}` (Phase 6)                                           |

---

## 3. Architecture (current code)

### Shared engine (top of file)

- `PALS` — six palettes `{top, bot, ring, acc, glow, sky}`. CHAIN and CHATTER cycle palettes per stage (see rule 11).
- Helpers: `wrap(a)` (angle → −π..π), `rr(x,y,w,h,r)` (rounded-rect path; caller fills/strokes), `segDist(px,py,ax,ay,bx,by)` (point→segment distance — used by SCAN's swept-beam collision).
- Audio: `beep`, semantic wrappers `sHit/sPerfect/sGold/sBad/sStage/sTick`; `buzz(ms)` haptics (no-op on iOS Safari — never rely on it).
- FX: module-level `parts`/`pops` with `burst()`, `pop()`, `updateFX()`, `drawFX()`. Games clear both arrays in `init()`.
- Background: `drawBG(pal, skylineY)` — gradient, 4px scanlines, two parallax star layers, procedural skyline, vignette.
- Chrome: `drawChrome(showHome)`, `homeRect()`, `muteRect()`, `inR()`.
- Screens: `drawReadyScreen(pal, name, lines)`, `drawOverScreen(...)`.
- Ranks: `rankFor(ranks, s)` / `nextRank(ranks, s)` over `{s, t}` tables.

### Game interface contract

Every game is an object literal implementing:

```
id        string, unique, lowercase
name      display name
tag       one-line menu subtitle
bestKey   localStorage key
ranks     [{s: threshold, t: 'TITLE'}, ...] ascending
init()    full state reset (incl. parts=[]; pops=[]), sets st='ready'
start()   st: 'ready' → 'play'
update(dt) dt seconds, clamped to 0.05 by the loop
render()  draws everything incl. HUD and ready/over overlays
tap(x,y)  ready-start, over-restart (450ms debounce via overAt), gameplay input
spaceTap  optional bool; when true the spacebar routes to tap() (positionless games: STACK, CHAIN, SWINGBALL). Absent/false ⇒ space is a no-op for that game.
```

State machine: `'ready' → 'play' → 'over'`. Restart requires `performance.now() - overAt > 450`.

**Contract extensions (all wired):**

- `par` — number, global par score for gauntlet normalisation. All seven games, all provisional: STACK 20, SCAN 35, CHAIN 50, SWINGBALL 40, CHATTER 600 (owner median — CHATTER plays far harder than its ranks), SCRAMBLE 120, KNUCKLEBONES 90.
- `seed(rng)` — accept a seeded PRNG for daily runs. Content-seeding only: spawn order, zone placements, disc timings. Physics stays live. Every game has `seed(rng)` (sets `this.srng`) + a shared `srnd(g)` = `g.srng ? g.srng() : Math.random()`. Only content calls route through `srnd` (SCAN spawn side/type/y, CHAIN+SWINGBALL zone, SCRAMBLE kid mix, BONES toss shuffles; STACK/CHATTER are already deterministic). `srng=null` ⇒ free play stays byte-identical; cleared at every non-daily entry.
- `onOver` — optional callback the daily controller sets to hook game-over. Each game calls `if(this.onOver) this.onOver()` at its over-transition; the controller registers it, clears it one-shot (double-fire safe), and free-play selection resets it to null.

### Router

- `mode`: `'menu' | 'game' | 'gauntlet'`; `cur` = active game (in gauntlet mode, the game the controller is running). Global `tick` drives all pulse/shimmer phases. Frame loop clamps dt to 0.05.
- Menu tap → `cur=g; cur.onOver=null; cur.srng=null; cur.init(); mode='game'`. Home → discard state, back to menu.
- **Menu (post-R3 layout)**: **page 0 is a single full-page TUCK SHOP RUN tile**; the seven game cards fill paginated 2×2 grids **from page 1 on** (`PER_PAGE=4` ⇒ 3 pages total: tile, 4 cards, 3 cards). Card selection resolves on `pointerup` so a horizontal drag reads as a page swipe; **in-game taps still fire on `pointerdown`** for zero latency. Swipe, tappable dots/arrows (`drawArrow()`), and ←/→ arrow keys all page. There is no `MENU_ITEMS` array any more — the grid iterates `GAMES` directly; the GAUNTLET menu card and the slim daily banner are both gone, replaced by the tile.
- **`paused`** (module-level): set on `visibilitychange` while a game is mid-play (in `game` **or** `gauntlet` mode); the loop then skips `update`, keeps rendering the frozen frame, and draws `drawPauseOverlay()`. A tap or space resumes; home/mute stay live.
- **`GAUNTLET` controller (post-R3: daily-only)**: `mode='gauntlet'` is entered **only** via `GAUNTLET.beginDaily()` (the tile). The controller sequences all games in `GAMES` order to first game-over each via each game's `onOver` hook, normalises (`round(score/par × 250)`, cap 625) into a running `total`, and renders interstitials → end card → share card. It always seeds (`cur.seed(mulberry32(hash(dayKey)+idx*101))` before `init()`), enforces one attempt/day via `arc_daily_state` (consumed at start; bailing forfeits), tracks `#N`/streak, and owns the `interstitial`→`end`→`sharecard` states. Reads only public fields (`score`/`par`/`ranks`/`name`); never mutates game internals. `begin()` (free-play mode), `drawEnd()`, and `arc_gauntlet_best` no longer exist after R3.
- **Bests cached** on menu entry via `refreshMenuBests()` → `menuBests` (over `GAMES` only; not `loadBest` per card per frame); refreshed on home-exit so a new best set mid-game shows.
- **`sc` is recomputed by `resize()`** on `resize`/`orientationchange`/`visualViewport.resize` only — backing store stays `W*dpr`, the single `setTransform` is never touched per frame (rule 2).
- **Frame loop starts only after the font is ready**: `document.fonts.load('8px "Press Start 2P"')` raced with a 1.5s timeout fallback, to kill the FOUT.

---

## 4. Current games — tuning constants

*Where Phase 4.5 changes a value, the table shows the **new target** with the pre-revision value in (parens). Everything is still `// PROVISIONAL` — retune from play data.*

- **STACK** — block height 42, start width 264, perfect window `5 + spd*0.4`px (scales with speed), speed 2.4 → cap 9.5 (+0.13/block). Perfect preserves width; ≤6px landing = topple. *(unchanged in 4.5)*
- **SCAN** — 3 lives (escape = life lost); $1/$2/$3 chips + 15% QR (+5 gold). Global speed mult `1 + score*0.02`. Beam extend/retract; collision tests the **segment swept by the tip** each frame (`segDist`), not just the tip point. *(unchanged in 4.5)*
- **CHAIN** — R=116, speed 1.7 (+0.05/link, cap 4.2), zone half-width 0.42 → floor 0.14 (−0.012/link, +0.07 on stage-up), stage = 10 links, perfect = inner 33% (+2), post-stage gold zone (+3). **300ms input grace after stage-up** (`grace` suppresses `breakChain`). Overshoot detection uses `wasInside`/sign-crossing logic — edit with extreme care. *(unchanged in 4.5)*
- **CHATTER** — 5 slots at 72° from −54° (top slot reserved for brand tag), 2 discs live at start, new disc every 2nd stage, stage every 14s, decay 4.0 +2.1/stage. Restrike: ≤20 energy = PERFECT SAVE (+45 energy, +8 pts); ≥80 = +8 energy only; else +30 energy. Score accrues at `3.6 × avgEnergy/100`/s, ×2 during FULL CHATTER (all ≥85). Miss increments `missStreak`; **`fumble()` (−4 all discs) fires only on the 2nd consecutive miss** (one stray tap is free), reset on any restrike. **Note: CHATTER is harder than its scores suggest — do not nerf it, and do not add a time cap. Its difficulty is its cap.** *(unchanged in 4.5)*
- **SWINGBALL** *(R4 targets)* — 45°-view **ellipse orbit** `CX:W/2, CY:300, RX:126, RY:54` (was top-down circle R:104 at CY:312); pole 120→470. Speed 1.7 (+0.05/hit, cap 4.2), zone half-width 0.42 → floor 0.14 (−0.012/hit), perfect = inner 33%, **slop band = 1.7× zone** (mistimed tap = unwind + survive). Wind meter 0→100 (+20 hit / +30 perfect / −25 mistimed); filling it **banks a LOOP** (score ×`(1+loop)`), resets wind, steps `baseSpeed +0.25` / `baseZone −0.03`. 300ms grace after a loop-bank. Break only on a wild tap (beyond slop) or an untapped overshoot — reuses CHAIN's `wasInside`/sign-cross logic **verbatim; angular timing is untouched by R4, only the projection changes**. New R4 visual constants: `COILS:6` (wind-meter rope rings under the pole cap), `wobble` (0–1, decays `−dt*1.8`, cap sway `sin(tick*0.45)*6*wobble`), depth scale `0.8→1.15` far→near. `par:40` PROVISIONAL. Backyard palette PALS[3] + gold accents.
- **SCRAMBLE** *(R1 targets)* — 5 kid slots (`KID_Y:532`), start **4 active (indices 0–3)** (was 3, indices 1–3); **stage-up wakes ANY inactive kid** (was edges [0,4] only — with 4 starters, the 5th kid arrives at the first stage-up). Tap a kid → a lolly **arcs** (~0.45s, `t += dt*2.2`) and adds `THROW_GAIN 34` to their haul (cap 100). Per-kid decay `DECAY_BASE 6.5` (was 5.0) `× decayMul (0.8–1.4)`, `+1.9`/stage (was +1.6), stage every **10s** (was 12). **Score rewards evenness, not volume**: `SCORE_RATE 5.0 × (1 − stddev/50)` over active hauls; **FAIR SHARE ×2** when live ≥2 and all ≥`FAIR_FLOOR 60` within `FAIR_BAND 20`. Empty meter → kid away `CRY_TIME 1.4`s (returns at `PITY 42`) and −1 life; **3 empties = over**. `par:120` PROVISIONAL — likely needs lowering after R1; retune. Party palette PALS[4]. **Deferred from spec:** spatial "greedy kids drift to dropped lollies" — v1 uses directed throws that always connect; per-kid decay carries the greedy-kid pressure.
- **KNUCKLEBONES** *(R5 targets)* — bones tossed on parabolas to `CATCH_LINE:480` from 5 `SLOTS`; base `BASE_LV:560`/`BASE_G:700` (was 660/1000 ⇒ opening airtime ~1.6s, was ~1.32s — noticeably floatier start). **Gold = catch** (tap airborne), **grey = skip** (leave). Missed catch OR grey-bone tap = a drop; **3 drops = over**. `RUNGS` ladder = ONESIES→…→OVER THE FENCE; **all rung `spd` values are now 1.00** (the rung table no longer sets speed — rungs only vary catch/skip counts; decoys from HORSES on). Speed ramps **with time only**: `baseSpd ×1.04` on every rung-up, with an **additional ×1.04 on loop** (≈×1.08 per loop total; was a flat ×1.12 per loop on top of a 1.00→1.45 rung table). `CLEARS_PER_RUNG 2` clean tosses advance a rung. Score `+2×(1+loop)`/catch, `+5×(1+loop)`/clean toss. No rotation sim — a horizontal frame-flip (every 6 ticks) sells the spin. Drawn grey **asphalt court** (chalk catch-line + play circle); PALS[5] amber accents, bone-cream sprites. `par:90` PROVISIONAL — retune post-R5.

---

## 5. Roadmap — implement in this order

### Phase 1 — Fixes ✅ IMPLEMENTED

*All seven items shipped: spacebar guard (space only to `spaceTap` games), resize/orientation handler, font preload, SCAN swept-segment beam collision, `visibilitychange` pause, menu best-cache + duplicate-filter removal, feel tweaks (STACK perfect window scaling, CHAIN stage-up grace, CHATTER free first stray tap).*

### Phase 2 — New games (three, plus menu redesign) ✅ COMPLETE

*SWINGBALL, SCRAMBLE, KNUCKLEBONES + paginated 2×2 menu shipped. Full order in `GAMES`: STACK → SCAN → CHAIN → SWINGBALL → CHATTER → SCRAMBLE → KNUCKLEBONES. Design intents preserved in §4; deferred SCRAMBLE spatial mechanic noted there.*

### Phase 3 — GAUNTLET (free play) ✅ SHIPPED, then ⚠️ SUPERSEDED by Phase 4.5 R3

*Shipped as specced (controller object, par normalisation ÷par ×250 cap 625, interstitials, `arc_gauntlet_best`, first menu tile). The owner has since decided the free-play gauntlet and the daily are the same thing: R3 folds it into TUCK SHOP RUN and removes the free-play entry point. The controller architecture (onOver hook, par normalisation, interstitials, bars) survives — only the free-play mode dies.*

### Phase 4 — THE DAILY (one attempt, shareable) ✅ COMPLETE — **TUCK SHOP RUN**

- A daily seeded gauntlet, **one attempt per calendar day** (local time), enforced via `arc_daily_state`. Individual games remain unlimited free-play. Streak counter in `arc_daily_streak` (+ `arc_daily_lastdone` for continuity).
- **Seeding**: `mulberry32` seeded from the local date (`YYYY-MM-DD`); each game gets an **independent** stream `mulberry32(hash(dayKey) + idx*101)`. Content-seeding only (see §3). Same challenge for everyone, not same replay.
- Daily numbering: `#N` where day 1 = `DAILY_EPOCH` **= 2026-07-16** (ship date, local time).
- **Name**: **TUCK SHOP RUN** (`DAILY_NAME`, owner pick). Other 90s-NZ candidates kept in a menu-copy comment: PLAYLUNCH, THE TUCK RUN, MORNING TEA, AFTER THE BELL.
- **One attempt = consumed at start** (owner choice): `beginDaily()` writes `{dayKey,played:true}` immediately, so bailing mid-run forfeits the day (locked re-entry shows the stored result, or COME BACK TOMORROW if forfeited).
- **Share (both A and B)** — shipped: **B — emoji grid** (⬛ <100 / 🟨 100–199 / 🟧 200–349 / 🟩 350–499 / 🟪 500+ per game, total, streak; `navigator.share({text})` → clipboard fallback + toast; emoji live in shared *text*, not canvas — rule 4 unaffected). **A — share card** (composed card painted to the visible canvas with chrome suppressed → `c.toBlob()` → `navigator.share({files})`; fallback "SCREENSHOT ME" frame).

### Phase 4.5 — REVISION PASS ⬅️ **IMPLEMENT THIS NOW** (owner playtest feedback, all approved)

All five revisions land together in `index.html` as one pass. Verify against the acceptance checks at the end of this section.

**R1 — SCRAMBLE is too easy: dial up both kid count and hunger.**
- `init()`: start with **4 active kids, indices 0–3** (`active:i<4, haul:i<4?70:0`) — was 3 (indices 1–3).
- `stageUp()`: wake **any** inactive kid (scan all kids, wake the first inactive), not just edges `[0,4]`. Net effect: the 5th kid arrives at the first stage-up (~10s in).
- Constants: `DECAY_BASE 5.0 → 6.5`, `DECAY_GROWTH 1.6 → 1.9`, `STAGE_INTERVAL 12 → 10`.
- Update the game's header comment to record the dial-up; leave `par:120` marked provisional (it will likely need to come down — owner to retune).
- Do **not** touch `THROW_GAIN`, scoring, FAIR SHARE, lives, or the ready-screen copy.

**R2 — iOS standalone (home-screen web app) bug: the screen shifts/jumps upward during play.**
Reported on iPhone with the URL added to the Home Screen; not a zoom — the whole viewport moves up quickly, and it reads as the game doing it. Root causes to eliminate: iOS rubber-band scroll, the double-tap scroll/zoom heuristic under rapid tapping, and `100vh` misbehaviour in standalone mode. Fix belt-and-braces:
- `<head>`: extend the viewport meta with `viewport-fit=cover`; add `apple-mobile-web-app-capable`, `mobile-web-app-capable`, `apple-mobile-web-app-status-bar-style` (black-translucent), and `theme-color #07050a` metas.
- CSS: `html,body{overscroll-behavior:none}`; body becomes `position:fixed; inset:0; width:100%; height:100%; height:100dvh` (keep the flex centering); add `-webkit-touch-callout:none`; give the canvas `touch-action:none` too.
- JS (top of file, near `resize()`): non-passive `touchmove` listener on `document` calling `preventDefault()`; non-passive `touchstart` `preventDefault()` when the target is the canvas; `gesturestart` `preventDefault()`; a `scroll` listener snapping `window.scrollTo(0,0)`; and `visualViewport.addEventListener('resize', resize)` when available.
- Safe because all game input runs on pointer events, which dispatch before touch defaults; comment this inline (rule: comment hard-won constraints).
- This becomes standing rule 10 in §1.
- **Side note (owner):** the accidental "screen shifts mid-game" effect is liked as a *deliberate* Phase 5 mutator idea — logged there as SCREEN QUAKE. Do not build it in this pass.

**R3 — Merge the free-play GAUNTLET into the daily. TUCK SHOP RUN is the only gauntlet.**
Owner accepts that this removes free-play gauntlet runs; the seven individual games stay unlimited.
- **Menu**: delete the GAUNTLET grid card, the `MENU_ITEMS` array, and the slim daily banner (`dailyBannerRect`/`drawDailyBanner`). **Page 0 becomes a single full-page TUCK SHOP RUN tile** (roughly `{x:15, y:172, w:W-30, h:344}` under the unchanged brand header); the seven game cards start on **page 1** (2×2 grids over `GAMES`, `PER_PAGE=4`, grid top ~y:190 now the banner is gone ⇒ 3 pages total). `menuPageCount() = 1 + ceil(GAMES.length/4)`; `menuCards()` returns `[]` on page 0. Dots/arrows/swipe/arrow-keys unchanged.
- **Tile content** (gold treatment, PALS[5], pulsing border, mark chip, drawn only — rule 4): `DAILY_NAME`, `DAILY #N`, "EVERY GAME · ONE SCORE" / "ONE ATTEMPT A DAY", streak when >0. Unplayed: pulsing `► TAP TO PLAY`. Played: `TODAY <total>` + `STREAK n` + `► TAP TO VIEW RESULT` (tap reopens the locked result via `beginDaily()`'s existing locked path). Forfeited: `ATTEMPT USED` / `COME BACK TOMORROW` (tile still tappable to the same locked screen).
- **Controller** becomes daily-only: delete `begin()`, `isGauntlet`, `bestKey`, `best`/`newBest`, and `drawEnd()`; `finish()` keeps only the daily branch (streak, emoji grid, `arc_daily_state` save); `launch()` always seeds; `render()`'s `end` state always draws `drawDailyEnd()`; `drawStrip()` labels with `DAILY_NAME` instead of "GAUNTLET". Input: remove the `isGauntlet` card branch and `dailyBannerRect` hit-test from `onUp`; page-0 tile tap → `GAUNTLET.beginDaily()`.
- `refreshMenuBests()` iterates `GAMES` only. `drawMenuIcon`'s `'gauntlet'` branch and palette-map entry are dead — remove them. Keep `PALMAP`/`gamePal` in the controller (result bars use them).
- `arc_gauntlet_best` is retired per §2 — never read or written again; leave stored values alone.
- Update the file's banner comment and the `GAMES` comment ("daily order").

**R4 — SWINGBALL redesign: 45° view, rope fixed at the top.**
Two owner complaints: (a) the top-down circle view doesn't work because the pole is drawn side-on; (b) the wind spiral climbing the pole is wrong — the rope should stay at the top. Mechanics, scoring, and the LOOP economy are untouched; this is projection + reskin.
- **Orbit becomes a squashed ellipse** (the arc reads as parabolic from a 45° viewpoint): replace `R:104` with `RX:126, RY:54` at `CX:W/2, CY:300`; `ballPos(a) = {x: CX+cos(a)*RX, y: CY+sin(a)*RY}`. Orbit ring, hit zone, and perfect arc all draw with `ctx.ellipse(CX,CY,RX,RY,0,start,end)` using the same angular values as before — **the angular timing logic (`wrap`, speeds, zone widths, the `wasInside`/sign-cross overshoot block) is copied verbatim; only the projection to screen changes.** The natural fore-shortening (ball visibly faster crossing front/back, slower at the sides) is correct and desired.
- **Depth ordering**: `sin(angle) >= 0` = near side. Far half (ball + rope + trail) draws **before** the pole; near half draws **after** it, so the ball passes behind then in front. Depth-scale the ball ~`0.8` (far) → `1.15` (near). The hit zone + perfect arc draw **in front of the pole** (floating UI) so the timing read is never occluded — readability beats occlusion realism here.
- **Rope stays at the pole top, permanently.** Remove `windHeight()` and the climbing spiral entirely. The rope draws from the pole-top attachment to the ball every frame.
- **Wind meter reskin**: a **coil of rope rings stacked under the pole cap** — `COILS:6` small ellipses; `round(wind/WIND_MAX*COILS)` rings lit (gold at ≥75% full, matching the old spiral's gold-near-full), unlit rings faint. Loop-bank `flash` boosts the coil glow.
- **Wobble on every direction change** (i.e. every time the ball is hit back — both `clean()` and `mistime()` flip `dir`): set `this.wobble=1` there; decay `wobble -= dt*1.8` in `update()`; cap + coil sway horizontally by `sin(tick*0.45) * 6 * wobble`. The pole shaft itself stays planted.
- Ready-screen copy: second line becomes "CLEAN HITS COIL THE ROPE AT THE TOP", third "FILL THE COIL TO BANK A LOOP". Banner repositioned above the ellipse (~`CY-RY-92`). The menu icon (rope from pole top to ball) already matches — leave it.

**R5 — KNUCKLEBONES is too hard: slow it down; speed ramps over time only.**
- **Flatten the rung speed table**: every `RUNGS` entry's `spd` becomes `1.00` (rungs continue to vary catch/skip counts and decoys — that's the ladder's difficulty axis now).
- **Slower opening toss**: `BASE_LV 660 → 560`, `BASE_G 1000 → 700` (airtime ~1.6s vs ~1.32s; peak height stays ~equivalent).
- **Time-only ramp** in `rungUp()`: `baseSpd *= 1.04` on every rung-up, plus an **additional** `*= 1.04` when the ladder loops (≈×1.08 per loop total; replaces the flat `×1.12` loop step). Nothing else sets speed.
- Update the game's header comment; `par:90` stays provisional — retune after the pacing change.

**Phase 4.5 acceptance checks**
1. SCRAMBLE opens with 4 kids; a 5th appears ~10s in; meters visibly drain faster than before.
2. On an iPhone home-screen install, sustained rapid tapping in CHATTER/SCRAMBLE never moves, scrolls, or zooms the viewport.
3. The menu opens on the full-page TUCK SHOP RUN tile; swiping right reveals game pages (4 cards, then 3); no GAUNTLET card or slim banner anywhere; after today's attempt the tile shows the result/streak and reopens the locked result on tap.
4. SWINGBALL: ball sweeps an ellipse, passing behind the pole (smaller) on the far half and in front (larger) on the near half; rope always hangs from the pole top; the coil under the cap fills with clean hits, goes gold near full, and the cap+coil wobble on every hit; free-play scoring/feel otherwise identical, and the daily still seeds it deterministically.
5. KNUCKLEBONES: the first tosses are noticeably floatier; speed creeps up only as rungs/loops accrue; ONESIES on loop 1 is faster than ONESIES on loop 0.
6. Free play remains byte-identical outside the changed games: `srng` stays null, `onOver` stays null, STACK/SCAN/CHAIN/CHATTER behaviour untouched.
7. `node --check` (or equivalent) passes on the extracted script; no references to `begin(`, `isGauntlet`, `arc_gauntlet_best`, `MENU_ITEMS`, `dailyBannerRect`, or `windHeight` remain.

### Phase 5 — Mutators (opt-in, experimental)

- **Free-play only, opt-in, never in the daily.** Gate the whole feature behind a single flag `const MUTATORS_ENABLED = true` so it can be switched off if playtesting kills it.
- Entry point: a small **paper fortune teller** icon on the menu (drawn, per rule 4). Tapping it plays a short fold/pick animation and deals a mutator; the player can accept (play any game with it) or dismiss.
- Mutator runs score to `arc_mut_best_<gameid>_<mutid>` — **never** to clean bests or the daily.
- Start with three, one line of code each where possible: GOLD RUSH (gold spawn chance ×2), MIRROR (CHAIN/SWINGBALL direction flips each stage), TREMOR (STACK slider sinusoidal wobble). Pass the active mutator to games as an optional `modifiers` object on `init()` — games ignore what they don't understand.
- **Candidate (owner request, from the R2 bug): SCREEN QUAKE** — the whole playfield drifts/jolts vertically during play (a deliberate, controlled version of the iOS bug the owner enjoyed). Implement as a render-space translate so input mapping is unaffected, or remap taps through the same offset. Candidate only — owner to confirm at Phase 5.

### Phase 6 — The creature

- A pixel tamagotchi-style creature living on the menu screen (bottom area, near the brand footer — on the **game pages**, not the daily tile page, unless space allows). Drawn sprite, palette-aware, idle animation on `tick`.
- **Growth-only. It never decays, never suffers, never guilts.** It sleeps when you're away.
- Feeding: XP from play — e.g. +1 per 100 normalised points across any mode, +25 per daily completed. Stages at XP thresholds (egg → hatchling → kid → teen → legend), each a visibly different sprite. Purely cosmetic companion — it gates nothing, sells nothing.
- State in `arc_creature`, device-bound localStorage — accepted for now.
- Creature reacts: hops on NEW BEST, celebrates daily completion, sleeps after 30s menu idle. No café framing — the creature is the hub's only meta-layer.

---

## 6. Decision log (settled — do not reopen)

| #   | Decision |
| --- | -------- |
| D1  | SWINGBALL is endless (loop-banking multiplier); it is a separate game from CHAIN, which remains in the roster. Fresh best key, no legacy inheritance. |
| D2  | Seven-game trial roster; no pruning yet. Menu redesign accepted. *(Revised by D12: the gauntlet no longer appears in the card grid.)* |
| D3  | Global par constants per game; par-based normalisation (÷par ×250, cap 625); no time caps anywhere; CHATTER par must be set from real play data, not rank tables. |
| D4  | Daily = one gauntlet attempt per day (consumed at start; bailing forfeits); individual games unlimited. Content-seeding only. Name settled: **TUCK SHOP RUN**. `DAILY_EPOCH = 2026-07-16`. |
| D5  | Mutators are opt-in, free-play only, feature-flagged, separate best tables, fortune-teller UI. Excluded from the daily until playtesting says otherwise. |
| D6  | Creature is growth-only, device-bound localStorage, cosmetic. |
| D7  | Café hub metaphor is dead. Creature only. |
| D8  | Social = share card (A) + daily emoji grid (B). No backend, no accounts. |
| D9  | **The free-play GAUNTLET is merged into the daily** (Phase 4.5 R3). TUCK SHOP RUN is the only gauntlet, presented as a full-page menu tile on page 0 with the game cards from page 1. `arc_gauntlet_best` retired in place. Individual games stay unlimited. |
| D10 | **SWINGBALL is viewed at 45°** (Phase 4.5 R4): elliptical orbit with pole-depth ordering; the rope is fixed at the pole top forever (it never climbs); the wind meter is a coil under the cap; cap+coil wobble on every direction change. Angular timing logic unchanged. |
| D11 | **KNUCKLEBONES speed ramps with time only** (Phase 4.5 R5): flat rung speed table, slower base toss (560/700), ×1.04 per rung-up with an extra ×1.04 on loop. |
| D12 | **iOS standalone viewport is hard-locked** (Phase 4.5 R2): fixed body, dvh, touch-default suppression. This is standing rule 10. The accidental screen-shift effect is preserved as the Phase 5 SCREEN QUAKE mutator candidate. |
| D13 | **SCRAMBLE dial-up** (Phase 4.5 R1): 4 starting kids, wake-any-kid stage-ups, decay 6.5/+1.9, 10s stages. |

## 7. Open decisions (owner to resolve — flag, don't guess)

- Final pars for all seven games after ~a week of owner playtesting (all seven currently `// PROVISIONAL`: STACK 20, SCAN 35, CHAIN 50, SWINGBALL 40, CHATTER 600, SCRAMBLE 120, KNUCKLEBONES 90). SCRAMBLE and KNUCKLEBONES pars in particular need a fresh look after R1/R5.
- Which game(s) get pruned post-trial, and whether the roster returns to four.
- SCRAMBLE/CHATTER attention-mechanic overlap: tolerated for the trial; revisit at prune time.
- Phase 5 mutator shortlist confirmation, incl. the SCREEN QUAKE candidate (D12 note).

## 8. Style conventions

- Compact canvas code; short locals (`p` palette, `g` gradient/game, `s` size/slot).
- All text in `Press Start 2P` at 5–30px; set `textAlign` explicitly before every text block.
- Glow via `shadowColor`/`shadowBlur`; always reset `shadowBlur=0` after.
- Keep each game's constants at the top of its object as UPPERCASE members (see CHAIN/CHATTER) so tuning never requires reading gameplay code.
- Comment hard-won constraints inline (see the home-icon comment) — future sessions read comments, not commit history.
