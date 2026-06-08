# Changelog

All notable changes to `quiver` are documented here. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this project adheres
to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
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
