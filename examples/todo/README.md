# todo — dynamic list (structural reactivity)

A small todo app that proves quiver's **structural reactivity**: the UI grows
and shrinks in response to state. Where `examples/settings` reacts on *values*
over a fixed layout, this one adds/removes *rows*.

## Run it

```bash
cd examples/todo
ruxen build
./target/debug/todo      # windowed if a display exists, else headless demo
```

## How it works

The list is a **`list_of`** node bound to a shared **`ListModel`**:

```ruxen
let todos = ui.list_model                       # a shared, mutable item model

root.button("Add", { |u| todos.add(u, "task") })  # handler CAPTURES the model
root.list_of(todos, row_height() * 6, { |c, m, i| # …and the list takes it too
  c.text(m.row_text(i))
})
```

- `ui.list_model` mints a `ListModel` — a `Send` class holding the items
  (labels + done flags) plus a reactive `State[Int]` **version**.
- `todos.add(ui, …)` / `toggle(ui, k)` / `remove(ui, k)` edit the items **and**
  bump the version through one `State.update` (single-lock RMW). The bound
  `list_of` subscribes to that version, so a mutation marks it dirty.
- On the next `flush`, the dirty `list_of` **rebuilds its child subtree** from
  the current items (clear its child chain, build one row per item), then
  re-`arrange`s + repaints — just that subtree; the header and buttons are
  untouched. It reuses the scroll/clip `list` machinery, so it scrolls.

### Two patterns worth copying

1. **Capture the model in handlers, then pass it to `list_of`.** A `ListModel`
   is a non-Copy handle and a *method call* consumes it, but a *closure capture*
   is a non-consuming pointer-copy. So capture it in the button handlers first,
   then pass it by value to `list_of` last:

   ```ruxen
   let todos = ui.list_model
   root.button("Add", { |u| todos.add(u, "x") })   # capture (non-consuming)
   root.list_of(todos, …, …)                        # move (consuming) — last
   ```

2. **Re-fetch the model for ad-hoc mutations.** Outside a captured handler, get a
   fresh copy with `app.list_model_of(list_node_id)` per mutation (each copy
   shares the same items).

## What's deferred (this is v1)

- **Keyed diffing** — a mutation rebuilds the whole subtree (coarse but correct);
  rows are not reused across changes, so node ids grow over a session.
- **Per-item reactive widgets** bound to per-item `State` — v1 rows are built
  from item strings + model-capturing handlers.
- **Row reordering / drag, animation.**

See `docs/decisions/structural-reactivity.md` for the full design and the ruxen
limitations it works around.
