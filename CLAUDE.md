# Level 0 — The Backrooms (repo: RyTheGuy18/Haunted-Pines)

Notes for future Claude sessions. Read this before changing anything.

First-person co-op Backrooms horror game for 1–4 players, played in the browser (desktop + phones, installable as a PWA). Survivors escape Level 0, then Level 1, while one player (or the computer) is **the Occupant** hunting them. The repo is still named Haunted-Pines, from an older scrapped concept (a foggy-woods campground); the game itself is "Level 0".

- Live: https://rytheguy18.github.io/Haunted-Pines/ (GitHub Pages, `main` branch, repo root)
- Owner: Ryan (GitHub `rytheguy18` / `RyTheGuy18`). Plays it with friends and younger brothers, mostly on phones.
- Current version: **v0.19.1** (shown bottom-right of the menu, `#ver`)

## Files

| Path | What |
|---|---|
| `index.html` | **The whole game** (~2900 lines): CSS, HTML screens, one `<script type="module">`. No build step. |
| `three.module.min.js` | three.js **r160**, mapped via importmap as `three` |
| `jsm/loaders/GLTFLoader.js`, `jsm/utils/{BufferGeometryUtils,SkeletonUtils}.js` | three r160 example modules (must match r160) |
| `peerjs.min.js` | PeerJS 1.5.x (falls back to unpkg) — data + voice |
| `models/snatcher.glb` | Snatcher skin: Mixamo-rigged, one idle clip, texture shrunk to 512 px |
| `models/crawler.glb` | Crawler skin: "Goose" by nodoxi (Sketchfab, **CC BY 4.0** — credit is in the menu footer, keep it), static mesh, rigged in code |
| `sfx/*.mp3` | crawler-steps (spider walking loop), heavy-steps (loop), spider-attack-1/2/4 (kill sounds). Mono, trimmed for clean loops. Pixabay-style sources |
| `sw.js` | Service worker. **Bump `const V='level0-vX-Y'` every release** or phones keep the old version. CORE list must include every new asset |
| `manifest.webmanifest`, icons | PWA |
| `README.md` | Short description + one line per version (see Publishing) |

`index.html` script sections (search for `/* ===`): utils, constants, renderer, textures, lights, level, collision/sight, the Occupant (built-in model), players (survivor avatar), Occupant skins (+ Snatcher/Crawler loaders), shared state (+ host logic, networking, voice, settings), audio (`sfx` object), input, menu, per-frame.

## Gameplay as it stands

**Level 0 (the office)** — 16×16 grid of 4 m cells (`N=16, CS=4`), wall height 2.8 m. Three zones: office (yellow wallpaper/carpet), parking corner (concrete), "the pitfalls" (pits with ~1.2 m walkways; falling = death). Maze from DFS + random openings (`rng()<0.43` = how open), plus big halls, tight corridor blocks, a red room and a **clean office** (desk, monitor, plant, ticking clock).
- Goal: find **3 fuses** → put them in the **breaker** → exit door opens → reach it.
- Walls **shift** where nobody is looking (every 20–35 s, never visible ones).
- Items: almond water (sanity), batteries (flashlight), cans (throw to make noise), lockers (hide; hold breath bar; **the Occupant can't open lockers**).
- Phones ring and give hints (sometimes "found you"). Lying EXIT signs, scrawls, doors to nowhere, wet footprints into walls, an off-the-hook phone (dial tone), a rolling office chair.
- When **every living survivor** has escaped, after 3.5 s the whole room goes to Level 1. **Everyone caught on Level 0 comes back** for Level 1 (`G.dead` is cleared).

**Level 1 (parking garage)** — `LK = 1`, `zoneOf()` returns 1 everywhere. Concrete, pillar grid, 8 parked cars (solid box colliders in `L.slabs`, block sight). All lights off. **3 generators**: stand next to one and press START GENERATOR for 3 s (`GEN_T`, loud: noise level 2). Each one lights only the lamp right above it; all 3 → all lights on + exit opens. Escaping ends the round.

**Survivors**: walk 2.6, sprint 5.0 (5.6 with adrenaline when the Occupant is within 10 m), crouch 1.35 (silent). ~7 s of sprint. Flashlight (battery). Hold the light on the Occupant's face to **blind** it (stun 3.6 s). Sanity drains in the dark → hallucinations, whispers, warped screen. Crouching in the dark with the light off hides you from the Occupant beyond 4 m.

**The Occupant** (player or computer `G.mo==='AI'`):
- Walks 4.0 (`MON_SPEED`), **sprint** ×1.15 for ~1.8 s with ~4 s recharge. Never faster than a sprinting survivor.
- Abilities (keys 1–5, buttons on phones): **Blackout** (30 s, cd 45), **Shift walls** (cd 22), **Lure** (fake footsteps run up *behind the nearest survivor*, cd 20), **Disguise** (looks like a random survivor from ≥5 m for 12 s; tells: no flashlight, lights flicker; cd 40), **Dash** (~3 m lunge, arms out, cd 28).
- **Vents**: 3 linked pairs of floor grates; 2 s crawl, loud clang on exit, cd 15.
- Hears survivors: walking 11 m, running 32 m, loud 60 m, talking (voice) counts as noise.
- Hears a loud clunk when a fuse is placed (survivors don't hear that one).
- View has a red/sepia tint; darker on Level 0, can see the garage dimly on Level 1.
- Head start at round start: 25 s (20 vs computer) with a huge countdown; Level 1: 15 s (12).
- The computer Occupant: wander → hunt (noise) → chase (sight), also dashes (2–4 m) and sprints.

**Occupant skins** (`SKIN_NAMES`, index): 0 Occupant, 1 Mannequin, 2 Wallpaper Man (all built from boxes in code), 3 **Snatcher** (glTF, idle clip + code-driven walk/dash/stun offsets on bones), 4 **Crawler** (glTF static mesh split into body/head/4 limbs with generated skin weights in `rigCrawler()`, crawls on all fours with a diagonal gait). Heights: built-ins ~2.7 m, Snatcher ~2.7 m (nearly touching the ceiling), Crawler ~2.15 m. Each skin object has `headH` (head height) and optionally `anim(M,t,moving,speed,stunned,dash)`. Model skins fall back to skin 0 until loaded. Survivor avatars (hazmat suits) are scaled 0.86 so their eyes sit at the 1.6 m camera height.

**Caught** → full-screen jumpscare: the real head of the current skin (`MON.gltf.B.Head`, `MON.bones[5]`, or `MON.P.head`) is placed 0.45 m in front of your eyes, zoomed camera, shake, strobe, red static overlay (`#scare`), then a hard cut to "YOU WERE FOUND", then spectate.

**Camera/look**: PSX vertex snapping, low render resolution, film grain, camcorder OSD. When a survivor **sees** the Occupant (or it's very close) the render resolution drops hard (`updateLowRes`). VHS tracking effect near it.

## Audio

All in the `sfx` object (Web Audio, HRTF panners). Procedural sounds for most things; recorded samples loaded in `sfx.loadSamples()`:
- Occupant footsteps are **looped samples** that follow it in 3D (`updateOccLoops`): spider walking for the Crawler, heavy steps for everyone else; faster when sprinting. Disguised + far = normal survivor steps instead.
- Kill sounds: Crawler = random spider attack 1/2/4, all others = spider attack 2. Others nearby hear it positionally (`G.kl` = list of players killed by the Occupant).
- Ambience: fluorescent hum (fades when the Occupant is near), room tone, drips, TV static, distant random events (quiet), elevator dings, ceiling creaks + dust, **silence events** (8 s of total quiet), dread drone, heartbeat + phone vibration when it's close.
- **Removed on purpose**: the musical "sting" when you spot the Occupant, and the 8-bit horror noise loop. Don't add them back.

## Multiplayer

PeerJS, host's browser is the hub (star). Room code = 4 letters, peer id `level0-v1-CODE`. Max 4 players.
- Host owns the authoritative state `G` (see `freshState()`), broadcasts it every 0.1 s in play; clients send `presence` (position, flags) every 0.1 s and `use` requests (`useReq` → `applyUse` on the host). One-off sounds go over `snd`.
- `G.lv` = level (0/1), `G.gen` generators, `G.fz` fuses, `G.dead/esc/kl`, `G.dg` disguise, `G.ov` Occupant voice option, etc. Level geometry is rebuilt identically on every client from a seed (`baseSeed`, `lvSeed(k)`), walls synced as a hex bitmask.
- Lobby: host picks who's the Occupant (random / computer / a player), skins per player, OCCUPANT VOICE option. End screen: PLAY AGAIN or **BACK TO LOBBY** (keeps the room together).
- `#localnet` in the URL swaps PeerJS for a BroadcastChannel transport (same-browser tabs) — used for automated tests. It has **no voice calls**.

**Proximity voice chat** (`voice` object): PeerJS audio calls, one per pair (lower peer id calls). Opt-in via VOICE CHAT button (menu, lobby, settings). Rules: lobby/end = everyone full volume; in a round survivors fade out by ~12 m (muffled through walls); the Occupant hears survivors; **nobody hears the Occupant** unless the host turned on **OCCUPANT VOICE: MONSTER** — then survivors within ~10 m hear it through `monsterVoice()` (delay-line pitch shifter ~10 semitones down + 38 Hz growl ring-mod + distortion + echo, applied on the receiving side); dead players only talk to dead players. Mic meter under the objective (quiet / HEARD / LOUD). V mutes. Confirmed working by Ryan with real players.

## Settings

Esc (desktop), ⚙ button (phones), or SETTINGS in menu/lobby. Volume, voice volume, voice on/off, mic mute, look sensitivity, FOV (60–100), resume, leave. Saved in `localStorage['lvl0_set']`. Solo pauses; online keeps running.

Other localStorage keys: `lvl0_name`, `lvl0_skin2` (skin), `lvl0_voice`, `lvl0_monvoice`.

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
