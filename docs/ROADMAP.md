# quiver — Roadmap

`quiver` is **L2** of the GUI stack. The first milestone is the **counter
app** — the thin vertical slice that proves the whole stack and, crucially,
de-risks the *language-level* reactive API.

## Milestone 0 — scaffold ✅

- `Ruxen.toml` (library + `canvas` path dependency).
- API skeleton and design docs (`DESIGN`, `ARCHITECTURE`, `REACTIVITY`,
  `DSL`, `ROADMAP`).

## Milestone 1 — the counter app, L2 side ✅

All five core-slice items are implemented and pinned by tests
(40 tests across `tests/`):

1. **Reactive core** — `Ui` graph + `State[T]` handles (`src/lib.rx`).
   Explicit `&var Ui` arena instead of the sketched global one (safe Ruxen
   has no globals); handle named `State`, not `Signal` (std collision).
2. **Builder DSL** — `Col` with `text` / `dyn_text` / `button`
   (`src/dsl.rx`), flat-arena tree, the static-vs-reactive rule pinned.
3. **Layout pass** — column stacking with char-metric widths
   (`App.arrange`; real text metrics arrive with Skia).
4. **Paint** — `PaintSurface` mixin + `RecordingSurface` + full/targeted
   paint passes (`src/paint.rx`).
5. **Event dispatch** — `pointer_down` → hit-test → handler → dirty →
   `flush` → `paint_dirty` of exactly the touched nodes (`src/app.rx`,
   `src/run.rx`).

**Demo:** `examples/counter` — runs the real pipeline headlessly today and
prints each paint command; the window path activates when canvas L1 lands.

## Milestone 1, canvas (L1) side ✅

Now that `canvas` Milestone 1 has landed (live SDL2 window + Skia surface +
event stream), `examples/counter` runs as a real windowed app:

- **Structured paint recording** — `RecordingSurface` records each primitive
  both as a string (test double) and as numeric fields (`op_at`/`x_at`/…). It
  remains the headless path + the suite's test double. (Originally the app
  binary `replay`ed those onto canvas's Skia `Canvas` because ruxen v1 resolved
  quiver's generic paint pass only against a single implementor. **ruxen Q17
  removed that limit** — see "Drop the single-`PaintSurface` workaround" below:
  each windowed example now defines its own `SkiaPaint` implementor and paints
  DIRECTLY onto canvas, no record→replay double pass.)
- **Live event loop** — `examples/counter` opens a scaled OS window, paints
  the first frame, and drives repaint from the real `poll_event` stream
  (`PointerDown` → dispatch → flush → repaint; `CloseRequested` → quit). It
  falls back to synthetic taps when there is no display, so headless/CI runs
  still exercise the whole pipeline.
- Verified end-to-end by offscreen pixel readback (white background + grey
  button box through real Skia, `skia_active = true`).

## Resolved open decisions

- **State API shape** — `count.get(&var ui)` / `count.set(&var ui, v)` /
  `count.update(ui, { … })`, plus non-subscribing `peek`. Call-style
  `count()` rejected; no-arg `get` impossible without ambient state.
- **Layout engine** — own simple column pass for the slice; the
  flex-vs-Yoga question moves to the widget-library cycle.
- **First-slice widget set** — `text`, `dyn_text`, `button` in one column.

## Later cycles

- **DSL ergonomics** (audit 2026-06-09, `docs/decisions/dsl-ergonomics.md`) —
  the builder stays Ruby-block-shaped; refinements live *within* that idiom.
  - 2026-06-09: inner STORED-closure params drop their type annotations
    (`{ |ui2| … }` / `{ |c| … }`), inferred from the builder signature.
  - 2026-06-10: **Ruby blocks + `alias`** adopted (CHANGELOG `[Unreleased]`,
    docs/DSL.md). Container builders (`row`/`col`/`list`/styled) are now
    `&block` + `yield`, called `root.row do |c: &var Col| … end`; `list`'s block
    is optional via `block_defined?`. Stored callbacks stay `{ }` braces (the
    do…end-vs-brace house rule). `String.from("literal")` swept to bare literals.
    `alias` added for the genuine synonyms (`Col.length`, `ListModel.size/length`,
    `App.type_char`/`key`).
  - Remaining ergonomic wins are **language-gated** and filed as Q-candidates:
    top-level `App.build` block param inference; the two-`&var`-arg `yield`
    miscompile (Q36, why `App.build` stays a closure param); `&block` param-type
    inference through the yield seam (why container block params must be typed);
    non-Copy call-result as a method arg; auto-reborrow of a `&var` used N>1
    times — to triage into the ruxen ledger, not hacked around here.
  - **Pre-existing example-build regression** (toolchain): `examples/*` fail
    `ruxen build` with `__block in function frame` on quiver's generic
    `frame[S: PaintSurface]` consumed as a library; the lib + 157-test suite are
    green. Filed in the ruxen ledger.
- **Widget library** — rows/containers, lists, inputs, nesting (needs
  child-id arrays in the arena), styling.
- **Text / i18n / accessibility** — via canvas's Skia paragraph /
  HarfBuzz / ICU and platform accessibility trees.
- **Platform matrix** — desktop → mobile → web (tracks `canvas`).
- **Language follow-ups** (filed from this slice's prototyping): flat
  symbol namespace, recursive class types, `&str`-vs-closure overloads,
  `Option[any Fn]` fields, `do…end` to free functions, dependency symbols
  in library/test builds, cross-package generic monomorphization, capture
  semantics under future `Drop`.

## Remaining — tracked checklist

Audited 2026-06-08 against `src/**` and CHANGELOG `[Unreleased]`. Ordered so the
arena/layout foundation lands before the widgets that need it. `→ ruxen #X`
marks a cross-repo dependency (see `../ruxen/docs/TASKS.md`).

### Phase 1 — prod-parity foundation (2026-06-11; `docs/LAYOUT.md` F1/F2 ADR)

**F1 — layout completeness** (the deferred half of the own-flex ADR):

- [x] **Flex grow** — per-child main-axis weight distributes the container's
      leftover main space proportionally (last grown child absorbs rounding).
      Scoped to styled containers; leaf-grow deferred. Pinned by `tests/flex.rx`.
- [x] **Main-axis alignment** — `justify`: start / center / end / space_between /
      space_around. Pinned by `tests/flex.rx`.
- [x] **Cross-axis alignment** — `align`: start / center / end / stretch. Pinned
      by `tests/flex.rx`.
- [x] **Gap** — per-container main-axis spacing between children (replaces
      hand-inserted spacers). Pinned by `tests/flex.rx`.
- [x] **Per-side padding** — `Style.pad_l/t/r/b` via `pad_sides`; `pad(n)` is the
      uniform shorthand (back-compat). Pinned by `tests/flex.rx`.
- [x] **Stack / positioned** — `kind_stack` (`stack` / `stack_styled`): children
      occupy the box (size = max both axes), each offset by `Style.offset`;
      z-order = build order, paint bottom→top, hit-test reverse (topmost wins).
      Pinned by `tests/stack.rx`.

**F2 — gestures & real scrolling:** (in progress — see below)

- [x] **Pixel-based scrolling** — `scroll(dx, dy)` moves `scroll_line` px/click
      (default `row_height`; `set_scroll_line` overrides). App boundary stays Int.
      Pinned by `tests/scroll_gestures.rx`.
- [x] **Velocity + fling momentum** — injected clock seam (`set_clock`, default =
      deterministic frame counter advanced by `tick`); drag-scroll captures a list,
      tracks velocity, `pointer_up` seeds a fling, `App.tick` steps + decays it
      (clamped at ends). Pinned by `tests/scroll_gestures.rx`.
- [x] **Tap vs long-press recognizers** — tap = immediate handler (unchanged);
      long-press via `c.long_press({ |ui| … })` (attaches to last-built node),
      fired by `tick` past `long_press_ms` within `gesture_slop`. Pinned by
      `tests/scroll_gestures.rx`.
- [x] **Scrollbars (paint-only)** — a thumb on the trailing edge when content
      overflows (right for a vlist, bottom for an hlist), sized/positioned from
      the viewport/content/scroll ratios, painted in viewport space after the
      clip restore. Pinned by `tests/scrollbar.rx`.
- [x] **Horizontal scroll** — `hlist(viewport_w)` reuses `kind_list` + a
      `horizontal` flag; all clip/scroll/hit/scrollbar/fling machinery is shared
      on an axis-tagged offset; the vertical path is byte-identical. Pinned by
      `tests/hlist.rx`.

**Deferred (explicit — do NOT implement this phase):** wrap, virtualization,
keyed diffing / row reuse, scroll animation/fades, double-tap, `shrink` /
overflow-shrink, leaf-level `grow`, sub-click Float wheel precision.

### Foundation (unblocks the whole widget library — do first)

- [x] **Nested tree in the arena** — `Col` now owns children via flat parallel
      tree-link hashes (`parent`/`child_first`/`child_last`/`child_next`, node-id
      → node-id, `-1`/absent ⇒ none) + a root chain + a build cursor. No
      recursive type; node id = build order. Pinned by `tests/nesting.rx`.
- [x] **`Row` + generic containers** — `Col.row { |c| … }` / `Col.col { |c| … }`
      build container nodes; block-built nodes become their children; nesting is
      arbitrary. `Row` of texts/`Col`s laid out + painted, AND reactive children
      (`dyn_text`/`button`) inside containers now re-render on state change
      (ruxen Q26 fixed). Pinned by `tests/row.rx` + `tests/nesting.rx`.

### Layout (needs a decision, then implement)

- [x] **Layout-engine decision (ADR)** — `docs/LAYOUT.md` (2026-06-08): adopt an
      **own simple main-axis stacking/flex pass in safe Ruxen**, shipped
      incrementally; Yoga (C/FFI) rejected as a charter violation and overkill.
- [x] **Real layout engine (own-flex v1)** — `App.arrange` recursively walks the
      nested arena: a flat pre-pass measures each leaf's text width once, then a
      measure/place recursion stacks each container's children along its axis
      (`Row` = X, `Col` = Y; top level = implicit column), producing the
      `geom_x/y/w/h` the paint pass reads. Grow/shrink/wrap/gap deferred
      (additive, per the ADR). Char-metric widths remain (next item).
- [x] **Real text metrics** — replace char-metric estimates with canvas's Skia
      `measure_text` true advance width, **injected through a seam** rather than
      imported (quiver is canvas-free by charter; `Ruxen.toml` no longer declares
      canvas). `App.set_measure({ |text, size| Int })` lets the shell binary —
      the only place quiver L2 + canvas L1 symbols meet — supply real metrics;
      `arrange`'s flat pre-pass routes leaf-text + checkbox-label widths through
      one `text_width` helper that calls the seam when set, else the char-metric
      fallback (so headless/tests/Skia-less runs are byte-identical). Measure runs
      **once per leaf per arrange** (pinned by an invocation-count double), not
      per flush. All three examples inject `measure_text_sized` on their windowed
      path. ADR: `docs/decisions/text-metrics-seam.md`; pinned by
      `tests/text_metrics.rx` (suite **142 → 150**). (Metric-aware caret/select
      paint + per-glyph hit positioning + font family/weight deferred — additive
      on the same seam.)

### Widgets (after foundation + layout)

- [x] Lists (vertical scroll) — `Col.list(viewport_h, build)`: a fixed-viewport
      container that clips + scrolls its children by an offset. Layout keeps the
      box at the viewport and lays children out in content space; paint wraps the
      children in `save`/`clip`/`translate(-offset)`/`restore`
      (`op_save`/`op_clip`/`op_translate`/`op_restore` → canvas
      `save`/`clip_rect`/`translate`/`restore`). Scroll via `scroll_to`/`scroll_by`
      (clamped) + drag-to-scroll (no wheel `Event`). **Hit-testing is
      scroll/clip-aware** — `hit_test` mirrors the paint walk, accumulating the
      scroll offset and intersecting each list's viewport as a clip, so clicks on
      scrolled buttons/inputs land right and clipped-away items are unhittable
      (nested lists accumulate). Pinned by `tests/list.rx` + `tests/list_interaction.rx`.
      (Horizontal scroll, virtualization, scrollbars deferred.)
- [x] Inputs (single-line text field) — `Col.input(value, width)`: a focusable
      field bound to a reactive `State[String]`. `App` tracks `focused` +
      per-input `caret`; `pointer_down` focuses, `key_down`/`type_char`/`key`
      route to the focused input (insert / backspace / left / right). Paint: box
      + text + caret rect. `Event.KeyDown` carries an opaque platform keycode, so
      quiver uses logical key codes the shell maps SDL onto. Pinned by
      `tests/input.rx`. (Selection, clipboard, multiline, real text metrics for
      the caret deferred.)
- [x] Checkbox — `Col.checkbox(state, label)` bound to a reactive `State[Bool]`:
      click toggles via a single-lock `state.update(ui, { |b| !b })`, the node
      re-renders on toggle, paint draws a bordered box + a filled inner mark when
      checked + the label (no new op). Composes in row/col/list/styled and
      hit-tests through list scroll. New `Ui.state_bool`. Pinned by
      `tests/checkbox.rx`. (A toggle/switch variant would be the same node with a
      pill-shaped paint — additive.)
- [x] Slider (horizontal) — `Col.slider(state, min, max, width)` bound to a
      reactive `State[Int]`. Click-to-set + drag-to-update via new **drag-capture
      infra** on `App` (`pointer_move`/`pointer_up`/`captured`, reusable). Screen
      x → value math (single-lock `set`, no peek+set); thumb re-renders on change;
      hit-tests through list scroll. Paint reuses `op_round_rect` (track + filled
      portion + thumb). Pinned by `tests/slider.rx`. (Vertical/range/float sliders
      deferred — additive.)
- [x] Dropdown/Select — `Col.select(state, options, width)` bound to a reactive
      `State[Int]` (selected index) over an `Array[String]`. Built on new
      **overlay/popup infra** on `App` (`open_overlay`/`close_overlay`/
      `overlay_open?` + an anchored slot) — a reusable framework primitive
      (dropdowns/menus/tooltips/modals): the overlay paints LAST (on top) and is
      hit-tested FIRST, so a click outside dismisses it and is consumed. Trigger
      reflects the selection; option-click writes the index (single-lock `set`)
      and closes. No new paint op. Pinned by `tests/select.rx`. (Multi-select,
      typeahead, nested overlays, scrollable popup deferred — additive.)
- [x] Styling — **padding, background, border** (v1) on `row`/`col` via a small
      `Style` value (`row_styled`/`col_styled`): padding insets children and
      grows the box; background (`fill_round_rect`) + border (`stroke_round_rect`)
      paint under the children with an optional corner radius. Maps onto canvas's
      `draw_round_rect`/`stroke_round_rect` in the example binary. Pinned by
      `tests/style.rx`. (Per-side padding, margins, gradient/shadow fills deferred
      — additive on the same `Style`.)
- [x] **Structural reactivity (dynamic lists)** — `Col.list_of(model, viewport_h,
      item_builder)` bound to a shared `ListModel` (`ui.list_model`). Mutating the
      model (`add`/`toggle`/`remove`, single-lock RMW that bumps a version) marks
      the list dirty; `flush` REBUILDS its child subtree wholesale (the one opt-in
      construct that grows/shrinks the tree — the build-once rule still holds for
      everything else). Reuses the scroll/clip list machinery. ADR:
      `docs/decisions/structural-reactivity.md`; pinned by `tests/dynamic_list.rx`;
      demo: `examples/todo`. (Keyed diffing / row reuse, per-item reactive
      widgets, reorder/animation deferred.)

### Blocked on ruxen (the no-GC promise + multi-package framework)

- [ ] **Real `Drop` so capture/teardown is sound** — landmines note capture is
      *"sound today because drops don't run yet."* Long-lived widget trees need
      deterministic teardown. **→ ruxen P0.2** (Drop elaboration discarded by
      both backends).
- [x] **Drop the single-`PaintSurface` workaround** (**ruxen Q17 landed**) —
      quiver's generic paint pass (`paint_all`/`paint_dirty`,
      `def …[S: PaintSurface]`) now monomorphizes against an implementor defined
      in the *consuming* binary. Each windowed example defines its own `SkiaPaint`
      implementor that issues Skia calls DIRECTLY as the pass visits each node;
      the per-frame record→`reset`→`replay` double pass is gone on the windowed
      path (one paint walk instead of walk+record+rewalk). `RecordingSurface`
      stays as the headless path + the suite's test double; both backends coexist
      in each binary. A canvas-free pin (`tests/multi_backend.rx`) drives a SECOND
      in-test implementor (`TallySurface`) through the SAME pass as
      `RecordingSurface` and asserts identical per-op streams — locking in "the
      framework drives N paint backends" forever. ADR:
      `docs/decisions/direct-paint-backend.md`. Suite **150 → 153** (additive);
      all three examples build. (Audited `src/paint.rx`'s generic pass for a
      single-impl assumption — none found; every paint fn was already
      `[S: PaintSurface]`. Only the now-obsolete single-impl doc comments
      changed.)
- [x] **Unit-test quiver's public API directly** (**ruxen Q16 fixed**) — `ruxen
      test` now compiles tests against the package's own + dependency symbols
      (library/`check`/`test` flat-merge dependency sources; verified by
      ruxen's `dep_visibility.rs`). quiver's headless suite was always able to
      name its own public types (`App`/`Ui`/`Col`/`RecordingSurface`); this
      cycle used that to pin previously under-tested public behavior **directly**
      rather than only through the example binaries:
      - **`tests/caret_edit.rx`** — the text-input editing surface
        `input.rx`/`window_events.rx` left uncovered: forward delete
        (`key_delete`), Home/End (`key_home`/`key_end`), mid-string insert/delete,
        no-op at the ends, control keys with nothing focused. (These had **zero**
        coverage before.)
      - **`tests/resize.rx`** — `App.resize` re-layout over a nested tree + a
        list: design-size tracking across multiple resizes, geometry stability
        (re-arrange is a no-op move, not a corruption), list viewport/content
        height + scroll offset surviving a resize, reactive state/cache
        undisturbed, hit-testing still correct.
      - **`tests/overlay.rx`** — the overlay/popup primitive (`open_overlay`/
        `close_overlay`/`overlay_open?`/`overlay_owner`/anchor) as a reusable
        primitive in its own right: open/close/owner/anchor, re-open replaces the
        owner, and the click-consumed-not-passed-below rule — independent of the
        `select` dropdown that `select.rx` already routes through it.
      Suite: **142 passed** (124 → 142, additive); the `app.rx`/`counter.rx`
      static-vs-reactive pins were not touched.
- [x] **Reactive children inside a container** (`dyn_text`/`button` in a
      `row`/`col`) — **ruxen Q26 fixed (2026-06-08)**: captures survive the
      container's `&var *self` reborrow, so reactive children re-render on state
      change just like top-level nodes. Pinned by `tests/nesting.rx`.
- [x] **Empty-hash lookups** (**ruxen Q25 fixed, 2026-06-08**) — `Hash.key?`/`get`
      on an empty hash no longer segfault; `&Hash`/`&Set` params now reject at
      compile time. quiver's `size > 0` workaround guards were removed (lookups
      are plain `match h.get(i)`). Repro kept for history in
      `tmp/test-cache/ruxen-empty-string-hash-segfault.md`.
