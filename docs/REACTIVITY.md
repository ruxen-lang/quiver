# Reactivity — fine-grained signals on a no-GC language

This is the headline L2 design constraint. It must be specified precisely
because it is where reactive UI meets Ruxen's borrow checker.

## Why fine-grained (not rebuild-diff)

Flutter rebuilds a widget subtree and diffs it against the previous tree. That
style leans on a garbage collector to absorb the per-frame allocation churn.
Ruxen has **no GC** — so instead we use **fine-grained signals**
(SolidJS/Leptos/SwiftUI-style): a change updates **only the exact bound nodes**,
with minimal per-frame allocation. That is the no-GC smoothness story made real.

## Signals vs. ownership (the crux)

Fine-grained reactivity needs shared, observable state with subscriptions —
which a borrow checker resists (shared mutable aliasing). The solution is the
pattern Leptos proved in Rust:

- An **arena-scoped signal runtime**, owned by the UI root.
- The app holds cheap **`Copy` handles** — *indices into the arena*, never
  references. So there is no aliasing/borrow fight: handles are values, freely
  copied and stored in closures.
- **Teardown is deterministic via the runtime's drop** — when the UI root drops,
  the arena drops, and all signals/effects go with it. No GC, no leaks.

```ruxen
# `Signal[T]` is a Copy handle (an index), not a reference to the value.
count: Signal[Int] = Signal.new(0)

count.get               # read  — subscribes the enclosing tracking scope
count.set(5)            # write — notifies subscribers
count.update { |c| c+1 }  # read-modify-write
```

## Tracking scopes

A *tracking scope* is a closure the framework runs while recording which signals
it reads. When any of those signals later changes, the framework re-runs **only
that scope** (and repaints only its node). Scopes are created by the DSL's block
form — see [`DSL.md`](DSL.md).

```
text { "count: #{count.get}" }
      └── tracking scope: reads `count` ⇒ re-runs ONLY this node on change
```

## Open detail (resolved in the core slice spec)

- **Signal API shape** — `count.get` / `count.set` vs call-style `count()`.
  Prototype both for ergonomics before committing.
