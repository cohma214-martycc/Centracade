# CLAUDE.md — CENTRARCADE

Single-file HTML5 arcade hub of quick Centrapay-branded games built on a shared 16-bit engine. One-thumb, portrait, mobile-first. The entire app is `index.html`.

**Phases 1–4.5 shipped**: fixes, the seven-game roster + paginated menu, the gauntlet, the once-a-day seeded **TUCK SHOP RUN** daily, and the 4.5 revision pass (R1 SCRAMBLE dial-up, R3 daily-only gauntlet + full-page daily tile, R4 SWINGBALL 45° view, R5 KNUCKLEBONES pacing).

**Phase 5's game roster is complete**: CHAIN is **pruned (Unit 5.0, shipped)**, **GUNGE has shipped (Unit 5.2)**, **HOWLER has shipped (Unit 4)**, **TAZO TYCOON has shipped (Unit 5)**, **DAIRY WARMER has shipped (Unit 6)**, and **WEAVER has shipped (Unit 7)**. The old roster was stylistically narrow (tap-timing and tap-target). Phase 5 added swipe-analog (HOWLER — the first swipe game in the hub), grid-rotation (GUNGE — the first puzzle), directional-merge (TAZO — the first 2048-style game), spatial-packing (DAIRY — the first Blockdoku-style packer), and path-drawing (WEAVER — the first continuous-drag game). Roster went **7 → 6 → 11** — now at **11**. Remaining Phase 5 work: the HOWLER revision TODO (§5.1) and the roster-wide par retune (§5.6/§7).

> **Branch policy** (owner instruction, 2026-07-18): all development and final pushes go to the repository's **default branch**. The former feature-branch-per-unit workflow is retired — commit units directly to the default branch.

This file has three jobs: (1) hard invariants you must never break, (2) an accurate map of the current code, (3) the settled roadmap. Decisions in the **Decision log** are final — do not relitigate them; implement them.

> **Source of truth**: `index.html`. Where this doc and the code disagree, the code wins and this doc is wrong — fix it in the same commit.

---

## 1. Non-negotiable rules

1. **Single file.** All markup, CSS, and JS in one HTML file. No libraries, no build step. Only external dependency is the `Press Start 2P` Google font import.
2. **Logical resolution 360×640.** All game code works in this coordinate space. DPR capped at 2. Physical scaling handled once via `sc` + `ctx.setTransform` — never per-frame math against window dimensions.
3. **The Centrapay mark is settled geometry.** `drawMark()` uses six blocks on a 100×71 grid:
   `[[54,0,29,18],[82,17,18,18],[17,18,33,18],[0,35,18,18],[50,35,33,18],[17,53,29,18]]`, corner radius `0.22 × block height`, skipped below 4px. Never adjust these numbers.
4. **Icons are drawn, not glyphs.** Unicode symbols render blank on too many mobile fonts (this bit us with the house icon). Any new icon — rotate button, wind arrow, fortune teller, creature — is drawn with canvas paths. (The mute speaker glyph is the one grandfathered exception; do not add more.)
5. **Audio only after a gesture.** `AudioContext` created lazily in `audio()`, resumed on tap (iOS requirement). All synthesis routes through `beep()`.
6. **Legacy best-score keys keep migrating.** `loadBest(key, legacy)` folds in old standalone keys: STACK ← `cps2_best`, CHAIN ← `chain_best`. Never remove this. **Pruned games keep their keys** — see rule 10.
7. **All localStorage access wrapped in try/catch.** Private browsing and sandboxes throw.
8. **`'use strict'`, no implicit globals.**
9. **New games must reuse the shared systems** — `drawBG`, `drawReadyScreen`, `drawOverScreen`, FX (`burst`/`pop`), rank helpers, chrome. Cohesion is the point of the hub. A bespoke over-screen is a code smell. This holds for Phase 5: the games may *play* differently, they must not *look* like guests.
10. **Pruning removes a game from the roster, never its data.** Drop it from `GAMES`/`PALMAP`; leave its `bestKey` and any legacy key untouched and documented as dormant (§2). A returning player's history must survive a prune and a re-add.
11. **iOS standalone viewport is hard-locked.** Fixed body, `100dvh`, `touch-action:none`, and the `touchmove`/`touchstart`/`gesturestart` preventDefaults exist because home-screen web-app mode rubber-bands and zoom-shifts without them. Game input runs on pointer events, which dispatch before touch defaults — that is why swallowing the touch defaults is safe. Do not "tidy" these away.

---

## 2. localStorage keys

| Key | Purpose |
| --- | --- |
| `arc_mute` | Mute toggle ('1'/'0') |
| `arc_stack_best` | STACK best (legacy: `cps2_best`) |
| `arc_scan_best` | SCAN best |
| `arc_chatter_best` | CHATTER best |
| `arc_swing_best` | SWINGBALL best (fresh — never inherited CHAIN's history) |
| `arc_scramble_best` | SCRAMBLE best |
| `arc_bones_best` | KNUCKLEBONES best |
| `arc_daily_state` | JSON: `{dayKey, played, total, result, emojiGrid}` — enforces one attempt/day (consumed at start) |
| `arc_daily_streak` | Consecutive daily completions |
| `arc_daily_lastdone` | dayKey of the last completed daily (for streak continuity) |
| **Dormant** | |
| `arc_chain_best` / `chain_best` | CHAIN best. **CHAIN is pruned in Phase 5 — these keys are retained, unread, never deleted** (rule 10). If CHAIN ever returns, `loadBest('arc_chain_best','chain_best')` restores history intact. |
| `arc_gauntlet_best` | Free-play gauntlet best. **Dead since 4.5 R3** (free-play gauntlet removed). Not written, not read, not deleted. |
| **Phase 5 (new)** | |
| `arc_howler_best` | HOWLER best |
| `arc_gunge_best` | GUNGE best |
| `arc_tazo_best` | TAZO TYCOON best |
| `arc_dairy_best` | DAIRY WARMER best |
| `arc_weaver_best` | WEAVER best |
| **Phase 6–7 (planned)** | |
| `arc_mut_best_<gameid>_<mutid>` | Per-game per-mutator bests, separate from clean bests |
| `arc_creature` | JSON creature state: `{xp, stage, hatchedAt}` |

---

## 3. Architecture (current code)

### Shared engine (top of file)

- `PALS` — six palettes `{top, bot, ring, acc, glow, sky}`: `0` ember/orange, `1` teal, `2` purple, `3` backyard green, `4` party pink, `5` amber.
- Helpers: `wrap(a)` (angle → −π..π), `rr(x,y,w,h,r)` (rounded-rect path; caller fills/strokes), `segDist(px,py,ax,ay,bx,by)` (point→segment distance — SCAN's swept-beam collision and WEAVER's hazard-vs-thread test), `segsCross(ax,ay,bx,by,cx,cy,dx,dy)` (strict segment-segment proper-crossing test, shared-endpoint = no cross — WEAVER's thread self-crossing test; added Unit 7 because point-to-segment distance is the wrong primitive for two thin segments crossing).
- Audio: `beep`, semantic wrappers `sHit/sPerfect/sGold/sBad/sStage/sTick`; `buzz(ms)` haptics (no-op on iOS Safari — never rely on it).
- FX: module-level `parts`/`pops` with `burst()`, `pop()`, `updateFX()`, `drawFX()`. Games clear both arrays in `init()`.
- Background: `drawBG(pal, skylineY)` — gradient, 4px scanlines, two parallax star layers, procedural skyline, vignette.
- Chrome: `drawChrome(showHome)`, `homeRect()`, `muteRect()`, `inR()`.
- Screens: `drawReadyScreen(pal, name, lines)`, `drawOverScreen(...)`.
- Ranks: `rankFor(ranks, s)` / `nextRank(ranks, s)` over `{s, t}` tables.
- Daily RNG: `mulberry32(a)`, `hashStr(s)`, `dayKeyOf(date)`, and `srnd(g)` = `g.srng ? g.srng() : Math.random()`.

### Game interface contract

Every game is an object literal implementing:

```
id        string, unique, lowercase
name      display name
tag       one-line menu subtitle
bestKey   localStorage key
par       number — global par for gauntlet normalisation (§5 Phase 3)
ranks     [{s: threshold, t: 'TITLE'}, ...] ascending
init()    full state reset (incl. parts=[]; pops=[]), sets st='ready'
start()   st: 'ready' → 'play'
update(dt) dt seconds, clamped to 0.05 by the loop
render()  draws everything incl. HUD and ready/over overlays
tap(x,y)  ready-start, over-restart (450ms debounce via overAt), gameplay input
seed(rng) accept a seeded PRNG (daily). srng=null ⇒ free play
onOver    optional callback the gauntlet controller sets to hook game-over
spaceTap  optional bool; true ⇒ spacebar routes to tap() (positionless games only)
```

State machine: `'ready' → 'play' → 'over'`. Restart requires `performance.now() - overAt > 450`.

- **`par`** — all provisional (`// PROVISIONAL`): STACK 20, SCAN 35, SWINGBALL 40, CHATTER 600 (owner median — CHATTER plays far harder than its ranks), SCRAMBLE 120, KNUCKLEBONES 90.
- **`seed(rng)`** — **content-seeding only**: spawn order, zone placements, toss shuffles, grid generation. Physics and cosmetics stay live. Currently routed through `srnd`: SCAN spawn side/type/y, SWINGBALL zone, SCRAMBLE kid mix, BONES toss shuffles. STACK/CHATTER are already deterministic. `srng=null` ⇒ free play stays byte-identical; cleared at every non-daily entry.
- **`onOver`** — each game calls `if(this.onOver) this.onOver()` at its over-transition; the controller registers it and clears it one-shot (double-fire safe); free play resets it to null.

#### Contract extension — pointer stream (Phase 5) ✅ WIRED (Unit 2)

The router implements the stream below (`gameGesture` + `gameDown()` in the Input section). **HOWLER (Unit 4) was the first consumer; WEAVER (Unit 7) is the first FULL consumer** — the only game where all three of `press`/`drag`/`release` carry live state (HOWLER's drag only updates the aim ghost; TAZO omits `drag` entirely). Games without `press` stay on tap-on-pointerdown, unchanged. The stream is forwarded in **both** `mode==='game'` and the gauntlet-playing path (Phase 5 games join the daily), so a `press`-game works in the daily too.

Four of the five Phase 5 games need more than a tap. Add **one** optional extension rather than five bespoke input hacks:

```
press(x,y)    optional — pointerdown
drag(x,y)     optional — pointermove while down
release(x,y)  optional — pointerup / pointercancel
```

Rules:
- A game **declares** the stream by defining `press`. If `press` is absent, behaviour is exactly as today: `tap(x,y)` on **pointerdown**, zero latency. Nothing about STACK/SCAN/CHATTER/SCRAMBLE/BONES/SWINGBALL changes.
- If `press` is defined, the router forwards down/move/up and **does not call `tap()`** during play. The game owns its gesture. `ready`-start and `over`-restart still route through `tap()` (the router calls `tap()` on release when `st!=='play'`) so the 450ms debounce and the shared ready/over screens keep working identically.
- Chrome (home/mute) is tested **before** the stream, on pointerdown, as it is now.
- `pointercancel` must call `release()` with the last known coords — iOS fires it on interruption and a game left mid-gesture will otherwise hang a drag forever.
- No Phase 5 game sets `spaceTap` (all are positional).
- Users: HOWLER (swipe), TAZO (swipe), WEAVER (continuous drag). GUNGE and DAIRY stay tap-only.

The existing `menuDrag` logic already proves the pattern: menu resolves on `pointerup` so a horizontal drag reads as a page swipe. Keep them separate — menu drag and game pointer stream must not share state.

### Router

- `mode`: `'menu' | 'game' | 'gauntlet'`; `cur` = active game (in gauntlet, the game the controller is running).
- Menu card tap → `cur=g; cur.onOver=null; cur.srng=null; cur.init(); mode='game'`. Home → discard state, back to menu. Global `tick` drives all pulse/shimmer phases. Frame loop clamps dt to 0.05.
- **Menu (4.5 R3)**: **page 0 is a full-page TUCK SHOP RUN tile** (`dailyTileRect()`, `drawDailyTile()`); game cards run in a 2×2 grid from **page 1** (`PER_PAGE=4`, `menuCards()` returns `[]` for page 0, `menuPageCount() = 1 + ceil(GAMES.length/PER_PAGE)`). Card selection resolves on `pointerup` so a horizontal drag reads as a page swipe; **game taps fire on `pointerdown`**. Swipe, tappable dots/arrows (`drawArrow()`), and ←/→ arrow keys all page.
- **`paused`** (module-level): set on `visibilitychange` while a game is mid-play (`game` **or** `gauntlet`); the loop skips `update`, keeps rendering the frozen frame, draws `drawPauseOverlay()`. A tap or space resumes; home/mute stay live.
- **`GAUNTLET`** — controller outside the game contract. **Daily-only since 4.5 R3**: `beginDaily()` is the sole entry; there is no free-play gauntlet. It sequences `GAMES` in order to first game-over each via `onOver`, always seeds (`cur.seed(mulberry32(seedBase + idx*101))`) before `init()`, normalises (`contribOf` = `round(score/par × NORM 250)`, cap `CAP 625`) into a running `total`, and renders `playing → interstitial → end → sharecard`. Reads only public fields (`score`/`par`/`ranks`/`name`); never mutates game internals.
- **Bests cached** on menu entry via `refreshMenuBests()` → `menuBests`; daily tile state cached via `refreshDaily()` → `dailyCache`. Neither is re-read per card per frame. Both refresh on home-exit.
- **`sc` is recomputed by `resize()`** on `resize`/`orientationchange`/`visualViewport` only — backing store stays `W*dpr`, the single `setTransform` is never touched per frame (rule 2).
- **Frame loop starts only after the font is ready**: `document.fonts.load('8px "Press Start 2P"')` raced with a 1.5s timeout fallback, to kill the FOUT.

### Palettes and roster growth (read before Phase 5)

**Ten palettes** (six original + four Phase-5), eleven games soon. Current assignment (the single module-level `PALMAP` (id→index) read via `gamePal(id)` — §7): STACK `0`, SCAN `1`, SWINGBALL `3`, CHATTER `3` (menu) cycling `CHATTER_PALS=[2,3,4,5,0,1]` in-play, SCRAMBLE `4`, BONES `5`. **PALS[2] (purple) is freed by CHAIN's removal and becomes the hub palette** — menu, interstitials, end card, pause.

With CHAIN gone, **CHATTER is the only palette-cycling game**. Its stage cycle is now an **explicit index list** (`CHATTER_PALS`, Unit 2) instead of `(2+stage)%PALS.length` — the list reproduces the old sequence byte-for-byte (`2,3,4,5,0,1,…`) but is decoupled from `PALS.length`, so appending palettes can no longer silently recolour CHATTER's stages. (Note: the shipped list preserves current colours; it is **not** the illustrative `[3,4,5,0,1,2]` that once appeared here, which would have recoloured stage 0.)

Phase 5 palette entries (added Unit 2): HOWLER reuses `0` (`PALMAP.howler=0`); **`6` GUNGE** slime lime (`PALMAP.gunge=6`), **`7` TAZO** electric holo blue (`PALMAP.tazo=7`; also the `TAZO_COL` tier ramp derives from this family), **`8` DAIRY** warm caramel (`PALMAP.dairy=8`), **`9` WEAVER** navy + red-sock (**now referenced** — `PALMAP.weaver=9`; shipped as-is with Unit 7, hues still `// PROVISIONAL`). (Palette lookup is the single module-level `PALMAP`/`gamePal` — §7; the old per-map "menu-icon + card lookups" are consolidated.) Drawing a surface directly (asphalt-court precedent) is still fine where a full palette isn't warranted.

---

## 4. Current games — tuning constants

*All constants provisional unless stated. Retune from play data, not from feel-in-a-vacuum.*

- **STACK** (`par:20`, PALS[0]) — block height `BH 42`, start width `SW 264`, perfect window `PFCT 5 + spd*0.4`px (scales with speed), speed 2.4 → cap 9.5 (+0.13/block). Perfect preserves width; ≤6px landing = topple. Deterministic — no seeded content.
- **SCAN** (`par:35`, PALS[1]) — 3 lives (escape = life lost); $1/$2/$3 chips + ~15% QR (+5 gold). Global speed mult `1 + score*0.02`. Beam extend/retract; collision tests the **segment swept by the tip** each frame (`segDist`), not just the tip point.
- **CHATTER** (`par:600`, PALS cycling) — 5 slots at 72° from −54° (top slot reserved for the brand tag), 2 discs live at start, new disc every 2nd stage, stage every `STAGE_INTERVAL 14`s, decay `DECAY_BASE 4.0` +`DECAY_GROWTH 2.1`/stage. Restrike: ≤20 energy = PERFECT SAVE (+45 energy, +8 pts); ≥80 = +8 energy only; else +30 energy. Score accrues at `3.6 × avgEnergy/100`/s, ×2 during FULL CHATTER (all ≥85). `fumble()` (−4 all discs) fires only on the **2nd consecutive miss** — one stray tap is free. **CHATTER is harder than its scores suggest — do not nerf it, and do not add a time cap. Its difficulty is its cap.**
- **SWINGBALL** (`par:40`, PALS[3] + gold) — **45° ellipse view (4.5 R4)**: orbit `RX 126`/`RY 54` at `CY 300`, pole `POLE_TOP 120`→`POLE_BOT 470`. Depth ordering: far half draws behind the pole, near half in front; ball scales 0.8→1.15. Cap and rope coil **sway** on a hit (`poleSway()`, `wobble`); the shaft stays planted. Speed 1.7 (+0.05/hit, cap 4.2), zone half-width 0.42 → floor 0.14 (−0.012/hit), perfect = inner 33%, **slop band `SLOP 1.7`× zone** (mistimed tap = unwind + survive). Wind meter 0→`WIND_MAX 100` (+20 hit / +30 perfect / −25 mistimed), drawn as a `COILS 6` rope coil under the cap; filling it **banks a LOOP** (score ×`(1+loop)`), resets wind, steps `baseSpeed +0.25` / `baseZone −0.03`. 300ms grace after a loop-bank. Break only on a wild tap (beyond slop) or an untapped overshoot.
- **SCRAMBLE** (`par:120`, PALS[4]) — **4.5 R1 dial-up**: 5 kid slots (`KID_Y 532`), **start 4 active (indices 0–3)**, stage-up **wakes any inactive kid** (the 5th arrives at the first stage-up). Tap a kid → a lolly **arcs** (~0.45s, `t += dt*2.2`) and adds `THROW_GAIN 34` to their haul (cap 100). Per-kid decay `DECAY_BASE 6.5 × decayMul (0.8–1.4)`, `+DECAY_GROWTH 1.9`/stage, `STAGE_INTERVAL 10`s. **Score rewards evenness, not volume**: `SCORE_RATE 5.0 × (1 − stddev/50)` over active hauls; **FAIR SHARE ×2** when live ≥2 and all ≥`FAIR_FLOOR 60` within `FAIR_BAND 20`. Empty meter → kid away `CRY_TIME 1.4`s (returns at `PITY 42`) and −1 life; **3 empties = over**. `par:120` **likely too high after R1 — expect to lower it**. **Deferred from spec:** spatial "greedy kids drift to dropped lollies"; per-kid decay carries that pressure for now.
- **KNUCKLEBONES** (`par:90`, PALS[5] accents on a drawn asphalt court) — **4.5 R5 pacing**: bones tossed on parabolas to `CATCH_LINE 480` from 5 `SLOTS`; `BASE_LV 560`/`BASE_G 700` (floatier than the original 660/1000). **`RUNGS` speeds are flat 1.00** — rungs vary catch/skip counts only, and **speed ramps with time**: `baseSpd ×1.04` per rung-up, plus an extra `×1.04` on loop (≈×1.08 total). **Gold = catch** (tap airborne), **grey = skip** (leave). Missed catch OR grey-bone tap = a drop; **3 drops = over**. Ladder ONESIES→…→OVER THE FENCE (decoys from HORSES on); `CLEARS_PER_RUNG 2` clean tosses advance a rung, then it loops. Score `+2×(1+loop)`/catch, `+5×(1+loop)`/clean toss. No rotation sim — a horizontal frame-flip (every 6 ticks) sells the spin.

---

## 5. Roadmap

### Phases 1–4 ✅ SHIPPED

1. **Fixes** — spacebar guard (positionless games only), resize/orientation handler, font preload, SCAN swept-segment beam, `visibilitychange` pause, menu bests caching, feel tweaks (STACK speed-scaled perfect window, CHAIN grace, CHATTER second-miss fumble).
2. **New games + menu redesign** — SWINGBALL, SCRAMBLE, KNUCKLEBONES; paginated 2×2 card menu.
3. **GAUNTLET** — par normalisation (`÷par ×250`, cap 625), interstitials, end card. *Superseded by 4.5 R3: the free-play gauntlet is gone; the machinery lives on as the daily.*
4. **THE DAILY — TUCK SHOP RUN** — one seeded attempt per local day, consumed at start (bailing forfeits), `#N` from `DAILY_EPOCH 2026-07-16`, streak, emoji grid + share card.

### Phase 4.5 — Revision pass ✅ SHIPPED

- **R1 — SCRAMBLE dial-up**: 4 starting kids, wake-any-kid stage-ups, decay 6.5/+1.9, 10s stages.
- **R3 — Daily-only gauntlet**: free-play gauntlet removed; `arc_gauntlet_best` retired; menu page 0 became the full-page TUCK SHOP RUN tile; game cards moved to page 1+.
- **R4 — SWINGBALL 45° view**: ellipse orbit, depth-ordered ball, swaying cap/coil.
- **R5 — KNUCKLEBONES pacing**: flat rung speeds, floatier toss, time-only speed ramp.

---

### Phase 5 — Prune CHAIN + five different-style games ⬅ **IN PROGRESS**

**Why**: every current game is a tap — either timing a moving thing (STACK, SWINGBALL, CHATTER) or hitting a target (SCAN, SCRAMBLE, BONES). The hub is cohesive but monotonous over a 5–10 min daily. Phase 5 buys variety of *verb*, not just of theme.

**Order of work**: (1) prune CHAIN ✅ **shipped**, (2) pointer-stream contract extension ✅ **shipped (Unit 2)**, (3) palette-cycle fix + new palettes ✅ **shipped (Unit 2)**, (4) games in the order below — GUNGE ✅ **shipped (5.2)**, HOWLER ✅ **shipped (Unit 4)**, TAZO ✅ **shipped (Unit 5)**, DAIRY ✅ **shipped (Unit 6)**, WEAVER ✅ **shipped (Unit 7)**. All five landed; each remains independently prunable.

#### 5.0 — Prune CHAIN ✅ SHIPPED

As-shipped (this is the record; code in `index.html` is the source of truth):

- **`CHAIN` object removed**; `chain` dropped from `GAMES`, `PALMAP`, both `drawMenuIcon`/card palette lookups, and its `drawMenuIcon` branch deleted. `refreshMenuBests` no longer special-cases `chain` (only STACK's `cps2_best` legacy remains).
- **`arc_chain_best` and `chain_best` retained, dormant** (rule 10, §2) — unread, never deleted, present nowhere in `index.html`. SWINGBALL keeps its fresh `arc_swing_best` with no inheritance (D1).
- **SWINGBALL now owns the `wasInside`/sign-crossing overshoot core** as the reference implementation; its comments say so and the "ported verbatim from CHAIN" note is gone. **Logic untouched** — only comments changed.
- **PALS[2] is the hub palette** — menu title glow, interstitials, end card, pause. It is unmapped in **`PALMAP`** (no game's menu/card lookup points at it; CHATTER's card is `3`), but **not** unmapped in play: CHATTER's stage cycle opens on it (see §7). "Freed by CHAIN's prune" means freed from `PALMAP`, not from every code path.
- **Menu subtitle derives from `GAMES.length`** → renders `6 QUICK GAMES · ONE THUMB` (digit, not a spelled word — owner-chosen). Can't drift again.
- Roster: STACK → SCAN → SWINGBALL → CHATTER → SCRAMBLE → KNUCKLEBONES (6). Daily is 6 games until the new ones land, then 11.
- **Beyond spec**: the source-file `GAME N` section labels were resequenced 1–6 to close the gap CHAIN left (comment-only). They track *file order*, not the `GAMES`/daily sequence — see §7.

#### 5.1 — HOWLER (`id:'howler'`, `arc_howler_best`, PALS[0]) ✅ SHIPPED (Unit 4)

**Hook**: the foam Mega Howler rocket, 90s/00s playground. **Verb**: swipe. **Ethos**: the analog break in a hub of digital taps — the one game where *how hard* matters, not *when*. As-shipped (code in `index.html` is the source of truth):

- **Input — first real consumer of the pointer stream (D12)**: `press` captures `(x,y,t)`; `drag` updates the live aim; `release` computes `dx,dy`, `dist=hypot`, `dt`, and `vel = dist/max(dt,8)` in **logical px/ms** (`getXY` already maps into 360×640, so the unit is correct without extra scaling). The aim **ghost** is an honest **direction dash + a drag-distance power bar** — deliberately *not* a landing arc, because the true landing depends on release velocity, which can't be known mid-drag. Below `MIN_DIST 20`px or above `MAX_DT 600`ms the gesture is a **free no-op** (§8), not a wasted life. `pointercancel` routes to `release(lastCoords)` (router); a stale/short cancel no-ops, a fast one may fire — accepted per D12 (no per-game input hacks).
- **Pseudo-3D (as-shipped)**: one rocket in flight at a time. Depth `z` runs 0→`landZ` over `FLIGHT_T 0.9`s; screen-y = `LAUNCH_Y + (TARGET_Y−LAUNCH_Y)·z − LOFT·sin(π·prog)` (the arc hump); `scale = max(0.14, 1−0.8·z)`. **Velocity → depth** (`power = clamp(vel/VREF, 0.35, 1.7)`, `landZ=power`) and **swipe angle → lateral** (`aimX = clamp(dx·0.72, ±150)`) are separate mappings, so the whistle can be retuned without touching how throws fly. No matrices (the BONES frame-flip precedent).
- **The whistle sweet spot**: `vel ∈ [WHISTLE_LO 1.85, WHISTLE_HI 2.15]` (px/ms) → `sGold()`, screenshake, gold burst, **wind immunity, ×2**. **These are a synthetic guess, ALL `// PROVISIONAL`** — the spec's `1.5–1.8` is a foreign coordinate space and this session could not measure on-device. A **`CALIBRATE` flag** ships in the object: flip it on to log every throw's logical-px/ms `vel` (console + on-screen min/max/count), gather ~50–100 one-thumb throws, set `VREF` at the comfortable median and the band ~±8% around it, then flip it off. **Owner still owes the on-device calibration pass** (§7).
- **Wind**: drawn HUD chevrons (rule 4); per-frame lateral **acceleration** on the airborne rocket (`WIND_BASE 90 + stage·45`, cap `340` px/s², sign/magnitude from `srnd`). Whistle throws are immune.
- **Scoring**: centre (`d ≤ centerR`) 100, rim (`d ≤ rimR`) 50, miss 0 + a life; **3 misses = over** (D14, endless). Each scoring hit advances a stage: `centerR 22→11`, `rimR 46→26`, wind strengthens, target re-drifts.
- **Seeded**: target x-drift + per-throw wind via `srnd(this)` (call **order** is deterministic, so everyone gets the same Nth-throw wind in the daily regardless of timing). **Swipe physics stay live** — free play byte-identical.
- **Ranks** (retitled by the revision pass to echo the distance ladder; thresholds unchanged): BACKYARD ARM → 30 METRE CLUB → 60 METRE CLUB → 90 METRE CLUB → ULTIMATE DISTANCE → WHISTLER (thresholds `0/250/650/1200/2000/3200`). Rank titles surface on the daily share card.
- **Roster**: slotted into `GAMES` **after SCAN** per §5.6; `PALMAP.howler=0` (added when this was still three maps — since consolidated to one, see §7). Menu count / emoji grid / `drawBars` derive from `GAMES.length`. Daily was **8 games** when this shipped.
- `par:500` PROVISIONAL — pure guess against ~100/hit; instrument early. The `whistles` over-screen stat counts sweet-spot-**velocity** throws (including ones that missed laterally), i.e. it measures the skill, not just scoring hits.

**Revision pass ✅ SHIPPED** (owner-requested; as-shipped record):

1. **Distance ladder — `30 M → 60 M → 90 M → ULTIMATE DISTANCE`** (homage: 2003 Dan Carter & Steve Devine Mega Howler TV ad). Built as **derived named milestones over the unchanged continuous difficulty ramp**: `tier() = min(3, floor(stage/TIER_HITS))`, `TIER_HITS:3 // PROVISIONAL` — 30 M holds for scoring hits 1–3, 60 M for 4–6, 90 M for 7–9, then **ULTIMATE DISTANCE from hit 10, open-ended forever** (D14 preserved: endless + 3 lives, nothing ends at the top; note the `centerR`/`rimR`/wind formulas hit their floors/cap around stage 8–12, so deep-ULTIMATE difficulty plateaus at max — pre-existing shipped behaviour, not a ladder regression). The `sStage()` fanfare + gold `pop()` banner + `buzz` now fire **only on tier crossings**; ordinary stage-ups keep just the hit sound from `resolve()`, so ladder moments stand out (deliberate feel change from Unit 4, owner-approved). HUD shows the current tier in gold at y152 under the wind meter; the over-screen `extra` is now `TIER · N WHISTLES` (was `STAGE n`). Ranks retitled to echo the ladder — see the Ranks bullet.
2. **Recolour — purple body, green/teal fins.** Took the cheapest path: `PALMAP.howler` stays `0` (ember) in the single consolidated `PALMAP` (§7 follow-up a) — card, background, HUD glow, aim ghost, and power bar stay ember; only the **drawn rocket** changed. `drawRocket()` and the `drawMenuIcon` howler branch now use literal hexes (body gradient `#4a1e8a→#8a4ae0`, fins `#2ad8b0`, outline `#0e0616`; cream nose + gold nozzle ring kept) — a distinct drawn object per the GUNGE-surface precedent, **not** reads of PALS[2], which remains the reserved hub palette. The lives icons render via `drawRocket(…,0.5)` so they recoloured for free. The icon's ember motion streaks were kept (they match the card's palette).

#### 5.2 — GUNGE (`id:'gunge'`, `arc_gunge_best`, PALS[6] slime lime) ✅ SHIPPED (Unit 5.2)

**Hook**: Sunday-morning kids' TV, the gunge tank. **Verb**: tap-to-rotate. **Ethos**: the only *puzzle* in the hub — thinking under a clock instead of reacting. As-shipped (code in `index.html` is the source of truth):

- **Grid**: N×N pipe tiles (straights, elbows, tees, dead ends). N=5, growing to **6 at stage 3** (spec's 5×5→6×6). Tap a cell → `r = (r+1) & 3` (90° step). **Tap-only** — no `press()`, so per D12/§3 it stays `tap()`-on-pointerdown; this was the cheapest of the five to build. Openings are a direction bitmask (N=1 E=2 S=4 W=8); shapes are base masks (straight 5, elbow 3, tee 7, dead 1) rotated by `r`.
- **Generation — solved-then-scrambled (D16), verified**: `genBoard()` carves **one** simple source→dest path (randomised DFS, `carve()`), assigns each path cell the SHAPE its two connections require (always a **2-opening** straight/elbow, so a correctly-oriented path never branches into a decoy and cannot leak), fills the rest with random decoy tiles, then **scrambles rotations only**. Every shape's solving orientation is one of its four rotations, so a solution provably exists after any scramble — a shared unsolvable daily board is structurally impossible. Confirmed across 4000 seeded boards (both grid sizes): all solvable and leak-free; carved path length 5–34.
- **Readability aid**: pipes currently connected to the source are **lit** (`flow()` caches `flowInfo` on every rotate). The board is always legible, so the gamble is time-vs-commit, not "did I misread my pipes."
- **RELEASE valve (D17)**: a **drawn** valve button (rule 4) sits well clear of the grid at the bottom (DAIRY-button style). Gunge only flows when the valve is released. Manual release banks the remaining clock; if the player never commits, it **auto-releases at `timeLeft` 0** and banks nothing. Outcome matrix (mirrored in the code comment):
  - release, **solved** → `score += remaining × segments`, advance a stage
  - release, **mis-solved** → leak, lose a life
  - auto-release@0, **solved** → gunge lands, **0 points, no life lost**
  - auto-release@0, **mis-solved** → leak, lose a life

  ("solved" = gunge reaches the tank cleanly with no spill; `segments` is fixed by the generated path. No confirm dialog — the snap decision is the point. Tapping the valve can cost a life, but §8 holds: stray taps elsewhere are free no-ops with a soft reject flash.)
- **Loss**: a leak (gunge hits a dead end / open edge, or a mis-solved release) costs a life; **3 fails = over**. Endless (D14). The per-board countdown is a **puzzle clock, not a score cap** — the game never time-caps scoring (D3), it just ends on 3 fails.
- **Timer (as-shipped)**: `stageTime() = max(TIME_FLOOR, TIME_BASE − stage·TIME_STEP)`. Shipped at **`TIME_BASE:48, TIME_STEP:3.6, TIME_FLOOR:21`** — i.e. 48s at stage 1, floored at 21s by ~stage 8. These are **3× the original 16/1.2/7** (owner asked for more solve time after the first build played too tight). Still `// PROVISIONAL`. This buff has par/run-length consequences — see §7.
- **Seeded**: source/dest, carve, decoy fill, scramble — all via `srnd(this)`. Deterministic thereafter (propagation uses no randomness), so the daily replays identically.
- **Roster**: slotted into `GAMES` **after SWINGBALL** to match the §5.6 final order; `PALMAP.gunge=6`; drawn menu icon (dripping pipe tile). Menu count, emoji grid, and `drawBars` all derive from `GAMES.length` — no manual bump needed.
- **Ranks**: STUDIO AUDIENCE → BUCKET CATCHER → PIPE PLUMBER → VALVE MASTER → GUNGE TANK LEGEND. `par:300`, ranks, and per-board timings all **PROVISIONAL** — instrument and retune with the roster. **`par:300` is now known to be too low post-timer-buff — see §7.**

#### 5.3 — TAZO TYCOON (`id:'tazo'`, `arc_tazo_best`, PALS[7] holo-blue) ✅ SHIPPED (Unit 5)

**Hook**: mid-90s chip-packet collectible craze. **Verb**: directional swipe. **Ethos**: pure spatial strategy — the only game with no time pressure at all. As-shipped (code in `index.html` is the source of truth):

- **Input — pointer stream (D12), `press`+`release`, `drag` deliberately absent (§5.3)**: `press` records the origin; `release` picks the dominant axis past `DIR_THRESH 30`px → U/D/L/R. No `drag` handler at all, so the router streams press→release and leaves `cur.drag` undefined (no live preview — a 2048 board doesn't want one). Sub-threshold and zero-delta (`pointercancel`) gestures are **free no-ops** with a soft reject flash (§8). `tap()` is only reached for ready-start / over-restart.
- **Grid**: 4×4 (`this.cells` = flat 16-int board, tier or −1). Tiles slide the vector's length; equal tiles merging along the path combine one tier up, **once per tile per move**. Tiers: **0 plastic → 1 cardboard → 2 holo → 3 gold → 4 mega slammer**. Each tier a drawn disc (`TAZO_COL` ramp + centred `drawMark` for hub cohesion, rule 4).
- **Juice**: every merge = **double-strike scale bounce** (`sin((1-b)·2π)·0.16·b`) + a **plastic clack** (`clack()`, two square/triangle `beep`s) + a `pop()` label; short slide tween (`SLIDE_T 0.09`). Mega slammer adds `sGold()` + screenshake.
- **Mega slammer (tier 4) — the RELIEF valve**: forming one (gold+gold) **detonates the low-tier CLUTTER around it** (orthogonal neighbours of tier ≤ `SLAM_MAXTIER 1`) and is **consumed** — a tier-4 tile never rests on the board. Clearing *only clutter* makes the valve **self-weaken late** (fewer low tiles as the board climbs), so it extends play without becoming an infinite pump. **This was hard-won**: a full-radius detonation (clears all neighbours regardless of tier) makes the board a perpetual-motion machine — sim showed greedy-optimal play never dies (slam-farming). The clutter-only rule fixed it. Do not "restore" full-radius clearing.
- **ESCALATING SPAWN PRESSURE — first-class mechanic (§5.3 design requirement), the run-length governor.** Two curves, both climbing as you play, spawn after **every effective move**:
  1. **Count** `= min(SPAWN_CAP 12, 1 + floor(moves / SPAWN_K 6))` — tiles/move rise with **moves elapsed** (skill-proof; free space inevitably collapses). Score-based escalation alone does **not** bound runs — a good player keeps score low relative to survival, so the pressure must track *time/moves*, not score.
  2. **Floor-tier** rises with **score** (`TIER_BANDS`: plastic-heavy → drops tier-0 by score 360) so the board clots with holo/gold a single low merge can't clear.
  A **time cap is forbidden (D3)** — retune run length via `SPAWN_K` / `TIER_BANDS`, never a clock.
- **Loss = board full, no legal merge in any direction: ONE terminal state** (`hasMove()` false). **Deliberate divergence from D14's "3-lives"** — 2048's natural loss is a single board-lock; par-normalisation still works on the resulting score; D14 explicitly allows revisit. Owner approved (2026-07-18).
- **Seeded** (D16-trivial — a 2048 board is always playable): spawn slots + tiers via `srnd(this)`; slide/merge math stays live so free play is byte-identical.
- **Ranks**: SWAPSIE ROOKIE 0 → LUNCHTIME TRADER 800 → HOLO HOARDER 2500 → GOLD SLAMMER 6000 → TAZO TYCOON 14000.
- **Roster**: slotted into `GAMES` **after CHATTER** (before SCRAMBLE) per §5.6; `PALMAP.tazo=7` (added when this was still three maps — since consolidated to one, see §7); drawn menu icon (fanned tazo stack). Menu count / emoji grid / `drawBars` all derive from `GAMES.length`. Daily is now **9 games**.
- **`par:4500`, ranks, all constants PROVISIONAL — SIM-DERIVED this session (no human play).** A headless replica of the exact core logic (`scratchpad`, not shipped) measured: greedy-optimal run **median ≈130 moves ≈3.3 min** (p90 ≈10.8 min), random ≈2.8 min; a mega slammer forms in a typical run. `par:4500` (≈greedy-median score 5257 minus a margin) puts a careful daily run at **~200–290 normalised**, with only god-runs pegging the 625 cap — the **GUNGE-par lesson** applied up front (§7). `CALIBRATE` flag logs real moves+seconds+score per run for the owner's on-device retune. **`par` and the pressure curve still owe a real-play pass.**

#### 5.4 — DAIRY WARMER (`id:'dairy'`, `arc_dairy_best`, PALS[8] warm caramel) ✅ SHIPPED (Unit 6)

**Hook**: the corner dairy pie warmer. **Verb**: tap-select, tap-rotate, tap-place. **Ethos**: cozy and geometric — the hub's only calm game. As-shipped (code in `index.html` is the source of truth):

- **Input — D13, tap-only, no `press()`**: **tap a tray pastry to select** (held preview + live rotation), **tap the drawn ROTATE button** (bottom-right, dimmed when nothing held) to cycle orientation, **tap a shelf cell to place**. No drag, no `press` — so per D12/§3 the router keeps it on `tap()`-on-pointerdown, exactly like GUNGE. Orientations are derived + deduped in `init()` (square 1, sausage 2, L 4). Placement anchors the piece's normalised bounding-box origin at the tapped cell; a **valid tap commits immediately** (single-tap place). An invalid tap, empty cell, or the rotate button with nothing held is a **free no-op** with a soft reject flash — never a strike (§8; a red ghost of the misfit footprint makes it legible).
- **Grid**: 6 wide × 4 tall glass shelf, 50px cells (`GX:30, GY:140`, 300×200 box). Pieces (`SHAPES`): **mince & cheese** 2×2, **sausage roll** 1×3, **potato-top** L-tromino. Blockdoku-style — place anywhere, **no gravity**. A tray of 3 is visible; **place in ANY order** (reading B).
- **Tray-of-3 / loss (reading B, owner-approved 2026-07-18 via "proceed as recommended")**: §5.4's "tap a queued pastry to select" reads as free choice — kept, over a forced-FIFO head, because it suits the calm ethos. The loss check is generalised from the spec's "the *head* has no placement" to **"no tray piece fits anywhere"** (`trayDead()`). A **STRIKE** (heat-escape) fires when the whole tray is dead: lose a life, **wipe the shelf** (fresh warmer) and deal a new tray. The wipe guarantees the fresh tray is placeable, so **a strike can never chain in one frame** — the reason it exists. **3 strikes = over** (D14, endless + 3-lives — no TAZO-style divergence).
- **Row clear**: a full **horizontal** row *sells* — clears with a neon **FRESH!** / **FRESH ×N!** banner (reuses `pop()` + a `banner` field like GUNGE's `GUNGED!`, not a bespoke overlay). Multi-row clears score **exponentially** (`score += BASE_CLEAR × (2^rows − 1)`, `BASE_CLEAR:60` → 60/180/420/900 for 1–4 rows). All `// PROVISIONAL`.
- **Seeded** (D16-trivial): tray order via `srnd(this)`; placement/clear math stays live so free play is byte-identical.
- **Roster**: slotted into `GAMES` **after SCRAMBLE** (§5.6 order); `PALMAP.dairy=8`; drawn menu icon (glass pie-warmer with a marked pie + heat wisps). Menu count / emoji grid / `drawBars` all derive from `GAMES.length`. Daily is now **10 games**.
- **Ranks**: AFTER-SCHOOL REGULAR → COUNTER HAND → WARMER WRANGLER → PIE ARCHITECT → DAIRY OWNER. `par:2200` **PROVISIONAL & SIM-INFORMED** (not the spec's 250): a headless greedy replica over 200 seeds scored **median ≈4620** (p10 ≈2580, p90 ≈7980), ≈147 placements/run — so `par:250` would peg the 625 normaliser cap for anyone competent (the GUNGE/TAZO par lesson, applied up front). **Caveat the other way**: DAIRY has no clock and a forgiving wipe-on-strike, so the greedy bot *over*-states a human — real play may sit well below 4620, i.e. `par` may need to come **down** after data. See §7.

#### 5.5 — WEAVER (`id:'weaver'`, `arc_weaver_best`, PALS[9] navy + red-sock) ✅ SHIPPED (Unit 7)

**Hook**: 1995 America's Cup red socks. **Verb**: continuous drag. **Ethos**: the most distinctive verb in the hub — built last as planned. As-shipped (code in `index.html` is the source of truth):

- **Input — the first FULL pointer-stream consumer (D12)**: all three handlers carry live state. `press` on a **red bollard** (one of the two carved endpoints) begins the thread; anywhere else is a free no-op + reject flash (§8). `drag` extends it: a knot within `SNAP_R 24` commits a segment only if it's unvisited, **rope-connected** to the current knot, and the segment crosses no committed segment; a blocked rope is a free visual reject. Dragging back onto the *previous* knot **unwinds one step** (Flow-Free-style). Snapping the final knot completes mid-drag. `release` with an unfinished thread **unravels it as a free retry** on the same board (§8 — lives are hazard-only); `release` unconditionally nulls the thread, and **`pointercancel` routes through the router to `release`**, so an abandoned drag can never hang a weave.
- **Generation — Hamiltonian path FIRST (D16), by inversion**: `carve(L)` runs a randomised DFS with backtracking (GUNGE's `carve()` DNA, length-targeted instead of dst-targeted) to lay a random simple path of exactly `L` cells on a **virtual 4×4 grid** — then **the carved cells become the knots** (cell centre + seeded jitter ≤12px), so the carve order is a Hamiltonian path over the knot set *by definition*. Decoy ropes are added after: seeded picks among grid-adjacent (chebyshev 1, incl. diagonal), non-consecutive knot pairs. Consecutive carve cells are orthogonal-adjacent and jitter is bounded below half a cell, so **the solution path can never geometrically self-cross** — the no-cross rule can't invalidate the intended solution. Cell 75px ≥ 2·`SNAP_R` + 2·`JITTER` = 72, so snap circles of neighbouring knots never overlap. Verified: 36,000 seeded boards (stages 0–8), zero assertion failures, carve never needed a fallback start.
- **Board**: knots `L = min(14, 7 + stage)`; decoy ropes `min(6, 2 + stage)`. Thread starts only on the two carved endpoints — the Hamiltonian guarantee holds from the endpoints *only*; a free-start could strand the player on a provably unwinnable attempt.
- **Rules**: thread cannot cross itself — tested at commit time with the new **`segsCross`** helper (§3; `segDist` is a point-to-segment primitive and cannot decide two segments crossing — the old §3 note claiming it could was wrong). Every knot exactly once completes the knit.
- **Hazards — the only life loss (§5.5 as specced)**: spy boats trace seeded rectangular patrol boxes (1 boat, +1 at stage 2, +1 at stage 5; speed ramps per stage); saltwater flares arc across the board on seeded routes/timings (period shrinks per stage). Contact with the **live thread** (committed segments + rubber-band, `segDist` per frame) = the thread snaps = 1 life; 3 lives = over (D14). Hazards only threaten an active thread — timing the weave around them is the game.
- **Clock — bonus meter only (GUNGE precedent, D3)**: `stageTime() = max(14, 30 − stage·1.5)` drains per board (across retries); completion banks `remaining × TIME_RATE 1` on top of `knots × NODE_PTS 3 × (stage+1)`; at 0 the board stays finishable for base points — expiry never costs a life or ends anything.
- **Seeded** via `srnd(this)`: carve, jitter, decoys, boat routes, flare routes — Nth-draw order is deterministic (HOWLER precedent); drag physics and cosmetics stay live, free play byte-identical.
- **Ranks**: SOCK KNITTER 0 → DECKHAND 120 → TACTICIAN 400 → HELMSMAN 900 → CUP HOLDER 1800. All PROVISIONAL.
- **`par:1000` PROVISIONAL & SIM-INFORMED** (not the spec's 200): a modelled player over 4000 runs scored p10/p50/p90 ≈ 310/1000/2600, median run ≈2.6 min, median stage 7 — `par:200` would peg the 625 cap for anyone reaching stage 4 (the GUNGE/TAZO/DAIRY lesson applied up front). The player model is synthetic; a `CALIBRATE` flag logs score/stage/knits/seconds per run for the real-play retune (§7).
- **Roster**: appended to `GAMES` **last** (§5.6 order); `PALMAP.weaver=9`; drawn menu icon (knot graph mid-weave). Menu count / emoji grid / `drawBars` derive from `GAMES.length`. Daily is now **11 games**.

#### 5.6 — Roster-wide consequences (do not miss these)

- **`GAMES` order** (daily sequence) — recommended, interleaving verbs so no two consecutive games feel the same: STACK → SCAN → HOWLER → SWINGBALL → GUNGE → CHATTER → TAZO → SCRAMBLE → DAIRY → KNUCKLEBONES → WEAVER.
- **Menu**: 11 cards → page 0 (daily) + 3 card pages. `menuPageCount()` already derives from `GAMES.length`; no change needed.
- **Daily length**: 11 games ≈ well past the 5–10 min target. Accepted for now (owner: all games join the daily; pruning comes later) — but **instrument run length from day one**, because this is the number that decides the prune. If it needs solving before the prune, the seeded-subset option is already available for free: draw N of 11 from the day's seed, same lineup for everyone, roster stays fresh. Flagged, not built (§7).
- **Emoji grid**: grows 6 → 11 tiles as games land. Grid length changes between days-of-different-rosters, so historical shares aren't comparable across a roster change. Acceptable; note it in the share code.
- **Interstitial/end-card bars**: measured at 11 rows (Unit 7): end card `drawBars(…,158,30,…)` puts row 11's rank text at y≈474 vs buttons at `H−104`=536; sharecard (32px pitch from 248) puts row 11 at y≈584 vs footer at 618. **Both fit — no pitch tighten was needed.** If a 12th row ever appears, tighten to ~26px rather than redesigning.
- **Pars**: every new game's par is a guess. They are the difference between a game being a rounding error and dominating the daily total. Retune all eleven together after the roster settles, not piecemeal.

---

### Phase 6 — Mutators (opt-in, experimental)

- **Free-play only, opt-in, never in the daily.** Gate behind `const MUTATORS_ENABLED = true` so playtesting can kill it in one line.
- Entry: a small **paper fortune teller** icon on the menu (drawn, rule 4). Tap → fold/pick animation → deals a mutator; accept (play any game with it) or dismiss.
- Scores go to `arc_mut_best_<gameid>_<mutid>` — **never** to clean bests or the daily.
- Start with three, ~one line each: GOLD RUSH (gold spawn ×2), MIRROR (SWINGBALL direction flips each loop), TREMOR (STACK slider sinusoidal wobble). Pass as an optional `modifiers` object on `init()`; games ignore what they don't understand.
- *Phase 5 adds obvious candidates — GREASY FINGERS (DAIRY rotate disabled), CROSSWIND (HOWLER wind ×2), FLOOD (GUNGE valve opens early). Note them; don't build them yet.*
- **MIRROR previously named CHAIN; CHAIN is pruned. SWINGBALL only.**

### Phase 7 — The creature

- A pixel tamagotchi-style creature on the menu screen (bottom, near the brand footer). Drawn sprite, palette-aware, idle animation on `tick`.
- **Growth-only. It never decays, never suffers, never guilts.** It sleeps when you're away.
- XP from play — e.g. +1 per 100 normalised points, +25 per daily completed. Stages: egg → hatchling → kid → teen → legend. Purely cosmetic: it gates nothing, sells nothing.
- State in `arc_creature`, device-bound localStorage — accepted.
- Reacts: hops on NEW BEST, celebrates daily completion, sleeps after 30s menu idle. The creature is the hub's only meta-layer.

---

## 6. Decision log (settled — do not reopen)

| # | Decision |
| --- | --- |
| D1 | SWINGBALL is endless (loop-banking multiplier), separate from CHAIN. Fresh best key, **no legacy inheritance — this survives CHAIN's prune**. |
| D2 | ~~Seven-game trial roster~~ — **superseded by D9/D10**. Menu redesign accepted. |
| D3 | Global par constants per game; par-based normalisation (÷par ×250, cap 625); no time caps anywhere; CHATTER par from real play data, not rank tables. |
| D4 | Daily = one gauntlet attempt per day (consumed at start; bailing forfeits); individual games unlimited. Content-seeding only. Name settled: **TUCK SHOP RUN**. `DAILY_EPOCH = 2026-07-16`. |
| D5 | Mutators are opt-in, free-play only, feature-flagged, separate best tables, fortune-teller UI. Excluded from the daily. |
| D6 | Creature is growth-only, device-bound localStorage, cosmetic. |
| D7 | Café hub metaphor is dead. Creature only. |
| D8 | Social = share card (A) + daily emoji grid (B). No backend, no accounts. |
| D9 | **Free-play gauntlet removed (4.5 R3). TUCK SHOP RUN is the only gauntlet.** `arc_gauntlet_best` retired, not deleted. |
| D10 | **CHAIN is pruned in Phase 5.** First prune; sets the pattern in rule 10 — roster entry removed, data keys kept dormant forever. |
| D11 | **Phase 5 = five new games, all joining the daily**, before mutators and the creature. Roster 7 → 6 → 11. Further pruning happens later, from play data. |
| D12 | **Input contract extends once**: optional `press`/`drag`/`release`. Games without `press` keep tap-on-pointerdown, unchanged. No per-game input hacks. |
| D13 | **DAIRY WARMER is tap-select / tap-rotate-button / tap-place.** No drag. |
| D14 | **All five Phase 5 games are endless + 3-lives semantics**, not discrete levels — required for par normalisation to mean anything. *(Owner: may revisit.)* **TAZO (Unit 5) is the first exception, owner-approved 2026-07-18**: 2048's natural loss is a single board-lock (grid full, no legal merge), so TAZO is **endless + one terminal state**, not 3 lives. Par-normalisation still works on the resulting score. HOWLER/GUNGE remain 3-lives. |
| D15 | **Nostalgia names stay literal**: TAZO TYCOON, HOWLER, GUNGE. Owner has cleared these references. |
| D16 | **Seeded content must be generated correct-by-construction**, never generate-and-hope: GUNGE solved-then-scrambled, WEAVER Hamiltonian-path-first. A shared unsolvable daily board is unacceptable. |
| D17 | **GUNGE releases the gunge on a drawn RELEASE valve button**, not on a timer-expiry auto-open alone. Manual release banks the remaining clock into score (`remaining × segments`); auto-release at `timeLeft` 0 banks **no** time. A clean connection always lands the gunge and advances (0 points if released at 0, no life lost); a mis-solved release always leaks and costs a life. Resolves the spec contradiction where "valve opens at zero" left "remaining time" always zero. Rule 4 (button is drawn) and D12 (still tap-only, no `press`) both still hold. |
| D18 | **DAIRY WARMER is a tray-of-3, place-in-any-order packer** (Unit 6, owner-approved 2026-07-18). Resolves the §5.4 tension between "tap a queued **pastry** to select" (free choice) and "the **head** is placeable-or-lose" (forced FIFO): the free-choice reading wins (suits the calm ethos), and the loss check generalises to **"no tray piece fits anywhere"** (`trayDead`). A strike **wipes the shelf** and deals a fresh tray so strikes can't chain in one frame; 3 strikes = over (D14 3-lives, no divergence). Tap-only (D13), no `press`. |
| D19 | **WEAVER's four §5.5 ambiguities are settled** (Unit 7, plan approved by owner 2026-07-18): (a) thread movement is **rope-constrained** — a segment may only follow a drawn edge, otherwise decoy edges would be decoration and D16's construction would guarantee nothing the player touches; (b) the thread may only **start on the two carved endpoints** (red bollards) — the Hamiltonian guarantee holds from the endpoints only; (c) **lives are hazard-only**: releasing an unfinished thread unravels it as a free retry (§8), the draining bonus clock is the cost of dithering; (d) the per-board clock is a **bonus meter, never a terminator** (D3, GUNGE precedent) — expiry zeroes the bonus, nothing else. Also: thread self-crossing is tested with the shared `segsCross` helper, not `segDist` (the old §3 parenthetical was geometrically wrong); hazard-vs-thread stays on `segDist`. |

## 7. Open decisions (owner to resolve — flag, don't guess)

- **Daily length (now 11 games — the full Phase 5 roster).** Already over the 5–10 min target — **TAZO is the longest single game** (sim median ≈3.3 min, p90 ≈10.8 min for a slam-farming god-run; see §5.3), the first game whose own run length can dominate the daily. **DAIRY is a second run-length watch item**: no clock + a forgiving wipe-on-strike makes it near-unlosable for a careful player (greedy sim ≈147 placements/run before 3 strikes), so its runs can trend long too. Options: accept until the prune; or draw a **seeded subset** (N of 11 per day, same for everyone). Instrument first — decide on data. TAZO ships a `CALIBRATE` flag (logs moves+seconds per run) precisely for this.
- **Final pars for all eleven** after playtesting. All `// PROVISIONAL`: STACK 20, SCAN 35, SWINGBALL 40, CHATTER 600, SCRAMBLE 120 *(likely too high post-R1)*, KNUCKLEBONES 90, HOWLER 500, GUNGE 300 *(now known too low — see next bullet)*, TAZO 4500 *(sim-derived, not the old spec guess of 400 — TAZO's score scale is O(thousands), so 400 would peg the cap for everyone; owes a real-play pass)*, DAIRY 2200 *(sim-informed, not the old spec guess of 250 — greedy replica median ≈4620, so 250 would peg the cap; but the greedy bot over-states a human given no clock + wipe-on-strike, so this may need to come DOWN after real play)*, WEAVER 1000 *(sim-informed, not the old spec guess of 200 — modelled-player median ≈1000, so 200 would peg the cap from stage 4; the player model is synthetic, `WEAVER.CALIBRATE` logs real runs for the retune)*.
- **GUNGE `par:300` is too low — the real par surprised me, and it moved after the timer buff.** Score is `remaining × segments`. Instrumented (seeded, both grid sizes): median path is **13 segments** (5×5, stages 0–2) / **17** (6×6, stage 3+); the shipped timer gives **48s → 21s**. So a single *cleanly-solved* board banks roughly `remaining(~25–35s) × segments(~13–17)` ≈ **300–500 points on its own**, and a multi-board run clears **1000–2500+**. At `par:300` the daily normaliser (`score/par × 250`, cap 625) **pegs the 625 cap for any competent player**, so GUNGE would dominate the daily total. Revised provisional guess: **~1200** (still a guess — needs real human play, not my sims). Two compounding causes I hadn't priced in: (a) `remaining × segments` is multiplicative, so long boards score super-linearly; (b) the 3× timer buff raised `remaining` across the board. **Retune `par` and re-measure once GUNGE has real play data.**
- **GUNGE run length is now clock-loose.** With `TIME_FLOOR:21` a careful player can solve even a 6×6 well inside the clock, so fails come almost entirely from *mistakes*, not time — runs trend long and open-ended (endless + 3 lives, no time cap by D3). This feeds directly into the daily-length concern above; instrument GUNGE run length specifically when deciding the seeded-subset question. (If GUNGE runs too long, the lever is `par`/timer, not a time cap — D3 forbids caps.)
- **Menu palette maps — CONSOLIDATED (follow-up (a) done).** *History:* a single render throw once froze the whole app — surfaced when GUNGE shipped, because `PALMAP`, `drawMenuIcon`'s lookup, and `drawMenu`'s card lookup were three separate hardcoded id→palette objects; missing GUNGE from the third made `p.acc` throw, and since the `requestAnimationFrame` loop only re-arms at the *end* of `frame()`, that one exception bricked the entire menu. *As-shipped now:* there is **one source of truth** — a module-level `const PALMAP` (id→index) + `function gamePal(id)` right after `const GAMES`, with a `PALS[2]` (hub) fallback so a missing game degrades instead of crashing. `drawMenu`'s card and the gauntlet end-card/`pid` persistence both read through it; `GAUNTLET`'s own `PALMAP`/`gamePal` were removed. **Investigating this turned up that `drawMenuIcon`'s map was dead code** — its `p` was assigned and never read (every icon branch hardcodes its colours), so it was deleted outright, not replaced. Adding a game is now **one line** in `PALMAP`. Verified: `gamePal` returns byte-identical palettes for all 9 ids, unknown-id → `PALS[2]`, and a full gauntlet renders its 9-row end-card with no throw. **Remaining follow-up (b, still deferred):** wrapping the `frame()` body in try/catch so one bad frame drops instead of freezing — held off (could mask real bugs; the single map + fallback already removes the known trigger).
- **HOWLER's whistle window** — the spec's 1.5–1.8 px/ms is not calibrated for 360×640. Must be measured on-device, not guessed. **Unit 4 shipped the calibration *harness* (`HOWLER.CALIBRATE` flag → logs logical-px/ms `vel` distribution) plus a marked-PROVISIONAL synthetic placeholder (`VREF 2.0`, band `1.85–2.15`). The on-device pass is still owed** — no human-throw data was available this session. Same for `par:500`.
- **Which games get pruned next**, and the target roster size. CHAIN was first (D10).
- **SCRAMBLE/CHATTER attention-mechanic overlap** — tolerated; revisit at prune time.
- **New `PALS` entries for Phase 5** — ✅ added (Unit 2) and now all referenced: `6` GUNGE, `7` TAZO, `8` DAIRY, `9` WEAVER (shipped with Unit 7). All four hue sets remain **provisional** — retune with the roster. Drawing surfaces directly (asphalt-court precedent) is still available where a palette isn't warranted.
- **CHATTER's in-play stage-0 colour is the "hub" PALS[2], and differs from its menu card** — surfaced when Unit 2 replaced `(2+stage)%PALS.length` with the explicit `CHATTER_PALS=[2,3,4,5,0,1]`. The list opens on `2`, so CHATTER's stage-0 (and every 6th stage) background/ring is the hub purple, while its **menu card** uses `PALMAP.chatter=3` (green). Both facts are **pre-existing** — the old modulo did exactly the same; the explicit list only made them visible, and Unit 2 preserved the sequence byte-for-byte. Open question: is CHATTER sharing the hub purple at stage 0 (and the card-vs-play colour mismatch) intended? If not, drop `2` from `CHATTER_PALS` — a deliberate, isolated one-line recolour, no longer a side effect of `PALS.length`.
- **Source `GAME N` section labels track file order, not roster order** (surfaced during the 5.0 prune). They were renumbered 1–6 after CHAIN's removal, but the file lays CHATTER's block *before* SWINGBALL's while `GAMES` runs SWINGBALL *before* CHATTER — so "GAME 3: CHATTER" is actually the 4th game in the daily sequence. This mismatch predates the prune (it was CHAIN-shaped before). Harmless — they're only comments — but do not read them as roster/daily indices. Decide whether to reorder the source blocks to match `GAMES`, or drop the numbers entirely, when the five new games land.

## 8. Style conventions

- Compact canvas code; short locals (`p` palette, `g` gradient/game, `s` size/slot).
- All text in `Press Start 2P` at 5–30px; set `textAlign` explicitly before every text block.
- Glow via `shadowColor`/`shadowBlur`; always reset `shadowBlur=0` after.
- Keep each game's constants at the top of its object as UPPERCASE members (see CHATTER/SWINGBALL) so tuning never requires reading gameplay code.
- Comment hard-won constraints inline (see the home-icon and iOS-viewport comments) — future sessions read comments, not commit history.
- Mark every unproven number `// PROVISIONAL`. An unmarked constant claims a confidence we haven't earned.
- Empty/stray inputs are free (no-ops), never penalties — SCRAMBLE, BONES, and CHATTER's second-miss rule all follow this. New games must too.
