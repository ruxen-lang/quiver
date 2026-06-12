# ADR: Layout engine — own simple flex vs binding Yoga

Status: **Accepted** (2026-06-08). Supersedes the "Layout engine" bullet under
*Resolved open decisions* in `docs/ROADMAP.md`, which deferred this very choice
to "the widget-library cycle" — this is that cycle.

## Context

quiver is **L2** of the Ruxen GUI stack and is, by charter, **100% safe Ruxen —
no `unsafe`, no FFI** (`docs/ARCHITECTURE.md`). FFI lives one layer down in
`canvas` (L1) and nowhere else.

Today's layout is a single vertical column: `App.arrange` stacks nodes top to
bottom, one row each, sizing each node to a char-metric width estimate
(`(label.size) * char_width + pad_x*2`). That is enough for the counter slice
but nothing more — there is no horizontal stacking, no nesting, no notion of a
container distributing space among children.

The next foundation work (this cycle) adds:

1. an **arena that can nest** — a node owning children (tracked separately, see
   the nesting ADR notes in `docs/DSL.md`), and
2. a **layout pass that walks that nested tree** and produces geometry for the
   paint pass — starting with `Row` (horizontal stacking) over the existing
   `Col` (vertical stacking).

The open question this ADR closes: when we grow that layout pass, do we **write
our own simple flex/stacking pass in safe Ruxen**, or **bind Yoga** (Facebook's
C flexbox engine, the one React Native / Litho use) through FFI?

### Forces

- **The safety charter is non-negotiable at L2.** Anything that pulls a C
  library into quiver violates the one architectural invariant that defines
  this layer.
- **The widget set we are building is tiny.** Row + Col + nesting. The geometry
  these need is *main-axis stacking with cross-axis alignment* — a single linear
  pass per container. We do not yet have wrapping, `flex-grow` ratios across
  remaining space, `aspect-ratio`, `position: absolute`, baseline alignment
  across mixed font sizes, or RTL — the features that make a real flex engine
  genuinely hard to get right.
- **ruxen v1 is young.** The landmines list in `CLAUDE.md` shows the compiler
  still miscompiles recursive types, `Option[any Fn]`, certain overloads, and
  `do…end` to free functions. A bound C engine would still need a *safe Ruxen
  wrapper* whose data marshalling hits exactly these sharp edges — and the FFI
  itself would have to live in `canvas`, not here, which means designing an L1
  surface for a layout concern that is conceptually L2's job.
- **Determinism / testability.** quiver pins geometry with exact-value tests
  (`tests/app.rx`: "layout stacks rows top to bottom", "layout sizes a node to
  its text width"; `tests/paint.rx` pins pixel-exact paint commands). An
  in-language pass is trivially unit-pinnable the same way. A C engine's output
  is pinnable too, but the binding adds a translation layer between "what we
  asked for" and "what got measured" that those tests would have to chase.

## Options

### Option A — Own simple flex/stacking pass (in safe Ruxen)

A linear constraint pass written in quiver. Per container, one walk over its
children: measure each child's intrinsic main-axis size, sum them, place them
along the main axis with the container's spacing, align each on the cross axis.
`Col` stacks on Y, `Row` stacks on X — the same routine parameterised by axis.
Grow/shrink (distributing leftover main-axis space by weight) is a later,
additive extension of the same pass; v1 ships fixed/intrinsic sizing only.

- **Pros:** Zero dependencies. Stays 100% safe Ruxen — honors the charter
  literally. Geometry is exactly what our tests assert; no marshalling layer.
  Scope matches need precisely (Row/Col/nesting want a linear pass, nothing
  more). Evolves additively as widgets demand more (rule 42-friendly).
- **Cons:** We will eventually re-implement pieces of flexbox (wrap, grow/shrink
  ratios, min/max clamping) ourselves, and must get edge cases right that Yoga
  already handles. Risk of a half-flex that subtly diverges from CSS semantics
  if we are not disciplined about which subset we claim to implement.

### Option B — Bind Yoga (C) through FFI

Vendor Yoga, expose a node-tree + calculate API, marshal quiver's arena into
Yoga nodes each layout, read computed rects back.

- **Pros:** Battle-tested, CSS-flexbox-accurate, handles wrap/grow/shrink/RTL/
  min-max out of the box. We never re-derive flex edge cases.
- **Cons:** **Forces a C dependency into the "100% safe" layer — a direct
  charter violation.** The FFI cannot even live in quiver; it would have to be
  bound in `canvas` (L1) and surfaced upward, coupling an L2 layout concern to
  the L1 engine boundary for no rendering reason. Bundle/build cost: a C
  toolchain dep and a vendored tree. Marshalling the arena ↔ Yoga node tree
  every frame is allocation churn on a no-GC language — exactly the cost
  fine-grained reactivity was chosen to avoid (`docs/REACTIVITY.md`). The safe
  Ruxen wrapper marshalling Ints/handles across FFI would hit the very v1
  compiler landmines we are already navigating. Massive overkill for Row + Col.

## Decision

**Adopt Option A — write our own simple main-axis stacking / flex pass in safe
Ruxen.** Ship it incrementally:

- **v1 (this cycle):** intrinsic-size main-axis stacking with cross-axis start
  alignment, generalised over axis so `Col` (Y) and `Row` (X) share one pass,
  walking the nested arena. No wrap, no grow/shrink yet.
- **later, additively:** spacing/gap, cross-axis alignment options, then
  grow/shrink weight distribution over remaining main-axis space, then
  min/max clamps — each a pin-tested extension of the same linear pass.

Rationale, in order of weight:

1. **The charter wins.** L2 is 100% safe Ruxen by definition; Yoga's C dep is a
   non-starter here. This alone is decisive.
2. **Need ≪ Yoga.** The counter-and-containers slice needs column + row
   stacking — a linear pass. Flexbox-complete behaviour is not on the near
   roadmap; paying its full binding cost now buys nothing.
3. **No FFI tax, no marshalling churn, no L1 coupling.** An in-language pass
   produces the geometry our tests already assert, with no per-frame
   arena→C→arena translation on a no-GC runtime.

## Consequences

- quiver gains a `layout`-style pass (note: `layout` is a reserved keyword per
  the landmines, so the method keeps the existing name `arrange`) that walks the
  nested arena and writes the same `geom_x/y/w/h` tables the paint pass already
  reads — so paint, hit-test, and the targeted-repaint span logic are unchanged
  in shape.
- The existing column tests (`tests/app.rx`: "layout stacks rows top to
  bottom" / "layout sizes a node to its text width") must keep passing
  unchanged — the new pass is a generalisation of the current column behaviour,
  with `Col` as the axis=Y specialisation. Changing those is a design event.
- We own correctness for the flex subset we claim. We mitigate by claiming a
  *small, documented* subset and pinning each behaviour with exact-geometry
  tests, exactly as the column pass is pinned today.
- Width still uses **char-metric estimates** until canvas's Skia `measure_text`
  returns true advance width; the layout-engine choice is orthogonal to that
  (tracked separately as "Real text metrics → canvas/ruxen" in the roadmap).

## What this defers (revisit triggers)

- **Flex completeness** — wrap, `flex-grow`/`flex-shrink` ratios, min/max
  clamping, `aspect-ratio`, absolute positioning, baseline alignment, RTL.
  Added additively to the own-pass as a widget demands each. *Revisit the
  Yoga question only if* we find ourselves needing most of CSS flexbox/grid
  semantics at once (e.g. a full data-grid or a constraint-heavy form layout)
  **and** ruxen has grown an FFI story that does not violate L2's safety
  charter — neither of which is true today.
- **Constraint solving beyond linear flex** (auto-layout / Cassowary-style)
  is explicitly out of scope; not anticipated for the widget library.

---

# Addendum: F1 layout completeness + F2 gestures/scrolling (2026-06-10)

Status: **Accepted** — this is the "later, additively" half the original ADR
deferred (gap, cross-axis alignment, grow/shrink, per-side padding), plus the
F2 gesture/scroll foundation (pixel scroll, fling, recognizers, scrollbars,
horizontal scroll). All still 100% safe Ruxen on the own-flex pass; no FFI.

The through-line for every F1 feature: it is **another read inside the existing
two-phase `measure` / `place` recursion**, writing the same `geom_x/y/w/h` tables
paint + hit-test already read. No new output surface, so paint/hit-test shapes
are unchanged where the math is unchanged.

## DSL shape decision — extend `Style`, not new yield shapes

The container-layout knobs (gap, main-/cross-axis alignment, per-side padding,
grow) all live as **new fields on the existing `Style` value**, consumed through
the existing `row_styled` / `col_styled` builders. Rationale:

- **It composes with build-once and the block idiom unchanged.** `Style` is a
  plain chainable value object unpacked into parallel `Int` hashes on `Col` (the
  flat-arena discipline); adding fields is purely additive — no new `&block`
  shape, so the two-`&var`-yield miscompile (Q36) and the typed-block-param
  constraint are both untouched. `row_styled(style) do |c: &var Col| … end` is
  the one surface.
- **A per-node post-hoc modifier (`c.text("x").grow(2)`) is NOT viable** — every
  builder returns `nil` (the arena is push-only; there is no returned node
  handle to chain on), and adding one would be a cross-cutting API change. A
  `c.grow(2) do … end` wrapper container is viable but adds a node per grown
  child; folding `grow` into `Style` (read by the *parent's* place pass) needs
  no extra node.
- **Defaults are inert.** Every new `Style` field defaults to the v1 behaviour
  (gap 0, align start, per-side pad = uniform `padding`, grow 0), so an unstyled
  `row`/`col` and every existing pin are byte-identical. New `Style` fields are
  stored in parallel `Int` hashes keyed by node id, absent ⇒ default — exactly
  the existing `pad_of` pattern.

New `Style` setters (each returns `self`):

| Setter | Field(s) | Default | Meaning |
|---|---|---|---|
| `gap(n)` | `gap` | 0 | px between adjacent children on the main axis |
| `justify(j)` | `justify` | `just_start` | main-axis distribution (see below) |
| `align(a)` | `align` | `align_start` | cross-axis alignment (see below) |
| `pad_sides(l,t,r,b)` | `pad_l/t/r/b` | uniform `padding` | per-side padding |
| `grow(n)` | `grow` | 0 | this container's main-axis grow weight in ITS parent |

`pad(n)` stays the uniform shorthand: it sets `padding` AND all four per-side
fields to `n` (so existing `pad(8)` pins hold). `pad_sides` overrides per side.

Alignment/justify constants are plain `Int`s (the node-kind idiom):

```
def just_start         -> Int { 0 }   # default — children packed at main-start
def just_center        -> Int { 1 }
def just_end           -> Int { 2 }
def just_space_between -> Int { 3 }   # first at start, last at end, gaps equal
def just_space_around  -> Int { 4 }   # equal space around each (half at ends)

def align_start   -> Int { 0 }   # default — children at cross-start
def align_center  -> Int { 1 }
def align_end     -> Int { 2 }
def align_stretch -> Int { 3 }   # children grow to the container's cross extent
```

## 1. Flex factors (`grow`)

A child's `grow` weight claims a share of the container's **leftover main-axis
space** (container inner main extent − sum of children main sizes − total gap),
distributed proportionally by weight. Single pass, after intrinsic measure:

- Place pass computes `leftover = inner_main − Σ child_main − gaps`. If
  `leftover > 0` and `Σ grow > 0`, each grown child's main size becomes
  `intrinsic + leftover * grow_i / Σ grow` (integer math; the LAST grown child
  absorbs the rounding remainder so the row exactly fills — no sub-pixel gap).
- A container's intrinsic main size still ignores grow (grow only acts when the
  parent has surplus). The TOP-LEVEL implicit column uses `design_w` as the
  available cross extent and `design_h` as available main extent for grow/justify
  (0 ⇒ no surplus ⇒ inert, so headless default geometry is unchanged).

**Scope landed:** grow on **styled containers** (`row_styled`/`col_styled`),
read by the parent place pass. **Deferred (filed):** grow on bare leaves — needs
a styled-leaf surface (`text` takes no `Style`); wrap a leaf in a 1-child
`row_styled(Style.new.grow(n))` to grow it today. `shrink` (overflow) is
deferred — v1 does not shrink below intrinsic; overflow clips (lists) or
overruns (rows). Documented as a revisit trigger.

## 2. Main-axis alignment (`justify`)

After children are measured and grow applied, the place pass distributes any
*remaining* leftover (when no child grew, or grow consumed none) along the main
axis per `justify`:

- `just_start` (default): pack at main-start, current behaviour.
- `just_center`: leading offset = `leftover / 2`.
- `just_end`: leading offset = `leftover`.
- `just_space_between`: gap between children += `leftover / (n−1)` (n ≥ 2; n=1
  behaves as `just_start`).
- `just_space_around`: each child gets `leftover / n` of surround, half before
  the first and after the last.

Grow and justify are mutually exclusive in effect: if children grew to fill,
leftover is 0 and justify is inert. Pinned both ways.

## 3. Cross-axis alignment (`align`)

Per child, on the axis perpendicular to the container's main axis:

- `align_start` (default): child at cross-start (current behaviour).
- `align_center`: cross offset = `(container_cross − child_cross) / 2`.
- `align_end`: cross offset = `container_cross − child_cross`.
- `align_stretch`: child's cross size becomes the container's inner cross extent
  (the one case that writes a child's `geom_w`/`geom_h` larger than intrinsic).

## 4. Gap

`gap(n)` inserts `n` px between adjacent children on the main axis (not before
the first or after the last). Implemented as `cx/cy += gap` after each non-last
child in the place walk, and added into the intrinsic main extent (`(n−1)*gap`)
so the container box accounts for it. Replaces hand-inserted spacer nodes — the
settings example's showcase uses it.

## 5. Per-side padding

`Style` grows `pad_l/pad_t/pad_r/pad_b`. The place pass insets children by
`(pad_l, pad_t)` and the box grows by `pad_l+pad_r` / `pad_t+pad_b`. `pad(n)` is
the shorthand that sets all four equal (back-compat: existing `pad_at(i)*2`
math becomes `pad_l_at(i)+pad_r_at(i)`, equal to `2*pad` when uniform). The
single `pad_at` reader is replaced by four side readers; `pad_at` is kept as an
alias returning `pad_l` for any incidental caller, then audited out.

## 6. Stack / positioned

A new `stack` container kind (`kind_stack`): every child occupies the container
box (all at the container origin + the child's optional `left`/`top` offset),
z-order = build order. The primitive for badges/overlap.

- **Layout.** The stack's intrinsic size is the **max** of its children on BOTH
  axes (not sum). Each child is placed at `(x + pad_l + left_i, y + pad_t +
  top_i)` at its intrinsic size. `left`/`top` are per-child `Style` offsets
  (`Style.offset(l, t)`), default 0.
- **Paint.** Children paint in BUILD order (first child at the bottom, last on
  top) — the existing `paint_tree` child walk already does this; a stack adds no
  clip/translate, so paint is the plain container walk.
- **Hit-test.** Children hit-test in **REVERSE** build order (topmost wins). This
  is the one place `hit_tree` must special-case a kind: for a stack, walk
  children last→first and return the first hit. (Normal containers are
  non-overlapping so order is immaterial; a stack overlaps by design.)

DSL: `root.stack do |c: &var Col| … end` and `stack_styled(style)`. A positioned
child sets `Style.offset` on its own (styled) sub-container.

## F2 — gestures & scrolling

### 7. Pixel-based scrolling

`App.scroll(dx, dy)` currently maps one wheel click → one `row_height` step. The
F2 change: scroll by **pixels**, with a configurable line-height factor. The
canvas seam delivers `Float32` wheel deltas; the example shells cast to `Int` at
dispatch. **Decision:** keep the `App.scroll(dx: Int, dy: Int)` signature (Int at
the App boundary — deterministic, headless-pinnable), and multiply by a
`scroll_line_px` factor (default = `row_height`, configurable via
`set_scroll_line(px)`) BEFORE clamping. So `scroll(0, 1)` moves `scroll_line_px`
px, not one row. `scroll_to`/`scroll_by` already take pixel offsets and stay
clamped — unchanged. Precision note in the ADR: wheel deltas are quantized to Int
clicks at the shell; sub-click precision is deferred until the clock/Float seam
lands (see #8).

### 8. Velocity + fling momentum

Pointer-drag scrolling (the existing drag-capture infra) tracks **velocity** over
recent samples and, on release, **decays** it per frame (exponential) advancing
the scroll until below a threshold.

- **Clock seam (injected, like `set_measure`).** `App.set_clock({ || Int })`
  supplies a monotonic ms timebase. **Default = a frame counter** that advances
  one unit per `tick`, so tests are fully deterministic without a real clock
  (canvas's clock binding is landing in parallel; we do NOT wait for it). A lone
  0-or-1-entry pool + presence flag (the `measure_fns` pattern — NOT
  `Option[any Fn]`).
- **Velocity tracking.** On a drag (`pointer_move` while a *list* is captured —
  new: a list captures on a press inside it that isn't on an inner widget), record
  (offset, clock) samples in a small ring (last K); velocity = Δoffset/Δtime over
  the window. On `pointer_up`, seed `fling_v[list] = velocity`.
- **Stepping.** `App.tick` (new per-frame entry the shell's loop calls) advances
  each active fling: `offset += v * dt; v *= decay` (decay ≈ 0.92/frame), stop
  when `|v| < threshold`. Each step `scroll_to`s (clamped — a fling into the end
  stops). Returns whether any fling is still active (so the shell knows to keep
  drawing). Deterministic under the fake clock: N ticks after release with
  velocity v → a predicted offset sequence, pinned exactly.

Drag-to-scroll for lists is **new infra** layered on the existing slider capture:
`captured` becomes "the captured interactive node" generalised — a list captures
when a press lands in its viewport but not on an inner handler/input/slider, and
`pointer_move` then scrolls it by the drag delta (and feeds the velocity ring).

### 9. Tap vs long-press recognizers

- **Tap** = `pointer_down`+`pointer_up` within slop (px) + time (ms via the
  clock). Today `pointer_down` fires button/checkbox handlers immediately; we
  keep that as the tap path (a click IS a tap) — no behavioural change for
  existing widgets.
- **Long-press** = pointer held past `long_press_ms` without moving past slop.
  New per-node handler slot (`on_long_press`, a stored `{ |ui| … }` closure in a
  new handler pool + `long_press_of` index hash — the `handlers` pattern). `App`
  arms a timer on `pointer_down` over a long-press node (records press node +
  press clock + press point); `App.tick` checks elapsed ≥ threshold && not moved
  past slop → fires the long-press handler once, disarms. A `pointer_move` past
  slop or a `pointer_up` before threshold disarms (→ a normal tap).

DSL: `c.on_long_press(node?, { |ui| … })` is awkward (no node handle). **Decision:**
long-press attaches to the LAST-built node via `c.long_press({ |ui| … })`
recorded against `self.size - 1` — the one post-hoc modifier, justified because
there is genuinely no other attach point and it reads cleanly right after the
widget. (This is the single exception to "builders return nil, no chaining"; it
attaches by node id, not by chaining.)

### 10. Scrollbars (paint-only)

When a list's content overflows its viewport, paint a **thumb**: a rounded rect
on the right edge whose size/position come from the ratios
`thumb_h = viewport * viewport/content`, `thumb_y = viewport * scroll/content`
(clamped to a min thumb height). No animation/fade. The thumb is recorded LAST
in the list's paint (after the clip/translate/restore, in viewport space) so it
overlays content and does not scroll with it. New `paint_scrollbar` helper; pin
asserts thumb geometry math for known viewport/content/scroll triples.

### 11. Horizontal scroll

`list` is vertical-only today (`scroll_of` is a Y offset; paint translates
`(0, -scroll)`; hit-test accumulates `oy`). **Decision:** add a **scroll-axis
param** to a horizontal list variant `hlist(viewport_w) do … end` (kind reuses
`kind_list` + a per-node `horizontal` flag, so all the clip/scroll/hit
machinery is shared). The place pass already stacks a list's children on Y; an
hlist stacks on X and clips/scrolls on X. `scroll`/`tick`/scrollbar all read the
node's axis flag. Audit first: `hit_tree`/`paint_tree` accumulate only `oy` and
translate only Y today — generalising to an axis-tagged offset is the bulk of
the work; pins mirror the vertical ones on the X axis.

## What F1/F2 keeps deferred (ticked into ROADMAP)

Wrap, virtualization, keyed diffing/row reuse, scroll animation/fades,
double-tap, `shrink`/overflow-shrink, leaf-level `grow` (wrap in a styled
container today), sub-click Float wheel precision. Each is additive on the same
pass + seams; none requires revisiting the own-flex-vs-Yoga decision.
