# CLAUDE.md — CENTRARCADE

Single-file HTML5 arcade hub of quick Centrapay-branded games built on a shared 16-bit engine. One-thumb, portrait, mobile-first. The entire app is `index.html`.

**Phases 1–5 are SHIPPED.** The roster is **12 games shipped** (STACK · SCAN · HOWLER · SWINGBALL · GUNGE · CHATTER · TAZO · SCRAMBLE · DAIRY · KNUCKLEBONES · WEAVER · **RECALL**) **targeting 15 gross** (D36), plus the once-a-day seeded **TUCK SHOP RUN** daily, which draws a day-seeded subset rather than the whole roster (D33; `DAILY_PICK` is **5** since RECALL landed, D37). The roster was reopened at D36 for **four** additions — RECALL ✅ · LOCK · ROUND · CHUTE (Phase 7.5) — after which it closes again pending prune calls (D26). Work now proceeds in this order:

- **Phase 6 — REFINEMENT ⬅ IN PROGRESS.** Telemetry first, then per-game polish (HOWLER redesign (2D distance course) + rocket redraw, TAZO difficulty + colour separation, GUNGE contestant-in-tank), housekeeping, the roster-wide par retune, and one shipped **determinism bug fix (B1)**.
- **Phase 7 — SKINS ⬅ IN PROGRESS.** Cosmetic era packs (default: NZ 90s/00s) over frozen mechanics. **7.0 (the string extraction) is SHIPPED** — display text now lives in `SKINS.nz90`, not in the game objects. Next: 7.1 selector + unlock, then 7.2's first alternate era pack.
- **Phase 7.5 — ROSTER EXPANSION.** Four new games, built pack-native after the 7.0 extraction and before the first alternate era pack (D38).
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
| `arc_recall_best` | RECALL best (Phase 7.5 — fresh key, no legacy inheritance, D1 precedent) |
| `arc_daily_state` | JSON: `{dayKey, played, total, result, emojiGrid}` — enforces one attempt/day (consumed at start). Each `result` row is `{name, id, contrib, rank, pid}` — `id` added 6.6/D34 so a locked/returning day's receipt can resolve its line labels; rows saved before 6.6 lack it and fall back to `shareMeta`'s generic tag. |
| `arc_daily_streak` | Consecutive daily completions |
| `arc_daily_lastdone` | dayKey of the last completed daily (for streak continuity) |
| **Phase 6** | |
| `arc_stats` | Telemetry (Unit 6.0 ✅) — JSON `{games:{<id>:{n,totalSecs,recent:[≤20],daily:{n,totalSecs,recent:[≤20]}}}, run:{n,totalSecs,recent:[≤20]}}`. Recorded on every game-over (free play + daily) and daily finish; dumped to console on menu entry. |
| **Phase 7 (planned)** | |
| `arc_theme` | Selected skin/era id (default `nz90`). Display-only (rule 12). **Not read yet** — 7.0 shipped the pack + a module-level `themeId` hardcoded to `nz90`; 7.1 wires this key to it. |
| **Phase 8–9 (idea-log placeholders — not built)** | |
| `arc_mut_best_<gameid>_<mutid>` | Per-game per-mutator bests, separate from clean bests |
| `arc_creature` | JSON creature state: `{xp, stage, hatchedAt}` |
| **Dormant** | |
| `arc_chain_best` / `chain_best` | CHAIN best. **CHAIN was pruned in Phase 5 — keys retained, unread, never deleted** (rule 10). If CHAIN returns, `loadBest('arc_chain_best','chain_best')` restores history intact. |
| `arc_gauntlet_best` | Free-play gauntlet best. **Dead since 4.5 R3.** Not written, not read, not deleted. |

---

## 3. Architecture (current code)

### Shared engine (top of file)

- `PALS` — **eleven** palettes `{top, bot, ring, acc, glow, sky}`: `0` ember/orange, `1` teal, `2` purple (**hub palette** — menu, interstitials, end card, pause, and the `gamePal` fallback), `3` backyard green, `4` party pink, `5` amber, `6` GUNGE slime lime, `7` TAZO holo blue, `8` DAIRY caramel, `9` WEAVER navy/red-sock, `10` RECALL dial-up magenta (~310° — the largest unused hue gap in the set; picked over reuse so the 12th card reads distinctly on the purple hub). The four Phase-5 hue sets are still `// PROVISIONAL`. **Appending is safe by construction** — `CHATTER_PALS` is an explicit index list, not a modulo over `PALS.length`, so new rows cannot recolour CHATTER (the only reference to `PALS.length` in the file is that comment).
- Helpers: `wrap(a)` (angle → −π..π), `rr(x,y,w,h,r)` (rounded-rect path; caller fills/strokes), `segDist(px,py,ax,ay,bx,by)` (point→segment — SCAN's swept beam, WEAVER's hazard-vs-thread), `segsCross(...)` (strict proper-crossing seg-seg test, shared endpoint = no cross — WEAVER's self-crossing test; `segDist` is the wrong primitive for that).
- Audio: `beep`, semantic wrappers `sHit/sPerfect/sGold/sBad/sStage/sTick`; `buzz(ms)` haptics (no-op on iOS Safari — never rely on it).
- FX: module-level `parts`/`pops` with `burst()`, `pop()`, `updateFX()`, `drawFX()`. Games clear both arrays in `init()`.
- Background: `drawBG(pal, skylineY)`; chrome: `drawChrome`, `homeRect`, `muteRect`, `inR`; screens: `drawReadyScreen`, `drawOverScreen`; ranks: `rankFor`/`nextRank` over `{s, t}` tables.
- Daily RNG: `mulberry32(a)`, `hashStr(s)`, `dayKeyOf(date)`, and `srnd(g)` = `g.srng ? g.srng() : Math.random()`.

### Game interface contract

Every game is an object literal implementing:

```
id        string, unique, lowercase
name      display name                     bound from the skin pack by applySkin() (7.0)
tag       one-line menu subtitle           bound from the skin pack by applySkin() (7.0)
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

**Pointer stream (D12, wired):** a game declares the stream by defining `press`. The router (`gameGesture` + `gameDown()`) then forwards down/move/up and never calls `tap()` during play; `ready`/`over` interactions still resolve through `tap()` **on release**. Works identically in free play and the gauntlet. `pointercancel` routes to `release(lastCoords)` — an abandoned drag can never hang a gesture. Consumers: HOWLER (press+drag+release, drag = aim arrow + power meter), TAZO (press+release, no drag by design), WEAVER (full consumer — all three carry live state). GUNGE, DAIRY (D13) and **RECALL** (7.5) are tap-only. Games without `press` keep zero-latency tap-on-pointerdown, untouched — so B6's release-latency cost does not apply to them. Keep `gameGesture` and `menuDrag` strictly separate.

### Determinism model for the daily (read before touching any `srnd` call)

The daily's promise is: **same day ⇒ same content for everyone**. The mechanism is Nth-draw determinism — the seeded stream's call *order* is what's shared, so every seeded draw site must consume a number of draws that does **not depend on player behaviour**, or the divergence must be inherent and acceptable:

- **Fixed-count-per-board draws** (GUNGE genBoard, WEAVER genBoard/rollBoats, DAIRY dealTray-per-placement, BONES per-toss shuffles, SCRAMBLE init, SWINGBALL per-hit zone): board/piece/throw N is identical for all players. ✅ *(HOWLER dropped from this list at the 6.1 redesign — no wind, no seeded content, deterministic like STACK.)*
- **Prefix-deterministic — RECALL (7.5)**, the strongest form in the codebase and worth understanding before copying it. The sequence is **one growing prefix of one stream**: `init()` draws item 1, each *cleared* round appends exactly one more, and a miss/replay consumes **nothing**. So draw count == rounds reached: a player who gets to round 20 and one who dies at round 8 share the first 8 items byte-for-byte. Skill changes how far down the sequence you get, never what it is. ⚠ Fragile in one specific way — a per-round decoy, shuffle or distractor draw would make consumption depend on the player again, which is B1 in a different costume. **One draw per round, or the property is gone.** Verified headless (a 2-miss run's sequence is a byte-exact prefix of a clean run's on the same seed).
- **Inherently divergent** (TAZO spawns — the board state, and thus empties, depends on player choices; SCAN's endless spawn stream): acceptable — the *sequence* of draws is shared, consumption naturally differs. ✅
- **Split-stream fix — WEAVER (B1, fixed 6.0)**: `spawnFlare`'s draw count varies with player pace, so it must never share a stream with board generation. WEAVER now derives a stable integer base (`dailySeed`, one fixed draw at seed time) and splits: a **per-stage board stream** (`brnd()` = `mulberry32(dailySeed ^ 'weaver-board' ^ (stage+1)·C)`, fixed draw count ⇒ boards identical for everyone) and a **separate hazard stream** (`hrnd()`, persistent ⇒ Nth flare route shared). Free play ⇒ `Math.random` throughout. Any future seeded feature must be checked against this model.
- **Daily game selection — D33 (shipped 6.6)**: `pickDailyGames(dayKey)` (defined beside `DAILY_NAME`/`DAILY_EPOCH`, just above `GAUNTLET`) shuffles `GAMES` on a stream salted `'daily-select'` — **separate** from `beginDaily`'s `seedBase=hashStr(dk)` used for each game's own board seed (`mulberry32(seedBase+idx*101)`) — so the day's 6-pick can never perturb any game's board draws, and vice versa. One fixed-size Fisher–Yates shuffle, so the pick is identical for every player on the same day; the TAZO/DAIRY-collision swap is a deterministic function of the same shuffle, not a second draw. ✅

### Router

- `mode`: `'menu' | 'game' | 'gauntlet'`; `cur` = active game. Menu card tap → `cur=g; cur.onOver=null; cur.srng=null; cur.init(); mode='game'`. Global `tick` drives pulses. Frame loop clamps dt to 0.05.
- **Menu**: page 0 = full-page TUCK SHOP RUN tile; game cards 2×2 from page 1 (`PER_PAGE=4`; `menuPageCount()` = `1+ceil(GAMES.length/4)`, so **12 games ⇒ 4 pages** — RECALL filled the gap the 11-game roster left on the last page and moved no existing card). The count next grows at **13 games** (LOCK). Card selection resolves on `pointerup` (drag = page swipe); game taps fire on `pointerdown`. Swipe, dots/arrows, ←/→ keys all page. Menu subtitle count derives from `GAMES.length`.
- **`PALMAP`/`gamePal(id)`** — the **one** module-level id→palette source of truth, with a `PALS[2]` fallback so a missing entry degrades instead of freezing the frame loop (the GUNGE-launch crash). Adding/removing a game = one line here. `drawMenuIcon` branches hardcode their own colours (its old lookup was dead code and was deleted).
- **`paused`**: set on `visibilitychange` mid-play (game or gauntlet); loop skips `update`, draws `drawPauseOverlay()`; tap/space resumes.
- **`GAUNTLET`** — daily-only controller outside the game contract. `beginDaily()` is the sole entry; consumes the attempt at start (bailing forfeits); sets `this.order = pickDailyGames(dayKey)` — the day's `DAILY_PICK=6` of 11 (D33) — and sequences it to first game-over via `onOver`; always seeds `mulberry32(seedBase + idx*101)` before `init()` (`idx` is now 0..5, per-game seed stream unaffected by the pick); normalises `contribOf = round(score/par × 250)` cap 625; renders `playing → interstitial → end → sharecard`; reads only public fields.
- Bests cached via `refreshMenuBests()`; daily tile via `refreshDaily()`; both refresh on home-exit. `resize()` only on resize/orientation events (rule 2). Frame loop starts after the font loads (1.5s timeout fallback).

### Skin packs (Phase 7.0)

`SKINS` sits between the shared screen helpers and the game blocks. A pack is `{label, hub:{dailyName}, games:{<id>:{…}}}`; `nz90` is the default and currently the only one. `DEFAULT_SKIN='nz90'` and a module-level `themeId` select it — 7.1 will load `themeId` from `arc_theme`.

- **`applySkin()`** binds `name`, `tag`, rank **titles** (into the game's own `ranks[i].t`, never the `{s}` thresholds), BONES' `LADDER`, and DAIRY's `SHAPES[i].name`/`.tag` onto the live objects. It runs **immediately after `const GAMES=[…]` and before the first `init()`** — `LADDER` is `[]` and `SHAPES` entries carry no name until it does. Re-run it whenever the skin changes.
- **`S(id,key)`** is the live lookup for copy read at draw/event time (ready lines, over titles, banners). Missing keys fall back through `SKINS[DEFAULT_SKIN]`, so a partial future pack degrades instead of drawing `undefined`.
- **`dailyName()`** replaces the old `DAILY_NAME` const (six call sites).
- **`shareMeta(id)`** reads `shareIcon`/`shareLabel` off the active pack (D34), keeping its generic fallback for daily-state rows saved before D34.
- Adding a game means adding a pack entry as well as a `PALMAP` line. Adding a *pack* means one more key under `SKINS` — nothing in the game objects changes, by construction (rule 12/D20).

### Palettes and CHATTER's cycle

CHATTER is the only palette-cycling game; its cycle is the **explicit list** `CHATTER_PALS=[2,3,4,5,0,1]` (decoupled from `PALS.length`, so appending palettes can't recolour it). Known quirk: stage 0 opens on the hub purple and mismatches its green menu card — pre-existing, owner call pending (§7). Drawing a surface directly (BONES asphalt precedent, HOWLER rocket livery) remains fine where a full palette isn't warranted.

---

## 4. Current games — as-shipped truth + tuning constants

*One entry per game, in `GAMES` order — which since D33 is the **menu/card** order, not the daily order (the daily is a day-seeded shuffle of a `DAILY_PICK` subset). All constants provisional unless stated — retune from telemetry (Unit 6.0), not feel-in-a-vacuum. Owner play observations so far are folded in bold.*

- **STACK** (`par:20`, PALS[0], `spaceTap`) — block height `BH 42`, start width `SW 264`, perfect window `PFCT 5 + spd*0.4`px, speed 2.4 → cap 9.5 (+0.13/block). Perfect preserves width; ≤6px landing = topple. Deterministic — no seeded content.
- **SCAN** (`par:35`, PALS[1]) — 3 lives (escape = life lost); $1/$2/$3 chips + ~15% QR (+5 gold). Speed mult `1 + score*0.02`. Beam collision tests the segment swept by the tip per frame (`segDist`).
- **HOWLER** (`par:1800` PROVISIONAL, PALMAP 0, press/drag/release) — **side-on 2D distance course** (6.1 redesign, D29–D32; supersedes the old pseudo-3D depth-swipe). Swipe sets **launch angle + power**; the howler flies a real parabola left→right and you clear four ground markers **in order** (30·60·90·ULTIMATE, `TARGETS_M`), camera panning right in flight then snapping back to the pad. `M2PX 7.5` (owner retune — pushed the targets further out; the old `6` sat them too close / needed too little power → 45° powers now ~0.65→1.30 across 30→ULT, up from ~0.50→0.99). `release` → `vel` (logical px/ms); `power = clamp(vel/VREF 2.0, 0.35, 1.9)` → launch speed `SPEED_REF 865`·power px/s; angle = `clamp(atan2(-dy,|dx|), ANG_MIN 0.25, ANG_MAX 1.48)` rad, always thrown rightward; integrate under `GRAVITY 1400`; land when it returns to `GROUND_Y 560`. Clear = landing within the target's band (`BAND_M 9`→`BAND_MIN_M 4.5`, narrows down the course); a miss just **costs time** — **no lives**; **two terminals** (D30): the ULTIMATE clear (a **win**) or the **time budget running out (~75s)** (a graceful budget-exhaust, banking whatever centre bonus was earned — the daily always advances), both firing `onOver` once (an in-flight throw at the buzzer resolves first). **Scoring (D31, higher-is-better):** a bleeding `TIME_BUDGET 3000` at `TIME_RATE 40`/s (fast = high) + per-clear centre bonus `round(CENTER_MAX 60 × closeness)`, all `× (1 + WHISTLE_STEP 0.25 × whistles)`; floored at 0, shown live. **Whistle (D32):** absolute swipe-power band `2.45–2.75` px/ms (centred on ULT's ~1.30 45° power after the retune), reachable at **every** distance via angle (near = steep lob, far ≈ 45°) — **co-tuned** so whistle power at 45° reaches ULTIMATE (`~1.30²×865²/1400 ≈ 903 ≥ 900px`) and `ANG_MAX` is wide enough that the near-target lob (30M needs ~79–85°) is reachable; higher-variance risk/reward; each whistle *clear* stacks toward a ×2 all-whistle WHISTLER run. **No wind — deterministic** (no seeded content, like STACK; §3). Feedback: aim **arrow** (angle) + **power meter** with the whistle band marked + a landing splash; **no landing predictor** (would trivialise a deterministic arc). HUD: score/best, run timer (reddens in the last quarter of the budget), 1-of-4 progress, off-screen active-target chevron. Rocket redrawn as a foam football head + long finned tail (purple `#4a1e8a→#8a4ae0` / teal `#2ad8b0`, owner-locked; `drawRocket(x,y,s,ang)` rotates to velocity). **All constants `// PROVISIONAL`; on-device `CALIBRATE` whistle pass still owed (§7).**
- **SWINGBALL** (`par:40`, PALS[3] + gold, `spaceTap`) — 45° ellipse orbit `RX 126`/`RY 54`, pole 120→470, depth-ordered ball (far half behind pole), swaying cap/coil. Speed 1.7 (+0.05/hit, cap 4.2), zone 0.42 → floor 0.14 (−0.012/hit), perfect = inner 33%, slop `1.7×` zone (mistime = unwind + survive). Wind meter 100 (+20/+30/−25) banks a LOOP (×(1+loop)), steps baseSpeed +0.25 / baseZone −0.03, 300ms grace. Owns the `wasInside`/sign-crossing overshoot core as the reference implementation (ex-CHAIN) — edit with care. Break only on wild tap or untapped overshoot.
- **GUNGE** (`par:300` — **known too low, revised guess ~1200, §7**; PALS[6]) — N×N pipe rotation puzzle (5, 6 from stage 3); tap rotates 90°; openings are direction bitmasks (N1 E2 S4 W8). **D16 solved-then-scrambled**: carve one src→dest path (randomised DFS), path cells get the exact 2-opening shape their connections require (never branches into a decoy → cannot leak), decoys fill the rest, rotations-only scramble ⇒ provably solvable (verified 4000 boards). Source-connected pipes are lit. **D17 RELEASE valve** outcome matrix: manual+solved → `+remaining×segments`, advance · manual+mis-solved → leak, −1 life · auto@0+solved → 0 pts, no life lost · auto@0+mis-solved → leak, −1 life. Timer `48 − stage·3.6`, floor 21 (3× original, post-buff). 3 fails = over; the clock is a puzzle clock, never a score cap (D3). **Contestant shipped (6.3): a chibi sits in a clear `tankRect` (x48–312, y456–508) directly below the grid; the RELEASE valve moved down to y514 to open the band (`GY` kept — the clock bar at y135 blocks moving the grid up). On a solved release the gunge pours from the dest lip into the tank (`this.pour`) and coats the contestant (`this.gunged` 0..1, drips off ~2.4s); a leak leaves them clean. Live `BANK +N` / `N PIPES LIT` readout added by the clock (surfaces the `remaining×segments` gamble). Owner to approve the look on-device.**
- **CHATTER** (`par:600` owner median, `CHATTER_PALS` cycle) — 5 slots at 72° from −54°, 2 discs live at start, +1 disc every 2nd stage, stage every 14s, decay 4.0 +2.1/stage. Restrike: ≤20 = PERFECT SAVE (+45, +8pts); ≥80 = +8 energy; else +30. Score `3.6 × avgEnergy/100`/s, ×2 during FULL CHATTER (all ≥85). `fumble()` (−4 all) only on the 2nd consecutive miss. **Harder than its scores suggest — do not nerf it, do not add a time cap. Its difficulty is its cap.**
- **TAZO TYCOON** (`par:2000` PROVISIONAL post-6.2b, PALS[7], press+release, no drag) — 4×4 directional-merge; tiers 0 slate → 1 orange → 2 holo cyan → 3 gold → 4 mega slammer white (`TAZO_COL` ramp, recoloured 6.2/6.2c for separation — all `// PROVISIONAL`; + centred `drawMark`). Merge once per tile per move; slide tween + double-strike bounce + `clack()`. **Mega slammer detonates orthogonal CLUTTER only (tier ≤ `SLAM_MAXTIER 1`) and is consumed — full-radius clearing makes the board a perpetual-motion machine (sim-proven); do not "restore" it.** Spawn count `min(12, 1+floor(moves/SPAWN_K 9))`. **Difficulty (6.2b, D28): ALL spawns are tier-0 (`SPAWN_TIER 0`) — a tier-0 can't merge into the mid/high tiles you build, so it clogs the board and bounds even careful play (sim: median 3.3→~1.3 min, p90 10+→~2.2 min). This REPLACED the old score-scaled `TIER_BANDS` (removed), which gave relief and never ended a skilled run. Tune length with `SPAWN_K`, not the pool.** Loss = board-lock, ONE terminal state (owner-approved D14 exception). Time caps forbidden (D3). **Owner reopened + delegated the difficulty; all-tier-0 was the owner's idea, sim-confirmed. Colour: tier-1 → vivid orange-red (6.2c, was kraft brown — still too close to gold on the dark board), tier-3 gold pushed yellower; screenshot-verified. All `// PROVISIONAL`.**
- **SCRAMBLE** (`par:120` — likely too high post-R1, and further down post-S2; PALS[4]) — 5 kid slots, **start 3 active** (S1/D35, was 4); lolly arcs ~0.45s, +34 haul; decay `6.5 × decayMul(0.7–1.6)` +2.8/stage, 7s stages (S2 pressure knobs, 2026-07-22 — was `decayMul(0.8–1.4)` +1.9/stage, 10s stages; all `// PROVISIONAL`, D24). Score = evenness: `5.0 × (1 − stddev/50)`, FAIR SHARE ×2 (all ≥60 within band 20). Empty meter = kid cries 1.4s (returns at 42) −1 life; 3 = over. Deferred from spec: spatial greedy-kid drift. **S1 arrivals ✅ SHIPPED (2026-08-01, D35) — a walk-in QUEUE over the same five fixed slots, no 6th slot:** a kid held at `FULL_THRESH 88` for `FULL_HOLD 1.8`s walks off **happy** (`kidLeaves` — slot freed, haul cleared, green/gold burst + `FULL UP!`, **never a life**, deliberately distinct from `kidEmpty`'s cry) and plays a `LEAVE_TIME 1.2`s walk-off (drawn by `drawKidBody(k,x,y,leaving)`, drifting out-and-forward, fading, no meter) which doubles as the refill gap. Slots refill via `wakeKid()` (haul 60, R1 precedent) at every stage boundary **and** once mid-stage at `MID_WAKE_AT 0.55` of the interval, so arrivals keep coming all run instead of exhausting after two. `MIN_ON_FIELD 2` floors the crowd so a well-fed board can't empty itself. **Both paths are RNG-free** — arrival/departure counts track player pace, so putting a draw there would break daily determinism; `init`'s five `srnd` `decayMul` draws remain the game's only seeded draws (§3, verified fixed at 5 regardless of run length). Sim-tuned (92/2.6 fired ~0×/run; 88/1.8 lands ~5–7 departures + ~9 arrivals per ~55s run). `xs`/`KID_Y`/`THROW_X`/`THROW_Y`/tap radius unchanged. All new constants `// PROVISIONAL`.
- **DAIRY WARMER** (`par:2200` sim-informed — greedy bot *over*-states a human, may need to come DOWN, more so now DD1/DD2 tightened the board; PALS[8], tap-only D13) — **5×4 shelf** (DD1, 2026-07-22, was 6×4), 50px cells, `GX` computed from `(W−COLS×CELL)/2` in `init()` (no hardcoded width); mince-&-cheese 2×2 / sausage roll 1×3 / potato-top L / **custard square S-tetromino / cheese roll T-tetromino (DD2, 2026-07-22)**; orientations derived+deduped in `init()` (generic over shape size). Tray of 3, **place in any order (D18)**; loss check = "no tray piece fits anywhere" (`trayDead`); STRIKE wipes the shelf + fresh tray (can't chain in one frame) — **DD4 (weakening this relief) is parked, not built** (§7, needs an explicit owner reopen); 3 strikes = over. Full horizontal row sells: `+60×(2^rows−1)`, FRESH! banner. **Redraw (2026-07-22, cosmetic):** shelf cells store `{col,shape}` and each placed cell draws a shape-specific texture (cheese-ooze corner / flaky hatch / mash-crown scallop / icing speckle / rolled swirl); warmer chrome adds an amber radial glow, faint condensation streaks on the glass, and a decorative `$` price tag per tray piece; a sold row spawns a small pastry sprite (`this.slides`) that slides off-shelf with a `$` and fades, instead of the row silently vanishing. Invalid taps are free no-ops with a red misfit ghost (§9).
- **KNUCKLEBONES** (`par:90`, PALS[5] on drawn asphalt) — parabolas to `CATCH_LINE 480` from 5 slots; `BASE_LV 560`/`BASE_G 700`; RUNG speeds flat 1.00, speed ramps with time only (`×1.04`/rung-up, extra `×1.04` on loop). Gold = catch, grey = skip; missed catch or grey tap = drop; 3 = over. Ladder ONESIES→OVER THE FENCE, `CLEARS_PER_RUNG 2`, loops. `+2×(1+loop)`/catch, `+5×(1+loop)`/clean toss. Frame-flip sells the spin.
- **WEAVER** (`par:1000` sim-informed, PALS[9], FULL pointer-stream) — **D16 by inversion**: carve a random simple path of exactly `L = min(14, 7+stage)` cells on a virtual 4×4 grid; carved cells BECOME the knots (+ jitter ≤12; cell 75 ≥ 2·SNAP_R 24 + 2·JITTER ⇒ snap circles never overlap) ⇒ carve order is Hamiltonian by definition; decoys (grid-adjacent, non-consecutive, `min(6, 2+stage)`) added after. Verified 36k boards. **D19**: rope-constrained moves; thread starts only on the two carved-endpoint bollards; lives are hazard-only (boats + flares vs live thread via `segDist`; release/blocked-rope/off-bollard all free); clock is a bonus meter only (`max(14, 30−stage·1.5)`, banks `remaining×1` on completion, expiry costs nothing). Self-crossing via `segsCross`. Unwind by dragging back onto the previous knot. **B1 fixed (6.0): board draws (`brnd`) and flare draws (`hrnd`) run on split seeded streams — daily boards are now identical for all players regardless of weave pace.**
- **RECALL** (`par:400` PROVISIONAL, PALS[10], tap-only) — **Phase 7.5, game 1 of 4; shipped 2026-08-02.** Five pads on CHATTER's ring geometry (`CX W/2`, `CY 288`, `RING_R 100`, slots at `-54+i·72°` so none sits under the brand tag; `PAD_R 26`, `HIT_R 34`). Each round replays the **whole** sequence, one item appended. Playback interval `max(PB_FLOOR 0.26, PB_BASE 0.62 − PB_STEP 0.03·(n−1))`, lit for `PB_LIT 0.62` of it, after a `PB_LEAD 0.5`s beat — the floor exists because watch time otherwise grows quadratically (round 14 would be a ~9s spectate). Phases `lead → watch → input → clear|miss`; taps outside `input` and taps off any pad are **free no-ops with a red reject flash**, never a miss and never a life (§9). **Scoring:** each correct item banks the current round length, so a **cleared round always pays exactly n²**; **RECITE ×1.5** (`RECITE_MULT 0.5` on top) when every tap in the completing pass lands within `FLUENT_GAP 0.55`s of the previous one (the first tap is timed from playback end). A **watermark (`deepest`) persists across a replay**, so re-walking already-banked items pays nothing — without it, failing on the last item of a long round and re-clearing would double-bank it (verified: clean and sabotaged round-6 clears both pay +54). A run that ends mid-round keeps its partial credit, which is what stops memory scoring being an all-or-nothing cliff that normalises badly against par. Miss = −1 life + same round replayed from the top; 3 = over (endless + 3 lives, D39 — no D14 exception). **The channel is visual** (rule 5): `TONES` are reinforcement only, and the lit pad alone reproduces the sequence with `arc_mute` on (verified headless; on-device pass still owed, §7). Pads carry a drawn waveform arc so a lit pad reads without relying on colour. Centre is a drawn modem whose LEDs blink while the line negotiates and hold steady while you answer; `LISTEN`/`REPEAT` readout plus per-item progress pips below the ring. **Theme (dial-up handshake) is D15-cleared** (owner, 2026-08-02) — pack copy is `CONNECTION LOST` / `HANDSHAKE!` / `LINE NOISE!`, ranks `ENGAGED TONE → 56K FLAT OUT`, receipt line `📞 dial-up`; no ISP or product is named anywhere in it. `par`/ranks `// PROVISIONAL` until D41's single all-15 retune.

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

#### 6.1 — HOWLER redesign: side-on 2D distance course — ✅ SHIPPED (owner to approve on-device)

**Shipped as designed below**, with two co-tuning deviations from the draft constants (both `// PROVISIONAL`): `SPEED_REF 900→1010` (so whistle power at 45° reaches ULTIMATE — the D32 co-tune) and `ANG_MIN/MAX 0.30/1.40→0.25/1.48` (so the near-target whistle lob, ~79–85° at 30M, stays reachable). Verified headless: physics reachability (every target clearable at ~45°, whistle band hittable at all four within angle limits), a full driven run (in-order clears, win-only terminal, `onOver` fires once, deterministic/seed-invariant, misses don't fail, speed-first floored scoring), and Chromium screenshots of ready/aim/flight/over. The §4 entry is the as-shipped truth. **Follow-up shipped same day:** the owner asked for the **budget-exhaustion terminal** (run ends when the time budget hits 0 at ~75s), so the daily always advances even if ULTIMATE is never cleared — D30 now has two terminals; the §7 flag is resolved. **Owner retune (2026-07-21):** targets were too close / needed too little power, so `M2PX 6→7.5` (further out) and `SPEED_REF 1010→865` (firmer swipes: 45° powers ~0.65→1.30 across the course, up from ~0.50→0.99), with the whistle band re-centred `1.85–2.15→2.45–2.75` px/ms to keep it on ULT's 45° power. Re-verified headless (reachability + the full driven run) and screenshot. All `// PROVISIONAL`. The design spec that shipped:


**Why.** The pseudo-3D depth-swipe played very hard and the "how hard, not when" idea never read clearly as *distance*. Owner call: rebuild HOWLER as a literal distance game — throw the Mega Howler left→right down a field, clear the ad's four markers in order, race the clock. The swipe verb (press/drag/release, D12) stays; everything downstream of the release changes.

**Mechanic (build to this):**

- **True 2D parabola — no `z`, no depth-scale, no LOFT-as-depth.** The release vector gives **both** launch angle and power. `power = clamp(vel/VREF, POW_MIN, POW_MAX)` → launch speed; swipe angle → launch angle (clamped `ANG_MIN..ANG_MAX`); integrate under `GRAVITY`. Range runs along the ground (x), loft is literal vertical arc (y).
- **Fixed course, four targets in order: 30 · 60 · 90 · ULTIMATE** (`TARGETS_M`). One active at a time; land inside its band → clear it, arm the next. Bands **narrow down the course** (`BAND_M` at 30 → `BAND_MIN_M` at ULTIMATE). Cleared markers grey out behind you.
- **Camera pans right** with the howler in flight, then snaps back to `LAUNCH_X` for the next throw. World is wider than 360; a marker sits at `LAUNCH_X + M2PX × distance`.
- **Timed, no lives (2nd D14 exception — see D30).** Clock runs from the first throw. A miss (outside the active band) clears nothing and just costs time — you re-throw. **Two terminals** (owner-extended at build): ULTIMATE cleared (a *win*), or the time budget running out (~75s, a graceful budget-exhaust so the daily always advances). Both fire `onOver` once (gauntlet/daily read `score`/`par` unchanged).
- **Scoring — speed-first, higher-is-better (D31).** `TIME_BUDGET` points bleed at `TIME_RATE`/s across the run; you bank the remainder (floored at 0) — fast = high. Each clear adds a small centre bonus `round(CENTER_MAX × closeness)` (0 at band edge → 1 dead-centre) — the "closer = more" layer, small next to time. Whistle stacks a final multiplier (below). Net: `score = round((timeScore + Σcenter) × (1 + WHISTLE_STEP × whistles))`.
- **Whistle — a power band, hittable at EVERY distance (D32).** Absolute swipe-velocity band `WHISTLE_LO..WHISTLE_HI`. Because angle is free, that one fixed power reaches any target: **lob steeper for near markers, flatter (~45°) for far ones** (owner's spec). Co-tune so whistle-power max range (≈45°) ≥ ULTIMATE, and so whistle power *overshoots* near targets at moderate angles → a near-target whistle demands a steep lob (or flat skim). High power amplifies angle error into big range error, so whistle throws are **higher-variance** — a risk/reward layer, not a free upgrade. Each whistle *clear* adds `WHISTLE_STEP` (0.25) to the final multiplier; whistling all four → ×2, the WHISTLER run. Wind-immunity is retired (no wind). Keep the `CALIBRATE` velocity-histogram harness — the band **must** be measured on a real phone (a window nobody can hit is worse than none).
- **No wind.** Delete `WIND_*`, `windStrength`, `rollWind`, `drawWind`, and the seeded-wind path. HOWLER becomes deterministic.
- **Aim feedback:** aim **arrow** (angle) + **power meter** (strength, with the whistle band marked so the power skill is learnable) — **no landing predictor** (keeps the old "direction is honest, a landing arc over-promises" rule; with no wind the arc is deterministic, so a predictor would trivialise it). Whistle-zone marker is `// PROVISIONAL` — it teaches the power target; angle stays the skill.
- **HUD:** drop the three life chips; add a **run timer** and a **1-of-4 progress** readout; keep score/best/rank/next-rank.

**Rocket redraw (unchanged from the prior 6.1 — still needed, no data required).** The real Vortex/Aero Howler is a **foam football-shaped head with a long tail shaft and three swept fins** — tail ≈ as long as the head, whistle holes in the head. The current sprite is a stubby rocket with base-corner fins. Redraw `drawRocket()`: ovoid purple head (`#4a1e8a→#8a4ae0`, owner-locked), slim shaft, three large swept **teal** fins (`#2ad8b0`) at the rear, whistle-hole dots, optional cream nose. Life icons reused it — but lives are gone, so only the flying rocket + the `drawMenuIcon` howler branch need to match. Livery stays purple/teal (settled).

**Constants (top of the HOWLER object, UPPERCASE per §9 — all `// PROVISIONAL`):**

```js
// ── course (metres → world px via M2PX) ──
M2PX:6,
TARGETS_M:[30,60,90,120],        // [3] = ULTIMATE_M; fixed order
BAND_M:9, BAND_MIN_M:4.5,        // clear half-width; narrows 30→ULT
CENTER_FRAC:0.35,                // inner fraction that reads as dead-centre

// ── launch physics (true 2D parabola — NO z/depth) ──
GRAVITY:1400, LAUNCH_X:40, GROUND_Y:560,
VREF:2.0,                        // swipe px/ms → reference launch speed
SPEED_REF:900,                   // px/s at power 1.0
POW_MIN:0.35, POW_MAX:1.9,
ANG_MIN:0.30, ANG_MAX:1.40,      // rad — allows steep lobs for near-target whistles

// ── gesture gates (kept from current HOWLER) ──
MIN_DIST:20, MAX_DT:600,         // stray/lazy swipe = free no-op (§9)

// ── whistle (absolute power band; reachable at ANY distance via angle) ──
CALIBRATE:false,                 // keep the velocity-histogram log (measures input)
WHISTLE_LO:1.85, WHISTLE_HI:2.15,// MEASURE ON DEVICE, then set
// co-tune: whistle-power range(≈45°) ≥ ULTIMATE, and whistle power overshoots
// near targets at moderate angles → near whistles need a steep lob. That's the reward.

// ── scoring (speed-first; higher = better) ──
TIME_BUDGET:3000, TIME_RATE:40,  // budget bleeds over the run; remainder = the spine
CENTER_MAX:60,                   // max centre bonus per clear (×4 targets)
WHISTLE_STEP:0.25,               // each whistle clear +25% final; 4 → ×2 (WHISTLER)
```

**Ready-screen copy** (replace the wind line):
`['SWIPE TO THROW — ANGLE + POWER','CLEAR 30 · 60 · 90 · ULTIMATE, FAST','FIND THE WHISTLE — LOB THE CLOSE ONES']`
**Over-screen** stays shared; title e.g. `ULTIMATE!`, sub-line `elapsed+'s · '+whistles+' WHISTLE'`.

**On-build doc updates (do these in the same commit that ships the redesign):**
- Rewrite the **§4 HOWLER entry** to as-shipped truth (it currently documents the depth-swipe — leave it until then; code is source of truth).
- **§3 determinism bullet:** remove `HOWLER per-resolve wind/target` from the fixed-count list — HOWLER becomes deterministic (no seeded content, like STACK).
- **§3 pointer-stream note:** HOWLER `drag` = aim arrow + power meter (was "aim ghost only").
- **§6.5 par table:** re-derive HOWLER `par` + rank thresholds from the *new* scoring model's telemetry (the old "500, likely down" is void — different scale).
- Confirm `TIERS`/`TIER_HITS`, `lives`/life-chips, and all wind/depth code are gone.
- `arc_howler_best` key is unchanged (§2). B8 (pointercancel) still applies.

**Still owed:** the on-device `CALIBRATE` pass for the whistle band (`VREF`/band are synthetic until measured).

#### 6.2 — TAZO refinement (owner: "easy" + colours too close) — ✅ SHIPPED (difficulty 6.2b + colours 6.2/6.2c)

- **Difficulty — ✅ SHIPPED 6.2b (D28, owner reopened 2026-07-20): ALL spawns are tier-0.** D27 had deferred difficulty because the sanctioned `SPAWN_K`/`TIER_BANDS`/`SPAWN_CAP` levers are inert for skilled play — a faithful sim showed `SPAWN_K 6→4` moves the greedy median only 3.30→3.05 min, lowering bands *raises* it, lowering `SPAWN_CAP` *lengthens* runs, and the ~12 min p90 tail is immovable (a good player out-clears spawn pressure). The owner then proposed making every spawn **tier-0**, and the sim confirmed it works where the levers couldn't: a tier-0 tile can only merge with another tier-0, so it can't be cleared into your accumulated mid/high tiles → the board clogs → even careful play is bounded. **Median 3.3→~1.3 min, p90 10+→~2.2 min** (greedy+survival). Shipped: `TIER_BANDS` removed, `SPAWN_TIER=0`, `SPAWN_K 6→9` (length comes from the clog, not a spawn flood), `par 4500→2000` + ranks rescaled — all `// PROVISIONAL`, final numbers at 6.5. Every tier-0 mix tested was *harder* than pure tier-0, so tune length with `SPAWN_K`, never by re-adding higher spawn tiers. Verified against the real game code (40 runs terminate, every spawn tier-0, no throw). **Caveat:** a rare optimal-survival merge loop can still run long (~1.5% of bot runs) — accepted vs the old *common* 10-min runs. (Sim reproducible from the §4 mechanics.)
- **Colour separation — ✅ SHIPPED 6.2 + 6.2c (`// PROVISIONAL`).** 6.2: tier-0 plastic blue → **slate grey** (`#9aa7b8`; old blue was ΔE 11.6 from the board ring — the clash). 6.2c (owner still found brown↔gold too close on the dark board): tier-1 kraft brown `#b06a34` → **vivid orange-red** `#ff6a2e`, and tier-3 gold pushed yellower `#ffd24a→#ffe23a` — tier-1↔tier-3 ΔE 49.4→68.9, tier-1↔bg ΔE 107. Tiers now read grey / orange / cyan / gold / white — **screenshot-verified on the dark board** (four unambiguous hues). Menu-icon fan updated to match; all retired hexes gone.

#### 6.3 — GUNGE contestant (owner request) — ✅ SHIPPED (owner to approve the look on-device)

- **Contestant + tank** ✅: a chibi (SCAN/SCRAMBLE outline+fill idiom, rule 4/9) sits in a clear `tankRect()` (x48–312, y456–508) directly below the grid. On a **solved** release the gunge pours from the dest lip down onto them (`this.pour` = wobbling stream + splash burst) and coats them green with drips (`this.gunged` 0..1, grimace + squeezed eyes while pouring, drips off over ~2.4s); on a **leak** they stay clean. Verified headless (solved→gunged, leak→clean, no throw) and by Chromium screenshot.
- **Layout — chosen: tank below the grid, valve nudged down.** Of the three options, this one (option 1 family) reads best — the pour lands straight from the dest onto the contestant. `GY` was **not** moved up (the doc's literal option-1 suggestion): the clock bar sits at y135 and the grid top is y150, so there's only ~8px, not the assumed ~10–14. Instead `valveRect()` moved y466→y514 (size 148×46 **unchanged** — tap target not shrunk); the freed band (450→514) holds the tank. Tile tap targets untouched (`GY`/`GRID_PX` unchanged).
- **Bank-early readout** ✅: a live `BANK +N` (green, when the board currently solves) / `N PIPES LIT` label sits by the clock bar, surfacing the `remaining × segments` gamble.

#### 6.4 — Remaining per-game polish (owner ideas land here)

Running list — append as observations arrive; each item stays independently shippable:
- SCRAMBLE: par likely down (post-R1); watch for the deferred greedy-kid-drift idea.
- CHATTER: resolve the stage-0 hub-purple question (§7) whenever CHATTER is next touched.
- **SCRAMBLE — harder (owner-directed, 2026-07-22): S2 ✅ SHIPPED, S1 ✅ SHIPPED (walk-in queue, D35).**
  - **S1 Arrivals ✅ SHIPPED (2026-08-01, D35 — walk-in queue, no 6th slot).** Start 3 active (was 4); a kid fed to sustained contentment (`FULL_THRESH 88` held `FULL_HOLD 1.8`s) walks off happy, freeing their slot with no life cost; the slot refills after a `LEAVE_TIME 1.2`s gap at the next stage boundary or the mid-stage wake (`MID_WAKE_AT 0.55` of `STAGE_INTERVAL`), so arrivals keep flowing all run. `MIN_ON_FIELD 2` stops a well-fed board emptying. Selection is deterministic (lowest free slot) and the departure trigger is threshold-driven — **no RNG in either path**, so no new seeded draw site (§3); `init`'s five `decayMul` draws remain the only seeded draws, verified fixed at 5 across short and long runs. As-shipped detail in §4. All magnitudes `// PROVISIONAL` pending 6.5 telemetry; `par:120` untouched here.
  - **S2 Pressure knobs ✅ SHIPPED** — `DECAY_GROWTH 1.9→2.8`, `STAGE_INTERVAL 10→7`, `decayMul` spread `0.8–1.4→0.7–1.6` (all `// PROVISIONAL`, D24). No layout change; `par:120` untouched here, still due for the 6.5 retune.
  - **Constraints held:** stays endless + 3-lives (SCRAMBLE is **not** a D14 exception — the cry-at-empty loss path is unchanged). No new seeded draw sites — the three knobs are static constants, §3 doesn't apply.
- **DAIRY WARMER — redraw + harder (owner-directed, 2026-07-22): redraw + DD1 + DD2 ✅ SHIPPED; DD4 parked (§7).**
  - **Redraw ✅ SHIPPED (cosmetic, no data):** shelf cells now carry `{col,shape}` (was a bare colour string) so each placed cell draws a shape-specific texture — mince & cheese gets a cheese-ooze corner, sausage roll a flaky diagonal hatch, potato-top a mash-crown scallop, + DD2's custard square (icing speckle) and cheese roll (rolled swirl). Warmer chrome added: amber radial glow behind the shelf, faint wobbling condensation streaks on the glass, a decorative `$` price tag per tray piece (`SHAPES[i].tag`, cosmetic only — not the D34 receipt price). A sold row no longer just vanishes: `this.slides` spawns a small coloured pastry sprite that slides off the right edge with a `$` and fades (`update`/`render`), then the burst/FRESH! banner fires as before.
  - **DD1 Tighter shelf ✅ SHIPPED** — `COLS 6→5`; `GX` is no longer a hardcoded constant, it's computed in `init()` as `(W−COLS×CELL)/2` so the shelf stays centred off the column count for any future width change.
  - **DD2 Nastier piece set ✅ SHIPPED** — added `CUSTARD SQUARE` (S-tetromino, 2 orientations) and `CHEESE ROLL` (T-tetromino, 4 orientations) to `SHAPES`; `buildOris()`'s rotate-normalise-dedupe is generic over shape size, no change needed there. Verified headless: every shape/orientation still fits somewhere on the tightened 5×4 board.
  - **Constraints held:** stays inside D18 (tray-of-3, `trayDead` loss, strike wipes, 3 strikes = over) and D13 (tap-only). **DD4 (strike relief) is untouched** — parked per §7, needs an explicit owner reopen. `dealTray()` still draws exactly one seeded index per tray slot regardless of `SHAPES.length` — fixed-count, §3-safe (wider value range, same draw count). Par is unchanged here (`2200`, still due to possibly come **down** at 6.5).
- *(empty slots — this is the refinement idea log)*

#### 6.5 — Roster-wide par retune (needs 6.0 data) — ⏸ DEFERRED by D41

**Blocked until all 15 games have telemetry, then runs once** (D41): par is the exchange rate between games in the daily total, so retuning 11 and re-retuning 15 wastes the data and produces two incomparable eras of totals. The skew notes below stay valid as the inputs to that single pass.

Retune **all eleven pars together** from `arc_stats`, not piecemeal — they are the exchange rate between games in the daily total. Current provisional set and known skew: STACK 20, SCAN 35, HOWLER — void; the 6.1 redesign changes the scoring model entirely (speed→points, not centre/rim hits). Re-derive par + ranks from the new model's telemetry, not by adjusting 500. SWINGBALL 40, GUNGE 300 (**up**, ~1200 revised guess — `remaining×segments` is multiplicative and the 3× timer buff raised `remaining`), CHATTER 600, TAZO **2000 provisional** (was 4500 — 6.2b all-tier-0 difficulty fix (D28) shortened runs to ~1.3 min median / ~2.2 min p90 and lowered scores; confirm par + ranks from telemetry here), SCRAMBLE 120 (likely down further post-S2), DAIRY 2200 (may need **down** — the greedy sim over-states a clock-less human game, and DD1/DD2 tightened the board since), BONES 90, WEAVER 1000. Also retune `DAILY_PICK` (6.6/D33) from telemetry once there's real run-length data.

#### 6.6 — D33 (daily 6-of-11) + D34 (receipt share) — ✅ SHIPPED

- **D33 — the daily draws a seeded subset ✅ SHIPPED.** *(Count superseded: `DAILY_PICK` is **5** since RECALL landed, D37 — the mechanism below is unchanged, only the constant.)* Resolves the old "daily length" flag via the sanctioned seeded-subset option. `const DAILY_PICK=6;` and `pickDailyGames(dayKey)` sit next to `DAILY_NAME`/`DAILY_EPOCH`; `GAUNTLET.beginDaily()` now sets `this.order = pickDailyGames(dk)` instead of `this.order = GAMES`. Implementation is the **per-day independent shuffle** (Fisher–Yates over a copy of `GAMES`, seeded `mulberry32(hashStr(dayKey) ^ hashStr('daily-select'))`) — coverage across days is statistical, not a guaranteed rolling cycle, per the build brief's explicit recommendation (a persistent rolling bag was considered and rejected as unneeded complexity, and awkward across the TAZO/DAIRY constraint at shuffle straddles). The **≤1 of {TAZO, DAIRY}** constraint is applied to the same shuffle after the fact: if both land in the first 6, the later-drawn of the two swaps with the first non-TAZO/DAIRY game waiting past index 5 — deterministic, no extra draw. **§3:** the selection shuffle runs on a stream salted `'daily-select'`, separate from `beginDaily`'s `seedBase=hashStr(dk)` (used for each picked game's own board seed, `mulberry32(seedBase+idx*101)`, `idx` now 0..5) — picking the day's games can never perturb any game's own board draws. Verified headless: same-day picks are byte-identical across repeated calls (and thus across devices/loads), different days differ, 200 simulated days had zero TAZO+DAIRY collisions, and per-game coverage over 200 days was roughly even (100–128 appearances each). `idx`/interstitial `N/6`/finish-check/emoji-grid/`arc_daily_state.result` all already read `this.order.length` or map over `this.results` — no further change needed there. The free-play menu subtitle (`GAMES.length` = 11) is untouched — that count is the arcade roster, not the daily.
- **D34 — the daily share is a priced tuck-shop receipt ✅ SHIPPED.** `GAUNTLET.shareText()` now calls `buildReceiptText()`: one line per game (`shareMeta(id).icon`/`.label` + `'$'+(contrib/100).toFixed(2)` + `tierTile(contrib)`), a dot-leader separator row, and a `TOTAL` line whose price is `(this.total/100).toFixed(2)` — always exactly Σ line prices, since both derive from the same `contrib`/`total` numbers already computed by `contribOf`. `SHARE_NZ90` (id → `{icon,label}`) sits near `pickDailyGames`, with the owner-proposed nz90 snack set from the doc-updates brief; `shareMeta(id)` falls back to a generic `{icon:'🍽',label:'mystery item'}` tag rather than throwing, for daily-state rows saved before this unit. **Impl fix (the brief's required item):** `GAUNTLET.capture()` already pushed `id` into `this.results`; `finish()`'s stored `result` row and `beginDaily()`'s locked-day rebuild now both carry `id` too (`this.results=(st.result||[]).map(r=>({...,id:r.id}))`), so a locked/returning day's receipt can resolve its line labels — old pre-D34 saved days fall back to `shareMeta`'s generic tag instead of crashing. Verified headless: line prices sum exactly to the printed total, and the missing-`id` fallback renders instead of throwing. `emojiGrid`/`buildEmojiGrid()` are unchanged and still stored (back-compat), just no longer what `shareText()` builds from. **On-canvas redesign ✅ SHIPPED (owner-requested follow-up, 2026-07-22):** the optional visual upgrade originally deferred is now built — `drawBars` (shared by the interstitial, end screen, and share card) draws `$`+`(contrib/100).toFixed(2)` per line plus a small drawn tier-tile swatch (`tierColor(c)`, the canvas counterpart of `tierTile`'s emoji — a filled rounded rect, never a glyph, rule 4) instead of the old raw contrib number. Every on-screen running/final total (`drawStrip`, `drawInterstitial`'s SCORE line + RUNNING TOTAL, `drawDailyEnd`, `drawSharecard`, and the menu's `drawDailyTile`) now also renders as `$X.XX` instead of a raw integer, so the whole daily flow reads as one consistent receipt, matching `buildReceiptText()`. Verified with Chromium screenshots (fabricated a 6-line result set) of the daily tile, interstitial, end screen, and share card — all four show `$` prices and coloured tier swatches correctly, bar fill/rank/streak unaffected. **Layout follow-up ✅ SHIPPED (owner-requested, 2026-07-22):** with the roster down to 6 lines/day (D33, was tuned for up to 11), `drawBars` rows were spread out — `drawDailyEnd`/`drawSharecard` row height `30/32→58`, `drawInterstitial` (max 5 rows shown, no rank line) `30→48` — and a faint dotted rule (`ctx.setLineDash`) now separates each line, echoing the dot-leader rows in `buildReceiptText()`'s copyable text. Name/price font bumped `6px→7px→9px` (owner asked for bigger again, same day) and the tier tile `8px→10px`, rank line `5px→6px`, for legibility at the wider spacing. Re-verified with Chromium screenshots at max row counts (6 on end/sharecard, 6 forced on interstitial as a conservative overflow check though real play caps it at 5), stress-tested with the roster's longest names (KNUCKLEBONES, DAIRY WARMER, TAZO TYCOON) — no overlap with the price/tile column, clear margin above the COPY/SHARE buttons, the NEXT/TAP-TO-CONTINUE text, and the sharecard's bottom border in every case. `shareLabel`/`shareIcon` are still owner-uncleared per D15 pending sign-off. **They moved into the pack at 7.0** (`SKINS.nz90.games.<id>.shareLabel`/`shareIcon`); the interim `SHARE_NZ90` object is gone and `shareMeta(id)` now reads the active pack, keeping its generic `{icon:'🍽',label:'mystery item'}` fallback for pre-D34 saved rows.

---

### Phase 7 — SKINS (cosmetic era packs)

**Model (D20–D23, settled):** a skin is a **content pack over frozen mechanics**. Every game keeps its verb, par, seed behaviour, scoring, `id`, and `bestKey`; the skin swaps what things are *called* and *look like*. The default pack is the current NZ 90s/00s theme (`nz90`). Future packs re-reference the same games into another era (e.g. an 80s or 10s NZ pack — HOWLER becomes that era's throwing toy, TAZO its collectible craze, and so on). The daily is identical across skins (D22) — share grids and totals stay comparable. v1 packs are **strings + palettes only** (D21); per-theme sprite overrides are a later unit (the drawn objects — rocket, pie warmer, red socks — are the expensive part).

#### 7.0 — String extraction (the reshaping — zero-visual-diff refactor) — ✅ SHIPPED (2026-08-01)

Display text no longer lives in the game objects. `SKINS` sits above the game blocks (full contract in §3); `nz90` is the only pack and the code's `themeId` is hardcoded to it, so this unit changed nothing a player sees.

- **Shipped pack shape**: `const SKINS={ nz90:{ label:'NZ 90s/00s', hub:{dailyName}, games:{ <id>:{ name, tag, ranks:[titles], ready:[lines], over|overWin/overTime, <banner keys>, shareIcon, shareLabel, (bones) ladder:[], (dairy) shapes:[{name,tag}] } } } }`. Accessors: `skin()`, `S(id,key)`, `dailyName()`, `applySkin()`, plus `DEFAULT_SKIN`/`themeId`.
- **What moved**: `name`, `tag`, ready-screen lines, over-screen titles (HOWLER's two terminals became `overWin`/`overTime`, plus its `whistle` noun), rank **titles** (game keeps `ranks:[{s}]` thresholds only), thematic banners (`GUNGED!`, `LEAK!`, `NEW DISC!`/`TEMPO UP!`, `FULL CHATTER!`, `SLAM!`, `CRYING!`/`FULL UP!`/`HI!`/`NEW KID!`/`HUNGRIER!`/`FAIR SHARE! ×2`, `FRESH`/`HEAT ESCAPED!`, `CLICK`, `KNITTED!`/`SNAPPED!`/`THREAD SNAPPED!`), the themed name lists (BONES `LADDER` rungs, DAIRY pastry names + their decorative price tags), D34's `shareIcon`/`shareLabel` (the interim `SHARE_NZ90` object is deleted), and `DAILY_NAME` → `dailyName()`.
- **What stayed inline** (audited, listed in the code banner): mechanical/universal labels (`PERFECT SAVE!`, `+N`, `UNWIND`, `LOOP ×N`, `RELEASE`, `ROTATE`, `BEST`, `STAGE`, `STREAK`, `TAP TO PLAY`…), anything fed by a number, and the **brand** (`CENTRARCADE`, `PAYMENTS BY CENTRAPAY`, the mark — rule 3: the mark is in every skin, it is not the theme).
- **Two delivery mechanisms, deliberately.** `applySkin()` *binds* the widely-read fields (`name`, `tag`, rank titles, `LADDER`, `SHAPES` names/tags) onto the live objects once, before first `init()`/render — so the menu, HUD, gauntlet and share code keep their existing reads and the diff stayed small. `S(id,key)` is a *live lookup* for copy read at draw/event time (ready lines, over titles, banners), with a fallback through the default pack so a future partial pack can never render `undefined` (the `gamePal` fallback philosophy). 7.1 switches skins by setting `themeId` and re-calling `applySkin()`.
- **Acceptance test — met.** A Chromium harness wraps `ctx.fillText`/`strokeText` and drives every screen (menu pages, all four daily-tile states, pause, every game's ready/play/over/new-best, the interstitial/end/sharecard, the receipt text) plus every thematic banner path poked directly, then dumps every string that reached the canvas: **944 strings, byte-identical before vs after, zero page errors.** Negative control: mutating three pack strings makes the diff fail, so the harness is not a no-op. Harness lives in the session scratchpad, not the repo (single-file rule 1).
- Palettes join the pack as index indirection later (a pack may remap `PALMAP` targets or supply new `PALS` rows) — still not needed.

#### 7.1 — Selector + unlock (D23)

- NZ-90s default. A **drawn** skin selector (rule 4) appears on menu page 0 once `arc_daily_streak ≥ 3`; freely switchable thereafter; choice in `arc_theme`. Before unlock, no UI hint beyond (optionally) a locked chip — keep it quiet.
- Switching is instant and touches display only. The daily share text may carry the pack's daily-name — totals/grid stay comparable regardless (D22).

#### 7.2 — First alternate era pack

- Strings + palettes for one more era (**owner to pick: 80s or 10s**, and the reference for each game — this is a naming/writing exercise, logged per game as ideas arrive). Nostalgia references stay literal per D15's spirit — owner clears each.
- Sprite overrides (per-theme draw functions) are **v2**, explicitly out of scope here.

*Skin idea log (append; don't build until 7.2):*
- *(2026-08-02, D43)* Banked alternates for the four Phase-7.5 games: RECALL = takeaway delivery jingle (Eagle Boys / Pizza Hut `0800 83 83 83`) · LOCK = Sky decoder parental lock · ROUND = school cross country · CHUTE = Video Ezy return chute. All pending D15.
- *(2026-08-02)* Observation for whichever era lands at 7.2: the jingle, the combination lock, the delivery round and the returns chute all **port cleanly across decades** — every era has all four — which makes them the safest structures if the goal is one skin architecture rather than four bespoke ones.
- *(empty slots — era candidates and per-game reference ideas go here)*

---

### Phase 7.5 — ROSTER EXPANSION (four games)

Four additions closing the roster's four biggest category gaps: **memory** (RECALL), **deduction** (LOCK), **scrolling dodge** (ROUND), **rule-switching discrimination** (CHUTE). Built after 7.0 so every display string is pack-native from commit one (D38).

Universal constraints — all four:

- Endless + 3 lives (D39). Rule 9: reuse `drawBG` / `drawReadyScreen` / `drawOverScreen` / `burst` / `pop` / rank helpers. Rule 4: every icon and symbol drawn, never a glyph.
- Seeded content **correct-by-construction with a fixed draw count per round/stage/board** (D16, §3). Player pace must never shift the stream — this is the WEAVER B1 bug class and it does not ship twice.
- §9 holds: a stray or ambiguous input is a free no-op with a soft reject, never a life.
- `par` + `ranks` are `// PROVISIONAL` placeholders until D41's retune.

All constants below are `// PROVISIONAL` design intent, not spec-final. Each game's **theme** sub-bullet is PARKED pending D15 clearance (D43), **except RECALL's dial-up main, cleared and shipped 2026-08-02**. The three remaining mains are brand-free as specified, so 7.5 may build against them — but clear each with the owner as its game lands, the way RECALL's was.

---

#### RECALL — memory sequence — ✅ SHIPPED (2026-08-02)

**As-shipped truth is the §4 entry.** The spec below is the brief it was built from; three things were decided at build and are worth carrying into LOCK/ROUND/CHUTE:
- **The re-bank hole.** "Items bank as you go" + "a miss replays the round from the top" meant failing on the last item of a long round and re-clearing paid it twice (up to 3× with 3 lives). Closed with a per-round **watermark**: a correct tap banks only past the deepest index already banked this round. Partial credit survives exactly as specced; a cleared round now always pays exactly n².
- **RECITE is per-round**, applied at round completion, not a run-level multiplier — the latter compounds against the n² curve and makes `par` meaningless.
- **`DAILY_PICK` 6 → 5 shipped with it** (D37). Dropping a receipt line strictly *frees* vertical space (5 rows bottom out 124px above the COPY/SHARE buttons vs 66px at 6), so there was no overflow risk to re-verify — only an aesthetic one. See §7.

- `id:'recall'`, `bestKey:'arc_recall_best'`. Verb: **TAP only** — no `press`, so it stays on tap-on-pointerdown (D12).
- **5 pads** on a ring, reusing CHATTER's 5-slot / 72° geometry (rule 9). 4 reads as generic Simon and carries too little information per item; 6 crowds 360px under a thumb.
- **Playback: the whole sequence replays each round**, one item appended. Playback interval **shrinks with length to a floor** (`PB_BASE ≈0.62s`, `−0.03/item`, `PB_FLOOR ≈0.26s`) — without this, watch time grows quadratically and round 12 is a ~9-second spectate.
- **Miss = −1 life, same length replayed from the top of the round.** 3 lives = over.
- **Score.** Each correct item banks the **current round length** — so a clean round of length *n* is worth *n²*, and a round failed halfway still banks partial credit. The all-or-nothing cliff is what makes memory games normalise badly against par; this removes it. Plus **RECITE ×1.5** on a round entered with every tap inside `FLUENT_GAP` (≈0.55s) of the last — rewards chunked recall over deliberate re-derivation.
- **Channel is visual and must stay fully playable with `arc_mute` on** (rule 5). Pad tones are reinforcement only; a muted player loses nothing.
- Seeding: one draw per round appended, fixed count. Replays consume nothing. Nth-draw safe.
- **Theme — main: the dial-up handshake. ✅ D15-CLEARED + SHIPPED (owner, 2026-08-02).** The five pads are five recognisable components of the 56k negotiation (the dial tone, the hiss, the two-note handshake). Each day's seeded sequence is a different negotiation — some days it connects fast, some days it's a long painful handshake. Fail copy: `CONNECTION LOST`. Brand-free as shipped — it's a sound everyone of that era knows, with no named product, which is why it cleared D15 first and most easily of the four. *(The spec's Xtra reference was dropped at build: the shipped pack names no ISP.)* *Banked alternate (7.2):* **the delivery jingle** — pads as notes building a takeaway jingle, hooks being Eagle Boys and the Pizza Hut `0800 83 83 83` number. Strong because it's a sequence people genuinely memorised as kids, and it ties into the tuck-shop/food thread. **Both are named brands — D15 clearance required.**

#### LOCK — deduction

- `id:'lock'`, `bestKey:'arc_lock_best'`. Verb: **TAP** — cycle a slot's symbol, tap a drawn SUBMIT (DAIRY rotate-button / GUNGE valve precedent).
- **Code: 3 slots × 6 symbols, repeats allowed** (216). Escalates to **4 slots from lock 3** (1296), capped at 4. Escalation is where run-length pressure lives.
- **Mastermind pegs.** Black = right symbol, right slot. White = right symbol, wrong slot. **Standard multiset counting: exact matches are removed first, then each remaining guess symbol consumes at most one remaining code symbol.** Spelled out so Code can't ship the naive double-count bug.
- **Guess budget: 5 at 3 slots, 6 at 4 slots.** Exhausted = **−1 life**, code revealed, new lock. 3 lives = over.
- **Soft clock = a per-lock BONUS METER only** (D3/D19, GUNGE and WEAVER precedent). It drains, it banks on solve, and hitting 0 **never** fails the lock or ends the run.
- **Score on solve:** `(guessesLeft × slots × 10) + clockRemaining`. A failed lock banks nothing.
- **Symbols must differ in SHAPE, not only hue** — the TAZO 6.2/6.2c ΔE lesson, plus six symbols on a dark field is where colourblind readability breaks.
- Seeding: `slots` draws per code, fixed count. Guesses consume nothing. Nth-draw safe.
- **Theme — main: big sister's diary padlock.** Cracking the code reveals one line of the diary on the solve screen — a reward beat per lock, which suits LOCK's chained structure better than a bare SOLVED banner. Three failed locks and she comes back into the room (the existing 3-lives terminal, reskinned). Diary lines are pack strings; they need writing as part of the pack and are subject to the same tone bar as the rest of the copy. *Banked alternate (7.2):* **the Sky decoder parental lock** — four digits between you and something you're not meant to be watching; the D19-style bonus meter reads as how long until someone comes down the hall. **Named brand — D15 clearance required.** (The bike-shed combination was the third option and was not selected; leaving it unlogged rather than half-parked.)

#### ROUND — lane dodge

- `id:'round'`, `bestKey:'arc_round_best'`. Verb: **coarse directional swipe L/R**, reusing TAZO's `press`/`release` + `DIR_THRESH` primitive (rule 9, D12). One lane-hop per swipe; a hop animates ~120ms and **is not cancellable mid-hop**.
- **3 lanes.** No held pointer state exists, so the B8 `pointercancel` class doesn't arise here.
- ⚠️ **Trade recorded (owner-chosen, 2026-08-01):** lane-snap makes ROUND a **discrete** game. It fills the scrolling-course/dodge gap; the roster still has **no continuous-input game**. Flagged in §7, not a defect.
- **Course: finite per stage, generated WHOLE at stage start from the seed, fixed draw count.** Spawn-as-you-scroll is explicitly **banned** here — it is exactly the WEAVER B1 divergence (§3). Clearing a stage regenerates a longer/faster one.
- **Hit = −1 life.** 3 lives = over.
- **Score: pickups are the score, distance is the clock.** Course length is fixed per stage; `+N` per pickup; stage completion banks a bonus scaled by stage; a stage cleared with zero hits banks a **CLEAN RUN** bonus. **Not** `distance × pickups` — multiplicative scoring is the GUNGE par lesson.
- **Theme — main: mowing the lawns.** Three strips up the section map onto the three lanes. Pickups are the rows still uncut; hazards are the hose, the sprinkler, and the brick left in the long grass. *Clearance note: brand-free.* *Banked alternate (7.2):* **school cross country** — three lanes of a borrowed farm paddock, dodging cow pats, an electric fence, and the kid who sprints the first hundred metres then walks. Brand-free, but era-agnostic to the point of being timeless, which makes it a weaker nostalgia slot than the lawns.

#### CHUTE — sort under a shifting rule

- `id:'chute'`, `bestKey:'arc_chute_best'`. Verb: **coarse directional swipe L / DOWN / R** on the item at the chute mouth → bins at bottom-left / bottom-centre / bottom-right. Direction maps to bin **position**, so the mapping is spatially self-evident. Reuses TAZO's swipe primitive (rule 9).
- **3 bins.**
- **The flip (D42): the sort CRITERION changes at stage-up** — colour → shape → size — announced by a **~2s telegraphed banner before it takes effect**. Bin mapping and labels never change and never lie.
- **Wrong bin = −1 life** (owner-chosen). 3 lives = over.
- **An item that falls through unsorted: OPEN — owner call (§7).** Recommendation: **no life, streak resets.** The §9 ethos is that a life is the price of a committed wrong decision, not of hesitation.
- **Score: per correct item, with a STREAK multiplier that resets on any wrong bin.** Differentiator, stated because CHUTE, SCAN and SCRAMBLE all look like "things arrive, act on them": **SCAN scores raw clear rate. SCRAMBLE scores evenness across recipients. CHUTE scores accuracy-gated throughput.**
- Seeding: **pre-generate the stage's item list at stage start** (fixed count) rather than streaming draws — keeps it Nth-draw safe instead of inheriting SCAN's accepted-divergence exemption.
- **Theme — main: the paper run (depot sort).** Three bags — the daily, the free community paper, the junk mail. The D42 criterion flip runs publication → street run → houses with a No Junk Mail sticker, telegraphed as always; bin mapping and labels never change. *Clearance note: brand-free if the publications stay generic* — naming a specific masthead pulls it into D15. *Banked alternate (7.2):* **the Video Ezy return chute** — new releases, weeklies, overnights, with the criterion flipping to genre, and an unrewound tape costing you. **Named brand — D15 clearance required.**

---

#### 7.5 build-time checklist (rewrite these ONLY in the commit that ships each game)

1. **Header roster line** — ✅ 12 at RECALL; → 13/14/15 as each lands.
2. **§2 keys table** — ✅ `arc_recall_best`; add `arc_lock_best`, `arc_round_best`, `arc_chute_best` as each ships. No legacy-key inheritance for any of them (D1 precedent).
3. **§3 determinism model** — ✅ RECALL added (as its own *prefix-deterministic* bullet — one draw per round, replays free; stronger than fixed-count and documented as such). Add each remaining game to the **fixed-count-per-board/round** bullet. None of the four may join the *inherently divergent* list.
4. **§3 pointer-stream note** — ✅ RECALL logged tap-only. ROUND and CHUTE become `press`+`release` consumers (no `drag`), TAZO-style. LOCK stays tap-only.
5. **§4** — ✅ RECALL. One as-shipped entry per game, written from the code, not from this brief.
6. **`GAMES` array** — ⚠ **pacing is moot, and the brief was wrong about this.** Since D33 the daily is a day-seeded *shuffle* of a subset, so this array no longer paces a run — it paces the **menu**. RECALL was therefore **appended**, not slotted mid-array: at 11 games the last page held 3 cards and a gap, so appending filled the gap and moved no existing card off its page or position. Do the same for LOCK/ROUND/CHUTE unless there's a menu-layout reason not to.
7. **`PALMAP`** — ✅ `recall:10` on a new `PALS[10]` (dial-up magenta, ~310°, the largest unused hue gap — reuse would have put a 12th card in an already-crowded band next to WEAVER's red). Appending `PALS` is safe by construction (§3). Three entries still to add; a missing one must still degrade to `PALS[2]`, never throw.
8. **`DAILY_PICK` 6 → 5 (D37)** — ✅ SHIPPED with RECALL. The layout re-verification the brief asked for was done and the concern was **inverted**: fewer rows strictly frees space (124px of clearance at 5 rows vs 66px at 6), so overflow was never the risk. Screenshot-verified at 5 lines including the longest names + max prices. The row heights (`58`/`48`) were left at their owner-tuned values; the leftover space is an open aesthetic call (§7).
9. **§6.5 par retune** — remains blocked until all 15 have telemetry (D41).

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
| D20 | **The base-game era is over; skins are cosmetic content packs over frozen mechanics.** Verbs, pars, rank thresholds, seeds, scoring, ids, bestKeys, and the daily sequence are theme-invariant (rule 12). Skins never swap games in or out. *(The "base-game era is over / no new games" clause is reversed by **D36** — four additions, Phase 7.5. Everything else in this row stands unchanged, including for the new games.)* |
| D21 | **Skin v1 = strings + palettes only.** Per-theme sprite/draw overrides are a later unit — the drawn objects are the expensive half of theming. |
| D22 | **The daily is identical across skins.** Same seed, same boards, same pars, same normalisation — share grids and totals comparable between players on different skins. Skin may relabel only. |
| D23 | **NZ-90s (`nz90`) is the default skin.** A drawn selector appears once `arc_daily_streak ≥ 3`; freely switchable; stored in `arc_theme`. No first-run choice screen. |
| D24 | **Phase 6 is telemetry-first.** Unit 6.0 ships before magnitude retunes; direction-known changes (HOWLER easier, TAZO harder) may proceed on owner observation but stay `// PROVISIONAL` until measured. |
| D25 | **Phases 8 (mutators) and 9 (creature) are placeholder idea logs.** Append ideas; do not build until the owner re-opens them. |
| D26 | **Pruning 1–2 games over time remains on the table**, decided from `arc_stats` telemetry (play counts, run lengths, daily-length pressure) — not from vibes. CHAIN's rule-10 pattern applies to any future prune. |
| D27 | **The sanctioned spawn LEVERS (`SPAWN_K`/`TIER_BANDS`/`SPAWN_CAP`) can't bound a skilled TAZO run** (owner call, 2026-07-20). A faithful Unit-6.2 sim proved they barely move the median and can't touch the ~12 min p90 tail (a skilled player out-clears spawn pressure). Do not re-attempt tuning those three levers for difficulty. *(Superseded on the difficulty **outcome** by D28: the fix was changing the spawn TIER, not the levers — this decision still stands as "don't bother tuning the three levers".)* |
| D28 | **TAZO difficulty = ALL spawns are tier-0** (owner reopened + delegated, 2026-07-20). The owner's own idea, sim-confirmed where D27's levers failed: a tier-0 tile can only merge with another tier-0, so it can't be cleared into your accumulated mid/high tiles → the board clogs → even careful play is bounded (median 3.3→~1.3 min, p90 10+→~2.2 min). Shipped 6.2b: `TIER_BANDS` removed, `SPAWN_TIER=0`, `SPAWN_K 6→9`, `par 4500→2000` + ranks rescaled — all `// PROVISIONAL`, final numbers at 6.5. Tune length with `SPAWN_K`, never by re-adding higher spawn tiers (every mix tested was harder). A rare optimal-survival merge loop is accepted vs the old common 10-min runs. |
| D29 | **HOWLER redesigned to a side-on 2D distance course** (owner-directed, Phase 6.1). Swipe = angle + power → a real parabola; range along x, loft along y; camera pans right. Retires the pseudo-3D depth-swipe and voids the old 6.1 difficulty-easing levers (the model they tuned is gone, not eased). The swipe verb (press/drag/release, D12) is kept. |
| D30 | **HOWLER is a fixed, timed course, no lives — the 2nd owner-approved D14 exception** (after TAZO). Four targets in order (30→60→90→ULTIMATE); a miss only costs time. **Two terminals** (owner-extended at build, 2026-07-21): the ULTIMATE clear (a *win*), or the **time budget running out (~75s)** — a graceful budget-exhaust end, not a fail and not a score cap (the bleeding budget *is* the score; it ends banking whatever centre bonus was earned). Budget-exhaust guarantees the daily always advances even for a player who never clears ULTIMATE. Both fire `onOver` once and par-normalise normally; an in-flight throw at the buzzer resolves first (can still win). |
| D31 | **HOWLER stays higher-is-better** — score = a bleeding time budget (fast = high) + a small per-clear centre bonus, × a whistle multiplier. **No raw stopwatch** — ranks, `par`, gauntlet, daily emoji all keep working with zero special-casing. |
| D32 | **Whistle = an absolute swipe-power band, achievable at ALL distances** (angle chooses range: steep lob for near markers, flat for far — owner spec), not an ultimate-only lock. It's higher-variance (power amplifies angle error) → risk/reward. Each whistle clear stacks `WHISTLE_STEP` toward a ×2 all-whistle WHISTLER run. Wind is **removed** from HOWLER; the daily course is deterministic (seed content-free). |
| D33 | **The daily draws a day-seeded subset, not the whole roster (owner-directed, 2026-07-22).** *(Shipped as 6 of 11; the count is now `DAILY_PICK=5` per D37 — mechanism unchanged.)* Resolves the §7 daily-length flag via the sanctioned "seeded subset" option (not a prune, not accept). Selection is a **deterministic bag**: shuffle all 11 from a day-seeded PRNG (`mulberry32(hashStr(dayKey))`), deal 6, **constraint: at most one of {TAZO, DAIRY} per day** (the two run-length watch items) so the run holds ~5–8 min; refill+reshuffle the bag when it drains so every game surfaces on a regular cadence. Same 6, same order, for everyone that day (rule 12 / D22 — selection is part of "the daily sequence," theme-invariant; seed from `dayKey` only, never from `arc_theme`). Contained change: the controller already sequences `this.order`, so this is "set `this.order` to the 6-pick" — `idx`, interstitial `/6`, finish check, and the par-normalised total all follow. Count is a constant `DAILY_PICK=6` for later retune. Orthogonal to pruning (D26): subset-of-N and prune-the-pool compose. A game's `idx`-derived seed now varies day to day → more board variety, no determinism cost. |
| D34 | **The daily share is a tuck-shop receipt with prices (owner-directed, 2026-07-22).** Refines D8's emoji grid (still card + text, no backend — not a reopen). Each of the day's 6 games is a receipt line: `<shareIcon> <shareLabel> <price> <tile>`, where **price = contrib ÷ 100** formatted `$X.XX` and the tile is the existing contrib tier. Prices sum to the run total like a real till; price bands and tile bands align automatically (both from the same contrib, so a line can never look self-contradictory). `shareLabel`/`shareIcon` are **skinnable display strings** (Phase 7.0 pack fields; inline under `nz90` until extraction) — a future era pack remaps the snacks; tiles/prices/total stay comparable (D22). Lineup + labels are shared for the day; prices + tiles are the personal result (the Wordle contract). **Rule 4:** emoji live only in the **copied/shared text**; the on-canvas receipt draws its own tiles + text (like the current end-screen chips). **Impl note:** add game `id` (or the resolved label) to each stored `arc_daily_state.result` row so the receipt rebuilds on a locked/returning day (today's stored row keeps `name`/`contrib`/`rank`/`pid`, no `id`). Nostalgia labels owner-cleared per D15 (proposed set below). |
| D35 | SCRAMBLE S1 arrivals use a walk-in queue over the existing 5 fixed slots (owner-directed) — not a 6th slot. A kid who reaches sustained high contentment walks off happy, freeing their slot; a new kid walks in after a short gap. No change to `xs`, tap targets, or THROW_X/THROW_Y. |
| D36 | **The roster is reopened for exactly four games** — RECALL, LOCK, ROUND, CHUTE. This reverses **only** D20's "base-game era is over / no new games" clause. D20's substantive invariant — *skins are cosmetic content packs over frozen mechanics* — and rule 12 stand **unchanged**: the four new games are frozen the same way once shipped. Target is **15 gross**; D26 pruning stays live and may land the net lower. |
| D37 | **`DAILY_PICK` 6 → 5**, shipped alongside the first new game. A 15-game roster at 6/day overruns the 5–10 min target; 5 holds it and gives a 5-line receipt. Per-game appearance rate moves ~55% → ~33% — accepted: variety over familiarity. |
| D38 | **Sequencing: 7.0 extraction → 7.5 roster expansion → 7.2 first alternate era pack.** New games are built **pack-native** (display strings live in the pack from the first commit, never inline). No second era pack may ship against an 11-game roster and be retro-fitted afterwards. |
| D39 | **D14 stands unamended — no new exceptions.** All four are endless + 3 lives: RECALL lengthens forever; LOCK chains locks (out of guesses = 1 life, new lock); ROUND chains finite courses as stages (WEAVER precedent); CHUTE is an endless sorted stream. TAZO remains the sole board-lock exception. |
| D40 | **The ≤1-of-{TAZO, DAIRY} daily constraint is unchanged.** LOCK is a §7 run-length **watch item**, not a constraint member — its fixed guess budget hard-bounds each lock and the life count bounds the run. Revisit only from telemetry. |
| D41 | **The 6.5 roster-wide par retune is deferred until all 15 games have telemetry, and then runs once.** Par is the exchange rate between games in the daily total; retuning 11 and then re-retuning 15 wastes the data and produces two incomparable eras of totals. |
| D42 | **CHUTE's flip is the sort CRITERION, telegraphed — never the bin mapping and never the labels.** CHUTE is the roster's rule-switching game; a game that lies about which bin is which is a reflex game with extra steps, and the roster already has two. Binding against drift at build. |
| D43 | **Each Phase-7.5 game ships with one main `nz90` theme and one banked alternate** (owner-directed, 2026-08-02). Main: RECALL = dial-up modem handshake · LOCK = big sister's diary padlock · ROUND = mowing the lawns · CHUTE = the paper run (depot sort). Banked alternates for the 7.2 pack: RECALL = delivery jingle · LOCK = Sky decoder parental lock · ROUND = school cross country · CHUTE = Video Ezy return chute. Mains and alternates were PARKED pending D15 clearance. **✅ RECALL's dial-up main is CLEARED (owner, 2026-08-02) — shipped, including its `📞`/`dial-up` receipt line.** The other three mains and **all four alternates remain parked** — none may ship until the owner clears those specific references. |
| D44 | **The paper run belongs to CHUTE, not ROUND** (owner-directed, 2026-08-02). Both games could carry it and D33's shuffle can't guarantee they land the same day, so splitting one artefact across two games that usually appear apart was rejected. ROUND takes mowing the lawns instead. |

## 7. Open decisions (owner to resolve — flag, don't guess)

- ~~**Daily length (11 games).**~~ — RESOLVED 2026-07-22 (D33): the daily now draws **6 of 11** via a day-seeded bag (at most one of TAZO/DAIRY per day). `DAILY_PICK=6` is tunable at 6.5 from telemetry.
- **Final pars for all eleven** (6.5) — see the skew table there. All `// PROVISIONAL`.
- ~~**HOWLER daily-termination guarantee**~~ — RESOLVED 2026-07-21 (owner said build it): shipped the **budget-exhaustion terminal** — when the time budget hits 0 (~75s) without an ULTIMATE clear, the run ends gracefully (score = banked centre bonus × whistle mult) and fires `onOver`, so the daily always advances. D30 updated to two terminals. Verified headless (budget-exhaust ends + fires once + floored score; in-flight-at-buzzer throw still resolves/wins).
- **HOWLER whistle band** — on-device `CALIBRATE` pass still owed; `VREF`/band are synthetic. Now a **power band hittable at any distance** (angle-chosen range); after the owner retune the co-tune is `M2PX 7.5` + `SPEED_REF 865` + band `2.45–2.75` + `ANG 0.25–1.48` (whistle@45° ≈ 120 M ≥ ULTIMATE; 30M whistle needs a ~79–85° lob) — reconfirm on device. *The old "6.1 easing magnitudes" item is void — the depth-swipe model was replaced, not eased.*
- **HOWLER scoring balance** — `TIME_BUDGET`/`TIME_RATE`/`CENTER_MAX`/`WHISTLE_STEP` split (speed vs precision vs whistle). Current draft makes speed dominant, whistling all four = ×2. Alternative under consideration: a hard "all-four-whistled" ×2 *gate* instead of the per-whistle `WHISTLE_STEP` stack. Decide from play.
- **HOWLER whistle-zone marker** on the power meter — show it (teachable) or hide it (purer)? Owner call.
- **HOWLER par + ranks** — re-derive at 6.5 from the redesigned model's telemetry (old `par:500` is void — different scale). Provisional new-scale guesses: `par:1800`; ranks `BACKYARD ARM 0 / 30 METRE CLUB 600 / 60 METRE CLUB 1100 / 90 METRE CLUB 1600 / ULTIMATE DISTANCE 2200 / WHISTLER 3200` — titles kept, thresholds `// PROVISIONAL`.
- **TAZO tier hexes** — tiers recoloured to slate / orange / cyan / gold / white (6.2 + 6.2c, `// PROVISIONAL`, ΔE + screenshot-verified on the dark board); owner to confirm the final hexes on-device.
- ~~**TAZO difficulty**~~ — RESOLVED 2026-07-20 (D28): owner reopened and shipped **all-tier-0 spawns** (6.2b). `par 2000` + ranks are `// PROVISIONAL` pending the 6.5 telemetry retune.
- **GUNGE contestant layout** — RESOLVED (6.3): shipped the tank-below-grid option (valve nudged to y514, `GY` kept). Owner to approve the look on-device (screenshots provided in the build session).
- **CHATTER stage-0 colour** — its in-play stage 0 uses hub purple (PALS[2]) and mismatches its green menu card. Pre-existing (the old modulo did the same). If unwanted: drop `2` from `CHATTER_PALS` — deliberate one-line recolour. Decide when CHATTER is next touched.
- **First alternate skin era** — 80s or 10s, and the per-game references (Phase 7.2 idea log).
- **Which games get pruned**, if any, and the target roster size (D26; after 6.0/6.5 data).
- **SCRAMBLE/CHATTER attention-mechanic overlap** — tolerated; revisit at prune time.
- **Share comparability (D33/D34).** The receipt is 6 lines, and the lineup changes daily, so **cross-day** shares aren't tile-comparable — this is now **by design** (each day is its own puzzle). **Same-day** shares stay perfectly comparable (everyone gets the same 6 games + labels). Noted in the share code.
- **DAIRY strike-relief (DD4) — PARKED, owner call (reopens D18).** The strike-wipes-the-whole-shelf-and-refreshes rule is *why* DAIRY is near-unlosable — it's the highest-leverage difficulty lever. Options if reopened: wipe only the bottom row / make the wipe cost banked points / 2 strikes instead of 3. **Not built** — D18 fixes "strike wipes shelf + fresh tray, 3 strikes = over," so this needs an explicit owner reopen before anyone touches it.
- **`shareLabel`/`shareIcon` per game (D34).** Proposed `nz90` set below; owner clears each snack reference (D15) before it ships.

*Phase 7.5 (roster expansion):*

- **ROUND continuous-control gap.** Lane-snap was the owner's pick; the roster consequently has no continuous-input game at all. Revisit at prune time or if a 16th slot ever opens.
- **CHUTE unsorted-item rule.** Life, points, or streak-only? Recommendation on file: streak-only. Owner call before build.
- **LOCK run length.** Deduction with only a soft clock is the new TAZO-shaped tail risk (D40). Measure at first telemetry; escalate to the ≤1-per-day constraint set only if data demands it.
- **RECALL playback ceiling.** Still open post-ship. At `PB_FLOOR 0.26` a round-14 playback is ~3.6s and round 20 ~5.2s — bounded, but judge it on device. If it drags, the fallback is replay-from-item-*k* rather than from item 1 — a real mechanic change, so owner-only.
- **RECALL par + ranks.** `par:400` ≈ reaching round 10 (a cleared round pays n², so cumulative ≈ R³/3). Ranks `ENGAGED TONE 0 / DIAL TONE 60 / HANDSHAKE 180 / NEGOTIATING 400 / CONNECTED 750 / 56K FLAT OUT 1200`. All `// PROVISIONAL` — folded into D41's single all-15 retune, not tuned alone.
- **Daily receipt whitespace at 5 lines.** Dropping to `DAILY_PICK 5` leaves ~124px between the last row and the COPY/SHARE buttons (was ~66px at 6). Nothing overlaps or clips, but the end card reads bottom-heavy. Row heights were left at the values the owner tuned for legibility (`58` end/sharecard, `48` interstitial); raising them to ~66 would fill the gap. Aesthetic call, not a bug.
- **D15 clearance for the Phase-7.5 themes (D43).** ✅ **RECALL's dial-up main is cleared and shipped** (owner, 2026-08-02) — the first of the four. Still owed: the three remaining **mains** (diary, lawns, paper run — brand-free as specified, so each can be built and cleared as its game lands, RECALL's precedent), plus **all four alternates**, which block 7.2 specifically: Eagle Boys and the Pizza Hut `0800 83 83 83` number (RECALL alternate), Sky (LOCK alternate), Video Ezy (CHUTE alternate), school cross country (ROUND alternate — brand-free but unconfirmed), and any named masthead if CHUTE's paper run stops being generic.
- **Share icons and share labels for the four new games** (D34 receipt lines, D15 clearance). ✅ RECALL shipped as `📞` / `dial-up`, cleared with its theme. LOCK / ROUND / CHUTE still owed — working names only until then.
- **RECALL muted-play check against the dial-up theme.** Verified headless at build — with audio off, the lit-pad channel alone reproduces the sequence byte-for-byte, and pads carry a drawn waveform arc so a lit pad doesn't rely on colour. **On-device confirmation still owed** (the headless test asserts state, not perceptibility at arm's length).
- **LOCK diary lines** — how many, who writes them, and whether they're seeded per lock or per day. Cosmetic, but it's the reward beat, so it needs more than filler.
- **Menu paging at 15 cards.** Confirmed harmless at 12 (still 4 pages — RECALL filled the last page's gap). The page count next grows at **13** (LOCK), reaching 5 pages at 15. Confirm the dots/arrows read cleanly at 5 when LOCK lands.
- **Byte budget.** Four games in a single file with no build step — check total size against iOS Safari load on-device before the last one lands.

**Receipt share — spec detail (for the D34 build unit).**

Line format (shared/copied text):
```
🧾 TUCK SHOP RUN #<N>
· · · · · · · · · · · ·
<icon> <label>   $<price>  <tile>
... (6 lines) ...
· · · · · · · · · · · ·
   TOTAL        $<total>  🔥<streak>
```
- `price = (contrib / 100).toFixed(2)`; `total = (this.total / 100).toFixed(2)` (== Σ line prices).
- `tile` = existing tier map: `<100 ⬛ · <200 🟨 · <350 🟧 · <500 🟩 · ≥500 🟪` (price bands align: ⬛ <$1.00 … 🟪 ≥$5.00).
- Dot-leaders keep columns aligned when pasted into a chat.
- On-canvas receipt (end/share screen): draw tiles as filled rects + text (rule 4 — no emoji glyphs on canvas). Emoji version is the copyable `shareText()` only.

Proposed `nz90` mapping (owner clears each per D15; emoji are the text-share icons):

| game | shareIcon | shareLabel | alt |
| --- | --- | --- | --- |
| STACK | 🥪 | club sammie | |
| SCAN | 🛒 | checkout | |
| HOWLER | 🏈 | mega howler | 📼 |
| SWINGBALL | 🎾 | swingball | |
| GUNGE | 🫗 | gunge tank | 🟢 |
| CHATTER | 💬 | chatter ring | 📞 |
| TAZO | 🎟 | tazo pack | 💿 |
| SCRAMBLE | 🍬 | lolly scramble | |
| DAIRY | 🥧 | mince pie | |
| KNUCKLEBONES | 🎲 | knucklebones | |
| WEAVER | 🧶 | jersey knit | |
| **RECALL** | **📞** | **dial-up** | — ✅ **D15-cleared + shipped** (2026-08-02) |

⚠ **Icon collision to avoid:** CHATTER's listed alt is `📞`, which is now RECALL's live icon. If CHATTER is ever switched to that alt, pick a different one (`☎️`, or keep `💬`) — two identical icons on the same day's receipt would make the lines unreadable.

These live in the Phase 7.0 pack as `SKINS.nz90.games.<id>.shareLabel` / `shareIcon` (the extraction shipped 2026-08-01; the interim `SHARE_NZ90` object is gone). The eleven rows above are still **owner-uncleared per D15**; RECALL's is the only cleared line so far.

## 8. Known bugs & tech debt (ledger — fix in the unit named)

- **B1 — WEAVER daily determinism bug. ✅ FIXED (6.0).** `spawnFlare` consumed seeded draws mid-board; flare count depends on player pace, so the shared stream drifted and players got **different boards from board 2 onward in the same daily**. Fixed by splitting the seeded stream: per-stage board stream (`brnd`, fixed draw count) + separate persistent hazard stream (`hrnd`) for flares. Verified headless — boards byte-identical across pace, Nth flare route shared. GUNGE/DAIRY/BONES/HOWLER were always safe (fixed draw counts per board/piece/throw); TAZO/SCAN divergence is inherent and acceptable.
- **B2 — Dead code. ✅ SWEPT (6.0):** removed `dailyPlayedToday()` (tile reads `dailyCache`), `HOWLER.lastOutcome` (written, never read), `CHATTER.nextStageAt` (init-only relic), `CHATTER.perfects` (counted, never shown), and `GAUNTLET.tap`'s `state==='playing'` branch (play input routes through `gameDown`). `CHATTER.restrikes`/`drops` kept — restrikes shows on the over-screen; drops was out of B2's named scope.
- **B3 — Interstitial header. ✅ FIXED (6.0)** — now renders `DAILY_NAME` instead of the stale `GAUNTLET`.
- **B4 — `GAME N` source labels. ✅ RESOLVED (6.0).** Game blocks reordered to match `GAMES`/daily order; the `GAME N` numbers were dropped from the banners (names kept) and the stale "labels track FILE order" caveats corrected. Source order and daily order now agree.
- **B5 — `frame()` try/catch — still deferred, deliberately.** One thrown frame kills the rAF loop (the GUNGE-launch freeze); the single-`PALMAP`-with-fallback fix removed the known trigger. A catch-and-drop-frame wrapper could mask real bugs — revisit only if another freeze class appears. Record kept so future sessions don't re-litigate blind.
- **B6 — Press-game ready/over latency (accepted, documented).** For games declaring `press`, ready-start and over-restart fire on *release*, not pointerdown — a one-frame-feel delay vs tap games. Cost of the D12 contract; do not special-case.
- **B7 — `refreshMenuBests` inlines STACK's legacy key.** Cleaner: an optional `legacyKey` field on the game object read generically. Cosmetic; fold into any 6.0 pass touching that function.
- **B9 — daily tile copy. ✅ FIXED (7.5/RECALL).** `drawDailyTile()` read the stale `EVERY GAME · ONE SCORE` (false since D33). Now renders `DAILY_PICK+' GAMES · ONE SCORE'` — **fed by the constant**, so it tracks any future `DAILY_PICK` change instead of going stale a third time, and number-fed copy stays inline per the 7.0 rules rather than becoming a pack string. Fixed here rather than at 7.0 because 7.0's acceptance test was byte-identical rendered text; shipping alongside the 6→5 change was the first commit that could carry it. Owner may still prefer different wording.
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
