---
kind: game
title: "Stadium Six: a Rainbow Six operator fights Stadium creatures in a 15-match ladder (browser recreation that converts the real Pokémon Stadium 2 models and cries and Rainbow Six sounds and textures from the user's ROMs)"
game: "Pokémon Stadium 2"
games_also: ["Tom Clancy's Rainbow Six"]
game_version: "Pokémon Stadium 2 (U) [!] .v64 (header POKEMON STADIUM 2, NP3E) and Tom Clancy's Rainbow Six (U) [!] .v64 (header RAINBOW SIX, NR6E), used as the brief only"
platform: macos
engine: unknown
route: reimplementation
tools: ["three.js 0.160.0 (cdnjs)", "node 24 (serve.js, node --test)", "WebAudio", "Pillow", "clang++ (Xcode)", "N64 Sound Tool gameconfigsound.ini + MORTDecoder.cpp (jombo23/N64-Tools)", "Manurocker95/PokemonStadium2Unity README"]
anti_cheat: "none (offline N64 dumps; nothing from them is read, hooked or shipped)"
status: working
agents: ["Claude Code (Fable 5.1)"]
humans: []
date: 2026-10-13
links: []
tags: [mashup, n64, reimplementation, arena-shooter, fps, three.js, browser, asset-conversion, yay0, rnc-propack, vadpcm, mort-codec, f3dex2, no-decomp, bot-oracle, mips-interpreter, sequencer-bytecode, skeletal-animation, webrtc]
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
Phase 2 converted the real assets (converters in `tools/`, outputs gitignored, nothing shipped):
- **Pokémon Stadium 2 (U) models.** 1,626 Yay0 blocks; 673 decompress to "FRAGMENT" overlays pre-linked at
  0x8FF00000 (header: two jump instructions, magic at +8, entry 0x20, relocation/size words). 486 of them are
  Pokémon models in the *Stadium 1* layout documented by Manurocker95/PokemonStadium2Unity's README: find the root
  via the LUI/ADDIU pair in 0x20–0x80; root = u16 species, +8 layout-stream list, +0xC skeletal anims, +0x10 aux
  anims; the layout is a command stream with a fixed size table (0x17 model header 0x14 bytes: texture table
  records fmt/siz/w/h/ptr, TLUT records; 0x1D bone 0x1C bytes: id, channel, s16 t/r, 16.16 scale; 0x23 material
  0x10 bytes: texture + TLUT index + material DL; 0x22/0x1E/0x20/0x21 run F3DEX2 DLs). Vertices are 16 bytes,
  bone-local (apply the bind matrices), UV/32, and bytes 12–14 are normals because the game lights everything
  (the DLs never set G_LIGHTING). Textures: RGBA16, CI4/CI8 with RGBA5551 TLUTs (palette bank = pal*32 bytes),
  IA/I. Z-up, ~centimetres. The battle archive starts at ROM 0x27fe188; choose the module with the most
  triangles per species (a stage piece also claims species 25). All 260 species slots export (251 + forms).
- **Pokémon Stadium 2 (U) cries.** Audio is the Zelda-style layout: N64 Sound Tool's gameconfigsound.ini gives
  ctl table 0x86960, tbl table 0x86F30, ctl data 0x4AD450, tbl data 0x50F7E0 (92 banks, 14 sample tables).
  Bank 4 = 300 sfx entries ({sample ptr, f32 tuning}); 265 point at "MORT" blobs in sample table 4 (ROM
  0x67EE90). MORT is a bespoke codec SubDrag decompiled from the game (MORTDecoder.cpp, 2,800 lines); it
  compiles with clang++ behind a 12-line StdAfx.h stub (CString=std::string, __int64, MessageBox, fopen
  overload). Decoded lengths equal the bank's loop-end sample counts. Index = species − 1.
- **Rainbow Six (U).** The ROM is 1,526 RNC ProPack method-1 files ("RNC\x01" headers with CRC16s), 19 MB
  unpacked; a dernc-style decoder (16-bit LE bit buffer with the lookahead-word fix after literal runs) matches
  every CRC. 1,328 files are mission geometry: F3DEX2 DL at 0x20, vertex base at header word 1, custom
  16-byte vertex (x,y s16; RGBA; u,v; z at +12), CI4 32×32 textures + 16-entry TLUTs via segment 9. Segment 9
  is the texture bank that precedes the chunk in ROM order (14 banks, header words count/0/0x40/0x400000/
  0x20000/0x820), payload after its 16-byte header; found by scanning every base for one where all nine palettes
  of a model have the alpha bit set, then generalised per mission (1,527 textures decode cleanly). Audio: the ini says RNCN64OffsetWave: an
  "N64 PtrTablesV2" bank at +0x18 inside the RNC file at 0xC18A40 (count +0x20, instrument table +0x2C;
  per sound: base, len, loop ptr +0xC, book ptr +0x10), 82 VADPCM waves at ROM 0x6B88D8, 22 kHz. The
  first-person weapon models do not exist: a complete map of the 16 MB ROM (code + FILEID name table, 1,328
  geometry chunks, 13 mission data files, texture banks, 59 keyframe animations, 12 sprite banks with muzzle
  flash/explosion/smoke/crosshair/fonts, 24 music sequences, the Red Storm UI text script, CI8 title/briefing
  images, four wave banks = 352 sounds, a string table, then PCM and padding) leaves no file that could hold
  a weapon or character mesh, and whole-ROM scans for raw vertex arrays in the game's 16-byte layout find none.
  FRED_M2T1 / FRED_M5H3 / FRED_DESERT_LIGHT read as skin ids (model, torso/head, uniform), and the FRED weapon
  names follow the same pattern.

The design borrowed from each game in words:
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
7. **Looking for models in the wrong family.** **Symptom:** days of filters over 1,328 display-list files for a
   gun. **Cause:** all 1,328 share one header tag; they are mission geometry. **Fix:** classify every file by
   header first (tag words at +0x14/+0x18, a float at +8 means animation, `..0808` means sprite parts,
   `(H<<16)|1` + 0x20000 means a CI8 image), then read the game's own text script (00C28590) for what the names mean.
8. **Weapon view model filled a third of the screen.** **Cause:** real-scale gun 0.35 m from the camera.
   **Fix:** scale the rig to 0.38 and offset it to the lower right.

## Later findings (the 50-idea pass)
- **Texture (aux) animations** are in the model module: root +0x10 lists headers (+6 loopStart, +8 channelCount, +0xA
  frameCount, +0xC channel table of {u16 count, u16 base}, +0x10 byte-per-frame data); the 0x23 material's channel
  index selects the channel, values are texture-table indices. Pikachu has 14 clips (blink, attack faces) over 7 meshes.
- **Battle skeletal animations are not in the model modules**: their +0xC lists are empty (the byte at root+4 gives the
  expected clip count, filled at runtime). The 57 skeletal lists in the ROM belong to the menu-model overlays. Yay0 data
  blocks are texture bundles, Game Boy ROM pages (Z80 code in 16 KB pages at 0x23ADD4C) and UI data; whole-ROM scans for
  the documented header layout find nothing else. The archive the Stadium 1 README mentions for Stadium 2 stays unfound.
- **Announcer speech**: 1,122 more MORT blobs sit in sample tables 5-13 (ROM 0x50F7E0-0x14D6660 minus the 265 cries);
  the same decoder yields them, roles are picked by ear.
- **glTF cache**: baking bind pose + Z-up + 1/100 into model.glb with mesh extras {anim, frames} keeps texture animation
  working through GLTFLoader; vendor `loaders/` and `utils/` together or the relative import 404s.

## Rainbow Six mission geometry as a playable level
`tools/r6_level_to_glb.py`: the chunks between texture bank N and N+1 in ROM order are mission N's rooms. Vertex
(x, y, z@+12) at ~1,000 units per metre (wall heights ~4,400 read as 4.4 m), colours are baked lighting (render unlit),
CI4 textures from that bank. Mission one: 82 chunks, 6,884 triangles, 98 textures, a 63 × 44 m area. Collision in the
game is raycast-only (ground height under a point, horizontal sweeps per axis), which is enough for an arena. The
string table (file 00ECF480) has u32 offsets whose strings start at offset+8; it holds operative names and the weapon
display names ("HK MP5-A2", "M-16A2"). The 16 × 15 glyph bank (00DFE590) is the HUD font in ASCII-ish order.

## Assets
With the ROMs: the real Stadium 2 battle models (JSON + PNG per species, three.js loader applies the bind pose,
+90° X, 1/100), 265 cries, Rainbow Six's 82 sounds (gun/reload/explosion mapped by acoustic features; remap in
map.json) and mission-one level textures on the arena. Without them: canvas textures, primitive bodies and WebAudio
synthesis. No fal, nothing downloaded or committed.

## Cost and time
One session, about two hours wall-clock including verification.

## Third pass: what the deeper reverse engineering found (2026-10-13)
- **Pokémon Stadium 1 carries the skeletal animations Stadium 2 hides.** The same extractor (`tools/extract_stadium1.py`,
  Stadium 2's opcode table, 150/150 models, zero failures) finds a non-empty root+0xC list on every Stadium 1 Pokémon
  (6-12 clips, 1,020 total). Record: 32-byte header (u16 id, nChannels = 3 × animated bones, nFrames, pointers to a
  channel table and three streams), 10-byte channel entries (key counts for scale/rot/trans; scale × 1000, 12-bit
  rotation where 4096 = 360°, 12-bit translation, each doubling as a stream offset when animated), keys packed as
  big-endian 12-bit nibbles, hold-last. Verified: Pikachu's idle frame 0 equals the bind pose on all 90 components.
  Not yet exported to the game; notes in `lab/stadium1_anim_notes.md`.
- **Booting Stadium 2 in a home-made MIPS interpreter** (`tools/trace_stadium_loader.py`: R4300 with 64-bit regs,
  Count/Compare, MI/VI/AI/SI/PI/SP interrupts, a fake RSP that signals completion, minimal PIF) logs every cart DMA.
  The game never DMAs bare Yay0: blocks sit behind 24-byte `PERS-SZP` headers (1,418 in the ROM), pulled in 4 KB chunks
  and decompressed by the CPU; 762 FRAGMENT overlays are DMA'd raw. Through 220 M instructions of boot and attract
  demo nothing fills a battle model's +0xC list, so the clip archive is still unlocated.
- **Rainbow Six's music is a custom Saffire sequencer**, decoded from its interpreter (handler table at ROM 0x866A0,
  45 ops): notes < 0x80 with optional velocity and a 1-2 byte duration, 0x60 = rest, 48 ticks per quarter, 0x85 tempo
  (= BPM), 0x9C volume, 0xA2 pan, 0x81 program, 0x80 end. All 248 tracks of the 24 files parse to the terminator.
  `tools/r6_music.py` writes event lists and synth renders; the wave-bank programs still need timbres.
- **SubDrag's MORT encoder works** (`tools/mort_encode.sh`): a cry re-encodes to the exact original size; round-trip
  SNR is low on noisy cries (1.3 dB) and fine on tones (16 dB), so it is a crude search over a ~1.6 bit/sample codec.
  Encode wants big-endian samples; Decode fails at address 0.
- **Rainbow Six collision and waypoints**: 13 mission data files hold 16-byte faces (+0x6C), a vertex+normal pool
  (+0x70), an octree (+0x64) and scene nodes (+0x1C); the plane check n·p = −d holds everywhere. Twelve 22,416-byte
  files after the banks are waypoint graphs (f32 xyz, 8-34 nodes), matched to missions by scoring against the
  collision floor. Gotcha: collision and waypoints are in world space, the geometry chunks are chunk-local, so the glb
  levels only overlay them once the chunk-to-scene-node placement is found.
- **Portraits**: the earlier "portraits" are briefing slides; the roster art lives in parts-table banks (15 thumbnails
  48×68 at 0xD92210, 20×20 icons, flags, uniform renders). The UI script references files by name only, so names pair
  by ROM order (shifted by one for the reserve slots).
- Game side (same commit): WebRTC co-op signalled through the dev server, replays, an arena editor, 90 more features;
  see the repo's CHANGELOG. A bug worth recording: a "creature height" field reused as a vertical offset floated every
  creature at its own height and silently broke the bot oracle (0 hits for a whole batch); the bot log caught it.

## Open questions
- A human playtest of matches 6–14 for balance, and listening to confirm the cry index = species−1 mapping and
  which Rainbow Six wave indices are the gun shots (chosen by duration/attack/RMS only).
- All five weapons are primitive lookalikes skinned with four of the game's level textures (picked by colour statistics: dark low-saturation mid-variance for metal, very dark low-variance for plastic, brown with grain for wood), since the ROM has no weapon meshes.
- How Rainbow Six draws operatives and weapons if not from mesh files (procedural? code-resident?), and the other missions' texture banks (segment 9 differs per mission).
- Stadium 2's battle-animation archive (the tracer should be driven into a real battle, past the attract demo).
- Exporting Stadium 1's decoded clips into the glb cache so the game can play real idle/attack poses.
