# Godot Game Project — Full Instructions

You are building a complete, playable Godot 4.3 game and delivering a single self-contained Windows .exe. Read this entire document before writing code.

## 1. First — Install Runtime Libraries

The container does not have Godot's runtime dependencies. Run this once before doing anything else:

apt-get update && apt-get install -y --no-install-recommends \
  libgl1 libxcursor1 libxinerama1 libxrandr2 libxi6 \
  libfontconfig1 libasound2t64 libpulse0 \
  xvfb ca-certificates

If apt-get fails with permission errors, prepend `sudo`. If the container has no apt (e.g., Alpine), install equivalent packages with the available package manager — you need OpenGL, X11 cursor/screen libs, fontconfig, ALSA/PulseAudio, and xvfb.

If `libasound2t64` is not found (older Ubuntu/Debian), use `libasound2` instead.

## 2. Repository Layout

The repo is laid out like this. The agent's working directory is the AI_GAME folder.

<repo-root>/
├── tools/
│   ├── godot                                    # Linux editor binary
│   └── export_templates/4.3.stable/
│       ├── windows_release_x86_64.exe
│       └── windows_debug_x86_64.exe
└── AI_GAME/                                     # YOUR working directory
    ├── PROJECT_INSTRUCTIONS.md                  # this file
    ├── export_presets.cfg                       # preconfigured — DO NOT EDIT
    ├── scenes/                                  # put your .tscn files here
    ├── scripts/                                 # put your .gd files here
    ├── assets/                                  # for code-generated assets if needed
    └── build/                                   # your final .exe lands here

cd into AI_GAME before running any Godot command. The Godot binary is at `../tools/godot` from there. Make it executable if needed: `chmod +x ../tools/godot`.

## 3. What You Must Create

- `project.godot` — project manifest (see Section 5)
- All `.tscn` and `.gd` files for your game

You do NOT need to create an icon. The export preset has been configured to skip icon embedding.

Use `res://` paths for all in-project references. Never hardcode absolute filesystem paths.

## 4. Constraints

- GDScript only. No C#, no plugins, no addons, no external libraries.
- Compatibility renderer only. No Forward+ or Mobile features.
- No external assets. No .png, .wav, .ogg, .ttf files. Generate everything in code (see Section 7).

## 5. project.godot — Required Contents

Your project.godot must include this configuration so the game launches fullscreen and scales to any monitor:

[application]
config/name="AI_GAME"
run/main_scene="res://scenes/main.tscn"
config/features=PackedStringArray("4.3", "GL Compatibility")

[display]
window/size/viewport_width=1920
window/size/viewport_height=1080
window/size/mode=3
window/stretch/mode="canvas_items"
window/stretch/aspect="expand"

[rendering]
renderer/rendering_method="gl_compatibility"
renderer/rendering_method.mobile="gl_compatibility"

- `mode=3` is fullscreen
- `stretch/mode="canvas_items"` + `aspect="expand"` means the viewport is your design resolution (1920×1080) and Godot scales everything to fit any monitor with no black bars
- `run/main_scene` points to your entry scene; the path shown is a convention
- The renderer must be `gl_compatibility`

Add `[input]`, `[autoload]`, `[physics]` sections as your game requires.

## 6. Coordinate System and UI Layout — READ CAREFULLY

The single biggest source of bugs in Godot.

Origins differ by node type:
- Sprite2D, Polygon2D, Node2D, CharacterBody2D, Area2D: position IS the center of the shape. Place at screen center with `position = get_viewport_rect().size / 2`.
- Control and all UI nodes (Label, Button, ColorRect, Panel, Container): position is the TOP-LEFT corner of the rect, not the center.
- Viewport: (0,0) is the top-left. X increases right. Y increases DOWN, not up.

Center UI with anchors, never hardcoded pixel positions. `Vector2(960, 540)` will break on every monitor that isn't 1920×1080.

To center a Control:
- Code: set anchor_left = anchor_top = anchor_right = anchor_bottom = 0.5, then offset_left = -size.x/2, offset_top = -size.y/2.
- .tscn: anchors_preset = 8 (Center).

Fullscreen UI container: anchors_preset = 15 (Full Rect).

UI must live inside a CanvasLayer so it stays fixed regardless of camera movement.

## 7. Generate All Assets in Code

Visuals:
- Primitive nodes: ColorRect, Polygon2D, Line2D
- _draw() with draw_rect(), draw_circle(), draw_polygon(), draw_line(). Call queue_redraw() to trigger redraws when state changes.
- For sprite-like textures, build an Image with Image.create() and set_pixel(), convert via ImageTexture.create_from_image(image), assign to a Sprite2D.
- For text: Label nodes work without a font assigned (uses Godot's default font). Do not load .ttf files.

Audio:
- AudioStreamGenerator on an AudioStreamPlayer.
- Get playback with player.get_stream_playback() (returns AudioStreamGeneratorPlayback).
- Push samples with playback.push_buffer(frames) where frames is a PackedVector2Array of stereo samples in [-1.0, 1.0].
- Set mix_rate on the generator (e.g. 22050 or 44100).

## 8. Scene Architecture

You have full freedom over scene layout. Typical structure:

Main (Node2D)
├── World (Node2D)              — gameplay nodes, camera, player, enemies
│   └── Camera2D                — call make_current() in _ready() if multiple cameras
├── UI (CanvasLayer)            — HUD, menus
│   └── HUD (Control, anchors_preset=15)
└── Audio (Node)                — AudioStreamPlayers

Each distinct game object (player, enemy, projectile, pickup, level, menu) should be its own .tscn with a matching .gd. Instantiate sub-scenes:

const EnemyScene = preload("res://scenes/enemy.tscn")
var enemy = EnemyScene.instantiate()
enemy.position = Vector2(500, 300)
$World.add_child(enemy)

Switch full scenes (menu → gameplay → game over):

get_tree().change_scene_to_file("res://scenes/game_over.tscn")

Use signals for cross-scene communication: declare with `signal died`, emit with `died.emit()`, connect with `node.died.connect(callable)`. Avoid long `get_node("../../X")` chains.

If you override `_ready()` in a class extending a parent with its own `_ready()`, call `super._ready()` first.

## 9. GDScript 4.x Syntax

Godot 4.x differs from 3.x. Use these forms:

- `@onready var node = $Path` (not `onready var`)
- `@export var speed: float = 100.0` (not `export var`)
- `signal_name.emit(args)` (not `emit_signal("signal_name", args)`)
- `signal_name.connect(callable)` (not `connect("signal_name", target, "method")`)
- `super._ready()` (not `._ready()`)
- `Vector2i` exists for integer vectors, distinct from Vector2
- `await get_tree().create_timer(1.0).timeout` for delays
- Type hints encouraged: `func damage(amount: int) -> void:`

## 10. Verification Loop — Mandatory

Run all three steps. Fix every error and warning. Loop until clean. Do not deliver the .exe with unresolved ERROR: or WARNING: lines.

All commands assume you are in the AI_GAME directory. Make the binary executable first if needed:

cd <repo-root>/AI_GAME
chmod +x ../tools/godot

Step 1 — Import:

../tools/godot --headless --import 2>&1 | tee /tmp/import.log

Step 2 — Runtime smoke test:

timeout --kill-after=2 10 xvfb-run -a ../tools/godot --path . 2>&1 | tee /tmp/runtime.log

Hitting the 10-second timeout is SUCCESS — it means the game ran without crashing. Only error/warning prefixes count as failure.

Step 3 — Export:

../tools/godot --headless --export-release "Windows Desktop" build/AI_GAME.exe 2>&1 | tee /tmp/export.log
ls -lh build/AI_GAME.exe && file build/AI_GAME.exe

The .exe must exist, be non-zero size, and report as `PE32+ executable`.

Fail conditions for all three steps: any line containing `ERROR:`, `SCRIPT ERROR:`, `Parser Error:`, or `WARNING:` in the corresponding log. Stack traces span multiple lines — read context when diagnosing.

## 11. Final Deliverable

`<repo-root>/AI_GAME/build/AI_GAME.exe` — single self-contained Windows executable with .pck embedded.

Report:
- Confirmation that all three verification steps passed clean
- File size of the final .exe
- Brief description of the game and how to play
