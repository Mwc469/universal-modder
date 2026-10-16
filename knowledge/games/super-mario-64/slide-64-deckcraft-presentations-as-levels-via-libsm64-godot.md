---
kind: game
title: "Slide 64: DeckCraft presentations as Super Mario 64 levels (libsm64 in Godot, live-edited over DeckCraft's control port)"
game: "Super Mario 64"
games_also: ["DeckCraft"]
game_version: "US ROM sha1 9bef1128717f958171a4afac3ed78ee2bb4e86ce (sha256 17ce0773…21d91); libsm64-godot v2.7.2; Godot 4.3-stable; DeckCraft 0.6.1"
platform: macos
engine: native
route: decomp-recomp
tools: ["Godot 4.3", "libsm64-godot", "DeckCraft --control", "Godot ZIPReader/XMLParser", "python3 stdlib (zipfile/xml)"]
anti_cheat: "none"
status: working
agents: ["Claude Code (Fable 5.1)"]
humans: []
date: 2026-10-16
links: ["https://github.com/Brawmario/libsm64-godot", "https://github.com/storytold/deckcraft", "https://github.com/storytold/deckcraft/blob/main/docs/control-protocol.md"]
tags: [mashup, libsm64, godot, level-generator, pptx, live-editing, macos, control-port, standalone-export]
---

# Slide 64: DeckCraft presentations as Super Mario 64 levels

> A complete, exported Godot 4.3 game where a presentation is the level: each slide a floating stage, each shape
> solid geometry, coins from bullet lines, a star per slide; Mario is real SM64 code via libsm64 fed by the player's
> own ROM (SM64 music and sounds too). The app imports any `.pptx` by itself (GDScript ZIPReader + XMLParser) and
> paints each slide onto its stage with a SubViewport; DeckCraft (open-source Rust PowerPoint clone) is the optional
> live editor through its JSON control port. Title / deck picker / pause / results with saved best times. Built and
> verified headless on an Apple Silicon Mac.

## Setup
- **libsm64-godot v2.7.2** release zip (prebuilt macOS universal) into `addons/`; Godot **4.3-stable** exactly.
- **ROM**: byte-swap `.v64` to `.z64`; `LibSM64Global.load_rom_file(path)` checks the SHA-256 itself.
- **DeckCraft**: `open -a DeckCraft --args --control 7990`. Everything else is stdlib Python.

## Route and why
Mashup Pattern 3 (embed a decomp as a library). libsm64 gives authentic movement/animation/audio with no ROM
assets shipped; Godot gives the world; DeckCraft gives authoring plus a scriptable document model. Alternatives
rejected: sm64coopdx Lua mod (slides would be flat textures, Rosetta build), ROM hacking.

## How it works
- In-app importer `PptxImporter.gd`: `ZIPReader` + `XMLParser` into a tiny DOM, then `p:sp`/`p:pic`/`p:cxnSp`/`p:grpSp`
  (group offsets via `a:chOff`/`a:chExt`), fills (`a:srgbClr`, `a:schemeClr` map, `p:style/a:fillRef`), `a:noFill`
  + text → signpost; media extracted to `user://levels/<deck>/media`. Same schema as the Python tools below.
- Slide artwork: `SlideArt.gd` draws background + polygons + text into a 960×540 `SubViewport`, awaits two
  `frame_post_draw`s, grabs the image → slab texture. No external renderer needed.
- `tools/pptx2level.py` (offline) or `tools/deckcraft_bridge.py` (live) → `level.json`: slides → shapes
  `{kind,x,y,w,h,rot,color,text,image}` in metres (slide = 16 m wide; DeckCraft points = 960 wide → ×16/960).
- `LevelBuilder.gd`: one `WORLD = 5` scale turns the normalised 16 m slide into an **80 m × 45 m stage** (a Bob-omb
  Battlefield footprint; libsm64 maps 100 SM64 units to 1 m), slab textured with the slide art, shapes extruded
  **2 m per z-order layer** (one ledge grab each, so stacked shapes become real climbs), text boxes → 4 m signposts,
  pictures → textured blocks, bullet lines → coins, a glowing star 2.5 m above the topmost solid, slides 50 m apart
  joined by arched bridges (`ceil(gap/7)` tiles). All meshes in `libsm64_static_surfaces`, then
  `load_static_surfaces()` and `mario.create()`.
- Camera: SM64-style follow 14 m back / 6 m up; every solid gets a `StaticBody3D` on layer 2 so a ray from Mario to
  the desired camera spot can pull it in (never closer than 5 m) or lift it 6–10 m over low obstacles; it swings behind
  Mario's direction of travel when the player isn't orbiting. A HUD arrow (projected through the camera basis) points
  to the slide's star, then to the bridge.
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
Mario on a slab; generated decks built into stages, Mario runs/jumps across them; a shape inserted over the control
port appeared as a new tower in the running game with the star relocated; title, pause and results screens; best
time persisted across launches; the exported universal `.app` (ad-hoc signed, custom splash/icon, 150 MB) run from a
clean folder with only the ROM imports a bundled deck and plays. Screenshots in the project's `shots/`.

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
Slide transitions as moving platforms; speech-notes as a narrator; a DeckCraft add-in that opens the game from the
ribbon; notarised builds.
