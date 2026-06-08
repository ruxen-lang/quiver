# The DSL — `dyn_text` = reactive, `text` = static

`quiver`'s builder DSL is plain Ruxen closures — **no JSX, no `view!` macro,
no language change.** The builder block runs **once** to construct the node
tree; reactivity granularity comes from which builder method you call.

## The one rule

| You write | Meaning |
|---|---|
| `root.text("Counter")` | **static** — built once, never re-evaluated |
| `root.dyn_text({ \|ui\| … })` | **reactive** — the closure is a *tracking scope*: it subscribes to the states it reads and re-runs **only that node** when they change |
| `root.button("tap", { \|ui\| … })` | static label; the closure is the **click handler** |

## Worked example (the shipped counter)

```ruxen
var app = App.build({ |ui: &var Ui, root: &var Col|
  let count = ui.state(0)
  root.text("Counter")                                          # static
  root.dyn_text({ |ui2: &var Ui| "count: #{count.get(ui2)}" })  # reactive
  root.button("tap", { |ui2: &var Ui|                           # handler
    count.update(ui2, { |c| c + 1 })
  })
})
```

- `App.build` runs the block **once**; `root` is the column under
  construction (a flat arena of nodes — see below).
- The `dyn_text` closure is a tracking scope. When `count` changes, only
  this node recomputes and repaints (`tests/app.rx`, `tests/paint.rx` pin
  this).
- Handler and reactive closures receive the reactive graph as
  `&var Ui` — safe Ruxen has no ambient runtime to reach for
  (docs/REACTIVITY.md).

## Deviations from the design sketch, and why

The sketch spelled this `text "x"` / `text { … }` / `button "tap" do … end`.
Prototyping against ruxen v1 forced three changes, each pinned by a probe:

1. **`text` vs `dyn_text` are separate methods.** Overloading one name on
   `&str` vs closure miscompiles (heap corruption) in ruxen v1.
2. **Stored closures use `{ … }` braces, not `do … end`.** `do…end` blocks
   passed to *free functions* with explicit closure parameters segfault;
   methods accept them, but braces work everywhere and we use them
   uniformly for stored callbacks.
3. **The tree is a flat arena** (parallel arrays in `Col`, node id = index),
   not a recursive `Widget` class: recursive class types crash the v1
   compiler, and `Option[any Fn]` fields miscompile. `-1` sentinels in
   `compute_of` / `handler_of` replace the option types.

None of these changes the model — build once, `dyn_text` closures are the
only things that ever re-run.

## Containers: `row` / `col` and arena nesting

A node can own children. The arena stays flat (no recursive type, per the
landmines): `Col` carries tree-link hashes `parent` / `child_first` /
`child_last` / `child_next` (node-id → node-id; absent ⇒ `-1`), a top-level
root chain (`root_first`/`root_last`), and a build cursor `parent_cursor`.
Node ids remain build-order indices — the "ids ARE node ids" invariant holds.

```ruxen
root.text("title")                  # node 0, a top-level (root) leaf
root.row({ |c: &var Col|            # node 1, a container; cursor moves onto it
  c.text("left")                    # node 2, child of the row
  c.col({ |c2: &var Col|            # node 3, a nested container
    c.text("a")                     # node 4, child of the col
    c.text("b")                     # node 5, child of the col
  })
})
```

`row` (X axis) and `col` (Y axis) push a container node, move the cursor onto
it, run the block, then restore the cursor — so nesting is arbitrary. The
layout pass (`App.arrange`, docs/LAYOUT.md) walks this tree; containers paint
nothing, their children paint themselves.

### Reactive children inside a container — fully supported

`dyn_text` and `button` work as container children, not just at the top level:
the closures they store keep their captured `State` handles, so a reactive
child re-renders on state change and a button child can mutate state, exactly
like a top-level node.

```ruxen
root.row({ |c: &var Col|
  c.dyn_text({ |ui| "n: #{count.get(ui)}" })            # re-renders on change
  c.button("tap", { |ui| count.update(ui, { |x| x + 1 }) })  # mutates state
})
```

This depended on ruxen **Q26** — a capturing closure stored through a
container's `build.(&var *self)` self-reborrow used to lose its captures
(wrong `Int`; SIGSEGV for a captured class handle). Q26 was fixed and
installed (2026-06-08); `tests/nesting.rx` pins it ("a dyn_text child
re-renders when a button child mutates its state"). The targeted-repaint
invariant holds through containers: a child state change flushes and repaints
exactly the subscribed child, never its container or siblings.

### Container styling: padding, background, border

A container can carry a `Style` — built with chainable setters and passed to
`row_styled` / `col_styled` (plain `row`/`col` stay zero-style):

```ruxen
root.row_styled(
  Style.new.pad(8).background(200, 200, 200).border(2, 0, 0, 255).radius(4),
  { |c: &var Col| c.text("x") })
```

| Setter | Meaning |
|---|---|
| `pad(n)` | uniform padding (px) on all four sides |
| `background(r, g, b)` | background fill colour (RGB 0-255) |
| `border(width, r, g, b)` | border stroke width (px) + colour |
| `radius(n)` | corner radius (px) for background **and** border |

Each setter returns the `Style` for chaining; unset fields draw nothing
(`background`/`border` are optional, padding defaults to 0).

- **Layout.** Padding insets the container's children (they lay out inside
  `box - padding`) and grows the container's own box by `2*pad` on each axis.
- **Paint.** The container records a background `fill_round_rect` then a border
  `stroke_round_rect` at its full box (each only if set), *before* its children
  — which paint later in tree order, so leaves land on top. Two display-list
  ops (`op_round_rect` / `op_stroke_rect`) carry a corner radius and (for the
  stroke) a width; the example binary replays them onto canvas's
  `draw_round_rect` / `stroke_round_rect`.

Style lives in parallel `Int` hashes on `Col` (the same flat-arena discipline
as the tree links; absent ⇒ unstyled), not a recursive/`Option`-typed field.
Pinned by `tests/style.rx`. Per-side padding, margins, and gradient/shadow
fills are deferred — additive on the same `Style`, no API break.

### Vertically-scrolling `List`

A `list` is a `col`-like vertical container with a **fixed viewport height**: it
clips its children and scrolls them when the content overflows.

```ruxen
root.list(96, { |c: &var Col|     # 96px viewport
  c.text("row 1")
  c.text("row 2")
  c.text("row 3")
  c.text("row 4")                 # below the viewport — clipped until scrolled
})
```

- **Layout.** The list's box is its viewport height (it does **not** grow to fit
  its children, unlike `row`/`col`). Children lay out in *content space* —
  natural, offset-free positions — and the list records its content height (sum
  of child heights) for clamping. So `app.y_of(child)` is the unscrolled
  position; the on-screen position is `y_of(child) - app.scroll_of(list)`.
- **Scroll state** lives on `App` (runtime, like geometry), keyed by the list's
  node id:

  | Call | Effect |
  |---|---|
  | `app.scroll_to(list_id, offset)` | set offset, clamped to `[0, content − viewport]`, then re-arrange |
  | `app.scroll_by(list_id, dy)` | scroll by a delta (clamped) |
  | `app.scroll_of(list_id)` | current offset |
  | `app.content_height_of(list_id)` | laid-out content height |

- **Paint.** The paint pass became a recursive tree walk (identical output to the
  old flat id loop for un-nested trees — the pins prove it). A list wraps its
  children in `push_clip(viewport)` + `push_translate(0, -offset)` and a closing
  `pop_state` — display-list ops `op_save` / `op_clip` / `op_translate` /
  `op_restore`, replayed onto canvas `save` / `clip_rect` / `translate` /
  `restore`. Off-viewport children still record, but the clip masks them.
- **Scroll input.** Canvas's `Event` enum has no wheel/scroll variant
  (PointerMove/Down/Up, KeyDown, Resize, CloseRequested), so a windowed shell
  drives scrolling by **drag** (pointer down then move adjusts the offset via
  `scroll_by`); the programmatic `scroll_to`/`scroll_by` are the headless-testable
  surface and what the drag handler calls.
- **Hit-testing is scroll/clip-aware.** `hit_test`/`pointer_down` mirror the
  paint walk: a recursive descent carrying an accumulated content offset.
  Entering a list adds its scroll offset (a screen point maps to that level's
  content space as `(x + ox, y + oy)`) and requires the point to be inside the
  list's viewport box (the clip) before any child is reachable. So a click fires
  the on-SCREEN child — not the unscrolled one — and a click outside the
  viewport (or on an item scrolled out of view) hits nothing. Nested lists
  accumulate: offsets sum, viewports intersect. Pinned by
  `tests/list_interaction.rx`.

Viewport height is stored in a per-node `Int` hash on `Col` (flat-arena
discipline; absent ⇒ not a list), keyed by `kind_list`. Pinned by
`tests/list.rx`. Horizontal scroll, virtualization, and scrollbars are deferred
(additive).

### Single-line text `Input`

`input` is a focusable, editable field bound to a reactive `State[String]`:

```ruxen
let name = ui.state_str("")
root.text("name:")
root.input(name, 160)             # 160px-wide field
```

The input node is a tracking scope (its compute reads `name`), so it
re-renders like a `dyn_text` when the value changes — reusing the reactive-child
machinery. The `State[String]` is the source of truth; the user holds it and can
read it.

- **Focus.** `App` tracks `focused` (the focused input's node id, `-1` = none).
  `app.pointer_down(x, y)` over an input sets focus to it (and `hit_test` now
  matches inputs as well as button handlers). `app.focused_node` reads it. Only
  the focused input consumes key input.
- **Key handling.** `app.key_down(code)` (alias `type_char(code)` / `key(code)`,
  the headless edit entry points) routes to the focused input:

  | `code` | Effect |
  |---|---|
  | `32`–`126` (printable ASCII) | insert that char at the caret, caret++ |
  | `key_backspace` (`8`) | delete the char before the caret, caret-- |
  | `key_left` / `key_right` | move the caret (clamped to `[0, len]`) |

  Editing mutates the value `State` → `flush` → repaints just the input;
  `app.caret_of(input_id)` reads the caret index.
- **What `KeyDown` carries.** `canvas`'s `Event.KeyDown(Int)` is an **opaque
  platform keycode** — no char, no modifiers. By SDL convention printable ASCII
  keys equal their ASCII code (so quiver treats `32`–`126` as the char), but
  arrow/edit keys are platform-specific, so quiver defines its OWN logical codes
  (`key_backspace`/`key_left`/`key_right`) and a windowed shell maps the SDL
  keycode onto them (the example binary's `map_platform_key`). quiver stays
  platform-agnostic.
- **Caret + paint.** `App` keeps a per-input `caret` index. Paint draws a field
  box (`fill_round_rect`), the text, and — only when focused — a thin caret
  `fill_rect` at `x + pad_x + caret * char_width` (the char-metric estimate
  layout already uses; real Skia advance widths later).
- **Layout.** Fixed `width` (stored in a per-node `Int` hash on `Col`), one line
  tall (`row_height`).

The value `State[String]` is held in a per-`Col` value pool
(`Array[State[String]]` + a node-id→index hash), the same index-pool pattern as
`computes`. **Edits go through a single `State.update(ui, { |cur| … })`** — a
`peek`-then-`set` pair on one handle would hold two Mutex locks across the frame
and corrupt the value (the "one lock per frame" landmine; see CLAUDE.md).
Pinned by `tests/input.rx`. Selection, clipboard, and multiline are deferred.

## Why it must stay pinned with tests

The static-vs-reactive boundary is the single rule users rely on for
performance and correctness. `tests/app.rx` ("pin: the static/reactive
boundary") asserts that across any number of clicks only the `dyn_text`
node is ever refreshed; `tests/counter.rx` asserts one repaint per tap,
zero on idle frames. If those tests change, the rule changed — that is a
breaking design event, not a refactor.
