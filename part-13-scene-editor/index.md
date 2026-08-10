# Part 13: Scene Serialization and Editor

Persist scenes to disk, load them on background threads, and build an in-engine editor that authors content without bypassing your render snapshot contract.

| # | Chapter |
|---|---------|
| 1 | [Scene File Format](01-scene-format.md) |
| 2 | [Component Snapshots](02-snapshots.md) |
| 3 | [Async Loading](03-async-load.md) |
| 4 | [Editor Application](04-editor-shell.md) |
| 5 | [Editor Panels](05-editor-panels.md) |
| 6 | [Scene Editor Demo](06-scene-editor-demo.md) |

**Arc:** `spark_scene_v4` text format → registry Capture/Apply handlers → `SceneManager` async Pump → `EditorApplication` shell + panels → `SceneEditor3DDemo` save/load regression.

Continue to [Part XIV — Physics](../part-14-physics/index.md).
