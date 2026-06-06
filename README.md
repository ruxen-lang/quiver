# quiver

**L2 framework for the Ruxen GUI stack.** Fine-grained reactive state, a
closure-builder DSL, layout, paint, and event dispatch — built on the L1
engine ([`canvas`](../canvas)).

`quiver` is **100% safe, platform-agnostic Ruxen**: no `unsafe`, no FFI, no
platform-specific code. All of that lives one layer down in `canvas`.

```
L0  Ruxen language        (unchanged — general purpose)
L1  canvas                    windowing + input + 2D canvas via C FFI   [unsafe/FFI]
L2  quiver   ← you are here   state + builder DSL + layout + paint     [100% safe]
L3  your app
```

## The feel

```ruxen
var app = App.build({ |ui: &var Ui, root: &var Col|
  let count = ui.state(0)
  root.text("Counter")                                          # static: built once
  root.dyn_text({ |ui2: &var Ui| "count: #{count.get(ui2)}" })  # reactive: re-runs ONLY this node
  root.button("tap", { |ui2: &var Ui|                           # click handler
    count.update(ui2, { |c| c + 1 })
  })
})
```

Run the demo: `cd examples/counter && ruxen run` — three synthetic taps
through the real dispatch → reactive → targeted-repaint pipeline, painting
to a recording surface (the desktop window activates when canvas L1 lands).

## The two ideas that make it work

- **Fine-grained reactivity (not rebuild-diff).** Updates touch *only* the
  exact bound nodes — minimal per-frame allocation. This is the no-GC
  smoothness story made real, and the reason we don't copy Flutter's
  rebuild-the-subtree-then-diff (which leans on a GC). See
  [`docs/REACTIVITY.md`](docs/REACTIVITY.md).
- **One DSL rule: `dyn_text` closures are the only things that re-run.**
  The builder block runs once; a `dyn_text` closure is a *tracking scope*
  that re-runs only its node when a state it read changes. No JSX, no
  macros — just Ruxen closures. See [`docs/DSL.md`](docs/DSL.md).

## Status

**Milestone 1, L2 side: complete.** Reactive core (`Ui` + `State[T]`),
column DSL (`Col`), layout, paint (`PaintSurface` + targeted repaint), and
pointer dispatch are implemented and pinned by 37 headless tests
(`ruxen test`). The desktop window path is blocked on `canvas` Milestone 1
(SDL + Skia + event stream). See [`docs/ROADMAP.md`](docs/ROADMAP.md).

Note: the implemented API differs from the early sketch in places (e.g.
`State[T]` rather than `Signal[T]`, explicit `&var Ui`, `dyn_text` as its
own method) — each deviation was forced by a measured ruxen-v1 behavior and
is documented in [`docs/DSL.md`](docs/DSL.md) and
[`docs/REACTIVITY.md`](docs/REACTIVITY.md).

## Docs

- [`docs/DESIGN.md`](docs/DESIGN.md) — the four concerns of L2 and why.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — the L0–L3 model and the L1/L2 boundary.
- [`docs/REACTIVITY.md`](docs/REACTIVITY.md) — state vs ownership: the explicit-arena pattern.
- [`docs/DSL.md`](docs/DSL.md) — the static-vs-reactive rule, precisely.
- [`docs/ROADMAP.md`](docs/ROADMAP.md) — milestones.

## Develop

```bash
ruxen test                      # 37 headless tests
ruxen build                     # library rlib (+ canvas dep piece)
cd examples/counter && ruxen run
```

## License

Dual-licensed under either of [MIT](LICENSE-MIT) or
[Apache-2.0](LICENSE-APACHE) at your option.
