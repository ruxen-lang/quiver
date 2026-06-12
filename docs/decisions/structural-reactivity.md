# ADR: Structural reactivity (reactive lists / dynamic children)

Status: **Accepted** (2026-06-09). Implemented in the same change.

## Context

quiver does **fine-grained content reactivity**: `App.build` runs the builder
block **once** to construct a fixed node tree; `flush` re-runs only the
`dyn_text`/compute closures of *existing* nodes (their tracking scopes) and
re-`arrange`s. The tree's *structure* is immutable after build.

That blocks every dynamic app — a todo list, a feed, any "add / remove a row"
UI. The `examples/settings` panel surfaced this: it can react on *values* over a
*fixed* layout, but cannot grow a list. We need a way for a node to **gain and
lose children in response to state**, without breaking the load-bearing
static-vs-`{}`-reactive rule (the `app.rx`/`counter.rx` pins must stay green —
static content is still built once; this is a **new opt-in construct**).

### Constraints discovered while validating (ruxen, today)

1. **No `State[Array[T]]` / `Mutex[Array[T]]` / `SharedSync[T]` for an
   unconstrained `T`.** `State[T: Send]` requires `T: Send`; `Array[Int]` *is*
   `Send` when checked through a generic bound, but the Send-ness of `Array`'s
   element type does **not** propagate, so `State[Array[Int]].new` and
   `Mutex[Array[String]]` both fail to construct (`E1011`/`E1101`). So we cannot
   back a reactive list with a `State[Array]`.
2. **A class CAN be shared if it `include Send`.** `SharedSync.new(Inner.new)`
   works once `Inner include Send`, and the shared handle is **capturable by
   multiple closures** (pointer-copy, like `State` handles) — verified: two
   closures mutating/reading one shared `Inner` agree. This is the escape hatch.
3. **The flat arena is push-only**; node ids are never reclaimed; only the tree
   *link* hashes (`parent`/`child_first`/`child_last`/`child_next`) are
   updatable. So "remove children" = **re-point the list's child chain**, not
   free ids.

## Options

### A — `State[Array[T]]` + diff (the natural shape, but blocked)

Bind a reactive list to `State[Array[T]]`; on change, diff old vs new items and
patch the child nodes. **Rejected:** `State[Array]` is not constructible
(constraint 1), and keyed diffing is out of scope for v1 anyway.

### B — Shared mutable model + version counter + wholesale subtree rebuild (chosen)

- A small **`ListModel`** class (`include Send`) owns the items as a plain
  `Array[String]` plus a `State[Int]` **version**. It is shared via
  `SharedSync`-style capture: closures (the "add" handler, a row's "delete"
  handler) capture the one `ListModel` and call its mutators.
- Each mutator (`add`/`remove`/`toggle`/`set`) edits the array **and** bumps the
  version in one `State.update(ui, { |v| v + 1 })` — a single-lock RMW.
- A **`list_of(model, viewport_h, item_builder)`** node subscribes to the
  version (its compute reads `model.version.get(ui)`) and stores the
  `item_builder` closure. So a mutation marks the list's scope dirty.
- On `flush`, a dirty `list_of` node **rebuilds its child subtree wholesale**:
  clear the list's child chain, set the build cursor to the list, run the
  `item_builder` once per current item (appending fresh child nodes linked under
  the list), restore the cursor — then re-`arrange` + repaint. The old child
  nodes stay in the push-only arrays but are **orphaned** (no longer in any
  chain), so layout/paint/hit-test never reach them.

**Why wholesale, not diff:** the task explicitly allows "coarse but correct".
Re-pointing the chain is O(items) and trivially correct; node-id growth is the
only cost (bounded by total mutations this session — acceptable for v1, and the
arena is reused across frames otherwise). Keyed diff / recycling / animation are
deferred.

### C — Give handlers `&var Col` to mutate the tree directly

**Rejected:** handlers run during dispatch with no safe re-entrant access to the
arena mid-frame; it would break the "build once, flush re-runs closures" model
and the pins. The version-counter indirection keeps the existing flush loop
intact (a `list_of` node is just another dirty scope).

## Decision

**Adopt Option B.** Structural reactivity is an **opt-in** node kind
(`kind_list_of`) layered on the existing scroll/clip `list` machinery; nothing
about `text`/`dyn_text`/`button`/the build-once rule changes, so the pins hold.

## Consequences

- New public surface: `Ui.list_model()` → a shared `ListModel`; its
  `add`/`remove`/`toggle`/`set` mutators (single-lock RMW); `Col.list_of(model,
  viewport_h, item_builder)`; the rebuild path inside `App.flush`.
- The item builder runs once per item **per rebuild** (not per frame). Items are
  plain `String`s for v1 (the model is a string list); a row can show the item
  text + a per-row delete/toggle affordance whose handler captures the model and
  a row identity. Rich per-row reactive widgets bound to per-item `State` are
  possible but deferred (needs per-item handles).
- Node-id growth across rebuilds is unbounded within a session (orphaned nodes
  aren't reclaimed). Fine for v1; a compaction/recycle pass is a later
  optimization.
- Reuses the `list` viewport/clip/scroll, so a reactive list scrolls and
  hit-tests through scroll like any list.

## What it defers

- **Keyed diffing / minimal patching** (reuse rows across mutations instead of
  wholesale rebuild) — and with it, per-row state preservation across reorders.
- **Node-id recycling / arena compaction** for long-lived dynamic lists.
- **Per-item reactive widgets** bound to per-item `State` handles (v1 rows are
  built from plain item strings + model-capturing handlers).
- **Animation** of row add/remove.

## Reported ruxen limitations (to the maintainer, not fixed here)

- `State[Array[T]]` / `Mutex[Array[T]]` not constructible (Send bound doesn't
  propagate through `Array`'s element type). Worked around with a `Send` class
  holding the array. (Repro: `tmp/test-cache/ruxen-state-array-send.md`.)
