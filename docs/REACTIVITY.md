# Reactivity — fine-grained state on a no-GC language

This is the headline L2 design constraint. It is where reactive UI meets
Ruxen's borrow checker — and it is now **implemented and pinned by tests**
(`tests/state.rx`, `tests/app.rx`).

## Why fine-grained (not rebuild-diff)

Flutter rebuilds a widget subtree and diffs it against the previous tree. That
style leans on a garbage collector to absorb the per-frame allocation churn.
Ruxen has **no GC** — so instead we use **fine-grained reactivity**
(SolidJS/Leptos/SwiftUI-style): a change updates **only the exact bound
nodes**, with minimal per-frame allocation.

## State vs. ownership — what the prototype resolved

The design sketch planned a Leptos-style *global* arena with `Copy` index
handles. Prototyping against today's ruxen settled it differently:

- **Safe Ruxen has no global/thread-local state**, so an ambient arena is
  impossible. The graph is an explicit value — `Ui` — passed as `&var Ui`
  into every reactive read/write. Same arena discipline, explicit handle.
- **The graph lives in `Ui`**: the currently-recording tracking scope, the
  `state id -> {scope ids}` subscription table, and the dirty-scope set.
  Plain owned fields; no shared mutability, nothing for the borrow checker
  to fight.
- **Values live in the handle**: `State[T]` holds its value behind
  `SharedSync[Mutex[T]]` (std's Arc/Mutex equivalents), so any number of
  stored closures can capture one handle and all observe the same value.
  Every `State` method is a *reading* method — interior mutability does the
  writes — so captured handles never need writable borrows.
- The handle is named **`State[T]`, not `Signal[T]`**: ruxen v1 has a flat
  global symbol namespace and `std.sync` already exports a `Signal` class.

```ruxen
var ui = Ui.new
let count = ui.state(0)          # State[Int]

count.get(&var ui)                    # read  — subscribes the recording scope
count.peek                            # read  — never subscribes
count.set(&var ui, 5)                 # write — marks subscribed scopes dirty
count.update(&var ui, { |c| c + 1 })  # read-modify-write

# inside a closure that received `ui2: &var Ui`, pass it along directly —
# and reborrow with `&var *ui2` if the closure uses it more than once:
#   count.update(ui2, { |c| c + 1 })
#   flag.get(&var *ui2)
```

## Tracking scopes

A *tracking scope* is a closure the framework runs while recording which
states it reads. In the core slice, scope ids are widget-tree node ids:
`App.refresh_node(i)` sets `ui.current_scope = i`, runs node `i`'s closure,
and every `State.get` inside subscribes node `i` to that state. When the
state later changes, only node `i` goes dirty; `App.flush` re-runs exactly
the dirty nodes and returns their ids — which is also the targeted-repaint
set (`paint_dirty`). Before a scope re-runs, all its previous subscriptions
are dropped (`Ui.unsubscribe_scope`), so a closure that conditionally reads
different states never stays subscribed to one it no longer reads — pinned
by the "stale subscriptions" test in `tests/app.rx`.

```
dyn_text({ |ui| "count: #{count.get(ui)}" })
          └── tracking scope: reads `count` ⇒ re-runs ONLY this node on change
```

## The locking rule

`Mutex` guards release at *function* exit in ruxen v1 (no early block-scope
drop), so every `State` helper takes the lock at most once and never calls
another locking helper while holding it. Keep to one lock per stack frame.

## Open detail (revisit as the language grows)

- **Handle lifetime under future `Drop` semantics.** Stored closures capture
  class handles by pointer-copy; today's runtime keeps the heap object alive
  for the process (drop semantics are still landing in ruxen v1). When
  deterministic drop arrives, captured handles need a keep-alive story —
  this is exactly the "signals + ownership ergonomics" risk DESIGN.md told
  the core slice to surface.
- **Implicit `ui`.** If ruxen ever grows safe ambient state (or capture-by-
  move of class values), `count.get` can drop its `&var Ui` parameter and
  return to the sketch's no-argument form.
