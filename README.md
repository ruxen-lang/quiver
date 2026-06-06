# quiver

**L2 framework for the Ruxen GUI stack.** Fine-grained reactive signals, a
Ruby-block DSL, layout, and widgets — built on the L1 engine
([`canvas`](../canvas)).

`quiver` is **100% safe, platform-agnostic Ruxen**: no `unsafe`, no FFI, no
platform-specific code. All of that lives one layer down in `canvas`.

```
L0  Ruxen language        (unchanged — general purpose)
L1  canvas                    windowing + input + 2D canvas via C FFI   [unsafe/FFI]
L2  quiver   ← you are here   signals + block DSL + layout + widgets    [100% safe]
L3  your app
```

## The feel

```ruxen
def counter_view(count: Signal[Int]) -> Widget
  column do
    text "Counter"                    # static: built once
    text { "count: #{count.get}" }    # reactive: re-runs ONLY this node
    button "tap" do                   # static label; trailing block = handler
      count.update { |c| c + 1 }
    end
  end
end
```

See [`examples/counter.rx`](examples/counter.rx).

## The two ideas that make it work

- **Fine-grained signals (not rebuild-diff).** Updates touch *only* the exact
  bound nodes — minimal per-frame allocation. This is the no-GC smoothness story
  made real, and the reason we don't copy Flutter's rebuild-the-subtree-then-diff
  (which leans on a GC). See [`docs/REACTIVITY.md`](docs/REACTIVITY.md).
- **One DSL rule: `{}` = reactive, value = static.** A plain value is built
  once; a block/closure becomes a *tracking scope* that re-runs only its node
  when a signal it read changes. No JSX, no macros — just Ruxen blocks. See
  [`docs/DSL.md`](docs/DSL.md).

## Status

**Scaffold.** API skeleton (`Signal[T]`, `Widget`, `column`/`text`/`button`,
`run`) + docs are in place; the signal arena runtime, layout, and paint/diff are
not yet implemented. First milestone: the desktop counter app. See
[`docs/ROADMAP.md`](docs/ROADMAP.md).

## Docs

- [`docs/DESIGN.md`](docs/DESIGN.md) — the four concerns of L2 and why.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — the L0–L3 model and the L1/L2 boundary.
- [`docs/REACTIVITY.md`](docs/REACTIVITY.md) — signals vs ownership: the arena + `Copy`-handle pattern.
- [`docs/DSL.md`](docs/DSL.md) — the `{}`-reactive / value-static rule, precisely.
- [`docs/ROADMAP.md`](docs/ROADMAP.md) — milestones.

## License

Dual-licensed under either of [MIT](LICENSE-MIT) or
[Apache-2.0](LICENSE-APACHE) at your option.
