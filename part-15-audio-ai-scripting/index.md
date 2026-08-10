# Part 15: Audio, AI, and Scripting

Wire playback and spatial audio, add steering-based movement AI, and host C# scripts that mutate game state—not Vulkan handles—each frame.

| # | Chapter |
|---|---------|
| 1 | [Sound Engine](01-audio.md) |
| 2 | [Spatial Audio](02-spatial-audio.md) |
| 3 | [AI Steering](03-ai-steering.md) |
| 4 | [C# Scripting](04-scripting.md) |

**Arc:** `SoundEngine` PumpFrame → spatial listener + `SoundCueComponent` flush → `SteeringComposer` on `AiBlackboard` → `SparkInterop` C ABI for managed gameplay.

Continue to [Part XVI — Production](../part-16-production/index.md).
