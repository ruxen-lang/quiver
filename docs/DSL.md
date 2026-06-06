# The DSL — `dyn_text` = reactive, `text` = static

`quiver`'s builder DSL is plain Ruxen closures — **no JSX, no `view!` macro,
no language change.** The builder block runs **once** to construct the node
tree; reactivity granularity comes from which builder method you call.

## The one rule

| You write | Meaning |
|---|---|
| `root.text("Counter")` | **static** — built once, never re-evaluated |
| `root.dyn_text({ \|ui\| … })` | **reactive** — the closure is a *tracking scope*: it subscribes to the states it reads and re-runs **only that node** when they change |
| `root.button("tap", { \|ui\| … })` | static label; the closure is the **click handler** |

## Worked example (the shipped counter)

```ruxen
var app = App.build({ |ui: &var Ui, root: &var Col|
  let count = ui.state(0)
  root.text("Counter")                                          # static
  root.dyn_text({ |ui2: &var Ui| "count: #{count.get(ui2)}" })  # reactive
  root.button("tap", { |ui2: &var Ui|                           # handler
    count.update(ui2, { |c| c + 1 })
  })
})
```

- `App.build` runs the block **once**; `root` is the column under
  construction (a flat arena of nodes — see below).
- The `dyn_text` closure is a tracking scope. When `count` changes, only
  this node recomputes and repaints (`tests/app.rx`, `tests/paint.rx` pin
  this).
- Handler and reactive closures receive the reactive graph as
  `&var Ui` — safe Ruxen has no ambient runtime to reach for
  (docs/REACTIVITY.md).

## Deviations from the design sketch, and why

The sketch spelled this `text "x"` / `text { … }` / `button "tap" do … end`.
Prototyping against ruxen v1 forced three changes, each pinned by a probe:

1. **`text` vs `dyn_text` are separate methods.** Overloading one name on
   `&str` vs closure miscompiles (heap corruption) in ruxen v1.
2. **Stored closures use `{ … }` braces, not `do … end`.** `do…end` blocks
   passed to *free functions* with explicit closure parameters segfault;
   methods accept them, but braces work everywhere and we use them
   uniformly for stored callbacks.
3. **The tree is a flat arena** (parallel arrays in `Col`, node id = index),
   not a recursive `Widget` class: recursive class types crash the v1
   compiler, and `Option[any Fn]` fields miscompile. `-1` sentinels in
   `compute_of` / `handler_of` replace the option types.

None of these changes the model — build once, `dyn_text` closures are the
only things that ever re-run.

## Why it must stay pinned with tests

The static-vs-reactive boundary is the single rule users rely on for
performance and correctness. `tests/app.rx` ("pin: the static/reactive
boundary") asserts that across any number of clicks only the `dyn_text`
node is ever refreshed; `tests/counter.rx` asserts one repaint per tap,
zero on idle frames. If those tests change, the rule changed — that is a
breaking design event, not a refactor.
