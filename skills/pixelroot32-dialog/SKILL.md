---
name: pixelroot32-dialog
description: The PixelRoot32 dialog system — DialogRunner headless state machine, DialogLine/DialogChoice flash scripts, DialogBox panel, speaker portraits and the selection caret. Use when writing branching dialogue, choice menus, paged text boxes or speaker faces.
license: MIT
compatibility: opencode>=0.1.0
metadata:
  domain: engine
  subsystem: dialog
  module: dialog
  feature_gate: PIXELROOT32_ENABLE_DIALOG
  platform: cross-platform
  engine_version: "1.11.0+unreleased"
---

## Overview

Dialog ships as **three independent pieces**, not one widget: **`TextLayout`** wraps and
measures text (always available, no flag), **`DialogRunner`** is a headless five-state
machine over a caller-owned flash script, and **`DialogBox`** is an optional default panel.
A game takes all three, or the runner alone with its own presentation. Nothing is copied,
nothing is heap-allocated, no piece owns another. Reference implementation:
`examples/dialog` (4bpp speaker faces, multi-column row, one-time riddle option).

**One flag gates all three dialog headers** (`gameplay/DialogTypes.h`,
`gameplay/DialogRunner.h`, `graphics/DialogBox.h`): `PIXELROOT32_ENABLE_DIALOG`, default
`0`. Without `-D PIXELROOT32_ENABLE_DIALOG=1` the headers are empty translation units —
adding the flag is part of writing the code, not a follow-up. Multi-color portraits need
their sprite flags too (see Portraits).

## Key APIs

### DialogTypes — the flat flash data model

**Header** `include/gameplay/DialogTypes.h` · **Namespace** `pixelroot32::gameplay` ·
**Feature gate** `PIXELROOT32_ENABLE_DIALOG`

```cpp
using LineId = uint16_t;   using ChoiceId = uint8_t;
inline constexpr LineId   kNoLine   = 0xFFFF;   // "no next line" — finishes the dialog
inline constexpr ChoiceId kNoChoice = 0xFF;     // "no choice" — no line may address choice >= 255
inline constexpr uint8_t kLineFlagAllowCancel  = 0x01;   // Cancel acts on a Choice line
inline constexpr uint8_t kLineFlagPortraitRight = 0x02;  // portrait top-RIGHT instead of top-left
enum class LineKind : uint8_t { Text = 0, Choice = 1, End = 2 };
enum class DialogState : uint8_t { Inactive = 0, ShowingText, AwaitingAdvance, ShowingChoices, Finished };
enum class DialogAction : uint8_t { None = 0, Advance, Up, Down, Confirm, Cancel };

struct DialogLine {                       // 36 B ESP32 / 64 B native — field order is fixed
    const char* text;                     // flash literal; nullptr = choice-only or End line
    const char* speaker;                  // nullptr = no speaker row
    LineId      next;                     // Text only; kNoLine finishes. Choice lines branch per-option
    uint16_t    tag;                      // opaque, carried on LineEnter; 0 is legal
    uint16_t    autoAdvanceMs;            // 0 = wait for player; non-zero = ShowingText timer
    ChoiceId    firstChoice;              // index into DialogScript::choices; must stay below 255
    uint8_t     choiceCount;              // clamped to DialogMaxChoices, the table, and below 255
    LineKind    kind;                     // discriminator — never inferred from populated fields
    uint8_t     flags;                    // unknown bits are IGNORED, not rejected (forward-compatible)
    const graphics::Sprite*     portrait = nullptr;      // 1bpp face, drawn in portraitInk
    const graphics::Sprite2bpp* portrait2bpp = nullptr;  // 4-color face, via portraitPaletteSlot
    const graphics::Sprite4bpp* portrait4bpp = nullptr;  // 16-color face, via portraitPaletteSlot
    uint8_t portraitPaletteSlot = 0;                     // sprite palette slot 0..7; ignored for 1bpp
};
struct DialogChoice {
    const char* text;            // flash literal; never wrapped — keep it inside the panel
    const char* detail = nullptr;// optional right-aligned 2nd column (a price); nullptr/"" draws nothing
    LineId      next;            // kNoLine ends the dialog
    uint16_t    tag;             // opaque, carried on ChoiceConfirmed
};
struct DialogScript { const DialogLine* lines; const DialogChoice* choices;
                      uint16_t lineCount, choiceCount; };   // 12 B ESP32; bound by pointer, never copied
```

The script, both tables and every string **must outlive every runner started against them** —
declare them `static const` at namespace scope (`.rodata`). A 9-value `DialogLine`
initializer still compiles: the portrait fields defaulted after it.

### DialogRunner — the headless five-state machine

**Header** `include/gameplay/DialogRunner.h` · **Namespace** `pixelroot32::gameplay` ·
**Feature gate** `PIXELROOT32_ENABLE_DIALOG`

```cpp
using DialogEventFn = void (*)(void* owner, const DialogEvent& event);  // plain fn pointer, no std::function
using ChoiceFilterFn = bool (*)(void* owner, LineId line, ChoiceId scriptIndex,
                                const DialogChoice* choice);            // pure: never call back into the runner
struct DialogEvent { DialogEventType type; LineId line; ChoiceId choice; uint16_t tag; };
class DialogRunner {
public:
    void configure(void* owner, DialogEventFn onEvent);   // static trampoline -> member, like StateMachine
    bool start(const DialogScript& script, LineId first = 0);  // false == rejected AND fully reset
    void stop();                                          // to Inactive; fires NO event
    void feed(DialogAction action);                       // total: illegal actions are silent no-ops
    void update(unsigned long deltaTimeMs);               // ticks autoAdvanceMs; no-op outside ShowingText
    void select(uint8_t visibleIndex);                    // direct selection, for touch hit-testing
    void setPageCount(uint8_t pages);                     // presenter declares pages; no-op with no current line
    void setChoiceFilter(void* owner, ChoiceFilterFn filter);  // nullptr (default) shows everything
    void refreshChoices();                                // renormalize after filter-relevant state changes
    DialogState state() const;  const DialogLine* currentLine() const;  LineId currentLineId() const;
    uint8_t page() const, pageCount() const;
    uint8_t choiceCount() const;                          // VISIBLE count under a filter (compacted indices)
    const DialogChoice* choice(uint8_t visibleIndex) const;  // nullptr once the runner leaves the line
    uint8_t selectedChoice() const;  uint16_t revision() const;  // change counter — compare by inequality only
};
```

`feed()` is total over state × action; `Confirm` aliases `Advance` in both text states; `Up`/`Down`
clamp, never wrap. One callback, one switch — events arrive synchronously:

| Event | `line` | `choice` | `tag` |
|---|---|---|---|
| `LineEnter` | entered line | `kNoChoice` | the line's tag |
| `ChoiceConfirmed` | the choice line | confirmed **visible** index | the **choice's** tag |
| `Cancelled` | the choice line | `kNoChoice` | the **line's** tag |
| `Ended` | last line, or `kNoLine` after a bad id | `kNoChoice` | — |

`enterLine()` assigns `state()` **before** emitting `LineEnter`. A callback's calls back into
`feed()`/`update()`/`start()` are **dropped, not queued** — but `stop()` and `select()` are
explicitly safe from inside the callback because they never emit.

### DialogBox — the optional default panel

**Header** `include/graphics/DialogBox.h` · **Namespace** `pixelroot32::graphics` ·
**Feature gate** `PIXELROOT32_ENABLE_DIALOG`

Deliberately **not** a `UIElement`: it works with `PIXELROOT32_ENABLE_UI_SYSTEM=0` and
allocates nothing. With UI on there are two panel paths — `DialogBox` for dialog, `UIPanel`
for structured HUDs (see **`pixelroot32-ui-system`**).

```cpp
struct DialogBoxStyle {
    const Font* font = nullptr;   // nullptr resolves FontManager's default
    int16_t x = 0, y = 0, w = 0, h = 0;   // h is used AS-IS, never clipped — size it via measureHeightPx()
    Color panel = Color::Black, border = Color::White, ink = Color::White;
    Color inkDim = Color::Gray;             // next-page cue
    Color inkSelected = Color::Yellow;      // selected choice only
    Color inkDetail = Color::Gray, inkDetailSelected = Color::Yellow;  // detail column
    Color portraitInk = Color::White;       // 1bpp portrait tint (2bpp/4bpp use portraitPaletteSlot)
    uint8_t borderWidth = 1;
    uint8_t padding = 4;        // inside the border AND above/below every choice row
    uint8_t textSize = 1, lineSpacing = 1;
    char choiceCaret = '>';     // 0 disables the caret AND its gutter, restoring old geometry
    bool fixedPosition = true;  // setOffsetBypass(true) while drawing, ignoring the camera
    bool portraitsEnabled = true;  // false draws every line portrait-less without touching scripts
    DialogPortraitSize portraitSize = DialogPortraitSize::Size16;  // fixed box: Size16/24/32
};
enum class DialogPortraitSize : uint8_t { Size16 = 16, Size24 = 24, Size32 = 32 };  // values ARE px
class DialogBox {
public:
    struct Layout { int16_t panelX, panelY, panelW, panelH, speakerX, speakerY, bodyX, bodyY,
                            choiceX, choiceY, choiceW, caretGutterPx, caretX, choiceTextX,
                            detailRightX, portraitX, portraitY, portraitW, portraitH,
                            bodyLineHeightPx, choiceRowHeightPx;
                    uint8_t bodyLineCount, choiceCount, pageCount, page;
                    bool hasSpeaker, hasPortrait, portraitRight;
                    TextLayout::WrappedLine bodyLines[platforms::config::DialogMaxWrappedLines]; };
    void setStyle(const DialogBoxStyle& style);  const DialogBoxStyle& style() const;
    static void computeLayout(const DialogBoxStyle&, const gameplay::DialogRunner&, Layout&);
    template <typename RendererT> void draw(RendererT& renderer, gameplay::DialogRunner& runner);
    bool needsRedraw(const gameplay::DialogRunner&) const;   // hint only — draw() is always safe
    bool choiceRect(const gameplay::DialogRunner&, uint8_t index, int16_t& x, int16_t& y,
                    int16_t& w, int16_t& h) const;           // false + untouched outputs when unusable
    static uint8_t pageCountFor(const gameplay::DialogLine&, const DialogBoxStyle&);  // Choice lines: always 1
    static int16_t measureHeightPx(const gameplay::DialogScript&, const DialogBoxStyle&);  // tallest single page
};
```

`draw()` is templated on the renderer type (`Renderer` declares no virtuals) and takes the
runner **non-const** solely to call `setPageCount()` — it changes no state and emits no event.
`computeLayout()` is the single geometry producer `draw()` and `choiceRect()` share, so drawn
and tappable geometry cannot drift apart.

## Authoring Scripts

```cpp
static const gameplay::DialogChoice kChoices[] = {
    {"Say hello",    nullptr, 3, 101},
    {"Ask a riddle", "once",  4, 102},   // detail draws right-aligned in inkDetail/inkDetailSelected
    {"Walk away",    nullptr, 5, 103},
};
static const gameplay::DialogLine kLines[] = {
    // text, speaker, next, tag, autoMs, first, count, kind, flags [, portraits..., slot]
    {"Welcome.", kGuide, 1, 0, 1800, 0, 0, gameplay::LineKind::Text, 0},
    {"Now try a choice.", kGuide, 2, 0, 0, 0, 0, gameplay::LineKind::Text, 0},
    {"What do you do?", kGuide, gameplay::kNoLine, 0, 0, 0, 3,
     gameplay::LineKind::Choice, gameplay::kLineFlagAllowCancel},
    {"Hello!", kGuide, 6, 0, 0, 0, 0, gameplay::LineKind::Text, 0},
    {nullptr, nullptr, gameplay::kNoLine, 0, 0, 0, 0, gameplay::LineKind::End, 0},
};
static const gameplay::DialogScript kScript{
    kLines, kChoices,
    static_cast<uint16_t>(sizeof(kLines) / sizeof(kLines[0])),
    static_cast<uint16_t>(sizeof(kChoices) / sizeof(kChoices[0]))};
```

`Text` with `autoAdvanceMs > 0` enters `ShowingText`, else `AwaitingAdvance`. `Choice` enters
`ShowingChoices` directly and its `next` is unused — branching comes from each option's `next`.
`End` finishes on entry. A `Choice` line may still carry prompt text, but it never pages.

## Driving the Runner

```cpp
runner.configure(this, &MyScene::onDialogEvent);   // static trampoline
runner.start(kScript, 0);

void MyScene::update(unsigned long deltaTime) {
    auto& input = engine.getInputManager();
    if (input.isButtonPressed(BTN_UP))      runner.feed(gameplay::DialogAction::Up);
    if (input.isButtonPressed(BTN_DOWN))    runner.feed(gameplay::DialogAction::Down);
    if (input.isButtonPressed(BTN_CONFIRM)) runner.feed(gameplay::DialogAction::Confirm);
    if (input.isButtonPressed(BTN_CANCEL))  runner.feed(gameplay::DialogAction::Cancel);
    runner.update(deltaTime);
}
```

**Choosing the next line from game state: restart in the same frame.** `DialogChoice::next`
is const flash and a `start()` from inside the callback is dropped — so read the pick
*before* `Confirm`, feed, then restart while still `Finished` (no drawn frame ever sees it):

```cpp
const gameplay::DialogChoice* picked = runner.choice(runner.selectedChoice());
runner.feed(gameplay::DialogAction::Confirm);
if (picked != nullptr && picked->tag == kTagBuy &&
    runner.state() == gameplay::DialogState::Finished) {
    runner.start(kShopScript, canAfford() ? kLineThanks : kLineNoMoney);
}
```

`Cancel` only acts on a `Choice` line carrying `kLineFlagAllowCancel`, and it **does not end
the dialog** — it emits `Cancelled` and stays on the line. The game calls `stop()` itself
(legal from inside the callback) when Cancel should close.

## Portraits

Set exactly one of `portrait` / `portrait2bpp` / `portrait4bpp` per line. The face draws 1:1
at the top of the content area — top-left by default, top-right with
`kLineFlagPortraitRight` — and the whole text column (speaker, body, choices) shifts to the
other side while the body wrap narrows by face width + one `padding`. Priority when several
are set: **4bpp, then 2bpp, then 1bpp**, shared by layout and draw. 1bpp draws in
`portraitInk`; 2bpp/4bpp resolve through the line's `portraitPaletteSlot` (0..7).

```cpp
// 4bpp face through palette slot 0, on the right:
{"Welcome.", kNarrator, 1, 0, 0, 0, 0, gameplay::LineKind::Text,
 gameplay::kLineFlagPortraitRight, nullptr, nullptr, &kNarratorFace, 0},
```

Three independent off-switches, all reproducing portrait-less geometry bit for bit: all
three pointers null (per line), `portraitsEnabled = false` (whole dialog), or an over-box
sprite (see below). The runner never reads any portrait field or the side flag.

**Multi-color faces need their sprite flags.** `PIXELROOT32_ENABLE_2BPP_SPRITES` /
`PIXELROOT32_ENABLE_4BPP_SPRITES` (default off) — without them the blit is a compiled-out
no-op *and* `DialogBox` reads that format as absent, so a missing face with unshifted text
means the flag (or a stale binary) rather than bad art. `examples/dialog` enables both in
`lib/platformio.ini`. Packing reminder for hand-authored art (see
**`pixelroot32-sprite-renderer`**): 4bpp is low-nibble-first, index 0 is transparent.

**Sizes are a closed set.** `portraitSize` (`Size16`/`Size24`/`Size32`, default `Size16`)
declares the square box every face must fit. Author faces at exactly the box; a smaller
sprite still draws 1:1, but a sprite larger than the box in either dimension is **ignored,
never drawn** — there is no 2bpp/4bpp scaler or clip rect, so drawing it would overflow the
panel. `measureHeightPx()` takes the taller of face and text block per line.

## Selection Caret, Touch and Paging

A non-zero `choiceCaret` (default `'>'`) reserves a gutter as wide as `{caret, ' '}` at the
current font/`textSize`. The caret draws at `Layout::caretX` — the text-column origin, past
a left-side face — in `ink`, **never** `inkSelected`: selection has two independent signals
because a zeroed Yellow slot once rendered a selected row invisible. `0` disables caret and
gutter, restoring pre-caret geometry. `choiceRect()` always reports the **full content-width
row**, face and gutter included, so the whole row stays tappable:

```cpp
for (uint8_t i = 0; i < runner.choiceCount(); ++i) {
    int16_t rx, ry, rw, rh;
    if (!box.choiceRect(runner, i, rx, ry, rw, rh)) continue;
    if (touchX >= rx && touchX < rx + rw && touchY >= ry && touchY < ry + rh) {
        runner.select(i);
        runner.feed(gameplay::DialogAction::Confirm);
        break;
    }
}
```

**Paging.** A `Text` line wrapping past `DialogMaxWrappedLines` (default `4`) splits into
pages; `draw()` publishes the count via `setPageCount()` and `Advance`/`Confirm` turns pages
before following `next`, with a `">"` cue in `inkDim` while a later page remains. A `Choice`
prompt never pages (`pageCountFor()` returns 1) — keep it to one page. `DialogMaxChoices`
(default `4`) clamps `choiceCount`. `measureHeightPx()` sizes for the **unfiltered** maximum,
so a pre-sized panel always fits, filtered or not.

**Sizing the panel** (draw uses `h` as-is, never clips):

```cpp
gfx::DialogBoxStyle style;
style.x = 4;
style.w = static_cast<int16_t>(platforms::config::LogicalWidth - 8);
style.borderWidth = 1;  style.padding = 4;  style.textSize = 1;  style.lineSpacing = 1;
style.h = gfx::DialogBox::measureHeightPx(kScript, style);   // reads w/font/padding/border/textSize
style.y = static_cast<int16_t>(platforms::config::LogicalHeight - style.h - 4);
box.setStyle(style);
```

`padding` costs `2 * padding * (1 + choiceRows)` px plus wrap narrowing — measure before
fixing `h`. `revision()` wraps (~18 min at 60 FPS): compare by inequality only, never order it.

## Feature Flags

Defined in `include/platforms/PlatformDefaults.h`, mirrored as `constexpr bool` in
`pixelroot32::platforms::config` (`include/platforms/EngineConfig.h`). Prefix with
`PIXELROOT32_ENABLE_`. Unlike the gameplay family, the dialog flag has no `GAMEPLAY_` infix
because it spans `gameplay/` and `graphics/`.

| Flag | Enables | Default | Mirror constant |
|---|---|---|---|
| `DIALOG` | `DialogTypes`, `DialogRunner`, `DialogBox` | `0` | `config::EnableDialog` (via `PIXELROOT32_ENABLE_DIALOG`) |
| `2BPP_SPRITES` | 2bpp blits incl. 2bpp portraits | off | `config::Enable2BppSprites` |
| `4BPP_SPRITES` | 4bpp blits incl. 4bpp portraits | off | `config::Enable4BppSprites` |

`DIALOG` has no dependency. The sprite flags pair with **`pixelroot32-sprite-renderer`**.

## Composition Patterns

**Full dialog scene — `examples/dialog`** (`DIALOG=1`, `2BPP_SPRITES`, `4BPP_SPRITES`): static
`kChoices`/`kLines`/`kScript` in `.rodata`, 4bpp faces (slime right, hero left), a
`ChoiceFilterFn` hiding the one-time riddle (renormalized via `refreshChoices()`), a
multi-column `"once"` detail, `measureHeightPx()` before fixing `h`, `box.draw()` after
`Scene::draw()`, button translation in `update()` plus `runner.update(deltaTime)`.

**Runner without the panel.** Drive `DialogRunner` alone and read `currentLine()` /
`page()` / `choice(i)` / `selectedChoice()` to feed a custom presenter (e.g. a
`UIPanel` HUD — see **`pixelroot32-ui-system`**); `pageCountFor()` + `TextLayout::wrap()`
(see **`pixelroot32-sprite-renderer`** for text measuring) give paging without `DialogBox`.

**Touch menus.** `choiceRect()` + `select()` + `Confirm` composes with
**`pixelroot32-touch-input`** hit-testing; `needsRedraw()` gates redraws on `revision()`.

## ESP32 Constraints

- Scripts are `.rodata` flash, never copied: `DialogLine` 36 B ESP32 / 64 B native,
  `DialogChoice` 12 B / 24 B, `DialogScript` 12 B ESP32. Growing them trips the
  `test_dialog_types_*_size_guard` tests by design.
- `DialogBox` 32 B ESP32 / 40 B native, `DialogBoxStyle` 28 B / 32 B — the portrait
  switch, box and detail colors all fit pre-existing tail slack (pinned by
  `test_dialog_box_sizeof_guard`).
- Zero heap in every method on all three pieces; `Layout` is stack-local.
- A 32x32 4bpp face costs 512 B flash; 16x16 costs 128 B. Faces are the dominant dialog
  flash cost — budget them like tiles.
- Any flag left off contributes **0 B**.

## Gotchas

- **A missing multi-color face with unshifted text is the sprite flag (or a stale binary),
  not bad art.** The blit compiles out without `2BPP/4BPP_SPRITES`, and the layout reads
  that format as absent. After changing engine headers, clean-rebuild examples (a running
  `program.exe` also blocks the native linker with "Access is denied" — kill it first).
- **Never call `start()`/`feed()`/`update()` from inside `DialogEventFn`** — the reentrancy
  guard drops them silently. `stop()` and `select()` are the only safe in-callback calls.
- **`ChoiceConfirmed`'s `choice` is a visible index** — valid only while the runner is still
  on the line (it is, during the callback). `choice()` returns `nullptr` afterwards.
- **`Cancel` never leaves the line by itself** — call `stop()` in the `Cancelled` branch.
- **Style colors are palette names resolved at draw time**, not RGB565. A zeroed slot draws
  black — keep Black/White/Gray/Yellow populated or point the style at slots the palette
  defines. The caret's `ink` (not `inkSelected`) coloring is the insurance.
- **`drawText()`/`drawSprite()` never clip and `h` is used as-is** — an undersized panel
  overflows. Always `measureHeightPx()` first; portraits never scale.
- **Over-box portraits vanish, they don't shrink.** A face larger than `portraitSize`
  draws nothing — during development a missing face means wrong box or wrong art size.
- **A `Choice` prompt that wraps past one page is unreachable** — the runner ignores
  `Advance` in `ShowingChoices`. Keep prompts short.
- **`select()` before `Confirm` from touch; never order `revision()`.**
- **Hand-packed 4bpp art is low-nibble-first** (even column in the low nibble), index 0
  transparent, row stride `ceil(width/2)` bytes. 2bpp packs LSB-first into 16-bit words.
  Verify by decoding back to the source art.

## Common Patterns

```cpp
// Minimal text + speaker line, no portrait, no choices
{"Hi there", "Bob", kNoLine, 0, 0, 0, 0, gameplay::LineKind::Text, 0},

// Timed auto-advance + right-side 4bpp face
{"Watch out!", kNarrator, 1, 0, 1800, 0, 0, gameplay::LineKind::Text,
 gameplay::kLineFlagPortraitRight, nullptr, nullptr, &kNarratorFace, 0},

// Choice row with right-aligned price detail
{"Sword", "120g", kNoLine, 12},

// Event trampoline + per-column selection colors
void MyScene::onDialogEvent(void* owner, const gameplay::DialogEvent& event) {
    static_cast<MyScene*>(owner)->handleDialogEvent(event);
}
// in init(): style.portraitsEnabled = false;  // dialogs without faces, scripts untouched
// in init(): style.portraitSize = gfx::DialogPortraitSize::Size32;  // faces authored at 32px
// in draw(): box.draw(renderer, runner);  // no-op with no current line; safe every frame
```

## Agent Constraints

- **Never write dialog code without `-D PIXELROOT32_ENABLE_DIALOG=1`** in the game's
  `platformio.ini`, plus `-D PIXELROOT32_ENABLE_2BPP_SPRITES` / `-D
  PIXELROOT32_ENABLE_4BPP_SPRITES` for any multi-color face. All default off.
- **Never invent APIs.** There is no `DialogBox::setPortrait()`, no `LineKind::Prompt`, no
  `DialogRunner::next()`, no `DialogLine::portraitSize` (the box lives on the style), no
  text interpolation, no per-character reveal, no localization table, no `drawSprite`
  clipping rect, and no 2bpp/4bpp scaler.
- **Scripts are `static const` at namespace scope** and must outlive every runner. Never
  build a script on the stack. Never `new`/`malloc`/`std::string`/`std::function` in
  dialog code paths — callbacks are plain function pointers with a static trampoline.
- **Respect the headless/runner split.** `DialogRunner` takes no `Renderer`/`Font`/pixels;
  `DialogBox` is not a `UIElement` and takes the runner non-const only for `setPageCount()`.
  Page counts come from `pageCountFor()`/`computeLayout()`, never from game-side arithmetic.
- **Keep routing honest.** `kNoChoice` (0xFF) forbids addressing choice index 255;
  `LineKind::Choice` ignores `Advance` (prompts never page); `choiceCount()`/`choice()` use
  compacted visible indices under a filter while `DialogLine::firstChoice`/`choiceCount`
  address the raw table.
- **C++17, `-fno-exceptions`, no RTTI.** PascalCase types, camelCase methods. Doxygen lives
  in headers (`@class`/`@struct` + `@brief`, `@param` per parameter).
- **Cross-reference, do not duplicate**: `pixelroot32-sprite-renderer` (sprite formats,
  palettes, `TextLayout` measuring), `pixelroot32-ui-system` (`UIPanel` vs `DialogBox`),
  `pixelroot32-touch-input` (hit-testing, `event.consume()`), `pixelroot32-scene-manager`
  (`Scene::init`/`draw` order), `pixelroot32-gameplay-framework` (sibling `gameplay::`
  primitives, `ChoiceFilterFn`'s purity rule), `pixelroot32-testing` (Unity + mocks).
