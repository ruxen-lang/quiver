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
