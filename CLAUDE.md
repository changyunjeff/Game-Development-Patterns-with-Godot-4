# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Companion code for the Packt book *Game Development Patterns with Godot 4* (Henrique Campos). It is written in GDScript for **Godot 4.3**. Every chapter builds on the same 2D platformer, which uses the "Kings and Pigs" art: the player is the King Pig and the enemies are Bumping Pigs.

## Layout: 24 independent Godot projects

```
NN.chapter-slug/
  01.start/      # project state at the beginning of the chapter
  02.finished/   # project state after applying the chapter's pattern
```

- Each `01.start` / `02.finished` folder is its own Godot project with its own `project.godot`. `res://` paths resolve relative to that folder. No code is shared between projects. Every chapter carries a full copy of `Actors/`, `Recipes/`, `Objects/`, `Assets/` and the rest.
- Chapter 3 (SOLID) has no code. The chapter folders go 01, 02, 04, 05 … 13.
- To find what a chapter teaches, diff its two projects: `diff -rq NN.*/01.start NN.*/02.finished -x '*.import' -x Assets`.
- Chapter N's `01.start` is usually close to chapter N−1's `02.finished`, but not identical. Start projects sometimes add new scaffolding, such as the `Game/` scene, doors, and `Level1`/`Level2` in ch05, or the extra bomb states in ch09.
- A bug fix to a file that appears in many projects (for example `Node2DFactory.gd`) is **not** propagated automatically. Choose deliberately which chapter copies to change. Past fix commits were scoped to specific chapters.
- The code mirrors the book's text. Keep changes minimal and faithful to what the chapter teaches. Don't refactor across the pattern a chapter introduces.

## Running / validating

There is no build system, linter, or test suite. Validation means loading or running a project in Godot 4.3:

```sh
godot -e --path 13.improving-game-feel-with-event-queue/02.finished   # open in editor
godot    --path 13.improving-game-feel-with-event-queue/02.finished   # run (main scene: Screens/MainMenu/MainMenu.tscn)
godot --headless --path <project> --import                            # import assets / surface script & resource load errors
```

`.godot/` is gitignored, so the first open of any project triggers a full reimport. The `*.import` sidecar files **are** committed. Keep them alongside their assets.

## Architecture (as of the final project, ch13 `02.finished`)

**Scene flow:** `Screens/MainMenu` → `Game/Game.tscn` (HUD wrapper whose `Level` child is `Levels/Level1`) → `Objects/Door` calls `change_scene_to_file()` to load another `Levels/LevelN.tscn` directly. `TeleportData.target_portal_name` tells the next level which door node to spawn the player at (`Level._ready`). `MainMenu` loads remapped controls from `user://control_settings.json`.

**Level contract:** every level inherits `Levels/Level.tscn`, and its root node must be named `Level`. The base scene provides named container nodes (`Bombs`, `Diamonds`, `Hearts`, `Crates`, `Enemies`, `PopLabels`, `Doors`, `Platforms`) and `PlayerCharacter2D`. `Node2DFactory.create()` uses `find_parent("Level").find_child(target_container_name)`, so renaming either the root or a container breaks spawning.

**Recipes/ = reusable generic building blocks; Actors/ = game-specific characters.** Player inheritance chain:
`BasicMovingCharacter2D` → `PassThroughCharacter2D` → `WallJumpingCharacter2D` → `Player2D` (input → commands) → `KingPigPlayer2D`.
The enemy body is `BumpingEnemy2D` (extends `BasicMovingCharacter2D` and emits `bumped` on walls). It is wrapped by the `BumpingPig` Node2D, which owns the state machine, commands and brain.

**Combat** uses paired areas in `Recipes/Hazard2D`. `HitArea2D` deals damage to a `HurtArea2D` only when their `team` exports differ. Actors react to `HurtArea2D.hurt(damage)`.

**Physics layers** are named in each `project.godot`: 1 Environment, 2 Pass Through, 3 Combat, 4 Interaction, 5 Visibility, 6 Pickable, 8 Detection. Layers 4 and 6 appear in later chapters.

**Where each pattern lives** (the chapter that introduces it):

| Ch | Pattern | Key files |
|---|---|---|
| 05 | Singleton | Autoloads `Score` (`Game/Score`), `TeleportData` (`Objects/Door/TeleportData`) |
| 06 | Observer | `KingPigPlayer2D` `lives_increased`/`lives_decreased` signals, wired in `Game/Game.gd` to `Interface/LivesBar` |
| 07 | Factory | `Objects/Node2DFactory` (Marker2D that instantiates `product_packed_scene`), `Objects/Loot`, `Objects/Heart` |
| 08 | State | `Actors/BumpingPig/States/*` (each subclasses `BumpingPigState`). `BumpingPig.set_state()` calls `exit`/`enter`, and every action method on `BumpingPig` delegates to `state`. Transitions use `context.state = context.find_child("XState")`. Animation is driven through `Actors/AnimationTree.gd` conditions. |
| 09 | Command | `Actors/Command.gd` (`receiver`, `execute`/`unexecute`). Commands are child nodes under a `Commands` node, and the actor's `_ready` assigns itself as `receiver`. The player's `_unhandled_input` and the enemy's `Brain.gd` both drive the actor only through commands. |
| 10 | Strategy | `Actors/BumpingPig/InteractionStrategies/*`. A `VisionArea2D` names a strategy script via `interaction_strategy_file` (set in the target's `.tscn`). `Brain` runs that strategy when it sees the area. |
| 11 | Decorator | `Recipes/BasicMovingCharacter2D/Stats.gd` (a Resource whose every property goes through a setter/getter) and `Actors/Decorators/*StatsDecorator.gd`. Autoload `PowerUpsSingleton` (`Game/PowerUps`) wraps `player.stats` and unwraps it on a timer. |
| 12 | Service Locator | Autoload `BackgroundMusic` (`Game/BackgroundMusic/BackgroundMusicLocator.gd`) cross-fades streams. Levels and the menu call `BackgroundMusic.play(stream)`. `Areas/AmbienceArea2D` switches the music while the player is inside the area and restores it on exit (used by `Areas/SecretRooms`). |
| 13 | Event Queue | `Actors/KingPig/InputBuffer` queues recent key events with expiry, so a jump pressed just before landing still fires. |

Autoloads differ per project. Check the `[autoload]` section of the specific `project.godot` before referencing one.

## Conventions

- GDScript is indented with tabs. Scripts typically put `class_name` and `extends` on the first two lines, in either order.
- Most signal connections are made in `.tscn` files rather than in code. When renaming a method like `_on_*`, update the `[connection]` entries in the scene files too.
- `.tscn`/`.tres` files are text and reference resources by `uid://` and `ExtResource("id")`. Keep hand edits small, and prefer the editor for structural scene changes.
