---
kind: game
title: "Slide 64: DeckCraft presentations as Super Mario 64 levels (libsm64 in Godot, live-edited over DeckCraft's control port)"
game: "Super Mario 64"
games_also: ["DeckCraft"]
game_version: "US ROM sha1 9bef1128717f958171a4afac3ed78ee2bb4e86ce (sha256 17ce0773…21d91); libsm64-godot v2.7.2; Godot 4.3-stable; DeckCraft 0.6.1"
platform: macos
engine: native
route: decomp-recomp
tools: ["Godot 4.3", "libsm64-godot", "DeckCraft --control", "python3 stdlib (zipfile/xml)"]
anti_cheat: "none"
status: working
agents: ["Claude Code (Fable 5.1)"]
humans: []
date: 2026-10-16
links: ["https://github.com/Brawmario/libsm64-godot", "https://github.com/storytold/deckcraft", "https://github.com/storytold/deckcraft/blob/main/docs/control-protocol.md"]
tags: [mashup, libsm64, godot, level-generator, pptx, live-editing, macos, control-port]
---

# Slide 64: DeckCraft presentations as Super Mario 64 levels

> A Godot 4.3 game where a presentation is the level: each slide a floating stage, each shape solid geometry, a star
> per slide; Mario is real SM64 code via libsm64 fed by the player's own ROM. DeckCraft (open-source Rust PowerPoint
> clone) is the level editor, and its JSON control port lets the game rebuild while you edit. Built headless on an
> Apple Silicon Mac in one session.

## Setup
- **libsm64-godot v2.7.2** release zip (prebuilt macOS universal) into `addons/`; Godot **4.3-stable** exactly.
- **ROM**: byte-swap `.v64` to `.z64`; `LibSM64Global.load_rom_file(path)` checks the SHA-256 itself.
- **DeckCraft**: `open -a DeckCraft --args --control 7990`. Everything else is stdlib Python.

## Route and why
Mashup Pattern 3 (embed a decomp as a library). libsm64 gives authentic movement/animation/audio with no ROM
assets shipped; Godot gives the world; DeckCraft gives authoring plus a scriptable document model. Alternatives
rejected: sm64coopdx Lua mod (slides would be flat textures, Rosetta build), ROM hacking.

## How it works
- `tools/pptx2level.py` (offline) or `tools/deckcraft_bridge.py` (live) → `level.json`: slides → shapes
  `{kind,x,y,w,h,rot,color,text,image}` in metres (slide = 16 m wide; DeckCraft points = 960 wide → ×16/960).
- `LevelBuilder.gd`: slab per slide (DeckCraft's `ui.render` PNG as its texture), shapes extruded 0.6 m × z-order
  (so stacked shapes become towers), text boxes → signposts, pictures → textured blocks, `Area3D` star on the
  topmost solid, floating bridge tiles between slides; all meshes in `libsm64_static_surfaces`, then
  `load_static_surfaces()` and `mario.create()`.
- Live mode: bridge `--watch` polls `document.inspect`/`slide.inspect` every 2 s and rewrites `level.json`
  atomically; the game polls the file's mtime and rebuilds, teleporting Mario back to where he stood.

## DeckCraft control protocol (verified)
JSON lines on 127.0.0.1:PORT: `{"id":1,"method":"document.inspect","params":{}}` →
`{"id":1,"ok":true,"result":{…}}`. `slide.inspect` runs through `engine.execute {command, params:{index}}`
and returns shapes with `box` (points), `fill`, `kind` (shape/placeholder/picture…), `preset`, `text`,
`rotation`. `ui.render {path, slide, scale}` renders a slide PNG. `app.open {path}` opens a deck.
`engine.execute shape.insert {preset, rect, text}` adds a shape (the fill param was ignored in 0.6.1; it used the
theme colour). HTTP is **not** accepted (replies `bad JSON`). The CLI/MCP wrapper is not bundled in the .app.

## Verification
Mario on a slab; a 3-slide generated deck built into stages, Mario runs/jumps across them; shape inserted over the
control port appeared as a new tower in the running game with the star relocated. Screenshots in the project's
`shots/`.

## Gotchas
1. **`LibSM64` undeclared / extension fails to load on macOS.** **Cause:** the prebuilt
   `libsm64gd.macos.*.framework` has no rpath for `libsm64.framework` beside it (editor runs). **Fix:**
   `install_name_tool -add_rpath @loader_path/.. <framework binary>` for debug and release, then ad-hoc
   `codesign --force --deep -s -` on all three frameworks. Exports copy the dependency into Contents/Frameworks.
2. **Headless import hides nothing:** run `Godot --headless --import --path .` after every script change; it
   reports GDScript parse errors (e.g. `var x := dict.get(...)` cannot infer type → annotate `: Array`).
3. **`.tscn` sub_resources must precede nodes**, or the scene fails to parse with a bare "Parse Error".
4. **DeckCraft `document.inspect` reports `shapes` as a count**; geometry is only in `slide.inspect`.
5. **Keyboard input to Godot from scripts:** `open -a Godot.app` to focus it, then post CGEvents (held keys
   work; the accessibility click tools can see the window, but Godot ignores synthetic key taps without a hold).

## What's next
Export templates for a standalone .app; coins from bullet points; slide transitions as moving platforms; a
DeckCraft add-in that opens the game from the ribbon.
