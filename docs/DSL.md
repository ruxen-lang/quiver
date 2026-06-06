# The DSL — `{}` = reactive, value = static

`quiver`'s builder DSL is plain Ruxen blocks/closures — **no JSX, no `view!`
macro, no language change.** (JSX was rejected: it would need a custom-syntax
macro facility; the Ruby-block form is native to a Ruby-flavored language.)

## The one rule

The builder block runs **once** to construct the node tree. Reactivity
granularity comes from a single, unambiguous rule:

| You write | Meaning |
|---|---|
| a **plain value** (e.g. `text "Counter"`) | **static** — built once, never re-evaluated |
| a **block / closure** `{ … }` (e.g. `text { "#{sig.get}" }`) | **reactive** — a *tracking scope*: subscribes to the signals read inside it and re-runs **only that node** when they change |

## Worked example

```ruxen
def counter_view(count: Signal[Int]) -> Widget
  column do
    text "Counter"                    # static: built once
    text { "count: #{count.get}" }    # reactive: re-runs ONLY this node when count changes
    button "tap" do                   # static label; trailing block = click handler
      count.update { |c| c + 1 }
    end
  end
end
```

- `column do … end` — a container; its block builds children **once**.
- `text "Counter"` — static text; never re-evaluated.
- `text { … }` — reactive text; the closure is a tracking scope. When `count`
  changes, only this node re-runs and repaints.
- `button "tap" do … end` — static label; the **trailing block is the click
  handler**, dispatched from L1's event stream.

This gives Solid/Leptos-grade granularity using only Ruxen's existing
blocks/closures.

## Why it must be pinned with tests

The static-vs-reactive boundary is the single rule users rely on for
performance and correctness. It must be unambiguous and well-documented, and
pinned with tests in the L2 core slice so it can never silently drift.
