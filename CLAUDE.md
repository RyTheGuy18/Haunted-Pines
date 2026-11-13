# Level 0 — The Backrooms (repo: RyTheGuy18/Haunted-Pines)

Notes for future Claude sessions. Read this before changing anything.

First-person co-op Backrooms horror game for 1–4 players, played in the browser (desktop + phones, installable as a PWA). Survivors escape Level 0, then Level 1, then Level 37 (the Poolrooms), while one player (or the computer) is **the Occupant** hunting them. The repo is still named Haunted-Pines, from an older scrapped concept (a foggy-woods campground); the game itself is "Level 0".

- Live: https://rytheguy18.github.io/Haunted-Pines/ (GitHub Pages, `main` branch, repo root)
- Owner: Ryan (GitHub `rytheguy18` / `RyTheGuy18`). Plays it with friends and younger brothers, mostly on phones.
- Current version: **v0.22.0** (shown bottom-right of the menu, `#ver`)

## Files

| Path | What |
|---|---|
| `index.html` | **The whole game** (~2900 lines): CSS, HTML screens, one `<script type="module">`. No build step. |
| `three.module.min.js` | three.js **r160**, mapped via importmap as `three` |
| `jsm/loaders/GLTFLoader.js`, `jsm/utils/{BufferGeometryUtils,SkeletonUtils}.js`, `jsm/objects/Reflector.js` | three r160 example modules (must match r160). Reflector = Level 37 water |
| `peerjs.min.js` | PeerJS 1.5.x (falls back to unpkg) — data + voice |
| `models/snatcher.glb` | Snatcher skin: Mixamo-rigged, one idle clip, texture shrunk to 512 px |
| `models/crawler.glb` | Crawler skin: "Goose" by nodoxi (Sketchfab, **CC BY 4.0** — credit is in the menu footer, keep it), static mesh, rigged in code |
| `sfx/*.mp3` | crawler-steps (spider walking loop), heavy-steps (loop), spider-attack-1/2/4 (kill sounds, boosted +14 dB with a limiter in v0.20), monster-roar (computer catch), monster-growl (one-shots while the computer chases), monster-groans (20 s loop while the computer chases). Mono. Pixabay-style sources Ryan uploaded |
| `sw.js` | Service worker. **Bump `const V='level0-vX-Y'` every release** or phones keep the old version. CORE list must include every new asset |
| `manifest.webmanifest`, icons | PWA |
| `README.md` | Short description + one line per version (see Publishing) |

`index.html` script sections (search for `/* ===`): utils, constants, renderer, textures, lights, level, collision/sight, the Occupant (built-in model), players (survivor avatar), Occupant skins (+ Snatcher/Crawler loaders), shared state (+ host logic, networking, voice, settings), audio (`sfx` object), input, menu, per-frame.

## Gameplay as it stands

**Level 0 (the office)** — 16×16 grid of 4 m cells (`N=16, CS=4`), wall height 2.8 m. Three zones: office (yellow wallpaper/carpet), parking corner (concrete), "the pitfalls" (pits with ~1.2 m walkways; falling = death). Maze from DFS + random openings (`rng()<0.43` = how open), plus big halls, tight corridor blocks, a red room and a **clean office** (desk, monitor, plant, ticking clock).
- Goal: find **3 fuses** → put them in the **breaker** → exit door opens → reach it.
- Walls **shift** where nobody is looking (every 20–35 s, never visible ones). **Level 0 only** — Ryan wants no moving walls on Level 1 or 37 (hostTick only shifts on `G.lv===0`; the Occupant's SHIFT ability says "The walls here don't move" on later levels).
- Items: batteries (flashlight), cans (throw to make noise), lockers (hide; hold breath bar; **the Occupant can't open lockers**). **Almond water, sanity and the rolling office chair were removed in v0.20 (Ryan's call — don't add back).** `G.aw`/`level.aw` still exist as empty arrays.
- Phones ring and give hints (sometimes "found you"). Lying EXIT signs, scrawls, doors to nowhere, wet footprints into walls, an off-the-hook phone (dial tone).
- When **every living survivor** has escaped, after 3.5 s the whole room goes to Level 1. **Everyone caught on Level 0 comes back** for Level 1 (`G.dead` is cleared).

**Level 1 (parking garage)** — `LK = 1`, `zoneOf()` returns 1 everywhere. Concrete, pillar grid, 8 parked cars (solid box colliders in `L.slabs`, block sight). All lights off. **3 generators**: stand next to one and press START GENERATOR for 3 s (`GEN_T`, loud: noise level 2). Each one lights only the lamp right above it; all 3 → all lights on + exit opens. When everyone left alive escapes, the room drops into Level 37 (same 3.5 s advance as 0→1; `advanceLevel()` goes to `G.lv+1`, max 2).

**Level 37 (the Poolrooms)** — rebuilt in v0.21 as a **free-form level with no grid** (Ryan: the first grid version felt too similar). `LK = 2`, `G.lv = 2`. Everything is in the `/* ===== Level 37 ... */` module (search `buildLevel37`):
- **Layout** (`gen37`, seeded): **224 × 224 m** (`S37`, grown in v0.22 because Ryan finished v0.21 before the first night; spawn→stairs ≈ 330 m of walking, ~40 rooms). Rooms on a jittered 7×7 grid of slots (centre = atrium, some skipped), corridors often bend at right angles: flooded **halls** (rounded rects, rows of tiled pillars, sometimes a railed deep lane-pool), **round rooms** (often a railed deep pool), dry tiled **locker rooms** (floor 0.3 m, lockers along both long walls, a bench), the **atrium** in the middle (r 17, 10 m ceiling, big oculus, railed deep pool r 7.5, a **balcony** arc at 3.2 m with a **ramp** up and a **slide** down — usable but never needed), the **stairwell** (farthest room from spawn: round, dry, a 300° **spiral staircase** around a column up to 7.2 m under open sky = **the exit**), and the survivors' **spawn** locker room in a corner. Corridors (capsules, ~45% are tiled tunnels with arches) = spanning tree over the flooded rooms + loops; **dry rooms only hang off it as dead ends**, so the water is one connected network (checked: 12/12 seeds). Balcony is placed away from corridor entrances; corridors also carve through it/dry rooms if needed.
- **Grids** (0.5 m, `G37 = 449` per side): `M37.D` distance to the nearest wall (from shape SDFs; >0 = open), `M37.H` floor height, `M37.K` ground kind (`K37`: SOLID, WATER, DRY, DEEP, BALC, RAMP, SLIDE, SPIRAL). Helpers `d37/h37/k37`. `floorY()` = `h37` on LK 2.
- **Collision** `collide37(x,z,r,ox,oz,eel,lunge)` (the normal `collide` forwards to it on LK 2 — pass the old position!): pushes out of walls along the SDF gradient, then **each deep pool pushes you out analytically** (`deepsNear37` + `sdShape(...) - 0.25`, so the railing is exact and can't be slipped through — v0.22 fix), circles/slabs as usual, then a **height rule** (no steps steeper than ~1.2 slope: balcony edges, deep pools, the spiral's sides block you; ramps/curbs/stairs are walkable), no walking up the SLIDE (and standing on it slides you down), and the **eel may only be on WATER** unless lunging. `losClear` → `los37` (walls only; railings don't block sight).
- **Visuals**: floor = one height-field mesh with vertex colours (deep pools mint); walls = **marching-squares contour of D**, extruded to the local ceiling (4.6, atrium 10, stairwell shaft 12.5) — rounded everywhere, no yellow; tunnel arches; ceiling ShapeGeometry with ~32 sunroofs + atrium drum/ring + stairwell shaft; sky lids, sun shafts and shader sun patches (`TU.uHoles` now 40 entries, `w = square + 2*round(ceilingHeight*10)`); metal **railings** (instanced posts + one merged tube mesh) around every deep pool, the balcony, ramp/slide and the spiral, built from `perim37(shape, offset, step)` so they're perfect circles / rounded rects; each deep pool is an exact basin (tiled walls, bottom, 1.6 m coping ring) cut out of the 1 m floor height-field; ~90 sunroofs merged into 3 meshes, only the nearest 40 uploaded to `TU.uHoles` (`upload37Holes`, every 0.5 s); one `Reflector` water plane over the whole level at 0.18 (dry floors sit above it).
- **Day/night**: `G.tod` (host clock), **3 min day / 3 min night, repeating** (`DAY_LEN=180`, `CYC=360`, 12 s dusk/dawn, `dayK()`, `isNight()`). Drives `TU.uDay` (sun patches → faint moonlight, caustics, sky lids get stars, shafts fade), sun light, hemisphere ambient, fog (`ZFOG/ZDENS/ZSKY/ZGND[3]` are rewritten each frame in `update37`), underwater pool lamps (the `PL` point lights go to the nearest `level.lamps` at night), `lightAtCell` (dark at night → crouch-hiding works), and messages at dusk/dawn.
- **The Occupant here is always the eel** (`EEL = 5`, `buildEel()`, a placeholder until Ryan sends a model; `useSkin(EEL)` is forced on LK 2, not in the skin picker). 16 swaying segments + fin, sits mostly under the shallow water; you see it by its **foam wake** (`wakeMesh`) and splashing swishes. `headH 0.6` (a player eel sees from just above the water). Jumpscare uses its head.
  - **Day**: it can't kill (host kill check needs `isNight()` and `!G.asub`). Computer eel (`aiTick37`): hidden (`G.asub=1`) most of the day, surfaces every ~40–80 s for 5–10 s at the edge of a survivor's vision 13–30 m away, swims past, and vanishes if anyone gets within 11 m or looks at it for >1.3 s (≈10% visible in tests). Player eel: slower (×0.72); if a survivor within 10 m looks right at it, it's forced under for 3 s (`G.asub`, hidden from everyone); staying within 13 m of a survivor who isn't looking fills **stalk** `G.stk` (8%/s) → at 100% night comes early. Abilities on LK 2: **SPLASH** (the lure, with splashes) and **LUNGE** (dash, night only); blackout/disguise/shift are off.
  - **Night**: computer eel hunts with the difficulty settings (hearing ×1.2 extra), pathfinds through water only (`nav37` = BFS on a 1 m water grid + string-pulling), lunges out of the water at close targets, then must slither back (`nearestWater`). Dry rooms/balconies/stairs are therefore safe-ish. Groans/growls while chasing as before.
- **Exit**: climb the spiral; escape when within 1.6 m of `level.exitPos` (top step) **and** `floorY > 6.2`. `G.open = 1` from the start. Phones point to "the stairs" (`level.stairBase`); HARD/IMPOSSIBLE computers use that as their objective.
- **Sounds on LK 2** (v0.22, Ryan: no leftover Backrooms sounds): distant events are pool sounds (`sfx.poolFar`: splash, drain gurgle, pipe knock, lifeguard whistle, drip echo); no whispers, ceiling creaks/dust or elevator dings; phones are ~10 **PA speakers on poles** (`sfx.chime` instead of ringing, "The intercom crackles…" hints); thrown cans `plop` in water. 22 cans, 16 batteries.
- Echo: a convolver reverb bus (`sfx.verbG`) fades in on LK 2. Wet-sounding steps, lapping-water bed.
- The old v0.20 grid poolrooms (drain valves, basin, `buildPool`) are gone; the generator/valve code paths for LK 2 are dead but harmless.
- **Ideas still open:** Ryan will send a real eel model; daytime player-eel play is "figure out later" (stalk + splash for now); rising tide was postponed; possible extra tasks besides "find the stairs".

**Survivors**: walk 2.6, sprint 5.0 (5.6 with adrenaline when the Occupant is within 10 m), crouch 1.35 (silent). ~7 s of sprint. Flashlight (battery). Hold the light on the Occupant's face to **blind** it (stun 3.6 s; shorter on HARD/IMPOSSIBLE computers). No sanity system anymore. Crouching in the dark with the light off hides you from the Occupant beyond 4 m.

**The Occupant** (player or computer `G.mo==='AI'`):
- Walks 4.0 (`MON_SPEED`), **sprint** ×1.15 for ~1.8 s with ~4 s recharge. Never faster than a sprinting survivor.
- Abilities (keys 1–5, buttons on phones): **Blackout** (30 s, cd 45), **Shift walls** (cd 22), **Lure** (fake footsteps run up *behind the nearest survivor*, cd 20), **Disguise** (looks like a random survivor from ≥5 m for 12 s; tells: no flashlight, lights flicker; cd 40), **Dash** (~3 m lunge, arms out, cd 28).
- **Vents**: 3 linked pairs of floor grates; 2 s crawl, loud clang on exit, cd 15.
- Hears survivors: walking 11 m, running 32 m, loud 60 m, talking (voice) counts as noise.
- Hears a loud clunk when a fuse is placed (survivors don't hear that one).
- View has a red/sepia tint; darker on Level 0, can see the garage dimly on Level 1.
- Head start at round start: 25 s (20 vs computer) with a huge countdown; Level 1: 15 s (12).
- The computer Occupant: wander → hunt (noise) → chase (sight) → (`cut` on HARD/IMPOSSIBLE). **Difficulty** `G.df` 0–3 = `AI_DIFF` (EASY/NORMAL/HARD/IMPOSSIBLE): hearing multiplier (EASY 0.45 … IMPOSSIBLE 1.8), sight range, speed (0.85…1.08 × 4.0, sprint ×1.15 only on HARD+, IMPOSSIBLE max 4.97 < survivor sprint 5.0), persistence, stun length (1.25…0.6), dash/blackouts off on EASY. HARD/IMPOSSIBLE: lead moving targets, read a survivor heading for an objective (`aiObjectives()`: fuses/breaker/generators/valves/exit) and wait a few metres short of it (`objCD` stops it re-camping the same spot for 20 s), pass by objectives while wandering, take **vents** as shortcuts (`G.avt` = computer in a vent: hidden, can't kill, clang on exit). Picked with the COMPUTER button (menu for solo, lobby for host; `localStorage['lvl0_diff']`). Note: before v0.20 the computer's chase branch was unreachable (it never sprinted); NORMAL keeps no sprint. `__hp.aiTick/aiReset` exist for unit tests.

**Occupant skins** (`SKIN_NAMES`, index): 0 Occupant, 1 Mannequin, 2 Wallpaper Man (all built from boxes in code), 3 **Snatcher** (glTF, idle clip + code-driven walk/dash/stun offsets on bones), 4 **Crawler** (glTF static mesh split into body/head/4 limbs with generated skin weights in `rigCrawler()`, crawls on all fours with a diagonal gait). Heights: built-ins ~2.7 m, Snatcher ~2.7 m (nearly touching the ceiling), Crawler ~2.15 m. Each skin object has `headH` (head height) and optionally `anim(M,t,moving,speed,stunned,dash)`. Model skins fall back to skin 0 until loaded. Survivor avatars (hazmat suits) are scaled 0.86 so their eyes sit at the 1.6 m camera height.

**Caught** → full-screen jumpscare: the real head of the current skin (`MON.gltf.B.Head`, `MON.bones[5]`, or `MON.P.head`) is placed 0.45 m in front of your eyes, zoomed camera, shake, strobe, red static overlay (`#scare`), then a hard cut to "YOU WERE FOUND", then spectate.

**Camera/look**: PSX vertex snapping, low render resolution, film grain, camcorder OSD. An always-on VHS camcorder overlay (scanlines, bleed, head-switching strip, REC timecode) was tried in v0.20.0 and **removed in v0.20.2 at Ryan's request** — `updateCamFx` only does the near-monster bleed/jitter again. Level 0 fog is ×1.45 for survivors only (`updateZoneEnv`). When a survivor **sees** the Occupant (or it's very close) the render resolution drops hard (`updateLowRes`). VHS tracking effect near it.

## Audio

All in the `sfx` object (Web Audio, HRTF panners). Procedural sounds for most things; recorded samples loaded in `sfx.loadSamples()`:
- Occupant footsteps are **looped samples** that follow it in 3D (`updateOccLoops`): spider walking for the Crawler, heavy steps for everyone else; faster when sprinting. Disguised + far = normal survivor steps instead.
- Kill sounds: Crawler = random spider attack 1/2/4, all others = spider attack 2. Others nearby hear it positionally (`G.kl` = list of players killed by the Occupant). **When the computer catches you, the monster roar plays on top of the bite.**
- Computer chasing (`G.ach`, set while it can see you): `monster-groans` loop follows it in 3D + `monster-growl` every 3.5–7.5 s. Player Occupants don't get these.
- Ambience: fluorescent hum (fades when the Occupant is near), room tone, drips, TV static, distant random events (quiet), elevator dings, ceiling creaks + dust, **silence events** (8 s of total quiet), dread drone, heartbeat + phone vibration when it's close.
- **Removed on purpose**: the musical "sting" when you spot the Occupant, and the 8-bit horror noise loop. Don't add them back.

## Multiplayer

PeerJS, host's browser is the hub (star). Room code = 4 letters, peer id `level0-v1-CODE`. Max 4 players.
- Host owns the authoritative state `G` (see `freshState()`), broadcasts it every 0.1 s in play; clients send `presence` (position, flags) every 0.1 s and `use` requests (`useReq` → `applyUse` on the host). One-off sounds go over `snd`.
- `G.lv` = level (0/1/2), `G.gen` generators, `G.fz` fuses, `G.dead/esc/kl`, `G.dg` disguise, `G.ov` Occupant voice option, etc. Level geometry is rebuilt identically on every client from a seed (`baseSeed`, `lvSeed(k)`), walls synced as a hex bitmask.
- Lobby: host picks who's the Occupant (random / computer / a player), skins per player, OCCUPANT VOICE option, COMPUTER difficulty (`G.df`), START AT level (`G.sl` 0/1/2 = Level 0/1/37; menu button `bLv1` for solo, lobby `bLv` for host; `localStorage['lvl0_start']`). `setupLevel(k)` builds a level into `G` (used by `advanceLevel` and by `hostStartRound` when starting later). End screen: PLAY AGAIN or **BACK TO LOBBY** (keeps the room together).
- `#localnet` in the URL swaps PeerJS for a BroadcastChannel transport (same-browser tabs) — used for automated tests. It has **no voice calls**.

**Proximity voice chat** (`voice` object): PeerJS audio calls, one per pair (lower peer id calls). Opt-in via VOICE CHAT button (menu, lobby, settings). Rules: lobby/end = everyone full volume; in a round survivors fade out by ~12 m (muffled through walls); the Occupant hears survivors; **nobody hears the Occupant** unless the host turned on **OCCUPANT VOICE: MONSTER** — then survivors within ~10 m hear it through `monsterVoice()` (delay-line pitch shifter ~10 semitones down + 38 Hz growl ring-mod + distortion + echo, applied on the receiving side); dead players only talk to dead players. Mic meter under the objective (quiet / HEARD / LOUD). V mutes. Confirmed working by Ryan with real players.

## Settings

Esc (desktop), ⚙ button (phones), or SETTINGS in menu/lobby. Volume, voice volume, voice on/off, mic mute, look sensitivity, FOV (60–100), resume, leave. Saved in `localStorage['lvl0_set']`. Solo pauses; online keeps running.

Other localStorage keys: `lvl0_name`, `lvl0_skin2` (skin), `lvl0_voice`, `lvl0_monvoice`, `lvl0_diff`, `lvl0_start`.

Controls (desktop): WASD, Shift sprint, C crouch, Left click / F flashlight, Right click / G throw can, E hide / vent / start generator, V mute, Esc settings, Occupant 1–5. Phones: left thumb move, right thumb look, buttons bottom-right (Occupant gets 6 buttons incl. RUN and DASH), USE pops up near things.

## How to work on it

- **Edit `index.html` with exact-match replacements** (Python `rep(old,new)` asserting one match worked well). Then syntax-check the module: extract the `<script type="module">` body to a `.mjs` and run `node --check`.
- **Module TDZ trap**: top-level code runs top to bottom; calling something that touches `sfx` (declared late) from earlier top-level code crashes. Defer with `queueMicrotask` or call only from event handlers/frame.
- **Testing**: Playwright + system Chrome (`/usr/bin/google-chrome`, flags `--use-gl=swiftshader --enable-unsafe-swiftshader`), serve the repo with `python3 -m http.server 8765 --directory <repo>` **started in the same command** (background servers die between turns). `window.__hp` exposes internals (G, me, level, MON, sfx, voice, ability, contextAction, …) for tests. Two-player tests: two pages in one context with `#localnet`; add `--disable-background-timer-throttling --disable-renderer-backgrounding --disable-backgrounding-occluded-windows` and `bring_to_front()` the page that must tick (the host runs the simulation). Headless runs at low FPS, so game time runs slow — poll for conditions instead of fixed waits. Fake mic: `--use-fake-ui-for-media-stream --use-fake-device-for-media-stream`.
- Can't test real PeerJS (cloud server unreachable from the sandbox) — voice/online needs Ryan to try it.

## Publishing (important quirk)

1. Bump `#ver` in `index.html` and `V` in `sw.js` (add new assets to `CORE`).
2. Commit with author `rytheguy18 <rytheguy18@users.noreply.github.com>`, `git fetch --depth=10 origin main && git rebase origin/main`, then `git -c push.negotiate=true push origin main`.
3. **Pushes from the Claude GitHub app do NOT trigger the GitHub Pages build.** A commit from Ryan's own account does. The fix used every time: in the built-in browser (signed in as RyTheGuy18), open `https://github.com/RyTheGuy18/Haunted-Pines/edit/main/README.md`, add a `vX.Y.Z: …` line at the end, Commit changes → Commit changes. Wait ~90 s.
4. Verify: fetch `index.html?bust=…` with `cache:'no-store'` on the live site and check `#ver`.

## Ryan's preferences / direction

- Wants it genuinely scary and uneasy the whole time, Lethal Company–style (low-res, PSX, proximity voice).
- Balance: survivors shouldn't get caught too easily; the Occupant should be interesting but not overpowered. Prefers tells/counterplay over raw power.
- Likes concise change summaries. When he says "don't add anything yet", plan only and wait for "add"/"go".
- Asks to "publish to github" when done — that means push + the README-edit trick above.
