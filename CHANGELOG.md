# Changelog

All notable changes to `quiver` are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this project adheres
to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **ADR: layout engine — own simple flex vs Yoga (`docs/LAYOUT.md`).**
  Decided **own simple main-axis stacking/flex pass in safe Ruxen**, shipped
  incrementally (intrinsic stacking now; grow/shrink/wrap later, additively).
  Yoga (C) rejected: it forces an FFI/C dependency into L2's 100%-safe charter,
  couples an L2 layout concern to the L1 boundary, and is overkill for the
  Row+Col+nesting slice. Closes the layout-engine open decision in `ROADMAP.md`.

### Blocked
- **Arena nesting + `Row` + the nested layout pass are NOT implemented this
  cycle** — blocked by an active toolchain regression (see below). They are
  TDD deliverables and cannot be written without a working `ruxen build`/`ruxen
  test`; shipping unverifiable code would violate the project's pin-test
  discipline. Carried forward in `ROADMAP.md`.

### Known issues
- **ruxen toolchain (`local-48c51aa`) cannot compile any program** — `ruxen
  build` and `ruxen test` fail inside the embedded std/prelude (move/borrow
  errors at fixed positions, identical for a bare `def main`). `ruxen check`
  still passes. Documented as **Q18** in `CLAUDE.md`'s landmines with a minimal
  repro (`tmp/test-cache/ruxen-prelude-miscompile.log`); owned by the ruxen
  toolchain. The full quiver suite, including the `app.rx`/`counter.rx` pin
  tests, is unrunnable in this environment until it is fixed.

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
