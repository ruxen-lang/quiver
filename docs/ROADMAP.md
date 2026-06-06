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
(37 tests across `tests/`):

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

## Milestone 1, remaining — the canvas (L1) side

Blocked on `canvas` Milestone 1 (SDL window + Skia surface + event pump):

- A `PaintSurface` adapter over canvas's `Canvas` (lives in the app shell /
  a `quiver-canvas` glue package — ruxen v1 library builds cannot see
  dependency symbols, so it cannot live in quiver's own `src/`).
- Swap `examples/counter`'s synthetic taps for the engine event stream.

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
