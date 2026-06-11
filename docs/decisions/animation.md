# ADR: Animation engine (F4) — explicit tweens + implicit transitions on the clock seam

Status: **Accepted** (2026-06-11). Phase 2, F4. Builds directly on the F2 clock
seam (`App.set_clock` / `App.tick` / the deterministic frame counter) and the
flat-arena + index-pool discipline that every other quiver subsystem uses.

## Context

F2 already proved the hard part: `App.tick` advances the fling stepper
deterministically off the injected `set_clock` seam, and the example window loops
already call `app.tick` once per frame. F4 generalises that single-purpose
stepper into a real **animation registry** — per-node, per-property tweens that
the same `tick` advances from the same clock delta, with the same determinism
guarantees the fling pins rely on.

The charter constraints are unchanged: 100% safe Ruxen, no FFI, flat parallel
arrays + `-1`/sentinel indices (no recursive types, no `Option[any Fn]` fields),
one `Mutex` lock per frame, all behaviour pinned under a fake clock.

## What paint actually consumes (the animatable-property audit)

Decisive for picking the Tier-1 property set. quiver's `PaintSurface` boundary
is **Ints + &String only** (`fill_rect`, `draw_text`, `fill_round_rect`,
`stroke_round_rect`, clip/translate/restore). Geometry crosses as Int px; colour
crosses as three Int channels (r, g, b 0-255). There is **no alpha/opacity op**
on the mixin — canvas *does* expose `save_layer_alpha` / `set_blend_mode`, but
surfacing group opacity through quiver's pass means a new op on the mixin, a new
op code, and a save-layer bracket in `paint_tree` around an animating subtree —
deep plumbing for one property.

**Decision — Tier-1 animatable set:**

| Property | What animates | Reads into paint via |
|---|---|---|
| `anim_offset_x` / `anim_offset_y` | a per-node paint-time translate (px) | a new translate applied in `paint_tree` |
| `anim_color_r/g/b` | an override colour (0-255 each) lerped | the node's existing fill/text colour |

- **Geometry (offset).** An *animation* offset is distinct from the static
  `Style.offset(l, t)` (which is a stack-positioning layout input). The animated
  offset is a **paint-time** translate, NOT a layout input — animating it must
  not re-run `arrange` (that would defeat targeted repaint and fight the
  build-once rule). So an offset tween shifts where a node *draws*, leaving its
  laid-out geometry — and every sibling's — untouched.
- **Colour.** RGB lerp is trivial Int math and the paint ops already take r/g/b,
  so a colour tween is a pure read-side override with zero new ops.
- **Size (w/h) is DEFERRED.** Animating a node's box *is* a layout input — it
  would force a re-`arrange` per frame (reflowing siblings), which is a different
  and much costlier contract than a paint-time offset/colour override. Filed as a
  revisit trigger; geometry-via-offset covers the slide-in/nudge motions the
  showcase needs without touching layout.
- **Opacity is DEFERRED** (no paint op; needs a mixin + pass change). Filed.

This keeps F4 entirely on the **read side** of paint: a tween never invalidates
geometry, only the colour/translate a node paints with. That is what lets a
single animating node repaint *itself* via the existing targeted-repaint path
without re-arranging anything.

## Fixed-point curves (quiver's design space is Int)

Curves compute in **scaled-Int fixed point**, never Float. The canonical
progress variable is `p ∈ [0, ANIM_SCALE]` where `ANIM_SCALE = 1000` (per-mille).
`p = elapsed * ANIM_SCALE / duration` (Int division; clamped to `[0, 1000]`).

Curve functions map `p → eased ∈ [0, 1000]`:

```
linear:        e = p
ease_in:       e = p*p / 1000                       # quadratic
ease_out:      e = 1000 - (1000-p)*(1000-p)/1000    # quadratic
ease_in_out:   p < 500 ? 2*p*p/1000
                       : 1000 - 2*(1000-p)*(1000-p)/1000
```

The lerped value is `from + (to - from) * eased / 1000` (Int). **Precision
note:** with `ANIM_SCALE = 1000` the worst-case rounding error on a single
channel/axis is < 1 unit — invisible at px / 0-255 granularity, and *exactly*
reproducible (no Float non-determinism). The end value is **clamped to `to`** on
retirement so a finished tween lands on its target byte-exactly regardless of
rounding along the way. Curve constants are plain Ints (the node-kind idiom):
`curve_linear = 0`, `curve_ease_in = 1`, `curve_ease_out = 2`,
`curve_ease_in_out = 3`.

## The registry (flat arrays + an index pool — the arena idiom)

A live animation is a row in parallel arrays on `App` (NOT a class with
`Option`/recursive fields). One row per (node, property-channel) tween:

```
anim_node:     Array[Int]    # target node id
anim_prop:     Array[Int]    # which property channel (prop_off_x / _off_y / _color)
anim_from:     Array[Int]    # start value  (px, or packed rgb — see below)
anim_to:       Array[Int]
anim_start:    Array[Int]    # clock time the tween began (from `now`)
anim_dur:      Array[Int]    # duration in clock units
anim_curve:    Array[Int]
anim_live:     Array[Bool]   # false = retired slot, reusable
anim_cb:       Array[Int]    # index into anim_callbacks pool, -1 = none
anim_callbacks: Array[any Fn[Fn(&var Ui) -> nil]]   # STORED completion closures
```

- **Colour packs into one row** as three sub-channels is awkward; instead a
  colour tween is **three rows** (one per r/g/b channel, `prop_color_r/g/b`) so
  every row is a scalar Int lerp — uniform stepping code, no packing math. The
  completion callback rides on the LAST channel row only (fires once).
- **Retirement frees the slot in place** (`anim_live[i] = false`); the next
  `tween` reuses the lowest dead slot (a free-list scan — the pool pattern, not
  a growing array). The slot's resolved value is written to the node's
  `anim_offset_*` / `anim_color_*` override hashes so paint reads the final.
- **Stored completion closures** live in `anim_callbacks` (the `handlers` /
  `long_press_handlers` pattern), indexed by `anim_cb` (`-1` = none) — never an
  `Option[any Fn]` field.

Per-node **resolved overrides** (what paint reads) are plain hashes, absent ⇒ no
override (paint uses the node's intrinsic value):

```
anim_off_x_of: Hash[Int, Int]   # paint translate dx for node i
anim_off_y_of: Hash[Int, Int]
anim_col_r_of: Hash[Int, Int]   # colour override channels (set together)
anim_col_g_of: Hash[Int, Int]
anim_col_b_of: Hash[Int, Int]
anim_has_col_of: Hash[Int, Bool]
```

## Stepping (`App.tick` drives it, exactly like fling)

`App.tick` already advances the deterministic clock and steps flings. F4 adds
`step_animations` to the same `tick` body (so fling + tween coexist, both
stepping off the one clock advance per tick — pinned):

```
def var tick -> Bool
  if !self.has_clock { frame_counter += 1 }
  var changed = self.step_flings
  if self.step_animations { changed = true }
  if self.fire_long_press { changed = true }
  changed
end
```

`step_animations` walks live rows: for each, `elapsed = now - start`; compute the
eased value; write it into the node's resolved-override hash; mark the node dirty
(`ui.mark_dirty(node)`) **only if its resolved value actually changed this step**
(dirty-set exactness — a non-animating sibling is never marked). When
`elapsed >= dur`: clamp to `to`, write the final override, fire the completion
callback if present, set `anim_live[i] = false`. Returns true if any row moved.

**Dirty-set exactness is the whole point** (quiver's reactivity charter): only
nodes with a live, *moving* tween get marked; `flush` then repaints exactly those
via the existing `paint_dirty` path. The pin asserts that animating node A
repaints while sibling B (no tween) records zero paint ops across the steps.

## Paint integration (read-side only)

`paint_tree` / `paint_node` read the override hashes:

- Before painting node `i`, if it has an `anim_off_x/y` override, the pass
  emits a `push_translate(dx, dy)` … `pop_state` bracket around the node's draw
  (offset is paint-time, so it shifts the node and its subtree without touching
  geometry). Default (no override) ⇒ no bracket ⇒ byte-identical to today.
- Colour-consuming draws (`draw_text`, button fill, container bg) read the
  node's `anim_col_*` override when `anim_has_col_of` is set, else the intrinsic
  colour. A single helper `paint_color_of(app, i, default_r/g/b)` centralises the
  override-or-default choice so every call site is one decision point.

`paint_dirty` (the targeted path the animation stepper triggers) repaints a node
the same way it does for a reactive text change — clears the node's row band and
re-paints it with its current override. The offset bracket applies there too.

## DSL surface

Explicit tweens are an **App-level** call (animations are runtime state on `App`,
like scroll/caret/capture — they are not tree structure, so they are NOT a `Col`
builder). The node is named by id (the same way `scroll_to`, `long_press`,
`drag_to` name nodes):

```
app.tween_offset(node, from_x, from_y, to_x, to_y, duration, curve)
app.tween_color(node, from_r,g,b, to_r,g,b, duration, curve)
app.tween_offset_done(node, …, { |ui| … })   # with a STORED completion closure
```

- Completion callbacks are STORED `{ |ui| … }` brace closures (the house
  do/brace rule — stored behaviour), pooled in `anim_callbacks`.
- A new tween on a node+property **supersedes** any live tween on the same
  node+channel (retire the old row, no double-step) — so re-triggering a hover
  animation mid-flight is well-defined.
- `app.animating?` / `app.animating_node?(i)` report liveness (so a shell knows
  to keep drawing; `tick`'s bool return already covers the loop condition).

**Repeat / yoyo: DEFERRED** — cheap to add later (a `repeat` flag + a direction
bit per row that re-seeds `start`/swaps `from`/`to` on retirement) but not needed
for the showcase; filed as a revisit trigger to keep F4 honest-scoped.

## Implicit transitions (Tier-1)

A node marked `transition(duration, curve)` animates *from its current resolved
value to a new one* instead of snapping, when an animatable property changes.
**Honest-scope decision:** implicit transitions on the *colour* channel are
landed (a `transition_color(node, dur, curve)` mark + a `set_color(node, r,g,b)`
that, if the node is marked, starts a colour tween from the current resolved
colour instead of writing instantly). Implicit *offset* transitions and
hooking transitions into reactive `dyn_text` colour changes are **filed
precisely** (the plumbing to intercept every colour write site is deep and not
needed for the showcase). The acceptance bar that may not slip — *explicit tweens
fully deterministic* — is met by the explicit-tween path above; implicit
transitions are the additive layer on top.

## Pins (all deterministic under the fake clock)

`tests/animation.rx`:

1. **Tween midpoints per curve** — at a known tick, `linear` / `ease_in` /
   `ease_out` / `ease_in_out` each yield their exact fixed-point value.
2. **Retirement** — a tween clamps to `to` byte-exactly at `elapsed == dur`,
   frees its slot, and a subsequent `tween` reuses the freed slot.
3. **Dirty-set exactness** — animating node A is marked dirty each moving step;
   a non-animating sibling B is never marked and records zero paint ops.
4. **Fling + tween coexist** — both step from one `tick`; offsets and scroll both
   advance on the same clock unit.
5. **Completion callback fires exactly once** — at retirement, not before, not
   twice (re-tick after retirement does not re-fire).
6. **Colour lerp** — three-channel lerp produces the expected r/g/b at a tick and
   the exact target at retirement.
7. **Supersede** — a second tween on the same node+channel retires the first
   (no double-step, no leaked live slot).

## Consequences

- `App` grows the animation arena (parallel arrays + override hashes + the
  callback pool) and `step_animations`; `tick` gains one call. No change to
  `arrange`, `flush`'s reactive contract, or the build-once rule — animation is
  pure runtime state read by paint, never tree structure.
- Paint gains override-aware colour + an offset translate bracket; with no live
  animation every existing paint pin is byte-identical (defaults inert).
- The showcase (`examples/settings` save button colour pulse OR `examples/todo`
  row slide-in) runs live for free — the window loop already calls `app.tick`.

## What this defers (revisit triggers)

- **Size (w/h) tweens** — need per-frame re-`arrange` (a layout-input animation),
  a different cost contract; revisit when a widget genuinely needs animated
  reflow.
- **Opacity / group alpha** — needs a `PaintSurface` opacity op + a save-layer
  bracket in the pass (canvas has `save_layer_alpha`); revisit when fades are
  required.
- **Repeat / yoyo / delay / spring curves** — additive on the same row schema.
- **Implicit offset transitions + reactive-colour transitions** — additive on the
  explicit-tween core.
- **Stagger / timeline sequencing** — compose multiple tweens; out of scope now.
