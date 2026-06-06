# quiver — Design (L2 framework)

**Status:** Design derived from the approved *Ruxen GUI* top-level design.
Only the **first vertical slice** (the desktop counter app) proceeds to
implementation next.

## Goal

Provide the L2 **framework** for the Ruxen GUI stack: a type-safe, no-GC,
reactive UI API in **100% safe, platform-agnostic Ruxen**, built on the L1
engine ([`canvas`](../../canvas)).

## Non-goals

- No windowing, input, or rendering — those are L1 (`canvas`).
- No `unsafe`, no FFI, no platform-specific code (the L1 invariant).
- No JSX / `view!` macro — the Ruby-block form is native to a Ruby-flavored
  language and needs no custom-syntax facility.

## The four concerns

### 1. Reactive model — fine-grained signals (not rebuild-diff)

Chosen over Flutter's rebuild-the-subtree-then-diff because that style leans on
a GC. Fine-grained signals (SolidJS/Leptos/SwiftUI-style) update **only the
exact bound nodes**, with minimal per-frame allocation — the no-GC smoothness
story made real. Full detail in [`REACTIVITY.md`](REACTIVITY.md).

### 2. DSL — Ruby-block, `{}` = reactive / value = static

The builder block runs **once** to construct the node tree. Reactivity
granularity comes from a single rule (full detail in [`DSL.md`](DSL.md)):

- **Plain value** → static content, built once, never re-evaluated.
- **Block / closure `{ ... }`** → a *tracking scope*: subscribe to the signals
  read inside it and re-run **only that node** when they change.

### 3. Layout

A constraint/flex layout pass over the widget tree, producing geometry consumed
by the paint pass. (Own simple flex vs binding a C layout lib like Yoga is a
spec-level open detail — see [`ROADMAP.md`](ROADMAP.md).)

### 4. Paint / diff + event dispatch

- The widget tree paints onto L1's `Canvas`.
- Signal changes invalidate only their tracking scopes → **targeted repaint**
  (no full-tree diff).
- L1's event stream is dispatched to the appropriate node's handlers (e.g. a
  `button do … end` click block).

## Risks

- **Signals + ownership ergonomics** — the arena / `Copy`-handle pattern must
  feel natural in Ruxen; prototype it in the core slice before committing the
  full API.
- **Block-DSL reactivity boundary** — the `{}`-reactive / value-static rule must
  be unambiguous and well-documented; pin it with tests in the core slice.
