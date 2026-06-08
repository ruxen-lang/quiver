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
  both as a string (test double) and as numeric fields (`op_at`/`x_at`/…).
  The app binary `replay`s those onto canvas's Skia `Canvas`. The bridge is a
  single `PaintSurface` impl on purpose: ruxen v1 only resolves quiver's
  generic paint pass when the mixin has one implementor (it devirtualises
  rather than monomorphising), and it cannot monomorphise a dependency's
  generic for a type defined in the consuming app — so the structured surface
  lives in quiver and the canvas glue lives in the binary.
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
- [ ] **Real text metrics** — replace char-metric estimates with canvas's Skia
      `measure_text` once it returns true advance width (**→ canvas / ruxen**:
      the FFI `&String` bug).

### Widgets (after foundation + layout)

- [ ] Lists (scrolling — needs canvas clip/layer, both now available).
- [ ] Inputs (text field; needs key events + caret).
- [x] Styling — **padding, background, border** (v1) on `row`/`col` via a small
      `Style` value (`row_styled`/`col_styled`): padding insets children and
      grows the box; background (`fill_round_rect`) + border (`stroke_round_rect`)
      paint under the children with an optional corner radius. Maps onto canvas's
      `draw_round_rect`/`stroke_round_rect` in the example binary. Pinned by
      `tests/style.rx`. (Per-side padding, margins, gradient/shadow fills deferred
      — additive on the same `Style`.)

### Blocked on ruxen (the no-GC promise + multi-package framework)

- [ ] **Real `Drop` so capture/teardown is sound** — landmines note capture is
      *"sound today because drops don't run yet."* Long-lived widget trees need
      deterministic teardown. **→ ruxen P0.2** (Drop elaboration discarded by
      both backends).
- [ ] **Drop the single-`PaintSurface` workaround** — multiple paint backends
      need cross-package generic monomorphization. **→ ruxen Q17.**
- [ ] **Unit-test quiver's public API directly** — `ruxen test` can't link the
      package, so behavior is pinned through the binary. **→ ruxen Q16.**
- [x] **Reactive children inside a container** (`dyn_text`/`button` in a
      `row`/`col`) — **ruxen Q26 fixed (2026-06-08)**: captures survive the
      container's `&var *self` reborrow, so reactive children re-render on state
      change just like top-level nodes. Pinned by `tests/nesting.rx`.
- [x] **Empty-hash lookups** (**ruxen Q25 fixed, 2026-06-08**) — `Hash.key?`/`get`
      on an empty hash no longer segfault; `&Hash`/`&Set` params now reject at
      compile time. quiver's `size > 0` workaround guards were removed (lookups
      are plain `match h.get(i)`). Repro kept for history in
      `tmp/test-cache/ruxen-empty-string-hash-segfault.md`.
