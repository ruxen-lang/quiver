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

> **⛔ BLOCKED (2026-06-08):** the Foundation + Row implementation below are TDD
> deliverables and cannot proceed — the installed ruxen (`local-48c51aa`)
> cannot `build`/`test` any program (Q18 in `CLAUDE.md`; repro in
> `tmp/test-cache/ruxen-prelude-miscompile.log`). Resume once a bare `ruxen
> build` succeeds. The layout ADR (design-only) was completed.

### Foundation (unblocks the whole widget library — do first)

- [ ] **Nested tree in the arena** *(blocked on ruxen Q18)* — today the arena is
      effectively a flat single column. Add **child-id arrays + `-1` sentinels**
      (recursive class types crash ruxen v1, so flat parallel arrays, per the
      landmines) so a node can own children. Prerequisite for every container below.
- [ ] **`Row` + generic containers** *(blocked on ruxen Q18)* — horizontal
      stacking and arbitrary nesting on top of the arena tree. First correct
      slice: `Row` of `Col`s.

### Layout (needs a decision, then implement)

- [x] **Layout-engine decision (ADR)** — `docs/LAYOUT.md` (2026-06-08): adopt an
      **own simple main-axis stacking/flex pass in safe Ruxen**, shipped
      incrementally; Yoga (C/FFI) rejected as a charter violation and overkill.
- [ ] **Real layout engine** *(blocked on ruxen Q18)* — only a simple column
      pass with **char-metric width estimates** exists. Per the ADR above,
      implement the own-flex pass (intrinsic main-axis stacking over the nested
      arena, generalised over axis for `Col`/`Row`) producing geometry for paint.
- [ ] **Real text metrics** — replace char-metric estimates with canvas's Skia
      `measure_text` once it returns true advance width (**→ canvas / ruxen**:
      the FFI `&String` bug).

### Widgets (after foundation + layout)

- [ ] Lists (scrolling — needs canvas clip/layer, both now available).
- [ ] Inputs (text field; needs key events + caret).
- [ ] Styling — padding, background, border (maps onto canvas rrect/gradient/shadow).

### Blocked on ruxen (the no-GC promise + multi-package framework)

- [ ] **Real `Drop` so capture/teardown is sound** — landmines note capture is
      *"sound today because drops don't run yet."* Long-lived widget trees need
      deterministic teardown. **→ ruxen P0.2** (Drop elaboration discarded by
      both backends).
- [ ] **Drop the single-`PaintSurface` workaround** — multiple paint backends
      need cross-package generic monomorphization. **→ ruxen Q17.**
- [ ] **Unit-test quiver's public API directly** — `ruxen test` can't link the
      package, so behavior is pinned through the binary. **→ ruxen Q16.**
