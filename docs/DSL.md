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
  root.text("Counter")                              # static
  root.dyn_text({ |ui2| "count: #{count.get(ui2)}" })  # reactive
  root.button("tap", { |ui2|                        # handler
    count.update(ui2, { |c| c + 1 })
  })
})
```

The **outer `App.build` block keeps its param types** (`{ |ui: &var Ui, root:
&var Col| … }`) — that one block is the app's entry point and its types are not
inferred today (a ruxen inference gap; see
`docs/decisions/dsl-ergonomics.md`). Every **inner** stored-closure / container
block, though, drops them — `{ |ui2| … }`, `{ |c| … }` — because ruxen infers
the parameter type from the builder method's signature. Use the terse form for
inner blocks; it reads as a plain Ruby block. (The annotated form still
compiles, so existing code is unaffected.)

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
root.text("title")        # node 0, a top-level (root) leaf
root.row({ |c|            # node 1, a container; cursor moves onto it
  c.text("left")          # node 2, child of the row
  c.col({ |c2|            # node 3, a nested container
    c.text("a")           # node 4, child of the col
    c.text("b")           # node 5, child of the col
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
root.row({ |c|
  c.dyn_text({ |ui| "n: #{count.get(ui)}" })                 # re-renders on change
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
  { |c| c.text("x") })
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
root.list(96, { |c|               # 96px viewport
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
- **Scroll input.** A windowed shell forwards canvas's `Event.Scroll(dx, dy)`
  (mouse wheel) to `app.scroll(dx, dy)`, which routes the wheel to the innermost
  scrollable list under the last hovered point (`pointer_move` records the
  hover) and `scroll_by`s it one row per click — `+dy` (wheel up/away) scrolls
  content toward the top (decreases the offset). The programmatic
  `scroll_to`/`scroll_by` remain the direct, headless-testable surface.
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
- **Typing.** `app.text_input(cp)` (alias `type_char(cp)`) inserts the Unicode
  codepoint `cp` at the focused input's caret (single-lock RMW) and advances it.
  This is the ONLY path that adds text — see "Windowed event contract" below.
- **Editing / caret.** `app.key_down(code)` (alias `key(code)`) routes a CONTROL
  key to the focused input:

  | `code` | Effect |
  |---|---|
  | `key_backspace` | delete the char before the caret, caret-- |
  | `key_delete` | delete the char at the caret (caret stays) |
  | `key_left` / `key_right` | move the caret (clamped to `[0, len]`) |
  | `key_home` / `key_end` | caret to start / end |

  `key_down` no longer inserts printable characters (that's `text_input`).
  Editing mutates the value `State` → `flush` → repaints just the input;
  `app.caret_of(input_id)` reads the caret index.
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

### `Checkbox`

`checkbox` is a clickable widget bound to a reactive `State[Bool]`:

```ruxen
let agree = ui.state_bool(false)
root.checkbox(agree, "I agree")
```

The node is **both** a tracking scope and clickable:
- its **compute** reads the bool (`state.get(ui)`), so the node subscribes and
  re-renders on toggle — reusing the reactive-child machinery;
- its **handler** toggles the bool with a single `state.update(ui, { |b| !b })`.
  This is a **single-lock read-modify-write** — never `peek` then `set`, which
  would hold two Mutex locks across the frame and corrupt the value (the "one
  lock per frame" landmine; see CLAUDE.md).

A click anywhere on the box+label region runs the handler → toggle → `flush` →
repaint just the checkbox. It composes inside `row`/`col`/`list`/styled
containers and hit-tests correctly through list scroll (it's a handler node, so
the scroll/clip-aware `hit_test` already covers it). `app.checkbox_checked(id)`
reads the live checked state (a lone `peek`, no `set` in the frame — safe).

- **Paint.** A bordered square (`stroke_round_rect`), a **filled inner mark**
  (`fill_rect`, inset 3px) only when checked, and the label beside it after a
  `pad_x` gap. No new display-list op — a filled inner square is the mark this
  round (a real check glyph waits for Skia text/paths).
- **Layout.** Leading pad + box (`check_box_size`) + gap + char-metric label
  width + trailing pad, one line tall.

The `State[Bool]` is held in a per-`Col` bool pool (`Array[State[Bool]]` + a
node-id→index hash), the same index-pool pattern as inputs. `Ui.state_bool`
mints the handle. Pinned by `tests/checkbox.rx`. A toggle/switch is the same
node with a pill-shaped paint — deferred (additive).

### Pointer dispatch + drag capture

`App` dispatches three pointer events, all returning the hit/captured node id
(or `-1`), plus key dispatch:

| Call | Effect |
|---|---|
| `pointer_down(x, y)` | hit-test; focus an input, **capture** + set a slider, or run a button/checkbox handler |
| `pointer_move(x, y)` | if a node is **captured**, drive it from `x` regardless of where the pointer is; else no-op |
| `pointer_up(x, y)` | release the capture |

`App` tracks a `captured` node id (like `focused` for inputs); `captured_node`
reads it. Capture is the reusable drag primitive: a widget that needs
press-drag-release (today the slider) captures on `pointer_down`, is driven by
every `pointer_move` until `pointer_up`. The windowed shell forwards real
`PointerMove`/`PointerUp` into these, the same way it forwards scroll/keys.

### `Slider`

`slider` is a draggable horizontal control bound to a reactive `State[Int]`
clamped to `[min, max]`:

```ruxen
let vol = ui.state(50)
root.slider(vol, 0, 100, 200)     # min 0, max 100, 200px wide
```

The node is a tracking scope (its compute reads the value), so the thumb
re-renders on change. Interaction:
- a **click on the track** jumps the value to the click fraction;
- **dragging** the thumb updates continuously via the capture above — a
  `pointer_move` anywhere (even off the slider) drives the captured slider.

Screen x → value: `track_left = x + thumb/2`, `track_w = width - thumb`, and
`v = min + round((px - track_left) / track_w * (max - min))`, clamped to
`[min, max]`. The value is computed from **geometry only — no state read** — so
writing it is a **single-lock `state.set(ui, v)`** (never `peek` then `set`,
which would collide on the Mutex; see CLAUDE.md). value → thumb center x inverts
the same map (`slider_thumb_center_x`, used by paint). Programmatic
`drag_to(id, x)` / `set_value(id, v)` / `slider_value(id)` are the
headless-testable surface.

It **hit-tests correctly inside scrolled lists**: `hit_tree` matches sliders and
already accounts for the list scroll offset/clip, and the X-axis value math is
unaffected by vertical scroll (capture stores the id; `pointer_move` maps screen
x against the slider's content-space x, identical for vertical-only scroll).

- **Paint** (no new op): a track bar (`fill_round_rect`), a filled portion from
  the track left to the thumb, and the thumb (a rounded square) centered at the
  value's x.
- **Layout**: fixed `width`, one row tall.

The `State[Int]` is held in a per-`Col` int pool (`Array[State[Int]]` + a
node-id→index hash) alongside per-slider `min`/`max`/`width`. Pinned by
`tests/slider.rx`. Vertical, range (two-thumb), and float sliders are deferred
(additive).

### Overlay / popup primitive

`App` has a single **overlay slot** — a node rendered above all normal content
that captures interaction. It is the reusable basis for dropdowns, context
menus, tooltips, and modals.

| Call | Effect |
|---|---|
| `open_overlay(owner, x, y)` | open the overlay owned by `owner`, anchored at screen `(x, y)` |
| `close_overlay` | close it |
| `overlay_open?` | whether one is open |

`overlay_owner` is the open overlay's owning node (-1 = none); `overlay_x` /
`overlay_y` its anchor. Two rules make it behave like a real popup:

- **Paint on top.** `paint_all` paints the main tree first, then — if open — the
  overlay LAST, so it sits above everything.
- **Hit-test first + click-outside-dismiss.** While an overlay is open,
  `pointer_down` routes to the overlay *before* the normal tree: a click inside
  the popup acts on it, a click **outside dismisses the overlay and is
  consumed** (it does NOT fall through to the content below).

One overlay at a time this round (no nested overlays).

### Dropdown / `Select`

`select` is a dropdown bound to a reactive `State[Int]` (the selected index)
over an `Array[String]` of options:

```ruxen
let choice = ui.state(0)
root.select(choice, options, 160)     # 160px trigger
```

- The **trigger** is a button-like box showing the current option + a ▾ arrow.
  Its compute reads the index, so the trigger reflects the selection reactively;
  `app.select_value(id)` / `app.select_text(id)` read the index / current option.
- **Clicking the trigger** opens the overlay popup anchored just below it (a
  panel + one row per option, the selected row highlighted).
- **Clicking an option** writes the index with a single-lock `state.set(ui, k)`
  (the index comes from the click row geometry — no state read, so no peek+set)
  and closes the overlay; the trigger then shows the new option.
- **Clicking outside** closes the overlay without changing the value.

Options are stored in a flat per-`Col` string pool (one array for all selects, a
per-select `(start, count)` slice); the selected index reuses the int pool.
Paint reuses existing ops (`fill_round_rect`/`fill_rect`/`draw_text`) — no new
op. Pinned by `tests/select.rx` (including a paint-order assertion that the popup
panel is recorded after the trigger box). Multi-select, typeahead, nested
overlays, and a scrollable popup (for very long option lists) are deferred.

## Why it must stay pinned with tests

The static-vs-reactive boundary is the single rule users rely on for
performance and correctness. `tests/app.rx` ("pin: the static/reactive
boundary") asserts that across any number of clicks only the `dyn_text`
node is ever refreshed; `tests/counter.rx` asserts one repaint per tap,
zero on idle frames. If those tests change, the rule changed — that is a
breaking design event, not a refactor.

## Writing a quiver app

The worked, multi-file reference is **`examples/settings`** — a reactive
settings panel exercising the whole widget set, organised the way a real app
is (model / views / shell), and heavily commented. Read it alongside this doc.
The patterns it teaches:

- **One build block, run once.** `App.build({ |ui, root| … })` constructs the
  tree once; quiver reacts by re-running `dyn_text` content closures of existing
  nodes, never by rebuilding. Structure is fixed; **values** are reactive.
- **Views are functions.** A view is a free function `view(root: &var Col, …)`
  that appends a subtree. Compose by calling them in order from the build block.
  (Files in a binary package flat-merge — no `use` between them.)
- **State by reference, deref'd to a local.** `State[T]` is non-`Copy`; a view
  takes `&State[T]` and does `let v = *handle`. Each consumer (a builder taking
  it by value, or a capturing closure) needs its **own** local copy.
- **RMW is a single `update`.** Editing state goes through one
  `state.update(ui, { |v| … })` / `set` — never `peek` then `set` (two locks per
  frame on the same mutex corrupt the value). The widgets do this for you.
- **The shell stays in the binary.** The window + event loop + paint `replay`
  (the canvas-facing glue) live in the example's `main.rx`; quiver itself is
  platform-free. Forward canvas events into `App`'s dispatch (see below); flush
  + repaint only when the dirty set is non-empty.

### Windowed event contract

A windowed shell forwards each `canvas` `Event` to the matching `App` method —
quiver consumes the events, the shell never interprets them:

| `canvas` event | `App` call | what it does |
|---|---|---|
| `PointerDown(x, y)` | `pointer_down(x, y)` | hit-test → focus / click / open / capture |
| `PointerMove(x, y)` | `pointer_move(x, y)` | record hover + drive a captured drag |
| `PointerUp(x, y)` | `pointer_up(x, y)` | release the drag capture |
| `TextInput(cp)` | `text_input(cp)` | **insert** the Unicode codepoint at the caret |
| `KeyDown(k)` | `key_down(k)` | **control only** — backspace/delete edit, arrows/home/end move the caret |
| `Scroll(dx, dy)` | `scroll(dx, dy)` | wheel → scroll the innermost hovered list |
| `Resize(w, h)` | `resize(w, h)` | record design size + re-arrange |
| `CloseRequested` | (quit the loop) | |

The split that matters: **`TextInput` is the only path that adds text** (it
carries the OS-composed, shift/layout-correct codepoint — `'A'`=65, `'é'`=233),
and **`KeyDown` is control-only** (real platform keycodes for Backspace/Delete/
arrows/Home/End, passed straight through — no remapping). Keep a `_ -> nil`
catch-all so new event variants don't break the match.

### Reactivity model: content + (opt-in) structure

quiver reacts on **content** by default: a state change re-runs exactly the
`dyn_text` closures that read it. The builder runs once and the tree is
otherwise fixed — `text`/`dyn_text`/`button`/`input`/… are built once. Most
apps (forms, dashboards, settings) react on *values* over a *fixed layout*.

For dynamic UIs (a todo list, a feed — anything that **adds/removes rows**),
there is one **opt-in structural-reactivity** construct: `list_of` bound to a
shared `ListModel`. See "Dynamic lists (`list_of`)" below and
`docs/decisions/structural-reactivity.md`. It does **not** change the
build-once rule for everything else — `list_of` is the only node whose children
are rebuilt.

### Dynamic lists (`list_of`)

`list_of` is a scrolling list whose children are **rebuilt from a `ListModel`**
whenever the model changes — the structural-reactivity primitive.

```ruxen
let todos = ui.list_model                         # shared, mutable item model
root.button("Add", { |u| todos.add(u, "task") })  # handler CAPTURES the model
root.list_of(todos, row_height() * 6, { |c, m, i| # one row per item
  c.text(m.row_text(i))
})
```

- **`ui.list_model`** mints a `ListModel` — a `Send` class holding the items
  (labels + done flags) plus a reactive `State[Int]` **version**.
- **`model.add(ui, …)` / `toggle(ui, k)` / `remove(ui, k)`** edit the items and
  bump the version through one `State.update` (single-lock RMW). The `list_of`
  node subscribes to that version, so a mutation marks it dirty.
- On `flush`, the dirty `list_of` **rebuilds its child subtree wholesale**:
  clear its child chain, build one row per current item, re-`arrange` + repaint
  — just that subtree (the rest of the tree is untouched). It reuses the
  `list` viewport/clip/scroll, so it scrolls and hit-tests through scroll.

Two handle rules (a `ListModel` is a non-Copy handle; a *method call* consumes
it, a *closure capture* does not):

1. **Capture in handlers first, pass to `list_of` last** — `let todos =
   ui.list_model`; capture it in the button handlers (non-consuming), then move
   it into `list_of` (consuming) last.
2. **Re-fetch for ad-hoc mutations** — outside a captured handler, get a fresh
   copy with `app.list_model_of(list_node_id)` per mutation (all share the
   items).

Worked example: **`examples/todo`**. Pinned by `tests/dynamic_list.rx`. Keyed
diffing (row reuse), per-item reactive widgets, and reordering/animation are
deferred — see the ADR.
