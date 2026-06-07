# Changelog

All notable changes to `quiver` are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this project adheres
to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
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
