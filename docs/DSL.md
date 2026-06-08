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

## Containers: `row` / `col` and arena nesting

A node can own children. The arena stays flat (no recursive type, per the
landmines): `Col` carries tree-link hashes `parent` / `child_first` /
`child_last` / `child_next` (node-id → node-id; absent ⇒ `-1`), a top-level
root chain (`root_first`/`root_last`), and a build cursor `parent_cursor`.
Node ids remain build-order indices — the "ids ARE node ids" invariant holds.

```ruxen
root.text("title")                  # node 0, a top-level (root) leaf
root.row({ |c: &var Col|            # node 1, a container; cursor moves onto it
  c.text("left")                    # node 2, child of the row
  c.col({ |c2: &var Col|            # node 3, a nested container
    c.text("a")                     # node 4, child of the col
    c.text("b")                     # node 5, child of the col
  })
})
```

`row` (X axis) and `col` (Y axis) push a container node, move the cursor onto
it, run the block, then restore the cursor — so nesting is arbitrary. The
layout pass (`App.arrange`, docs/LAYOUT.md) walks this tree; containers paint
nothing, their children paint themselves.

### Deviation: reactive children inside a container are deferred (ruxen Q21)

The first slice ships **static `text`** inside `row`/`col`. A `dyn_text` or
`button` built *inside* a container would store a closure that captures state
(e.g. a `State` handle), and in this ruxen build a capturing closure stored
through the container's `build.(&var *self)` self-reborrow has its captures
**corrupted** (wrong value for an `Int`; SIGSEGV for a captured class handle).
The top-level `App.build` is unaffected (it invokes the builder as
`f.(&var local.field)`, never a self-reborrow). So reactive *children* are
held behind ruxen Q21 — `tests/nesting.rx` keeps that assertion as an `xit`
pending, and reactive content lives at the top level until ruxen fixes
capture under the reborrow. See `CLAUDE.md` landmines (Q21) and
`tmp/test-cache/ruxen-closure-capture-reborrow.md`.

## Why it must stay pinned with tests

The static-vs-reactive boundary is the single rule users rely on for
performance and correctness. `tests/app.rx` ("pin: the static/reactive
boundary") asserts that across any number of clicks only the `dyn_text`
node is ever refreshed; `tests/counter.rx` asserts one repaint per tap,
zero on idle frames. If those tests change, the rule changed — that is a
breaking design event, not a refactor.
