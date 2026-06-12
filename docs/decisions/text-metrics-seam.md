# ADR: Real text metrics via an injected measure seam

Status: **Accepted** (2026-06-10). Implemented in the same change.

Role trail (this change): **ARCHITECT** (this ADR) → ENGINEER (seam + plumb
through `arrange`) → REVIEWER (adversarial pins) → OPTIMIZER (measure-once proof).

## Context

`App.arrange` sizes a text leaf to a **char-metric estimate**:
`leaf_w(i) = (label.size) * char_width() + pad_x() * 2`, and a checkbox to
`pad_x + box + pad_x + (label.size) * char_width + pad_x`. Every glyph is
assumed `char_width()` (8px) wide. That is the last unchecked **Layout** item in
`docs/ROADMAP.md`: *"Real text metrics — replace char-metric estimates with
canvas's Skia `measure_text`."*

canvas (L1) now exposes true advance width:
`Canvas#measure_text(text) -> Int` and `Canvas#measure_text_sized(text, size)
-> Int` over a sound `&String` FFI (ruxen Q29 audit). The naive move would be to
call it from `arrange`. **We cannot:** quiver is **canvas-free by design** — the
charter (`docs/ARCHITECTURE.md`, the layout ADR `docs/LAYOUT.md`) forbids any
FFI/`unsafe` at L2, and `Ruxen.toml` no longer even declares canvas as a
dependency (commit `4311393`). `src/` and `tests/` must reference **zero** canvas
symbols; only a **binary** package (an example shell) is allowed to name both
quiver and canvas symbols at once (ruxen v1 Q16/Q17 — library/test builds don't
merge a dependency's sources; binaries do).

So real metrics must be **injected through a seam**, never imported.

## Decision

### The seam shape

Add a measure hook on `App`, set by the **shell binary** that owns the canvas
handle:

```
app.set_measure({ |text: String, size: Int| Int })   # advance width in px
```

- The closure takes the display string and a logical font size (px) and returns
  the advance width in px. Quiver passes its single slice font size
  (`text_size()`, a new layout constant = the size the shell renders at) so the
  injected metric matches what canvas will actually paint.
- **Default = the existing char-metric estimate.** With no measure set,
  `arrange` produces byte-for-byte the geometry it produces today. Headless runs,
  the test suite, and any example built without Skia keep working **identically** —
  no `Err`, no behavioural cliff, no required wiring. Absence is the zero value.

### Respecting Q2 (no `Option[any Fn]` fields)

An `Option[any Fn]` field stores garbage in ruxen v1. quiver already solves this
everywhere (dyn_text computes, button handlers, item builders): **a pool array +
a presence flag/index.** The measure closure follows the exact same pattern —
NOT an `Option[Fn]`:

```
measure_fns: Array[any Fn[Fn(String, Int) -> Int]]   # 0 or 1 entry
has_measure: Bool                                    # presence flag
```

`set_measure` pushes into a freshly-reset `measure_fns` (always index 0) and sets
`has_measure = true`. The measure path reads `measure_fns.get(0)`. This keeps the
established "store closures in a pool, index/flag into it" discipline; no new
landmine surface.

### Where measurement happens (the flat pre-pass property is preserved)

`measure_leaves` is already the **single** flat pre-pass that resolves each
leaf's intrinsic width exactly once per `arrange` and stashes it on the arena
(`Col.push_iw`); the recursive measure/place reads it back via `iw_of` and
re-measures nothing. That property is the whole point of the OPTIMIZER bar, and
it is **unchanged**: the only edit is *how* the per-leaf width is computed inside
that one pre-pass —

- `leaf_w(i)` → `text_width(text_of(i))`
- `checkbox_w(i)`'s label term → `text_width(label_at(i))`

where `text_width(s)` is a new private helper: call the injected closure when
`has_measure`, else fall back to `(s.size) * char_width()`. So a node is measured
**once per `arrange`** — and `arrange` runs only on build, on a flush that
changed something, on scroll, and on resize, exactly as before. A `dyn_text`
whose text changes is re-measured because `flush` re-`arrange`s (its new cached
string flows through `text_of` → `text_width` on the next pre-pass). Static text
never re-measures after the first `arrange` unless the measure fn is (re)set
(which would call `arrange` again). No per-frame, per-flush re-measure of static
text.

### What gets measured (audit of `arrange`)

- **text / dyn_text / button** → `leaf_w` → now `text_width`. **In scope.**
- **checkbox** → `checkbox_w`'s label term → now `text_width`. **In scope.**
- **input / slider / select** → **FIXED** box widths (`input_width_at`,
  `slider_width_at`, `select_width_at`) declared at build time; their *box*
  geometry does not depend on text content, so layout does not char-measure them.
  The select trigger and input caret paint text *inside* a fixed box — a caret /
  trigger refinement that uses metrics is a **separate, additive** follow-up
  (filed under "deferred" below), not part of this layout item. **Out of scope.**

So the seam touches precisely the two char-metric width computations that feed
`arrange`: leaf text and checkbox labels.

### Failure / absence semantics

- No injected measure → char-metric fallback. No `Err`, no panic, no log.
- The closure returns an `Int`; quiver treats it as an opaque px width. A
  pathological 0/negative is the shell's contract to avoid (canvas returns a real
  advance ≥ 0); quiver does not second-guess it (same trust we give
  `viewport_h`, `width`, etc.).

## Consequences

- **`app.rx` / `counter.rx` stay green untouched.** Those pin char-metric widths
  (`7 * char_width() + pad_x()*2`); with no measure injected, the default path is
  byte-identical, so the design pins do not move. Changing them would be a design
  event — this change does not.
- The three example shells (counter, todo, settings) — the only places both
  symbol sets meet — inject `measure_text_sized` through the seam right after
  `App.build`, capturing the canvas handle under the known capture landmines
  (plain capture, not `move`; `let`-bind non-Copy call results before passing
  them as method args). Headless fallback in each shell simply skips the
  injection, so the headless path is the char-metric path.
- New public API: `App.set_measure(f)` + the `text_size()` layout constant. DSL.md
  / REACTIVITY.md note the growth.

## What this defers (revisit triggers)

- **Metric-aware caret + select-trigger text fit** — the input caret x and the
  select trigger's text/▾ placement still use char metrics in *paint*. Additive:
  route them through the same `text_width` once a paint-side metric query is
  wanted. Not a layout-geometry concern, so out of this item.
- **Per-glyph / cursor hit positioning inside an input** (click-to-place caret by
  x) — needs prefix-width measurement; additive on the same seam.
- **Font family / weight / per-node size** — the seam passes one slice font size
  today (one font for the slice, per `app.rx`'s header). A richer measure
  signature (family, weight) is an additive widening of the closure type when
  styling grows fonts.
- **Caching measured widths across arranges** — today each `arrange` re-runs the
  one pre-pass (measure-once *per arrange*, the established contract). A content-
  keyed metric cache (skip the FFI call when a leaf's string is unchanged between
  arranges) is a pure optimization, additive, and only worth it if the FFI call
  shows up in a profile.
