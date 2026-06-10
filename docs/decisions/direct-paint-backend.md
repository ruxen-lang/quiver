# ADR: Direct-paint backend (drop the single-`PaintSurface` workaround)

Status: accepted — 2026-06-10
Context: quiver L2, branch `feat/l2-arena-nesting-layout`
Supersedes: the "single `PaintSurface` impl on purpose" note in
`docs/ROADMAP.md` (Milestone 1, canvas side) and the matching comment block in
`src/paint.rx` (lines ~51–63).

## Context

Until now every windowed example painted **twice** per frame:

1. quiver's generic paint pass (`paint_all` / `paint_dirty`) walked the node
   tree and recorded each primitive into `RecordingSurface` — the **sole**
   `PaintSurface` implementor, living inside quiver.
2. The example binary's `replay` then walked that recorded mirror
   (`op_at`/`x_at`/… parallel arrays) a **second** time and re-issued every op
   onto canvas's Skia `Canvas`.

The reason was a ruxen v1 limitation: the compiler only resolved quiver's
generic paint pass when `PaintSurface` had a **single** implementor (it
devirtualised the call rather than truly monomorphising), and it could not
monomorphise a dependency's generic for a type defined in the **consuming**
binary. So a second implementor — e.g. a Skia-backed one in the app binary —
made the generic body emit an unresolved `S: PaintSurface_*` symbol that failed
to link. The record→replay split was the workaround: keep the only implementor
inside quiver, mirror the ops as plain Ints/Strings, rewalk in the binary.

**ruxen Q17 has landed and is installed.** A generic FREE FUNCTION bounded by a
mixin (`def paint_all[S: PaintSurface](…)`) now monomorphizes per concrete
implementor — including an implementor defined in the consuming binary. Verified
on the installed CLI (`ruxen 0.1.0`) both by the two-implementor shape note in
the task brief and by a local probe in this repo (a second in-test
`PaintSurface` class driven through `paint_all` alongside `RecordingSurface`
compiled and ran clean). The reason for the double pass is gone.

## Decision

Each example binary defines its **own** `PaintSurface` implementor —
class `SkiaPaint` — that issues canvas/Skia calls **directly** as quiver's
generic paint pass visits each node. The windowed frame becomes a single walk:

```
begin_frame → paint_all(&app, &var skia_paint) → end_frame → present
```

No `RecordingSurface`, no `surface.reset`, no `replay` on the windowed path.

### Implementor shape

`SkiaPaint` is a **class** (ruxen Q35: a struct's `include` does not yet satisfy
a generic bound — classes do). It:

- `include`s `PaintSurface` and has an explicit `def init` (landmine: classes
  need explicit `init`).
- holds a borrow of the canvas as a field. The window owns the `Canvas`
  (`win.canvas`); `SkiaPaint` is constructed **per frame**, inside the
  `begin_frame`/`end_frame` bracket, borrowing `&var win.canvas` so the lifetime
  is exactly the frame. (A per-frame value class is cheap — no allocation of an
  op list, which is the whole point.)
- maps each `PaintSurface` op 1:1 onto the **already-proven** canvas calls that
  currently live in `replay` — same `Rect.new`, same `Color.rgb`, same
  `draw_rect`/`draw_round_rect`/`stroke_round_rect`/`save`/`clip_rect`/
  `translate`/`restore`/`draw_text`. The mapping is a transliteration of the
  existing `replay` body from "read the i-th recorded op" to "handle this op
  directly", so behaviour is identical by construction.

The op→canvas mapping (unchanged from `replay`):

| PaintSurface method        | canvas call                                            |
|----------------------------|--------------------------------------------------------|
| `fill_clear(r,g,b)`        | `clear(Color.rgb)`                                     |
| `fill_rect(...)`           | `draw_rect(Rect, Color)`                               |
| `draw_text(&t,x,y,...)`    | `draw_text(t, x, y, Color)`                            |
| `fill_round_rect(...)`     | `draw_round_rect(Rect, radius, Color)`                 |
| `stroke_round_rect(...)`   | `stroke_round_rect(Rect, radius, width, Color)`        |
| `push_clip(x,y,w,h)`       | `save` then `clip_rect(Rect)`                          |
| `push_translate(dx,dy)`    | `translate(dx, dy)`                                    |
| `pop_state`                | `restore`                                              |

Note `push_clip` issues TWO canvas calls (`save` + `clip_rect`) — matching how
`RecordingSurface.push_clip` records two ops and `replay` replayed them. The
direct backend keeps that exact pairing, so save/restore nesting is identical.

### What stays — the recording path is load-bearing, not legacy

`RecordingSurface` remains, unchanged, as:

- **quiver's test double** — the entire headless suite (`tests/*.rx`) pins paint
  output by formatted-string narration (`command_at`) and structured fields
  (`op_at`/`x_at`/…). That coverage is canvas-free and must not regress.
- **the headless example path** — each example's `run_headless` (CI / no
  display) still paints into `RecordingSurface` and narrates, so the apps run
  anywhere. Only the **windowed** path switches to `SkiaPaint`.

So both backends coexist in each binary: `RecordingSurface` for headless,
`SkiaPaint` for the live window. This is precisely the "framework drives N paint
backends" property Q17 unlocks — and it is what the new pin test asserts inside
quiver itself (see below).

### src/paint.rx audit

Audited the generic pass for any accidental single-impl assumption. **None
found** — every paint free fn is already correctly bounded
(`paint_all[S: PaintSurface]`, `paint_dirty[S]`, `paint_tree[S]`, `paint_node[S]`,
and every per-widget `paint_*[S]`). The only concrete `RecordingSurface`
references in `src/` are its own class body and a doc line in `lib.rx`; none sit
in a paint-pass signature. **No code change to the generic pass is required** —
the only edits to `src/paint.rx` are the doc comment block (lines ~51–63), which
described the now-obsolete single-impl constraint.

### Pinning the generic-ness inside quiver (canvas-free, forever)

The suite cannot see canvas, so it cannot exercise `SkiaPaint`. To stop a future
change from silently regressing back to a single-implementor assumption (e.g.
someone narrowing a signature to `&var RecordingSurface`), a new test
`tests/multi_backend.rx` defines a **second** in-suite `PaintSurface`
implementor — `TallySurface`, a per-op counting fake — and asserts quiver's
paint pass drives **both** `RecordingSurface` and `TallySurface` correctly in
**one** program over the **same** tree: the tally's per-op counts equal the op
breakdown of the recording's op list. This pins "the framework supports N paint
backends" without ever touching canvas.

## Consequences

### Win (the point of the change)

The windowed hot loop no longer allocates and walks a recorded op list each
frame. Before: **walk tree (record) + rewalk op list (replay)** = two passes
plus an op-array allocation (`reset` then re-`push` every op) every frame.
After: **one** tree walk that paints directly. For an N-op frame this removes N
array pushes, N array reads, and the whole `RecordingSurface` reset/regrow
churn from each repaint. Structural win, not a micro-opt: one paint walk instead
of walk+record+rewalk.

`paint_dirty` (targeted repaint) goes through the same generic pass, so dirty
repaints get the direct backend too — no per-node mirror round-trip.

### Cost / risk

- Each example now defines `SkiaPaint`. It is ~30 lines of mechanical op→canvas
  mapping per binary, lifted verbatim from the `replay` body it replaces. Net
  lines roughly neutral (delete `replay` + `blit`'s reset/replay, add
  `SkiaPaint`).
- `SkiaPaint` is constructed per frame holding `&var win.canvas`. The closure /
  field captures the canvas borrow plainly (never `move` — non-Copy class,
  landmine). One borrow of `win.canvas` per frame, released at frame end.
- Landmine watch during implementation: (a) never pass a non-Copy class
  call-result directly as a method arg — let-bind `Rect.new(...)` / `Color.rgb`
  before passing if a method arg ever takes one (canvas's are methods on
  `canvas`, taking `Rect`/`Color` by value — so `let rect = Rect.new(...)` then
  `canvas.draw_rect(rect, col)`, exactly as `replay` already does); (b)
  paren-less call landmines; (c) capture of the `Window`/`Canvas` handle is a
  plain capture.

## Alternatives considered

- **Keep record→replay, just delete one pass.** Not possible — the two passes
  exist *because* the second (canvas) backend couldn't be a `PaintSurface`. With
  Q17 it can, so the mirror is pure overhead on the windowed path.
- **Make `RecordingSurface` itself hold a canvas handle.** Rejected: that drags
  canvas symbols into quiver's `src/` (charter violation — quiver is canvas-free)
  and couples the test double to a backend. The implementor belongs in the
  binary, the only place L2+L1 symbols meet.
- **A struct `SkiaPaint`.** Rejected: ruxen Q35 — a struct's `include` does not
  yet satisfy a generic bound. Use a class.
