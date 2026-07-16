# CLAUDE.md — CENTRARCADE

Single-file HTML5 arcade hub of quick Centrapay-branded games built on a shared 16-bit engine. One-thumb, portrait, mobile-first. The entire app is `index.html`. Roster is the full **seven** games (STACK, SCAN, CHAIN, SWINGBALL, CHATTER, SCRAMBLE, KNUCKLEBONES) — Phase 2 complete.

This file has three jobs: (1) hard invariants you must never break, (2) an accurate map of the current code, (3) the settled roadmap of revisions to implement. Decisions in the **Decision log** are final — do not relitigate them; implement them.

---

## 1. Non-negotiable rules

1. **Single file.** All markup, CSS, and JS in one HTML file. No libraries, no build step. Only external dependency is the `Press Start 2P` Google font import.
2. **Logical resolution 360×640.** All game code works in this coordinate space. DPR capped at 2. Physical scaling handled once via `sc` + `ctx.setTransform` — never per-frame math against window dimensions.
3. **The Centrapay mark is settled geometry.** `drawMark()` uses six blocks on a 100×71 grid:
   `[[54,0,29,18],[82,17,18,18],[17,18,33,18],[0,35,18,18],[50,35,33,18],[17,53,29,18]]`, corner radius `0.22 × block height`, skipped below 4px. Never adjust these numbers.
4. **Icons are drawn, not glyphs.** Unicode symbols render blank on too many mobile fonts (this bit us with the house icon). Any new icon — creature, fortune teller, share, coins — is drawn with canvas paths.
5. **Audio only after a gesture.** `AudioContext` created lazily in `audio()`, resumed on tap (iOS requirement). All synthesis routes through `beep()`.
6. **Legacy best-score keys keep migrating.** `loadBest(key, legacy)` folds in old standalone keys: STACK ← `cps2_best`, CHAIN ← `chain_best`. Never remove this.
7. **All localStorage access wrapped in try/catch.** Private browsing and sandboxes throw.
8. **`'use strict'`, no implicit globals.**
9. **New games must reuse the shared systems** — `drawBG`, `drawReadyScreen`, `drawOverScreen`, FX (`burst`/`pop`), rank helpers, chrome. Cohesion is the point of the hub. A bespoke over-screen is a code smell.

---

## 2. localStorage keys

| Key | Purpose |
|---|---|
| `arc_mute` | Mute toggle ('1'/'0') |
| `arc_stack_best` | STACK best (legacy: `cps2_best`) |
| `arc_scan_best` | SCAN best |
| `arc_chain_best` | CHAIN best (legacy: `chain_best`) |
| `arc_chatter_best` | CHATTER best |
| `arc_swing_best` | SWINGBALL best (starts fresh, does NOT inherit CHAIN's history) |
| `arc_scramble_best` | SCRAMBLE best |
| `arc_bones_best` | KNUCKLEBONES best |
| `arc_gauntlet_best` | Best free-play gauntlet total (new) |
| `arc_daily_state` | JSON: `{dayKey, played, result, emojiGrid}` — enforces one attempt/day (new) |
| `arc_daily_streak` | Consecutive daily completions (new) |
| `arc_mut_best_<gameid>_<mutid>` | Per-game per-mutator bests, separate from clean bests (new) |
| `arc_creature` | JSON creature state: `{xp, stage, hatchedAt}` (new) |

---

## 3. Architecture (current code)

### Shared engine (top of file)
- `PALS` — six palettes `{top, bot, ring, acc, glow, sky}`. CHAIN and CHATTER cycle palettes per stage.
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

**Contract extensions required by the roadmap** (add to all games):
- `par` — number, global par score for gauntlet normalisation (see §5). *Now on all seven games, all provisional: STACK 20, SCAN 35, CHAIN 50, SWINGBALL 40, CHATTER 600 (owner median — CHATTER plays far harder than its ranks), SCRAMBLE 120, KNUCKLEBONES 90.*
- `seed(rng)` — optional; accept a seeded PRNG for daily runs. Content-seeding only: spawn order, zone placements, disc timings. Physics stays live.
- `onOver` — optional callback the gauntlet controller sets to hook game-over. *Wired: each game calls `if(this.onOver) this.onOver()` at its over-transition; the controller registers it, clears it one-shot (double-fire safe), and free-play resets it to null.*

### Router
- `mode`: `'menu' | 'game' | 'gauntlet'`; `cur` = active game (in gauntlet, the current game the controller is running).
- Menu tap → `cur=g; cur.init(); mode='game'`. Home → discard state, back to menu. Global `tick` drives all pulse/shimmer phases. Frame loop clamps dt to 0.05.
- **Menu is a paginated 2×2 card list** over `MENU_ITEMS` = `[GAUNTLET, ...GAMES]` (`PER_PAGE=4`, `menuPage`, `menuCards()`, `menuNav()`, drawn dots + arrows via `drawArrow()`). The GAUNTLET card is the **first tile** (eight cards → two pages). Card selection resolves on `pointerup` so a horizontal drag reads as a page swipe; **game taps still fire on `pointerdown`** for zero latency. Swipe, tappable dots/arrows, and ←/→ arrow keys all page.
- **`paused`** (module-level): set on `visibilitychange` while a game is mid-play (in `game` **or** `gauntlet` mode); the loop then skips `update`, keeps rendering the frozen frame, and draws `drawPauseOverlay()`. A tap or space resumes; home/mute stay live.
- **`GAUNTLET`** (Phase 3 controller, outside the game contract): `mode='gauntlet'` sequences all games in `GAMES` order to first game-over each via each game's `onOver` hook, normalises (`round(score/par × 250)`, cap 625) into a running `total`, and renders interstitials + an end card writing `arc_gauntlet_best`. Reads only public fields (`score`/`par`/`ranks`/`name`); never mutates game internals. Free-play resets `cur.onOver=null` on selection, so the hook is a no-op outside the gauntlet. See §5 Phase 3.
- **Bests cached** on menu entry via `refreshMenuBests()` → `menuBests` (not `loadBest` per card per frame); refreshed on home-exit so a new best set mid-game shows.
- **`sc` is recomputed by `resize()`** on `resize`/`orientationchange` only — backing store stays `W*dpr`, the single `setTransform` is never touched per frame (rule 2).
- **Frame loop starts only after the font is ready**: `document.fonts.load('8px "Press Start 2P"')` raced with a 1.5s timeout fallback, to kill the FOUT.

---

## 4. Current games — tuning constants

- **STACK** — block height 42, start width 264, perfect window `5 + spd*0.4`px (scales with speed), speed 2.4 → cap 9.5 (+0.13/block). Perfect preserves width; ≤6px landing = topple.
- **SCAN** — 3 lives (escape = life lost); $1/$2/$3 chips + 15% QR (+5 gold). Global speed mult `1 + score*0.02`. Beam extend/retract; collision tests the **segment swept by the tip** each frame (`segDist`), not just the tip point.
- **CHAIN** — R=116, speed 1.7 (+0.05/link, cap 4.2), zone half-width 0.42 → floor 0.14 (−0.012/link, +0.07 on stage-up), stage = 10 links, perfect = inner 33% (+2), post-stage gold zone (+3). **300ms input grace after stage-up** (`grace` suppresses `breakChain`). Overshoot detection uses `wasInside`/sign-crossing logic — edit with extreme care.
- **CHATTER** — 5 slots at 72° from −54° (top slot reserved for brand tag), 2 discs live at start, new disc every 2nd stage, stage every 14s, decay 4.0 +2.1/stage. Restrike: ≤20 energy = PERFECT SAVE (+45 energy, +8 pts); ≥80 = +8 energy only; else +30 energy. Score accrues at `3.6 × avgEnergy/100`/s, ×2 during FULL CHATTER (all ≥85). Miss increments `missStreak`; **`fumble()` (−4 all discs) fires only on the 2nd consecutive miss** (one stray tap is free), reset on any restrike. **Note: CHATTER is harder than its scores suggest — do not nerf it, and do not add a time cap. Its difficulty is its cap.**
- **SWINGBALL** — orbit R=104 at CY=312, pole 120→470. Speed 1.7 (+0.05/hit, cap 4.2), zone half-width 0.42 → floor 0.14 (−0.012/hit), perfect = inner 33%, **slop band = 1.7× zone** (mistimed tap = unwind + survive). Wind meter 0→100 (+20 hit / +30 perfect / −25 mistimed); filling it **banks a LOOP** (score ×`(1+loop)`), resets wind, steps `baseSpeed +0.25` / `baseZone −0.03`. 300ms grace after a loop-bank. Break only on a wild tap (beyond slop) or an untapped overshoot — reuses CHAIN's `wasInside`/sign-cross logic. `par:40` PROVISIONAL. Backyard palette PALS[3] + gold accents. **Constants are provisional — retune from play data.**
- **SCRAMBLE** — 5 kid slots (`KID_Y:532`), start 3 active (indices 1–3), wake edges [0,4] on stage-up. Tap a kid → a lolly **arcs** (~0.45s, `t += dt*2.2`) and adds `THROW_GAIN 34` to their haul (cap 100). Per-kid decay `DECAY_BASE 5.0 × decayMul (0.8–1.4)`, `+1.6`/stage, stage every 12s. **Score rewards evenness, not volume**: `SCORE_RATE 5.0 × (1 − stddev/50)` over active hauls; **FAIR SHARE ×2** when live ≥2 and all ≥`FAIR_FLOOR 60` within `FAIR_BAND 20`. Empty meter → kid away `CRY_TIME 1.4`s (returns at `PITY 42`) and −1 life; **3 empties = over**. `par:120` PROVISIONAL. Party palette PALS[4]. **Deferred from spec:** spatial "greedy kids drift to dropped lollies" — v1 uses directed throws that always connect; per-kid decay carries the greedy-kid pressure. **Constants provisional — retune from play data.**
- **KNUCKLEBONES** — bones tossed on parabolas to `CATCH_LINE:480` from 5 `SLOTS`; base `BASE_LV:660`/`BASE_G:1000` (airtime ~1.2s/spd, peak height constant as spd rises). **Gold = catch** (tap airborne), **grey = skip** (leave). Missed catch OR grey-bone tap = a drop; **3 drops = over**. `RUNGS` ladder = ONESIES→…→OVER THE FENCE (catch/skip/spd per rung; decoys from HORSES on); `CLEARS_PER_RUNG 2` clean tosses advance a rung, then it **loops** (`baseSpd ×1.12`, restart at ONESIES). Score `+2×(1+loop)`/catch, `+5×(1+loop)`/clean toss. No rotation sim — a horizontal frame-flip (every 6 ticks) sells the spin. Drawn grey **asphalt court** (chalk catch-line + play circle); PALS[5] amber accents, bone-cream sprites. `par:90` PROVISIONAL. **Constants provisional — retune from play data.**

---

## 5. Roadmap — implement in this order

### Phase 1 — Fixes ✅ IMPLEMENTED
*All seven items shipped. Kept below as the record of what changed and why.*
1. **Spacebar guard**: space currently calls `cur.tap(-999,-999)` — fires a wasted beam in SCAN and triggers a punishing `fumble()` in CHATTER. Route space only to positionless games (STACK, CHAIN, SWINGBALL); no-op elsewhere. Keyboard is convenience-only; touch is the design target.
2. **Resize/orientation handler**: recompute `sc` and canvas CSS size on `resize`.
3. **Font preload**: `document.fonts.load('8px "Press Start 2P"')` (with timeout fallback) before starting the frame loop — kills the FOUT.
4. **SCAN beam tunnelling**: check the swept segment per step, not just the tip.
5. **`visibilitychange` pause**: on hide during play, freeze; on return show a tap-to-resume overlay. Matters most for SCAN/CHATTER/SCRAMBLE.
6. **Menu perf**: cache bests on menu entry instead of `loadBest` per card per frame. Remove SCAN's duplicate `targets` filter.
7. **Feel tweaks**: STACK perfect window scales gently with speed (`5 + spd*0.4`); CHAIN gets 300ms input grace after stage-up slow-mo; CHATTER fumbles only on the second consecutive miss (single stray tap = free).

### Phase 2 — New games (three, plus menu redesign) ✅ COMPLETE
Seven games total during the trial period (all four existing + three new). None are pruned yet — pruning happens later from play data. The 2×2 menu must become a scrollable or paginated card list; keep card visual language identical.

*Status: all three new games (SWINGBALL, SCRAMBLE, KNUCKLEBONES) + the menu redesign shipped. Menu is a **paginated** 2×2 list (`PER_PAGE=4`, now two pages), card language unchanged, swipe/dots/arrows/arrow-keys. Full gauntlet order wired in `GAMES`: STACK → SCAN → CHAIN → SWINGBALL → CHATTER → SCRAMBLE → KNUCKLEBONES.*

**SWINGBALL** (`id:'swing'`, fresh `arc_swing_best`, no CHAIN inheritance) — ✅ IMPLEMENTED (see §4 for shipped constants; all provisional)
- Adapted from CHAIN's ring-timing core but a separate game — CHAIN stays in the roster unchanged.
- Ball orbits a pole; tap to strike when it crosses the hit zone. The differentiator: a **spiral wind meter** — clean hits wind the rope up the pole (visible spiral climbing), mistimed hits unwind it. It is **endless**: reaching the top doesn't end the run, it banks a LOOP (score multiplier tier +1), rope resets to bottom, speed and zone difficulty step up. Runs end only on a miss/break, CHAIN-style.
- Backyard palette (green/gold), pole and rope drawn, ball carries the mark.
- Ranks should ladder through backyard-cricket-adjacent NZ summer vocabulary.

**SCRAMBLE** (`id:'scramble'`, `arc_scramble_best`) — ✅ IMPLEMENTED (see §4 for shipped constants; all provisional)
- The lolly scramble, inverted: player is the adult throwing. Kids (chibi sprites, SCAN-clerk style) line the bottom, each with a visible haul meter. Tap a kid to throw a lolly to them.
- **Scoring rewards evenness, not volume**: continuous accrual scaled by how *equal* the haul meters are (e.g. `rate × (1 − stddev/maxStddev)`), with a FAIR SHARE bonus state when all meters are within a tight band (FULL CHATTER analogue — glow + ×2).
- Pressure: kids' meters decay at different rates, greedy kids drift toward dropped lollies, more kids join per stage. Lose condition: any kid's meter empty 3 times (they go home crying) — 3 lives semantics.
- Design intent: generosity as plate-spinning. It overlaps CHATTER's attention-juggling, which is fine during the trial — the data decides.
- *Shipped v1: directed throws (tap a kid, lolly arcs and always connects), per-kid decay, evenness scoring + FAIR SHARE ×2, 3-lives. **Deferred:** the spatial "greedy kids drift toward dropped lollies" mechanic — per-kid decay carries the greedy-kid pressure until it lands.*

**KNUCKLEBONES** (`id:'bones'`, `arc_bones_best`) — ✅ IMPLEMENTED (see §4 for shipped constants; all provisional)
- Digitised knucklebones. Five bones toss up; tap-catch them in the pattern the current level demands before they land. The traditional sequence **is the rank/level ladder**: ONESIES → TWOSIES → THREESIES → FOURSIES → CLICKS → HORSES IN THE STABLE → OVER THE FENCE. Endless via speed/height variation loops after the ladder completes.
- Discrete timing with combinatorial catch patterns (catch 2-then-2, catch all-but-one, etc.). Missed catch = drop; 3 drops = over.
- Bone-coloured sprites on a schoolyard-asphalt palette; keep physics simple (parabolic, no rotation sim needed — a spin frame-flip sells it).
- *Shipped v1: gold bones = catch (tap airborne), grey = skip decoys (leave them, tapping one faults); missed catch OR grey-bone tap = a drop, 3 drops = over. Ladder rungs set catch/skip counts + speed (decoys from HORSES on); 2 clean tosses advance a rung, then loops faster. Frame-flip spin, drawn asphalt court, PALS[5] accents. **Palette note:** no grey palette exists and adding a 7th `PALS` entry would shift CHAIN/CHATTER's `stage % PALS.length` cycling, so the asphalt court is drawn directly and PALS[5] supplies accents.*

### Phase 3 — GAUNTLET (free play) ✅ COMPLETE
- Menu card **GAUNTLET — EVERY GAME · ONE SCORE**, shipped as the **first tile** (owner request; `MENU_ITEMS = [GAUNTLET, ...GAMES]`). Plays ALL roster games in fixed order (STACK → SCAN → CHAIN → SWINGBALL → CHATTER → SCRAMBLE → BONES), each to first game-over.
- **Par normalisation**: contribution = `round(score / game.par × 250)`, capped at `625` (2.5× par). No time caps on any game, including CHATTER.
- Pars are **global constants** on each game object, all shipped provisional (`// PROVISIONAL`): STACK 20, SCAN 35, CHAIN 50, SWINGBALL 40, **CHATTER 600** (owner median, not the rank ladder — CHATTER plays far harder than its ranks imply), SCRAMBLE 120, BONES 90. All still to be retuned from real play data.
- Between games: interstitial showing running total + per-game contribution bars (`drawInterstitial`). End: total, per-game bars with rank titles, `arc_gauntlet_best` (`drawEnd`).
- Implemented as a `GAUNTLET` controller object outside the game contract (see §3) — it sequences `cur`, hooks `onOver`, accumulates, renders interstitials. Game internals untouched.
- *Shipped: `mode='gauntlet'` wired into the frame loop, input (home exits via `GAUNTLET.exit()`, taps forward to the controller), Space, and the `visibilitychange` pause. New key `arc_gauntlet_best` (already in §2). Drawn gold card + "seven palette chips under one gold mark" icon.*

### Phase 4 — THE DAILY (one attempt, shareable)
- A daily seeded gauntlet, **one attempt per calendar day** (local time), enforced via `arc_daily_state`. Individual games remain unlimited free-play. Streak counter in `arc_daily_state`/`arc_daily_streak`.
- **Seeding**: seed a small PRNG (mulberry32) from the local date string. Content-seeding only via each game's `seed(rng)`: STACK slider start sides, SCAN spawn schedule/types, CHAIN zone placements + gold order, SWINGBALL zone placements, CHATTER slot wake order, SCRAMBLE kid mix, BONES toss patterns. Same challenge for everyone, not same replay.
- Daily numbering: `#N` where day 1 = the launch date constant (`DAILY_EPOCH`, set at ship time).
- **Name**: needs a 90s-NZ word. Candidates to surface in a menu-copy comment for the owner to pick: **PLAYLUNCH**, **THE TUCK RUN**, **MORNING TEA**, **AFTER THE BELL**. Use `PLAYLUNCH` as the working title until overridden.
- **Share (both A and B)**:
  - **B — emoji grid**: on completion, build a text block — title, `#N`, one emoji tile per game coloured by contribution tier (⬛ <100 / 🟨 100–199 / 🟧 200–349 / 🟩 350–499 / 🟪 500+), total, streak. `navigator.share` text, clipboard fallback with a COPIED! toast.
  - **A — share card**: render a composed end card to an offscreen canvas (total large, per-game bars + rank titles, date + daily number, Centrapay mark as crest, palette-consistent). `canvas.toBlob()` → `navigator.share({files})`; fallback: draw it full-screen with a "screenshot me" frame. Card must look deliberate — it is the product's face in group chats.

### Phase 5 — Mutators (opt-in, experimental)
- **Free-play only, opt-in, never in the daily.** Gate the whole feature behind a single flag `const MUTATORS_ENABLED = true` so it can be switched off if playtesting kills it.
- Entry point: a small **paper fortune teller** icon on the menu (drawn, per rule 4). Tapping it plays a short fold/pick animation and deals a mutator; the player can accept (play any game with it) or dismiss.
- Mutator runs score to `arc_mut_best_<gameid>_<mutid>` — **never** to clean bests or gauntlet/daily.
- Start with three, one line of code each where possible: GOLD RUSH (gold spawn chance ×2), MIRROR (CHAIN/SWINGBALL direction flips each stage), TREMOR (STACK slider sinusoidal wobble). Pass the active mutator to games as an optional `modifiers` object on `init()` — games ignore what they don't understand.

### Phase 6 — The creature
- A pixel tamagotchi-style creature living on the menu screen (bottom area, near the brand footer). Drawn sprite, palette-aware, idle animation on `tick`.
- **Growth-only. It never decays, never suffers, never guilts.** It sleeps when you're away.
- Feeding: XP from play — e.g. +1 per 100 normalised points across any mode, +25 per daily completed. Stages at XP thresholds (egg → hatchling → kid → teen → legend), each a visibly different sprite. Purely cosmetic companion — it gates nothing, sells nothing.
- State in `arc_creature`, device-bound localStorage — accepted for now.
- Creature reacts: hops on NEW BEST, celebrates daily completion, sleeps after 30s menu idle. No café framing — the creature is the hub's only meta-layer.

---

## 6. Decision log (settled — do not reopen)

| # | Decision |
|---|---|
| D1 | SWINGBALL is endless (loop-banking multiplier); it is a separate game from CHAIN, which remains in the roster. Fresh best key, no legacy inheritance. |
| D2 | Seven-game trial roster; no pruning yet; gauntlet spans all games. Menu redesign accepted. |
| D3 | Global par constants per game; par-based normalisation (÷par ×250, cap 625); no time caps anywhere; CHATTER par must be set from real play data, not rank tables. |
| D4 | Daily = one gauntlet attempt per day; individual games unlimited. Content-seeding only. 90s-NZ name (working title PLAYLUNCH). |
| D5 | Mutators are opt-in, free-play only, feature-flagged, separate best tables, fortune-teller UI. Excluded from the daily until playtesting says otherwise. |
| D6 | Creature is growth-only, device-bound localStorage, cosmetic. |
| D7 | Café hub metaphor is dead. Creature only. |
| D8 | Social = share card (A) + daily emoji grid (B). No backend, no accounts. |

## 7. Open decisions (owner to resolve — flag, don't guess)

- Final name for the daily (PLAYLUNCH working title).
- Final pars for all seven games after ~a week of owner playtesting.
- Which game(s) get pruned post-trial, and whether the roster returns to four.
- SCRAMBLE/CHATTER attention-mechanic overlap: tolerated for the trial; revisit at prune time.

## 8. Style conventions

- Compact canvas code; short locals (`p` palette, `g` gradient/game, `s` size/slot).
- All text in `Press Start 2P` at 5–30px; set `textAlign` explicitly before every text block.
- Glow via `shadowColor`/`shadowBlur`; always reset `shadowBlur=0` after.
- Keep each game's constants at the top of its object as UPPERCASE members (see CHAIN/CHATTER) so tuning never requires reading gameplay code.
- Comment hard-won constraints inline (see the home-icon comment) — future sessions read comments, not commit history.
