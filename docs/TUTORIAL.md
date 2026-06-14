# quiver tutorial — build a GUI app, step by step

This is a **beginner-friendly, copy-as-you-go** guide. By the end you'll have a
working reactive GUI app (a counter, then a small settings panel) and understand
every line. No prior Ruxen-GUI knowledge assumed.

> **Mental model in one sentence:** you write **components** (small functions
> that add widgets) and hand them to **`run_app`**; quiver opens the window,
> paints the widgets, tracks **state**, and when state changes only the affected
> widget repaints — the Flutter model, in 100% safe Ruxen.

You write **two things**: components, and a `run_app` call. quiver owns
everything else — the window, the paint bridge, the event loop. There is **no
boilerplate shell to copy**.

Contents:
1. [What you need](#1-what-you-need)
2. [Create the project](#2-create-the-project)
3. [Your whole first app](#3-your-whole-first-app)
4. [Static text](#4-static-text)
5. [State + reactivity: the counter](#5-state--reactivity-the-counter)
6. [Components: reusable UI](#6-components-reusable-ui)
7. [Layout: rows and columns](#7-layout-rows-and-columns)
8. [More widgets: checkbox, slider, text](#8-more-widgets-checkbox-slider-text)
9. [The rules (read this — it saves hours)](#9-the-rules-read-this--it-saves-hours)
10. [Run & build](#10-run--build)
11. [Where to go next](#11-where-to-go-next)

---

## 1. What you need

- The `ruxen` toolchain (`~/.ruxen/bin/ruxen` — `ruxen --version` to check).
- One dependency: **quiver** — the GUI framework. It pulls in its renderer
  (**canvas**) automatically, so you only declare quiver.

---

## 2. Create the project

```bash
ruxen new my-app          # makes my-app/ with Ruxen.toml + src/main.rx
cd my-app
```

Edit `Ruxen.toml` so it's a **binary** that depends on quiver. You can pull it
straight from GitHub or point at a local checkout:

```toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2026"

[build]
type = "binary"

[dependencies]
# From GitHub (ruxen clones, checks out the ref, and builds it). No release tags
# yet, so track a branch — `master` — or pin an exact commit:
quiver = { git = "https://github.com/ruxen-lang/quiver", branch = "master" }
# Pin to an exact commit instead (reproducible):
# quiver = { git = "https://github.com/ruxen-lang/quiver", rev = "<full-sha>" }
# …or a local checkout during development:
# quiver = { path = "../quiver" }
```

Or from the CLI: `ruxen add quiver --git https://github.com/ruxen-lang/quiver --branch master`.

> **Why only quiver?** quiver depends on canvas (the renderer), and a **binary**
> automatically pulls in the whole dependency chain — so canvas comes along for
> free. A git dep accepts **`branch`**, **`tag`**, or **`rev`**; `Ruxen.lock`
> records the exact resolved commit so a branch dep still builds reproducibly
> until you `ruxen update`.

---

## 3. Your whole first app

Put this in `src/main.rx`. **This is the entire app** — no shell, no window code,
no paint code:

```ruxen
def main
  run_app("My App", 240, 120, { |ui: &var Ui, root: &var Col|
    root.text("Hello, quiver!")
  })
end
```

```bash
ruxen run
```

A 240×120 window titled **My App** opens showing **Hello, quiver!**.

`run_app(title, width, height, build)` does it all: opens the window, builds your
UI from the `build` closure, paints it, and runs the event loop until you close
the window. The `build` closure hands you two things:

- **`ui`** — the state arena (Section 5).
- **`root`** — the top-level **column** you add widgets to.

> **Why the param types?** The `run_app` build closure keeps
> `|ui: &var Ui, root: &var Col|` — it's the app entry and its types aren't
> inferred. (Inner closures you'll see later drop their types.)

No `use` statements are needed: a binary flat-merges its dependencies' sources,
so quiver's names (`run_app`, `Ui`, `Col`, …) are already in scope.

---

## 4. Static text

`root` is a column; add widgets top-to-bottom. `root.text(...)` adds a static
label (built once, never changes):

```ruxen
def main
  run_app("My App", 240, 120, { |ui: &var Ui, root: &var Col|
    root.text("Welcome")
    root.text("This is quiver.")
  })
end
```

---

## 5. State + reactivity: the counter

State lives in `ui.state(initial)`. A **`dyn_text`** closure *reads* state and
re-runs only itself when that state changes. A **`button`**'s second argument is
its click handler.

```ruxen
def main
  run_app("Counter", 240, 120, { |ui: &var Ui, root: &var Col|
    let count = ui.state(0)                              # State[Int]
    root.text("Counter")
    root.dyn_text({ |ui| "count: #{count.get(ui)}" })   # reactive: re-runs on change
    root.button("tap +1", { |ui|                        # click handler
      count.update(ui, { |c| c + 1 })
    })
  })
end
```

- `ui.state(0)` → an `Int` state, initially 0. (`ui.state_str("")`,
  `ui.state_bool(false)` for String/Bool.)
- `count.get(ui)` inside a `dyn_text` **subscribes** it: that one line repaints
  when `count` changes — nothing else.
- `count.update(ui, { |c| c + 1 })` reads-modifies-writes in one call.

Tap the button → only the `count: …` line repaints.

---

## 6. Components: reusable UI

As your UI grows, the `build` closure gets long. Pull pieces into **components** —
plain functions that take `c: &var Col` (where to add widgets) and `ui: &var Ui`
(the state arena), and build into `c`:

```ruxen
## A component: the counter UI, reusable anywhere.
def counter(c: &var Col, ui: &var Ui) -> nil
  let n = ui.state(0)
  c.text("Counter")
  c.dyn_text({ |ui2| "count: #{n.get(ui2)}" })
  c.button("tap +1", { |ui2| n.update(ui2, { |x| x + 1 }) })
end

def main
  run_app("Counter", 240, 120, { |ui: &var Ui, root: &var Col|
    counter(&var *root, &var *ui)     # drop the component into the app
  })
end
```

`counter(&var *root, &var *ui)` passes the column and arena into the component.
The `&var *x` is a **reborrow** — required when you forward a `&var` parameter as
an argument (see rule 8). You can call a component multiple times, or call other
components from inside one — that's how you compose larger UIs.

---

## 7. Layout: rows and columns

`root` (and any `c: &var Col`) is a column. Nest with `row`/`col` builders.
**Container builders use `do … end` and you must type the param** `|c: &var Col|`:

```ruxen
def counter(c: &var Col, ui: &var Ui) -> nil
  let count = ui.state(0)
  c.text("Counter")
  c.row do |r: &var Col|
    r.button("-", { |ui| count.update(ui, { |x| x - 1 }) })
    r.dyn_text({ |ui| " #{count.get(ui)} " })
    r.button("+", { |ui| count.update(ui, { |x| x + 1 }) })
  end
end
```

> **`do…end` vs `{ }`** is the house rule:
> **`do…end`** = a container that runs *right now* while building (`row`/`col`/
> `list`). **`{ }`** = a callback you *store* for later (`dyn_text`, button
> handlers). Don't mix them.

---

## 8. More widgets: checkbox, slider, text

These bind directly to typed state:

```ruxen
def settings(c: &var Col, ui: &var Ui) -> nil
  let dark = ui.state_bool(false)
  let vol  = ui.state(50)
  let name = ui.state_str("")

  c.checkbox(dark, "Dark mode")          # State[Bool]
  c.slider(vol, 0, 100, 200)             # State[Int], min, max, width-px
  c.textarea(name, 200, 1)               # State[String], width-px, rows

  c.dyn_text({ |ui| "volume: #{vol.get(ui)}" })
end
```

Typing in the textarea / dragging the slider / toggling the checkbox updates the
bound state, and any `dyn_text` reading it repaints.

---

## 9. The rules (read this — it saves hours)

quiver is built on Ruxen v1, which has sharp edges. These are the ones a beginner
hits; following them avoids crashes/miscompiles:

1. **The `run_app` build closure keeps its types**: `{ |ui: &var Ui, root: &var Col| }`.
   **Stored inner closures drop types**: `{ |ui| … }`. **Container `do…end`
   blocks must type the param**: `row do |c: &var Col| … end`.
2. **A component is a function** `def name(c: &var Col, ui: &var Ui) -> nil` that
   builds into `c`. Call it as `name(&var *root, &var *ui)`.
3. **`text` (static) and `dyn_text` (reactive) are different methods** — never
   overload one name on a string vs a closure.
4. **`do…end` = build-now container; `{ }` = stored callback.**
5. **Read-modify-write state through ONE `update`** — `count.update(ui, { |c| c+1 })`,
   never a `peek` then `set` in the same frame (they'd self-deadlock and corrupt
   the value).
6. **`.size` returns `USize`** → write `(xs.size as Int)`.
7. **No `arr[i]`** for a non-literal index (parses as generics) → use `xs.get(i)`.
   There's no index assignment → push-only arrays or `Hash.insert`.
8. **Forwarding a `&var` parameter as an argument needs a reborrow**:
   `counter(&var *root, &var *ui)`, not `counter(root, ui)`.
9. **String literals already ARE `String`** — write `"hi #{x}"` directly; only use
   `String.from(v)` to convert a `&str` *variable*.

(The full landmine list lives in `../CLAUDE.md` and `docs/DSL.md`.)

---

## 10. Run & build

```bash
ruxen run                 # build + open the window (debug)
ruxen build --release     # optimized binary in target/release/
ruxen check               # fast type-check, no codegen — your tightest loop
```

> **Web (wasm):** running quiver apps in the browser via CanvasKit is in
> progress (see `../../canvas/docs/WEB_BACKEND.md`). The *same* components and
> `run_app` code is the goal there too — you won't rewrite your app for the web.

---

## 11. Where to go next

- **`examples/counter`** — the canonical app, exactly the shape this tutorial
  teaches (a `counter` component + `run_app`). **`examples/todo`**,
  **`examples/settings`** — lists, checkboxes, sliders.
- **`docs/DSL.md`** — every builder method + why the API looks like it does.
- **`docs/REACTIVITY.md`** — how `State`/`Ui` tracking works.
- **`docs/LAYOUT.md`** — the layout model (sizes, rows/columns, lists).
- The Ruxen language tutorial: `../../ruxen/docs/tutorial/` (33 chapters).

Build something — write a component, drop it in `run_app`, `ruxen run`.
