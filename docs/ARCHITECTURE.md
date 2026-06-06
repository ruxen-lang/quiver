# Architecture — the Ruxen GUI stack (and where `quiver` sits)

The stack mirrors Dart/Flutter's layering 1:1, delivered as **packages, not a
language change**. The Ruxen language (L0) stays general-purpose.

| Layer | Package | Flutter analogy | `unsafe`/FFI? |
|---|---|---|---|
| **L0 — Language** | ruxen core (unchanged) | Dart | — |
| **L1 — Engine** | [`canvas`](../../canvas) | `dart:ui` / the engine | Yes — the only such layer |
| **L2 — Framework** | **`quiver`** | the Flutter framework | **No — 100% safe ruxen** |
| **L3 — Apps** | your app | a Flutter app | No |

## The L1/L2 boundary

`quiver` talks to the world *only* through the surface `canvas` exposes:

- `Window` / surface — where the UI is drawn.
- an **event stream** (`Event`) — pointer/keyboard/resize/lifecycle.
- a `Canvas` — `begin_frame`/`end_frame`, `clear`, `draw_rect`, `draw_text`, …

`quiver` adds **no** platform code and **no** FFI. If it needs a new drawing
primitive, that primitive is bound in `canvas` (L1) and surfaced as a `Canvas`
method — never reached around `canvas` directly.

## Consistency model

**Draw-everything** (Flutter-style): `quiver` draws *all* widgets onto L1's
`Canvas` — no native controls — for pixel-identical output across platforms.

## Data flow (one frame)

```
SDL event pump ─► canvas Event stream ─► quiver dispatches to widget handlers
                                              │
                                  signal changes invalidate tracking scopes
                                              │
quiver paints dirty nodes ─► canvas Canvas (Skia) ─► Window surface ─► screen
```

## Internal shape of `quiver`

```
App.build  ──►  Ui (reactive graph: subscriptions + dirty scopes, &var Ui)
                  │
   DSL (Col: text/dyn_text/button) builds the flat node arena ONCE
                  │
   arrange ──► geometry ──► paint pass ──► PaintSurface (──► canvas Canvas)
                  │
   pointer_down ──► handler closures ──► State updates ──► flush ──► targeted repaint
```

See [`REACTIVITY.md`](REACTIVITY.md) for the arena/`Copy`-handle pattern and
[`DSL.md`](DSL.md) for the static-vs-reactive rule.
