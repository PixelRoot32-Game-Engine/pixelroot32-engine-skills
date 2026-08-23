# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A catalog of **Agent Skills** (structured Markdown rulebooks for LLMs) covering the subsystems of the **PixelRoot32 Game Engine** — a C++17 game engine for ESP32 microcontrollers. There is no buildable code here: the deliverables are the `skills/*/SKILL.md` files themselves, distributed to agent configs via the [skills.sh](https://skills.sh/) CLI:

```bash
npx skills add PixelRoot32-Game-Engine/pixelroot32-engine-skills/skills/<skill-name>
```

There are no build, lint, or test commands. Validation is manual review of the Markdown.

`.agents/` and `skills-lock.json` are local artifacts of the skills CLI and are gitignored — edit only the tracked files under `skills/`.

## SKILL.md File Format

Every skill lives at `skills/pixelroot32-<name>/SKILL.md` and starts with YAML frontmatter:

```yaml
---
name: pixelroot32-<name>
description: <one-line summary; ends with "Use when ..." guidance>
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: <audio|physics|...>      # subsystem skills only
  module: core
  feature_gate: PIXELROOT32_ENABLE_<X>  # if the subsystem is gated
  platform: cross-platform
  engine_version: "1.9.0+unreleased"    # engine revision the skill was verified against
---
```

`metadata.engine_version` is mandatory on every skill. The catalog previously carried no engine-version anchor at all — nothing recorded which engine a skill had last been checked against — which is exactly why a two-minor-version drift went unnoticed. The format is the engine version string, with `+unreleased` appended when the skill also documents APIs that are on `main` but not yet in a tagged release (e.g. `"1.9.0+unreleased"`). Update it whenever a skill is re-verified against a newer engine, and only then: it records verification, not intent.

Engine-subsystem skills (audio, physics, particles, camera2d, sprite-renderer, scene-manager, touch-input, ui-system, entity-actor, projection, gameplay-framework) follow a canonical section order — keep it when editing or adding skills:

1. `## Overview`
2. `## Key APIs` — per class: **Header** path, **Namespace**, **Feature gate**, then a compact C++ usage snippet
3. Subsystem-specific sections (formats, pipelines, enums, etc.)
4. `## Composition Patterns` — how the subsystem combines with others
5. `## ESP32 Constraints` — hard hardware limits and budgets
6. `## Gotchas`
7. `## Common Patterns`
8. `## Agent Constraints` — the strict do/don't list the consuming agent must obey

Cross-cutting skills (`cpp-code-generation`, `memory-optimization`, `testing`) do not follow that order. All three open with `## Overview` and then `## Source of Truth` — naming the engine docs they are derived from — before their own topic sections, and all three still end with `## Agent Constraints`. Keep the `## Source of Truth` section: it is what makes a cross-cutting skill auditable.

## Engine Rules the Skills Must Encode

All skill content documents (and must stay consistent with) the engine's core pillars — contradicting these across skills is the main failure mode to avoid:

- **Zero-allocation game loop**: no `new`, `malloc`, `std::vector`, or `std::shared_ptr` in `update()`/`draw()`; object pools and fixed arrays instead.
  - Scoped to the loop on purpose. **UI layout containers are the documented exception**: `UILayout` stores children in `std::vector<UIElement*>` (`include/graphics/ui/UILayout.h:141`) and `UIAnchorLayout` adds a second `std::vector<std::pair<UIElement*, Anchor>>` — heap-backed and unbounded, with no max-children constant. The rule the skills encode is therefore *build the layout tree once in `Scene::init()`/`initUI()` and never add or remove layout children per frame*. Do not "correct" `pixelroot32-ui-system` back into claiming the engine has no `std::vector` anywhere; that claim is false and was already disproved by a code read.
- **Fixed-point math**: `toScalar()` rather than `float` for determinism on non-FPU chips (e.g. ESP32-C3).
- **Feature gates**: subsystems are wrapped in `#if PIXELROOT32_ENABLE_*` macros; skills state the gate for each API they document.
- **No exceptions/RTTI** (`-fno-exceptions`); C++17; PascalCase types, camelCase methods/members, no `m_` prefixes; public namespaces are `pixelroot32::{core,graphics,graphics::ui,input,physics,math,audio,gameplay}`.
- **Event consumption**: touch/UI events must call `event.consume()` to prevent input bleeding into game logic.
- Hard limits are contractual (e.g. 50 particles per emitter, 8 audio voices) — when documenting APIs, keep the stated limits, header paths, and namespaces exact; these skills exist specifically to stop agents from hallucinating APIs.

## Keeping the Catalog Consistent

- `README.md` has a table listing every skill — update it when adding or renaming a skill.
- `pixelroot32-cpp-code-generation` holds the **canonical subsystem routing table** (`## Subsystem Skills`). Add new subsystem skills there; every other skill points at it rather than restating it.
- Skills cross-reference each other by name (e.g. `pixelroot32-cpp-code-generation` lists the subsystem skills; composition sections name sibling skills) — keep references valid when renaming.
