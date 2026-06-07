# PixelRoot32 Agent Skills

Welcome to the **PixelRoot32 Skills Catalog**. This repository contains a collection of specialized "Skills" designed to supercharge AI Agents (like Gemini, Cursor, or Copilot) when developing video games for the **PixelRoot32 Game Engine**.

## 🤖 What is this?

This is not just standard documentation. These are **Agent Skills**—strict, context-rich rulebooks formatted specifically for LLMs. 

When you ask an AI to "create a new level" or "add a touch UI" for PixelRoot32, the AI reads the corresponding skill from this repository. This guarantees that the AI:
- Writes C++17 code highly optimized for the ESP32.
- Adheres strictly to the **Zero-Allocation** architecture (no `new`, no `std::vector`).
- Respects hardware limits (memory, spatial grids, fixed arrays).
- Avoids hallucinations by following exact syntax for drawing, physics, and audio.

## 🚀 How to Use These Skills

We use the official [skills.sh](https://skills.sh/) CLI to distribute these skills into your local agent configuration.

1. **Install a specific skill via npx:** 
   Run the following command for the subsystem you need. For example, to install the audio skill:
   ```bash
   npx skills add PixelRoot32-Game-Engine/pixelroot32-engine-skills/skills/pixelroot32-audio
   ```
2. **Contextualize your Agent:** The CLI will automatically copy the skill to your agent's config folder (e.g. `.cursor/rules` or `.gemini/config/skills`).
3. **Prompt freely:** Ask your AI to build features. For example: *"Create a new Scene with a pause menu using `pixelroot32-scene-manager` and `pixelroot32-ui-system`"*.
4. **Automatic Compliance:** The AI will read the requested skills, apply the engine's strict `Agent Constraints`, and generate production-ready code that complies with the ESP32 limits.

## 📚 Skill Catalog

The suite is divided into specific engine subsystems. Your AI will dynamically load these as needed:

| Skill | What the AI learns... |
|-------|------------------------|
| `pixelroot32-sprite-renderer` | How to draw 1bpp/2bpp/4bpp sprites, tilemaps, and use the Dirty Grid rendering without dropping frames. |
| `pixelroot32-physics` | How to set up rigid bodies, kinematic controllers, and use the Flat Solver without relying on floating-point math (FPU). |
| `pixelroot32-ui-system` | How to build HUDs, touch menus, and grids using pre-allocated elements to prevent dangling pointers. |
| `pixelroot32-audio` | How to synthesize NES-style 8-voice audio, ADSR envelopes, and queue sounds safely across threads. |
| `pixelroot32-memory-optimization`| **CRITICAL:** The strict rules of embedded C++ (No exceptions, no dynamic heap, pool allocation). |
| `pixelroot32-scene-manager` | How to handle game states, scene transitions (Fade/Iris), and Arena allocation. |
| `pixelroot32-particles` | How to spawn explosion or dust effects respecting the hard limit of 50 particles per emitter. |
| `pixelroot32-touch-input` | How to map raw touch events to gestures and prevent UI events from bleeding into the game logic. |
| `pixelroot32-camera2d` | How to manipulate the viewport, apply screen shake, and bounds clamping. |
| `pixelroot32-entity-actor` | How to structure the game objects using the Godot-inspired node hierarchy. |
| `pixelroot32-testing` | How to generate Unity framework unit tests using Mocks for isolated validation. |
| `pixelroot32-cpp-code-generation`| General C++17 formatting, naming conventions, and `-fno-exceptions` enforcement. |
| `pixelroot32-docs` | Doxygen documentation standards for the PixelRoot32 ecosystem. |

## 🛡️ Engine Constraints Enforced by these Skills

By using these skills, your AI is strictly instructed to follow the core pillars of PixelRoot32:
- **Zero-Heap Fragmentation:** Agents are forbidden from using `malloc`, `new`, or `std::shared_ptr` in the game loop.
- **Data-Driven Compilation:** Agents will correctly wrap subsystems in `#if PIXELROOT32_ENABLE_*` macros to save Flash and RAM.
- **Fixed-Point Math:** Agents will use `toScalar()` instead of `float` to guarantee determinism on non-FPU microcontrollers like the ESP32-C3.
- **Event Consumption:** Strict enforcement of `event.consume()` to prevent touch-input bleed.

---
*Built to empower developers to create highly optimized embedded games at the speed of thought.*
