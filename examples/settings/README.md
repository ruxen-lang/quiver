# settings — a multi-file quiver app (and tutorial)

A small **Settings panel** that exercises the whole quiver widget set, written
the way a real quiver app is organised: **model**, **views**, and a thin
**shell**, across multiple files. Read the files in this order to learn how to
build apps in quiver — they're heavily commented.

```
examples/settings/
  Ruxen.toml        # a BINARY package — the only place quiver (L2) + canvas (L1) meet
  src/
    state.rx        # the MODEL: option lists, the fixed row set, pure label helpers
    views.rx        # the UI, split into view functions that append subtrees
    main.rx         # the SHELL: build the app from views, window + event loop, headless fallback
```

## Run it

```bash
cd examples/settings
ruxen build
ruxen run            # opens a window if a display is available …
./target/debug/settings   # … otherwise prints the headless demo (CI-safe)
```

With a display it opens a live OS window driven by real pointer/keyboard events.
With no display it falls back to **synthetic events through the same pipeline**
and prints each frame's targeted-repaint count, so it runs anywhere.

## What it demonstrates

Every widget, composed idiomatically:

| Widget / feature | Where |
|---|---|
| `text` (static) vs `dyn_text` (reactive) — **the one DSL rule** | `views.rx` `header` |
| `input` (text field, bound to `State[String]`) | `name_field` |
| `checkbox` ×2 (bound to `State[Bool]`, toggle = single-lock RMW) | `toggles` |
| `slider` (bound to `State[Int]`, click + drag) | `volume_control` |
| `select` (dropdown + overlay popup) | `theme_picker` |
| `list` (fixed viewport, clip + scroll) | `device_list` |
| `row` / `col` + `row_styled` / `col_styled` (padding / bg / border) | every card |
| `button` + handler + targeted repaint | `save_bar` |
| composing views, passing state by `&State` | `main.rx` `build_app` |

## The patterns to copy

1. **One build block, run once.** `App.build({ |ui, root| … })` runs your
   builder **once** to construct the node tree. quiver reacts by re-running the
   `dyn_text` *content closures* of existing nodes — it does **not** rebuild the
   tree. So the *structure* is fixed; the *values* are reactive.

2. **Static vs reactive is the one rule.** `text("…")` is baked in at build
   time; `dyn_text({ |ui| … })` is a tracking scope — it re-runs (and only that
   node repaints) whenever a state it `.get(ui)`-read changes.

3. **Views are functions.** A view takes `root: &var Col` plus the `&State`
   handles it needs, and appends a subtree. Compose by calling them in order.

4. **State handles are passed by reference, deref'd to a local.** `State[T]` is
   a non-`Copy` handle. A view takes `&State[T]` and does `let v = *handle` to
   get a local copy (a cheap pointer-copy of the shared cell — every copy sees
   the same value). Each *consumer* (a widget builder that takes it by value, or
   a closure that captures it) needs its **own** local copy; reusing one local
   twice moves it.

5. **Read-modify-write is a single `update`.** Toggling/editing state goes
   through one `state.update(ui, { |v| … })` (or one `set`) — never a `peek`
   then `set` (two locks on the same mutex in one frame corrupt the value).
   The widgets do this for you.

6. **`let`-bind a function result before passing it to a method.** In this ruxen
   build, handing a function's class-value result *directly* to a method
   argument segfaults — so `let style = card_style; root.row_styled(style, …)`,
   not `root.row_styled(card_style, …)`. (See *Known gaps* below.)

## Reactivity model: values, not structure

quiver does **fine-grained reactivity on content**: change a state, and exactly
the `dyn_text` nodes that read it recompute and repaint. It does **not** support
**structural reactivity** — you cannot reactively add or remove nodes (e.g.
"add a todo row" growing a list). The builder runs once; handlers receive
`&var Ui` (the reactive graph), not `&var Col` (the tree), so there is no API to
append a node after build.

That's why this is a settings panel (reactive **values** over a fixed layout)
and not a todo app (which needs to grow/shrink a list). When quiver gains
**reactive lists / keyed children / remount**, a dynamic-structure app becomes
possible — that's the recommended next framework feature.

## Known gaps (framework / toolchain, today)

- **No structural reactivity** (add/remove nodes) — see above. Next big feature.
- **`let`-bind function results before a method call** — a non-`Copy` class
  value returned by a call and passed *directly* as a by-value method argument
  segfaults in the current ruxen build (binding to a `let` first is the fix;
  repro under `tmp/test-cache/`). Idiomatic anyway, and used throughout here.
- **Char-metric text widths** — widths are estimated per character until canvas
  exposes real Skia advance widths.
