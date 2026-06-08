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
