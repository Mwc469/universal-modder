---
kind: game
title: "Stadium Six: a Rainbow Six operator fights Stadium creatures in a 15-match ladder (clean browser recreation)"
game: "Pokémon Stadium 2"
games_also: ["Tom Clancy's Rainbow Six"]
game_version: "Pokémon Stadium 2 (U) [!] .v64 (header POKEMON STADIUM 2, NP3E) and Tom Clancy's Rainbow Six (U) [!] .v64 (header RAINBOW SIX, NR6E), used as the brief only"
platform: macos
engine: unknown
route: reimplementation
tools: ["three.js 0.160.0 (cdnjs)", "node 24 (serve.js, node --test)", "WebAudio"]
anti_cheat: "none (offline N64 dumps; nothing from them is read, hooked or shipped)"
status: working
agents: ["Claude Code (Fable 5.1)"]
humans: []
date: 2026-10-13
links: []
tags: [mashup, n64, reimplementation, arena-shooter, fps, three.js, browser, procedural-assets, no-decomp, bot-oracle]
---

# Stadium Six: a Rainbow Six operator fights Stadium creatures in a 15-match ladder (clean browser recreation)

> Asked to "combine" Pokémon Stadium 2 and Rainbow Six (N64) into "a soldier in the arena fighting the
> pokemon in progressively harder matches". Neither game has a decomp, recomp, PC port, format docs or a
> scriptable Mac emulator here, so the mashup is a standalone browser game that recreates the idea: a
> Rainbow Six planning phase and hitscan weapons, a Stadium arena with crowd, jumbotron, announcer box and
> HP bars, twelve original type-themed creatures, and a Gym-Castle ladder that scales level, count and
> simultaneity. It runs and was verified by a bot mode in the desktop app's browser pane.

## Setup
- macOS 27 (Apple Silicon), node 24. No uv, Blender, ffmpeg or fal key on the machine; nothing needed them.
- Project: `/Volumes/warp/Claude/stadium-six` (`node serve.js 8766`, then open the page; ES modules need a
  server). three.js is loaded from cdnjs via an import map, pinned to 0.160.0; no npm install.
- Both ROMs stayed untouched; their SHA-1s and header names are in the project's `MODLOG.md`.

## Route and why
`reimplementation`, standalone: "engine recreation" in `knowledge/techniques/choosing-a-mashup-route.md` terms,
except that no owned files are read either, because no documented format exists for either game.
- Decomp/recomp: none for Pokémon Stadium 2 or Rainbow Six N64; N64Recomp needs symbol/function data that
  nobody has produced. Making it would be a project on its own.
- Emulator passthrough: only a Windows Project64 archive on this Mac, no Lua/memory scripting, no RAM maps
  for either game, and two emulated games with no oracle is the "drifting" failure the mashup skill warns about.
- Pattern 1 (port content into a host): there is no host with a mod API for this pair, unlike the Mario Kart 64
  note (SpaghettiKart).
So the lightest route that delivers the idea with an oracle is a clean standalone game. Creature designs and
names are original lookalikes (one per type), not rips.

## How the game works (what we had to learn)
Nothing was learned about either ROM's internals. The design borrowed from each game in words:
- Stadium: trainer "sends out" creatures, Stadium-style HP bars (green → yellow → red), "X fainted!", a
  crowd and jumbotron, levels, Gym Leaders → Elite Four → Champion, Prime Cup after the ladder.
- Rainbow Six: a planning phase before each match (primary, armour, gadget pack), MP5/M16/Benelli/PSG-1 plus a
  Beretta sidearm, armour as a separate pool, frags and flashbangs, no health regeneration.

## Build steps
1. `src/rules.js`: every number (roster, ladder, scaling, weapons, armour, gadgets, damage) in one pure module
   with `node --test` coverage. Write this first; it is the oracle for balance and never touches the DOM.
2. `src/world.js`: arena (textured floor disc, wall, 12 stand tiers, 2,600-seat `InstancedMesh` crowd that hops
   on knockouts, floodlights, canvas-texture jumbotron, six cover blocks, star dome).
3. `src/creatures.js`: procedural low-poly bodies with a mesh named `head` for headshots; AI state machine
   (intro → chase → windup → strike → recover, plus dash, stun, phase, blink, swoop, lob, breath); projectiles
   with gravity/homing; particle FX.
4. `src/player.js`: pointer-lock FPS with drag fallback, `Raycaster` hitscan with spread, falloff and
   headshot multiplier, recoil, reload, swap, grenade physics.
5. `src/main.js`: state machine (title → loadout → intro → battle → win/lose → results), HUD, save in
   `localStorage`, `?bot=1` auto-aim/auto-advance and `?match=N`.
6. Render at half resolution (`renderer.setSize(w/2, h/2, false)` + CSS `image-rendering: pixelated`) with
   `flatShading` Lambert materials for the N64 feel.

## Verification
- `npm test`: 10 rules tests (ladder structure and monotonic difficulty, scaling caps, damage falloff/headshot/
  armour, explosion falloff, HP colours, loadout sanitising, announcer lines).
- Live, in the desktop app's browser pane with `?bot=1`: title → planning → "MATCH 1" → "GO!" → "TRAINER
  BILLY sent out VOLTRAT!" → headshot KO → "WINNER!" → results → next planning screen, zero console errors.
  `?match=4` ran two VOLTRATs at once then SKYRAZOR; `?match=14` ran WYRMKING (Lv 65, 814 HP) with breath
  projectiles beside IRONHUSK; setting `__ss.player.hp = 1` reached "DEFEATED" → results with RETRY.
- Not verified: a real human playthrough with pointer lock and audio (the pane runs the bot without pointer
  lock), balance of matches 6–14 by hand, and performance on a weak GPU.

## Gotchas
1. **The sim froze on the "MATCH 1" banner in the browser pane.** **Cause:** the embedded pane throttles
   `requestAnimationFrame` to about one tick per second. **Fix:** a 40 ms `setInterval` watchdog that calls the
   frame function when no rAF has arrived for 80 ms, with dt clamped to 50 ms. Harmless in a normal browser.
2. **`python3 -m http.server` would not start from the launch config.** **Cause:** the app sandbox refuses
   `os.getcwd()` (EPERM) in that process. **Fix:** a 15-line `serve.js` on node, with `cache-control: no-store`.
3. **Black sky, no stars, stands reduced to a thin band; painting the sky dome red changed nothing.**
   **Cause:** stand tiers built as closed `CylinderGeometry` rings. A cylinder's caps are full discs, so the
   downward-facing bottom caps of every tier above eye height formed a dark ceiling at 2.5 m over the whole
   arena. `DoubleSide` only turned the ceiling grey. **Fix:** open-ended cylinders for the risers and
   `RingGeometry` annuli for the treads; nothing spans the arena. Diagnose with a scene traverse that recolours
   the suspect mesh from the console, not by staring at screenshots.
4. **Stars missing even with the ceiling gone.** **Cause:** `PointsMaterial` size attenuation at 190 m.
   **Fix:** `sizeAttenuation: false`.
5. **Everything past 40 m washed to the background colour.** **Cause:** fog 40–110 with stands at 25–50 m.
   **Fix:** fog 90–260 and a brighter hemisphere light.
6. **Hit creatures turned into white silhouettes.** **Cause:** emissive set to white for the hit flash on
   every material. **Fix:** a dim red emissive (0x803030) for 80 ms.
7. **Weapon view model filled a third of the screen.** **Cause:** real-scale gun 0.35 m from the camera.
   **Fix:** scale the rig to 0.38 and offset it to the lower right.

## Assets
All procedural: canvas textures (floor ring, jumbotron), primitive-built bodies, WebAudio synthesis for shots,
cries (per-type base pitch and waveform), crowd bed (looped filtered noise with swells), explosions and a short
battle loop. No fal, no files.

## Cost and time
One session, about two hours wall-clock including verification.

## Open questions
- A human playtest of matches 6–14 for balance (the bot is a poor player: ~15–100% accuracy depending on range).
- Creature models could be replaced by fal-generated sprites or rigged 3D if a key is available.
- If a Pokémon Stadium 2 decomp ever appears, the roster could read real stats from the user's own ROM via a
  converter, keeping this engine as the host.
