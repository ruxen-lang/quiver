# Changelog

All notable changes to `quiver` are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this project adheres
to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Direct public-API unit tests (ruxen Q16 unblocked).** `ruxen test` now
  compiles tests against the package's own + dependency symbols, so previously
  binary-only behavior is pinned directly. Three new headless files
  (+18 tests; suite **124 → 142**, additive — the `app.rx`/`counter.rx`
  static-vs-reactive pins were not touched):
  - `tests/caret_edit.rx` — the text-input editing surface that had **zero**
    direct coverage: forward delete (`key_delete`), Home/End
    (`key_home`/`key_end`), mid-string insert/delete, no-op at the ends, and
    control keys with nothing focused.
  - `tests/resize.rx` — `App.resize` re-layout over a nested tree + a list:
    design-size tracking, geometry stability across a resize, list
    viewport/content height + scroll offset survival, reactive state/cache left
    undisturbed, hit-testing still correct.
  - `tests/overlay.rx` — the overlay/popup primitive
    (`open_overlay`/`close_overlay`/`overlay_open?`/`overlay_owner`/anchor) as a
    reusable primitive in its own right (re-open replaces the owner; an outside
    click is consumed, not passed to the content below) — independent of the
    `select` dropdown that already routes through it.

### Changed
- **DSL: terser builder blocks (no API change).** Inner stored-closure and
  container-block parameters no longer need a type annotation — ruxen infers
  `{ |ui2| … }` / `{ |c| … }` / `{ |c, m, i| … }` from the builder method's
  signature, so the blocks read as plain Ruby blocks instead of
  `{ |ui2: &var Ui| … }`. The annotated form still compiles (every existing test
  proves it); this is purely a cleaner spelling. Applied across `examples/counter`,
  `examples/todo`, `examples/settings`, and the worked snippets in `docs/DSL.md`.
  The outer `App.build({ |ui: &var Ui, root: &var Col| … })` block keeps its
  annotations — a narrow ruxen top-level-inference gap, filed as a Q-candidate in
  the new `docs/decisions/dsl-ergonomics.md` (which also audits the remaining
  language-gated ergonomic wins: non-Copy call-result method args, `&var`
  auto-reborrow).

### Added
- **Windowed event consumption — text input, scroll, resize.** quiver now
  consumes canvas's full event set:
  - `App.text_input(cp)` inserts a Unicode codepoint at the focused input's
    caret (single-lock RMW) — the ONLY text-adding path, fed by
    `Event.TextInput`.
  - `App.key_down` is now **control-only** (fed by `Event.KeyDown`): backspace
    (`key_backspace`), forward delete (`key_delete`), caret move
    (`key_left`/`key_right`/`key_home`/`key_end`). The control key constants are
    canvas's real platform keycodes, passed straight through (no remapping;
    `map_platform_key` removed from the examples). Printable insertion was
    removed from `key_down`.
  - `App.scroll(dx, dy)` routes a wheel event (fed by `Event.Scroll`) to the
    innermost scrollable list under the last hovered point (`pointer_move` now
    records the hover); `+dy` scrolls content toward the top, one row per click;
    a no-op when nothing scrollable is hovered.
  - `App.resize(w, h)` records the design size and re-arranges (fed by
    `Event.Resize`); `design_width`/`design_height` expose it.
  All three examples (counter, settings, todo) route the new events
  (`TextInput`/`KeyDown`/`Scroll`/`Resize` → the matching `App` call, with a
  `_ -> nil` catch-all) and **build against the current canvas**. Pinned by
  `tests/window_events.rx` (10 tests). Suite: **124 passed**; the
  `app.rx`/`counter.rx` pins stayed green and untouched.
- **Structural reactivity — dynamic lists (`list_of`).** The first opt-in
  construct that grows/shrinks the tree in response to state, without breaking
  the build-once rule for everything else. `Col.list_of(model, viewport_h,
  item_builder)` binds to a shared `ListModel` (`ui.list_model`), a `Send` class
  holding the items (labels + done flags) + a reactive `State[Int]` version.
  `model.add/toggle/remove(ui, …)` edit the items and bump the version through a
  single `State.update` (single-lock RMW); the bound list subscribes to it, so a
  mutation marks the list dirty and `App.flush` REBUILDS its child subtree
  wholesale (clear the child chain, build one row per item, re-arrange + repaint
  — just that subtree). Reuses the scroll/clip `list` machinery. Design:
  `docs/decisions/structural-reactivity.md` (an ADR). Pinned by
  `tests/dynamic_list.rx` (7 tests); demoed by the new **`examples/todo`** (a
  real add/toggle/remove app). Suite: **114 passed**; the `app.rx`/`counter.rx`
  pins stayed green and untouched. Deferred (per the ADR): keyed diffing / row
  reuse, per-item reactive widgets, reorder/animation.
- **`examples/settings` — a multi-file example + "how to write a quiver app"
  tutorial.** A reactive settings panel split into `state.rx` (model: option
  lists, fixed rows, label helpers), `views.rx` (the UI as composable view
  functions), and `main.rx` (the shell: builds the app from views, runs the real
  window + event loop, falls back to synthetic events headlessly). Exercises the
  whole widget set idiomatically — `text`/`dyn_text` (the static-vs-reactive rule
  shown side by side), `input`, two `checkbox`es, a `slider`, a theme `select`,
  a scrollable `list`, `row_styled`/`col_styled` cards, and a Save `button` —
  all heavily commented. Builds (`cd examples/settings && ruxen build`) and runs
  windowed (live Skia) and headless (CI-safe). A new "Writing a quiver app"
  section in `docs/DSL.md` walks through the patterns. **Honest reactivity note:
  quiver reacts on content, not structure** — the builder runs once and handlers
  get `&var Ui` (not `&var Col`), so reactively adding/removing nodes is not
  supported; reactive lists / keyed children / remount is the recommended next
  framework feature (documented in the example README + DSL.md).
- **Overlay/popup infra.** `App` gained a single open-overlay slot
  (`overlay_owner` + anchor `overlay_x`/`overlay_y`, -1 = none) with
  `open_overlay(id, x, y)` / `close_overlay` / `overlay_open?`. The overlay
  paints **last** (`paint_all` paints it after the main tree, so it sits on top)
  and is hit-tested **first**: when open, a `pointer_down` inside the popup picks
  an option, a click **outside dismisses it and is consumed** (not passed to the
  content below). Reusable framework primitive for dropdowns/menus/tooltips/
  modals.
- **Dropdown/`Select` widget.** `Col.select(state, options, width)` binds to a
  reactive `State[Int]` (selected index) over an `Array[String]` of options
  (`let choice = ui.state(0); root.select(choice, options, 160)`). The trigger is
  a box showing the current option + a ▾ arrow (reactive — its compute reads the
  index). Clicking the trigger opens the overlay popup anchored below it; clicking
  option k writes the index via a single-lock `state.set` and closes; clicking
  outside closes without change. Paint: trigger (box + text + arrow), then the
  popup panel + option rows (selected highlighted) on top — no new display-list
  op. Options stored in a flat per-`Col` string pool; the index reuses the int
  pool. Pinned by `tests/select.rx` (8 tests, incl. paint-order). Suite:
  **107 passed**; the two `app.rx`/`counter.rx` pins stayed green and untouched.
- **Drag-capture event infra.** `App` gained `pointer_move(x, y)` /
  `pointer_up(x, y)` dispatch and a `captured` node id (like `focused` for
  inputs). `pointer_down` on a draggable widget sets capture; subsequent
  `pointer_move`s drive THAT node regardless of pointer position; `pointer_up`
  clears it. `captured_node` reads it. Reusable by any future drag widget. The
  example binary's event loop now forwards real `PointerMove`/`PointerUp`.
- **Horizontal `Slider` widget.** `Col.slider(state, min, max, width)` binds to
  a reactive `State[Int]` clamped to `[min, max]` (`let vol = ui.state(50)`;
  `root.slider(vol, 0, 100, 200)`). The node is a tracking scope (its compute
  reads the value, so the thumb re-renders on change). A click on the track
  jumps the value to the click fraction; dragging the thumb updates continuously
  via the capture above. Screen x → value:
  `v = min + round((x - track_left) / track_w * (max - min))`, clamped — computed
  from geometry only (no state read), so the write is a **single-lock `set`** (no
  peek+set). Programmatic `drag_to(id, x)` / `set_value(id, v)` / `slider_value(id)`
  for headless use. Paint: a track (`fill_round_rect`), a filled portion to the
  thumb, and the thumb centered at the value's x — no new display-list op. Hit-
  tests correctly inside scrolled lists (routes through the offset/clip-aware
  `hit_tree`; X-axis value math is unaffected by vertical scroll). Pinned by
  `tests/slider.rx` (11 tests). Suite: **99 passed**; the two `app.rx`/`counter.rx`
  pins stayed green and untouched.
- **`Checkbox` widget.** `Col.checkbox(state, label)` binds to a reactive
  `State[Bool]` (`let agree = ui.state_bool(false)`; `root.checkbox(agree,
  "I agree")`). The node is both a tracking scope (its compute reads the bool,
  so it re-renders on toggle) and clickable (its handler toggles via a single
  `state.update(ui, { |b| !b })` — single-lock RMW, not peek+set). A click on
  the box/label region toggles → `flush` → repaints just the checkbox. Paint: a
  bordered square (`stroke_round_rect`), a filled inner mark only when checked
  (`fill_rect`), and the label beside it — no new display-list op. Layout: box +
  gap + char-metric label width, one line tall. Composes inside row/col/list/
  styled containers and hit-tests correctly through list scroll. New
  `Ui.state_bool`. Pinned by `tests/checkbox.rx` (7 tests). Suite: **88 passed**;
  the two `app.rx`/`counter.rx` pins stayed green and untouched.

### Fixed
- **Hit-testing through a scrolled `List`.** `hit_test`/`pointer_down` now apply
  the same transforms the paint walk applies — a recursive walk mirroring
  `paint_tree` that carries an accumulated scroll offset and enforces each
  list's viewport box as a clip. Entering a list subtree adds its scroll offset
  to the accumulated content offset (a screen point maps to content space by
  adding the offset back) and requires the point to be inside the viewport box,
  so: a click fires the on-SCREEN child (not the unscrolled one), a click
  outside the viewport (or on an item scrolled out) fires nothing, and nested
  lists accumulate (sum offsets, intersect clips). Previously a scrolled list of
  buttons/inputs hit the wrong child or a clipped-away one. The non-scrolled
  path is unchanged (the `app.rx`/`counter.rx` pins and existing nesting/row/
  style/list tests stay green). Pinned by `tests/list_interaction.rx` (7 tests).
  Suite: **81 passed**.

### Added
- **Single-line text `Input`.** `Col.input(value, width)` builds a focusable
  field bound to a reactive `State[String]` (`let name = ui.state_str("")`;
  `root.input(name, 160)`). The node is a tracking scope (its compute reads the
  value) so it re-renders like a `dyn_text`. `App` owns the runtime state:
  `focused` (the focused input's node id; `pointer_down` on an input focuses it,
  only the focused input consumes keys) and a per-input `caret` index.
  `App.key_down(code)` / `type_char(code)` / `key(code)` route to the focused
  input — printable ASCII (32-126) inserts at the caret, `key_backspace` deletes
  before it, `key_left`/`key_right` move it (clamped); edits mutate the value →
  `flush` → repaint just the input. `canvas`'s `Event.KeyDown(Int)` carries an
  opaque platform keycode (no char/modifiers), so quiver defines logical key
  codes and the example binary maps SDL keycodes onto them. Paint: a field box +
  the text + a thin caret rect (only when focused) at the char-metric caret x.
  Pinned by `tests/input.rx` (9 tests). Suite: **74 passed**; the two
  `app.rx`/`counter.rx` pins stayed green and untouched.
- **Vertically-scrolling `List`.** `Col.list(viewport_h, build)` builds a
  `Col`-like container with a FIXED viewport height that clips its children and
  scrolls them by an offset. Layout: the list's box is its viewport (it does
  NOT grow to fit children); children lay out in content space (offset-free) and
  the content height is tracked for clamping. Scroll is runtime state on `App`:
  `scroll_to(list_id, offset)` / `scroll_by(list_id, dy)` clamp to
  `[0, content - viewport]` and re-arrange; `scroll_of` / `content_height_of`
  expose it (all headless-testable). Paint (recursive walk now) wraps a list's
  children in `push_clip(viewport)` + `push_translate(0, -offset)` and a closing
  `pop_state`, so off-viewport children are masked and content is scrolled —
  four new display-list ops `op_save` / `op_clip` / `op_translate` /
  `op_restore`, replayed onto canvas `save` / `clip_rect` / `translate` /
  `restore` in the example binary. Canvas's `Event` has no wheel variant, so
  scroll input is **drag-to-scroll** (pointer down→move) plus the programmatic
  API. Pinned by `tests/list.rx` (6 tests). Suite: **65 passed**; the two
  `app.rx`/`counter.rx` pins stayed green and untouched.
- **Container styling: padding, background, border.** A small `Style` value
  (`Style.new.pad(8).background(r,g,b).border(w,r,g,b).radius(n)`, chainable)
  applied to a container via `Col.row_styled(style, build)` /
  `col_styled(style, build)`. Layout: padding insets a container's children and
  grows its box by `2*pad`. Paint: the container records a background
  `fill_round_rect` then a border `stroke_round_rect` at its full box (each only
  if set, optional corner radius), under its children so leaves land on top.
  Two new display-list ops `op_round_rect` / `op_stroke_rect` with `rad_at` /
  `sw_at` (corner radius + stroke width) on `RecordingSurface`; the example
  binary's `replay` maps them to canvas `draw_round_rect` / `stroke_round_rect`.
  Style lives in parallel Int hashes on `Col` (flat-arena discipline; absent ⇒
  unstyled). Pinned by `tests/style.rx` (5 tests). Suite: **59 passed**; the two
  `app.rx`/`counter.rx` pins stayed green and untouched.
- **ADR: layout engine — own simple flex vs Yoga (`docs/LAYOUT.md`).**
  Decided **own simple main-axis stacking/flex pass in safe Ruxen**, shipped
  incrementally (intrinsic stacking now; grow/shrink/wrap later, additively).
  Yoga (C) rejected: it forces an FFI/C dependency into L2's 100%-safe charter,
  couples an L2 layout concern to the L1 boundary, and is overkill for the
  Row+Col+nesting slice. Closes the layout-engine open decision in `ROADMAP.md`.
- **Arena nesting — a node can own children.** `Col` gained flat parallel
  tree-link hashes (`parent` / `child_first` / `child_last` / `child_next`,
  node-id → node-id, `-1`/absent ⇒ none) plus a top-level root chain
  (`root_first`/`root_last`) and a build cursor. No recursive type anywhere;
  node id is still build order. New arena accessors: `parent_of`,
  `first_child_of`, `next_sibling_of`, `root_head`, `container_at`. Pinned by
  `tests/nesting.rx`.
- **`Row` / `Col` containers + nested own-flex layout pass.** `Col.row { |c| … }`
  and `Col.col { |c| … }` build container nodes whose block-built nodes become
  their children. `App.arrange` now recursively walks the nested arena: a flat
  pre-pass measures each leaf's intrinsic text width once, then a measure/place
  recursion stacks each container's children along its axis (`Row` = X,
  `Col` = Y) and the top level as an implicit column — writing the same
  `geom_x/y/w/h` the paint/hit-test passes already read. Containers paint
  nothing (their children paint themselves). Pinned by `tests/row.rx`.
- **Reactive children inside containers.** `dyn_text` and `button` work as
  `row`/`col` children: a reactive child re-renders on state change, a button
  child mutates state, and the targeted-repaint invariant holds (a child state
  change repaints exactly the subscribed child, not its container or siblings).
  Pinned by `tests/nesting.rx` ("a dyn_text child re-renders when a button
  child mutates its state"). Suite: **54 passed, 0 pending** (the two
  `app.rx`/`counter.rx` pins stayed green and untouched).

### Changed
- **`Event.KeyDown` no longer inserts characters** (control-only); typing now
  arrives via `Event.TextInput` → `App.text_input`. The `App` `key_*` constants
  are canvas's real control keycodes (Backspace 8, Delete 127, arrows / Home /
  End), and the examples' `map_platform_key` shim was removed.
- Dropped the defensive empty-hash `size as Int > 0` / `key?` guards from the
  arena and geometry accessors (`Col.first_child_of`/`next_sibling_of`/
  `parent_of`/`append_child`, `App.x_of`/`y_of`/`w_of`/`h_of`/`text_of`): ruxen
  **Q25** is fixed, so empty-hash `get` is safe and these are plain
  `match h.get(i)` again.

### Fixed (ruxen, now resolved)
- **Q26** — capturing closures stored through a container's `&var *self`
  reborrow now keep their captures, unblocking reactive children in containers.
- **Q25** — `Hash.key?`/`get` on an empty hash no longer segfault; `&Hash`/
  `&Set` params reject at compile time. Repros kept for history under
  `tmp/test-cache/`.

- **Milestone 1 (L1 integration): the counter runs as a live window.**
  `examples/counter` opens a scaled SDL2 window over canvas's Skia `Canvas`
  and drives repaint from the real engine event stream (`PointerDown` →
  dispatch → flush → repaint, `CloseRequested` → quit), falling back to
  synthetic taps headlessly so CI still runs the whole pipeline.
- **Structured paint recording** on `RecordingSurface`: alongside the string
  command log, each primitive is recorded as numeric fields
  (`op_at`/`x_at`/`y_at`/`w_at`/`h_at`/`r_at`/`g_at`/`b_at`/`text_at`) plus
  `op_clear`/`op_rect`/`op_text` codes and a `reset`. This is the canvas-free
  bridge an app binary `replay`s onto a real backend — kept as the single
  `PaintSurface` impl because ruxen v1 only resolves quiver's generic paint
  pass with one implementor. Pinned by 2 new tests (42 total).
- **Milestone 1 (L2 side): the counter core slice**, fully implemented and
  pinned by 37 headless tests:
  - Reactive core: `Ui` subscription graph + `State[T]` handles
    (`get`/`peek`/`set`/`update`) with values behind `SharedSync[Mutex[T]]`.
  - Builder DSL: `Col` with `text` (static), `dyn_text` (tracking scope),
    `button` (stored handler); flat-arena tree.
  - `App`: build-once mount, targeted refresh (`flush` returns exactly the
    dirty node ids), column layout (`arrange`), hit-testing and
    `pointer_down` dispatch.
  - Paint: `PaintSurface` mixin, `RecordingSurface` test double,
    `paint_all` / `paint_dirty` (targeted repaint), `first_frame`/`frame`
    drivers.
  - `examples/counter` binary package: the milestone demo running the real
    pipeline headlessly (window path activates with canvas L1).
- Initial package scaffold: `Ruxen.toml` (library + `canvas` path dependency),
  API skeleton, and full design docs (`DESIGN`, `ARCHITECTURE`, `REACTIVITY`,
  `DSL`, `ROADMAP`).

### Fixed
- Targeted repaint erases the full (monotonic) row span and `flush`
  re-arranges geometry, so labels that grow or shrink can never leave ghost
  pixels (pinned by the 9 → 10 transition test).
- Tracking scopes drop all previous subscriptions before re-running, so
  conditional reads cannot leave stale subscriptions (over-notification).
- `RecordingSurface.fill_rect` records its color, so erase vs. button
  background are distinguishable in tests.

### Changed
- The `canvas` (L1) dependency now resolves from its published git repository
  (`https://github.com/ruxen-lang/canvas.git`, `master`) instead of a local
  `../canvas` path, so anyone depending on `quiver` pulls L1 transitively
  without vendoring or path-linking it by hand.
- Public API renamed/reshaped from the design sketch where ruxen v1 forced
  it: `State[T]` (not `Signal[T]` — std collision), explicit `&var Ui`
  parameter (no globals in safe Ruxen), `dyn_text` as its own method (the
  `&str`-vs-closure overload miscompiles), braces closures for stored
  callbacks. Rationale recorded in `docs/DSL.md` / `docs/REACTIVITY.md`.
