# Changelog

All notable changes to `quiver` are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this project adheres
to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **F4 — animation engine** (`docs/decisions/animation.md`): explicit tweens +
  colour implicit transitions, stepped deterministically off the F2 clock seam
  (`App.tick`). Default-inert — with no live animation every paint pin is
  byte-identical.
  - **Animatable set** (after auditing what paint consumes — `PaintSurface` is
    Ints + &String, no opacity op): **geometry via a paint-time offset** (a
    translate that shifts where a node draws — NOT a layout input, so no
    re-arrange and siblings never move) + **colour** (RGB lerp into the existing
    fill/text ops). Size/opacity deferred (filed).
  - **Fixed-point curves** — linear / ease_in / ease_out / ease_in_out, per-mille
    Int math (`anim_scale = 1000`); `ease` + `anim_lerp`. Clamp-to-end on
    retirement → byte-exact landing, fully deterministic (no Float).
  - **Registry** — flat **slot-keyed Hashes** + a free-list (`Array.insert` is
    insert-and-shift in ruxen v1, not replace — so mutable rows live in Hashes
    that update in place; `anim_count` is the slot high-water). A retired slot is
    reused (row count never grows unbounded). Completion callbacks in an
    `anim_callbacks` pool (never `Option[any Fn]`).
  - **API** — `tween_offset` / `tween_color` (+ `_done` completion variants),
    `animating?` / `animating_node?`, `transition_color` + `set_color` (colour
    implicit transitions Tier-1). App-level (animations are runtime state, not
    tree structure). A new tween on a node+channel supersedes the live one.
  - **Stepping** — `App.tick` advances every live tween from the one clock delta
    (fling + tween coexist); a node is marked dirty ONLY when its resolved value
    actually moved (dirty-set exactness — non-animating siblings never repaint).
  - **Paint** — colour-consuming draws read the override (default ⇒ intrinsic
    colour); an animated offset wraps the subtree in a `push_translate` /
    `pop_state` bracket (absent ⇒ no bracket). `paint_tree` split into
    `paint_tree` (offset bracket) + `paint_subtree` (body).
  - **Showcase** — `examples/settings`' Save button colour-pulses on click (flash
    green-highlight, fade to grey via `tween_color`; the window loop already
    `tick`s); pinned by an exact-mirror test. The settings shell also wires F3's
    clipboard seam to canvas's `Window.clipboard_text` / `set_clipboard_text`.
  - Suite **220 → 236** (`tests/animation.rx`, 16); all three examples build.
- **F3 — production text editing** (`docs/decisions/text-editing.md`): the
  single-line `input` grew a full editing surface, all App-level runtime state
  reached through App entry points or injected seams (quiver stays canvas-free).
  Every piece is default-inert — a plain `input` is byte-identical to before.
  - **Selection (anchor + caret).** Per-input `anchor` + `caret`; selection =
    their `[min, max]`. Pointer drag-select reuses the `captured` drag primitive
    (generalised from sliders to inputs); keyboard shift-extend keeps the anchor.
    Caret/selection x come from **measured prefixes** (`prefix_width`, the measure
    seam; char-metric fallback — default caret position unchanged). New readers:
    `anchor_of` / `sel_start_of` / `sel_end_of` / `has_selection?` /
    `selected_text` / `select_all` / `caret_x_of` / `place_caret`.
  - **Modifier keys.** `key_down_mod(code, shift, ctrl)` — shift extends the
    selection, ctrl runs clipboard/undo shortcuts (c/x/v/z, a). **`key_down` (and
    the `key` alias) now forward to `key_down_mod(code, false, false)`** — fully
    back-compat. canvas delivers no key modifiers yet (filed gap); shells pass
    `false`/`false`, so keyboard selection-extend + ctrl-shortcuts are reachable
    via this API + headless tests but dormant on a real window.
  - **Clipboard seam.** `set_clipboard(get, set)` (the measure/clock 0-or-1-pool
    pattern) + App-level `copy` / `cut` / `paste` on the focused selection
    (single-lock RMW). Headless backend: **`ClipboardCell`** (a shared `Send`
    String cell, like `RecordingSurface` for paint — a bare `Mutex[String]`
    corrupts through a closure on this ruxen build). `clipboarded?` reports state.
  - **IME composition.** `text_editing(start, len, text)` stores marked text;
    paint renders it inline + underlined at the caret; the committing `TextInput`
    clears the mark and inserts (mirrors canvas/SDL's `TextEditing` →
    `composition_text` → `TextInput` sequence). Reader: `marked_text_of`.
  - **Multi-line `textarea`.** `root.textarea(state, width, rows)` — one
    `State[String]` with `\n` separators, flat caret index, `rows * row_height`
    box. Up/down cross lines preserving the column; home/end act on the caret's
    line. Reuses all single-line machinery. New `multiline_at` / `rows_at` on
    `Col`.
  - **Undo.** Per-input bounded `(value, caret)` snapshot ring with clock-distance
    coalescing (rapid typing = one entry); `undo` restores the exact prior state;
    a redo stack is kept. `undo_group_ms` tuning constant.
  - Suite **200 → 220** (`tests/text_editing.rx`, 20); all three examples build.
    The single-line `input`/`caret_edit` pins stay green unchanged.
  - **Toolchain note (filed):** a non-Copy class CALL-RESULT passed directly as a
    by-value method argument corrupts (already a known landmine) — `cut`'s
    `clip_write(selected_cached(i))` hit it; fixed by `let`-binding. AND a bare
    `Mutex[String]` set/get THROUGH A CLOSURE corrupts (new observation, 2026-06-11)
    — `ClipboardCell` (a `Send` class owning the `String`) is the sound workaround.
- **F2 gestures & scrolling (2/2) — scrollbars + horizontal scroll**
  (`docs/LAYOUT.md` F2), completing Phase 1.
  - **Scrollbars (paint-only).** When a list's content overflows its viewport, a
    rounded thumb is painted on the trailing edge (right for a vlist, bottom for
    an hlist), AFTER the clip/translate restore so it sits in viewport space and
    does not scroll with the content. Geometry from the ratios:
    `thumb_len = viewport² / content` (min-clamped), `thumb_pos = start +
    (viewport − len) * scroll / max_scroll`. No fade/animation. New
    `scrollbar_visible?` / `scrollbar_thumb_h` (length) / `scrollbar_thumb_y`
    (position) on `App`; `paint_scrollbar` in the pass. Pinned by
    `tests/scrollbar.rx` (7). **Design event:** `tests/list.rx`'s two
    "restore is last" assertions became "restore is second-to-last, scrollbar
    round-rect is last" — the overflowing 4-item list now draws a thumb (noted
    inline).
  - **Horizontal scroll (`hlist`).** `root.hlist(viewport_w)` reuses `kind_list`
    + a per-node `horizontal` flag, so all the clip/scroll/hit/scrollbar/fling
    machinery is shared on an axis-tagged offset. Children stack + clip + scroll
    on X; the box is the fixed viewport width; paint translates on X; hit-test
    and `list_under` accumulate the X offset; the wheel scrolls on `dx` (dy
    fallback); drag-scroll tracks the X position; the scrollbar is a horizontal
    bar. The vertical path is byte-identical (axis generalization defaults to Y).
    Pinned by `tests/hlist.rx` (7) — layout, content extent, X wheel, X
    clip/translate, X-scroll hit-test, horizontal scrollbar, X drag.
  - Suite **186 → 200**; all three examples build. **Phase 1 (F1 + F2) complete.**
- **F2 gestures & scrolling (1/2) — pixel scroll, fling momentum, tap vs
  long-press** (`docs/LAYOUT.md` F2). Built on an injected clock seam that
  mirrors `set_measure`, so all of it is deterministic and headless-pinnable.
  - **Pixel-based wheel scroll.** `App.scroll(dx, dy)` now moves `scroll_line`
    px per wheel click (default `row_height`; `set_scroll_line(px)` overrides,
    `scroll_line_px` reads it). The App boundary stays Int (the shell quantizes
    canvas's Float32 wheel deltas to Int clicks); the line factor multiplies
    before clamping. `scroll_to`/`scroll_by` already took pixel offsets — unchanged.
  - **Clock seam.** `App.set_clock({ || Int })` injects a monotonic-ms timebase
    (0-or-1-entry pool + `has_clock`, not `Option[any Fn]`). With none injected,
    `now` returns a frame counter that `tick` advances — fully deterministic in
    tests. `clocked?` reports which is active.
  - **Drag-scroll + fling.** A press that lands in a scrollable list's viewport
    (not on an inner widget) captures it; `pointer_move` scrolls it by the drag
    delta and tracks velocity over the clock; `pointer_up` seeds a fling. The new
    per-frame `App.tick` (the shell loop calls it) steps each active fling
    (`offset += v`, `v *= 0.92`) until below threshold or clamped at an end, and
    returns whether anything changed (so the shell keeps drawing). Generalised
    pointer capture: `scroll_captured` (a list) is distinct from `captured` (a
    slider).
  - **Tap vs long-press recognizers.** A press fires button/checkbox handlers
    immediately (a click IS a tap — unchanged). A new `c.long_press({ |ui| … })`
    DSL modifier attaches a STORED handler to the LAST-built node (the one
    post-hoc modifier — the arena is push-only, so a gesture handler attaches by
    node id; lives in a `long_press_handlers` pool). `tick` fires it once when the
    press is held past `long_press_ms` without moving past `gesture_slop`; a
    move-past-slop or early up cancels it. Long-press nodes are hit-testable
    (`interactive_at`).
  - Pins: `tests/scroll_gestures.rx` (10 — pixel scroll, clock default/injected,
    drag-scroll, deterministic fling decay + end-clamp, tap, long-press fire +
    slop-cancel). Suite **176 → 186**; all three examples build.
- **F1 layout completeness — flex grow, alignment, gap, per-side padding,
  stack** (the deferred half of the own-flex ADR, `docs/LAYOUT.md` F1 addendum).
  All additive on the existing two-phase `measure`/`place` recursion in
  `App.arrange`, writing the same `geom_x/y/w/h` the paint + hit-test passes read;
  every new knob defaults to the v1 behaviour, so existing geometry pins are
  byte-identical.
  - **DSL shape: extend `Style`** (not new yield shapes). New chainable setters
    `gap(n)`, `justify(j)`, `align(a)`, `pad_sides(l,t,r,b)`, `grow(n)`,
    `offset(l,t)`; `pad(n)` now sets all four per-side fields equal (back-compat).
    Consumed through `row_styled`/`col_styled`/`stack_styled`. New `Int` constants
    `just_start/center/end/space_between/space_around`,
    `align_start/center/end/stretch`. Stored in parallel `Int` hashes on `Col`
    (the flat-arena discipline; absent ⇒ default).
  - **Grow** — a child's main-axis weight claims a proportional share of the
    container's leftover main space (last grown child absorbs integer rounding so
    the row exactly fills). Scoped to styled containers; leaf-grow deferred (wrap
    a leaf in a 1-child styled container today).
  - **Main-axis `justify`** — start / center / end / space_between / space_around,
    over any leftover main space (inert when grow consumed it).
  - **Cross-axis `align`** — start / center / end / stretch (stretch resizes the
    child to the container's inner cross extent).
  - **Gap** — px between adjacent main-axis children (accounted in the box +
    content-height); replaces hand-inserted spacers.
  - **Per-side padding** — `pad_l/t/r/b` insets children + grows the box per side.
  - **`stack` / positioned** (`kind_stack`, `root.stack` / `stack_styled`) — a
    container whose children all occupy its box (size = max on both axes), each
    optionally offset by `Style.offset(left, top)`; z-order = build order.
    Children PAINT in build order (bottom→top, the existing walk) and HIT-TEST in
    REVERSE (topmost wins — `hit_stack_children`).
  - Pins: `tests/flex.rx` (12 — padding/gap/align/justify/grow geometry + a
    hit-test pin), `tests/stack.rx` (7 — layout/offset/paint-order/reverse
    hit-test). Suite **157 → 176**; all three examples build.

### Changed
- **Owned-`String` label params (drop the last 5 `String.from` sites).** Post-Q38
  a bare/interpolated literal IS an owned `String`, so the label-taking builders
  `Col.text` / `Col.button` / `Col.checkbox` and `ListModel.seed` / `add` now take
  `label: String` (by value) instead of `label: &str` + a `String.from(label)`
  copy. Every call site passes a literal, which coerces directly — no `.clone`
  needed anywhere. `String.from` is now entirely gone from `src/`. No
  runtime-behaviour change; suite stays **157**, all three examples build.
- **DSL adopts Ruby blocks + `alias`; bare string literals replace
  `String.from("…")`.** An idiom migration (no runtime-behaviour change — the
  build-once invariant holds):
  - **Container builders are `&block` + `yield`.** `row` / `col` / `list` /
    `row_styled` / `col_styled` now declare `&block: Fn[(&var Col) -> nil]` and
    `yield(&var *self)` (private `container` → `open_container`/`close_container`
    primitives, since block-forwarding is staged in ruxen). Call sites read
    `root.row do |c: &var Col| … end`. The param MUST be typed (`&block` type
    does not infer through the yield seam). `list`'s block is **optional**
    (`block_defined?`): `root.list(h)` with no block builds an empty viewport.
  - **Stored callbacks stay `{ … }` brace closures**, NOT blocks — `dyn_text`,
    `button` handlers, `set_measure`, and `list_of`'s item builder (re-invoked on
    every rebuild). House rule: **do…end = immediately-invoked structure; { } =
    stored behaviour** (docs/DSL.md "Blocks & the do/brace idiom").
  - **`App.build` stays an explicit closure param** — its two-`&var`-arg block
    miscompiles under `yield` (filed ruxen Q36; repro
    `tmp/test-cache/ruxen-two-var-yield.md`).
  - **`alias` synonyms** (one body, zero extra codegen): `Col.length`→`size`;
    `ListModel.size`/`length`→`count`; `App.type_char`→`text_input`,
    `App.key`→`key_down` (the last two replaced hand-written delegating methods).
  - **`String.from("literal")` → bare `"literal"`** across `src/` / `tests/` /
    `examples/` (154 call sites; interpolated literals too). Bare literals coerce
    to `String` everywhere; `String.from(...)` kept ONLY for `&str`-variable →
    `String` conversions (5 sites: `String.from(label)`).
  - New pins: optional-block `list` (`tests/list.rx`), `Col.length` alias
    (`tests/dsl.rx`), `ListModel.size`/`length` aliases (`tests/dynamic_list.rx`).
    Suite **153 → 157** (additive). Stale "do…end → free-fn segfault" landmine
    deleted from CLAUDE.md (ruxen Q3 fixed upstream).
  - **Known issue (pre-existing, NOT from this change):** `examples/{counter,
    settings,todo}` do not currently `ruxen build` on the installed toolchain —
    a `could not infer type for parameter __block in function frame` error on
    quiver's generic `frame[S: PaintSurface]` (`src/run.rx`) when quiver is
    consumed as a library by a binary. Reproduces at pristine HEAD (clean
    rebuild), independent of this migration; the quiver **library** builds and
    the 157-test headless suite is green. Filed for the toolchain.
- **Direct-paint backend: dropped the single-`PaintSurface` workaround (ruxen
  Q17).** quiver's generic paint pass (`paint_all`/`paint_dirty`,
  `def …[S: PaintSurface]`) now monomorphizes against a `PaintSurface`
  implementor defined in the *consuming* binary, so each windowed example
  defines its own `SkiaPaint` implementor that issues canvas/Skia calls DIRECTLY
  as the pass visits each node. The per-frame record→`reset`→`replay` double pass
  is gone on the windowed hot loop (one paint walk instead of walk+record+rewalk;
  no per-frame op-array allocation). `RecordingSurface` is unchanged and stays as
  the headless example path + the test suite's double — both backends coexist in
  each binary. No change to `src/paint.rx`'s generic pass was needed (audited: it
  was already correctly `[S: PaintSurface]` throughout); only the obsolete
  single-impl doc comments changed. New canvas-free pin `tests/multi_backend.rx`
  drives a SECOND in-test implementor (`TallySurface`) through the SAME pass as
  `RecordingSurface` and asserts identical per-op streams (incl. `paint_dirty`),
  locking in "the framework drives N paint backends". Suite **150 → 153**
  (additive); `examples/{counter,settings,todo}` all build and run windowed via
  `SkiaPaint`. ADR: `docs/decisions/direct-paint-backend.md`.
- **quiver no longer declares `canvas` as a library dependency — it is now
  platform-agnostic at the manifest level.** quiver's `src/`/`tests/` never
  referenced a canvas symbol (paint crosses the boundary as plain Ints/Strings);
  the dependency existed only for transitive convenience. Removing it (a) fixes
  `ruxen test`, which broke once a Q16-enabled toolchain flat-merged canvas's
  FFI-calling bodies into quiver's test **executable** without canvas's C runtime
  linked (undefined `ruxen_canvas_*`), and (b) is the correct L2 layering: an app
  picks its rendering engine and declares both quiver + an engine itself (the
  examples already do). Suite back to **142 passed**. Filed the underlying
  flat-merge-of-FFI-dependency link gap as a ruxen Q-candidate.

### Added
- **Real text metrics via an injected measure seam** (the last Layout roadmap
  item). `App.set_measure({ |text, size| Int })` lets a shell binary inject true
  advance widths (canvas's Skia `measure_text_sized`); quiver stays canvas-free
  and calls *through* the seam. `arrange`'s flat pre-pass routes leaf-text and
  checkbox-label widths through one `text_width` helper — the injected metric
  when set, else the char-metric estimate, so headless/tests/Skia-less examples
  are byte-identical (the `app.rx`/`counter.rx` width pins are untouched). The
  closure lives in a 0-or-1-entry pool + presence flag (not an `Option[any Fn]`
  field — ruxen Q2). Measurement stays **once per leaf per `arrange`** (a
  `dyn_text` change re-measures on flush because flush re-`arrange`s; static text
  never re-measures per frame) — pinned by an invocation-count double. New
  `text_size()` layout constant + `App.measuring?` introspection. All three
  examples (counter, todo, settings) inject `measure_text_sized` on their
  windowed path; headless keeps char metrics. ADR:
  `docs/decisions/text-metrics-seam.md`; pinned by `tests/text_metrics.rx`
  (+8 tests; suite **142 → 150**, additive).
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
