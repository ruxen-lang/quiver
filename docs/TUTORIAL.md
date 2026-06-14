# quiver tutorial — build a GUI app, step by step

This is a **beginner-friendly, copy-as-you-go** guide. By the end you'll have a
working reactive GUI app (a counter, then a small settings panel) and understand
every line. No prior Ruxen-GUI knowledge assumed.

> **Mental model in one sentence:** you describe a tree of *widgets* (text,
> buttons, rows/columns), quiver **paints** them and tracks **state**, and when
> state changes only the affected widget repaints — the Flutter model, in 100%
> safe Ruxen.

Contents:
1. [What you need](#1-what-you-need)
2. [Create the project](#2-create-the-project)
3. [The app skeleton (copy once)](#3-the-app-skeleton-copy-once)
4. [Your first UI: static text](#4-your-first-ui-static-text)
5. [State + reactivity: the counter](#5-state--reactivity-the-counter)
6. [Layout: rows and columns](#6-layout-rows-and-columns)
7. [More widgets: checkbox, slider, text](#7-more-widgets-checkbox-slider-text)
8. [The rules (read this — it saves hours)](#8-the-rules-read-this--it-saves-hours)
9. [Run & build](#9-run--build)
10. [Where to go next](#10-where-to-go-next)

---

## 1. What you need

- The `ruxen` toolchain (`~/.ruxen/bin/ruxen` — `ruxen --version` to check).
- Two packages: **quiver** (this — the widgets/state framework, L2) and
  **canvas** (the renderer, L1). You depend on both.

quiver is platform-agnostic; canvas does the drawing. An app declares both.

---

## 2. Create the project

```bash
ruxen new my-app          # makes my-app/ with Ruxen.toml + src/main.rx
cd my-app
```

Edit `Ruxen.toml` so it's a **binary** that depends on quiver + canvas. You can
point at local checkouts **or pull them straight from GitHub** — both work:

```toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2026"

[build]
type = "binary"

[dependencies]
# From GitHub (ruxen clones, checks out the ref, and builds them). There are no
# release tags yet, so track a branch — `master` — or pin an exact commit:
quiver = { git = "https://github.com/ruxen-lang/quiver", branch = "master" }
canvas = { git = "https://github.com/ruxen-lang/canvas", branch = "master" }
# Pin to an exact commit instead (reproducible):
# quiver = { git = "https://github.com/ruxen-lang/quiver", rev = "<full-sha>" }
# …or local checkouts during development:
# quiver = { path = "../quiver" }
# canvas = { path = "../canvas" }
```

You can also add them from the CLI:

```bash
ruxen add quiver --git https://github.com/ruxen-lang/quiver --branch master
ruxen add canvas --git https://github.com/ruxen-lang/canvas --branch master
```

> A git dep accepts **`branch = "..."`**, **`tag = "..."`**, or **`rev = "<sha>"`**
> (default branch if none given). `Ruxen.lock` records the exact resolved commit,
> so a branch dep still builds reproducibly until you `ruxen update`.
> A **binary** is the only place quiver and canvas symbols may meet — that's why
> your app is `type = "binary"`. Libraries don't merge dependency sources.

---

## 3. The app skeleton (copy once)

Every quiver desktop app needs a small **shell**: open a window, bridge quiver's
paint to canvas, and run the event loop. **Copy this into `src/main.rx` verbatim
once** — you'll only ever edit the `App.build` block (Section 4 onward).

This shell is copied straight from the shipped `examples/counter` (it's the exact
verified code) — just with a simpler UI block. No `use` statements: a binary
flat-merges its dependencies' sources, so quiver and canvas names are in scope.

```ruxen
## The paint backend: quiver calls these as it walks the widget tree; each one
## issues the matching canvas draw. Copy verbatim — it's the canvas↔quiver bridge
## (quiver itself is renderer-free, so every app supplies this once).
class SkiaPaint
  include PaintSurface
  canvas: &var Canvas

  def init(canvas: &var Canvas)
    self.canvas = canvas
  end

  def var fill_clear(r: Int, g: Int, b: Int) -> nil
    let col = Color.rgb(r as UInt8, g as UInt8, b as UInt8)
    let _ = self.canvas.clear(col)
  end

  def var fill_rect(x: Int, y: Int, w: Int, h: Int, r: Int, g: Int, b: Int) -> nil
    let col = Color.rgb(r as UInt8, g as UInt8, b as UInt8)
    let rect = Rect.new(x as Float32, y as Float32, w as Float32, h as Float32)
    let _ = self.canvas.draw_rect(rect, col)
  end

  def var draw_text(text: &String, x: Int, y: Int, r: Int, g: Int, b: Int) -> nil
    let col = Color.rgb(r as UInt8, g as UInt8, b as UInt8)
    let label = text.clone
    let _ = self.canvas.draw_text(label, x as Float32, y as Float32, col)
  end

  def var fill_round_rect(x: Int, y: Int, w: Int, h: Int, radius: Int, r: Int, g: Int, b: Int) -> nil
    let col = Color.rgb(r as UInt8, g as UInt8, b as UInt8)
    let rect = Rect.new(x as Float32, y as Float32, w as Float32, h as Float32)
    let _ = self.canvas.draw_round_rect(rect, radius as Float32, col)
  end

  def var stroke_round_rect(x: Int, y: Int, w: Int, h: Int, radius: Int, width: Int, r: Int, g: Int, b: Int) -> nil
    let col = Color.rgb(r as UInt8, g as UInt8, b as UInt8)
    let rect = Rect.new(x as Float32, y as Float32, w as Float32, h as Float32)
    let _ = self.canvas.stroke_round_rect(rect, radius as Float32, width as Float32, col)
  end

  def var push_clip(x: Int, y: Int, w: Int, h: Int) -> nil
    let _ = self.canvas.save
    let rect = Rect.new(x as Float32, y as Float32, w as Float32, h as Float32)
    let _ = self.canvas.clip_rect(rect)
  end

  def var push_translate(dx: Int, dy: Int) -> nil
    let _ = self.canvas.translate(dx as Float32, dy as Float32)
  end

  def var pop_state -> nil
    let _ = self.canvas.restore
  end
end

## Paint one frame to the window: open a frame, walk quiver's paint pass through
## SkiaPaint (which issues the draws), end + present.
def paint_frame(win: &var Window, app: &var App) -> nil
  match win.canvas.begin_frame
    Ok(_)  -> nil
    Err(e) -> puts "begin_frame: #{e}"
  end
  var skia = SkiaPaint.new(&var win.canvas)
  paint_all(app, &var skia)
  let _ = win.canvas.end_frame
  let _ = win.present
end

def main
  match Window.open("My App", 240, 120)
    Ok(win) -> run_app(win)
    Err(e)  -> puts "cannot open window: #{e}"
  end
end

def run_app(win0: Window) -> nil
  var win = win0

  # >>> YOUR UI GOES HERE — this block is the whole app <<<
  var app = App.build({ |ui: &var Ui, root: &var Col|
    root.text("Hello, quiver!")
  })

  let _ = win.show_gpu_scaled(3)
  # Give quiver real text metrics so layout is accurate.
  app.set_measure({ |t: String, sz: Int| win.canvas.measure_text_sized(t, sz as Float32) })
  paint_frame(&var win, &var app)

  var running = true
  while running
    match win.poll_event
      Some(ev) -> match ev
        Event.PointerDown(x, y) -> app.pointer_down(x as Int, y as Int)
        Event.PointerMove(x, y) -> app.pointer_move(x as Int, y as Int)
        Event.PointerUp(x, y)   -> app.pointer_up(x as Int, y as Int)
        Event.TextInput(cp)     -> app.text_input(cp)
        Event.KeyDown(k)        -> app.key_down(k)
        Event.CloseRequested    -> running = false
        _                       -> nil
      end
      nil -> nil
    end
    let dirty = app.flush
    if (dirty.size as Int) > 0
      paint_frame(&var win, &var app)
    end
    win.sleep_ms(16)
  end
  let _ = win.hide
end
```

Run it: `ruxen run`. A window opens showing **Hello, quiver!**. Everything below
only changes the `App.build({ … })` block.

---

## 4. Your first UI: static text

The `App.build` block builds your widget tree **once**. `root` is a column; add
widgets to it. `root.text(...)` adds a static label:

```ruxen
var app = App.build({ |ui: &var Ui, root: &var Col|
  root.text("Welcome")
  root.text("This is quiver.")
})
```

> **Why the param types?** The outer `App.build` block keeps `|ui: &var Ui, root:
> &var Col|` — it's the app entry and its types aren't inferred. (Inner closures
> you'll see later drop their types.)

---

## 5. State + reactivity: the counter

State lives in `ui.state(initial)`. A **`dyn_text`** closure *reads* state and
re-runs only itself when that state changes. A **`button`**'s second argument is
its click handler.

```ruxen
var app = App.build({ |ui: &var Ui, root: &var Col|
  let count = ui.state(0)                                  # State[Int]
  root.text("Counter")
  root.dyn_text({ |ui| "count: #{count.get(ui)}" })       # reactive: re-runs on change
  root.button("tap +1", { |ui|                            # click handler
    count.update(ui, { |c| c + 1 })
  })
})
```

- `ui.state(0)` → an `Int` state, initially 0. (`ui.state_str("")`,
  `ui.state_bool(false)` for String/Bool.)
- `count.get(ui)` inside a `dyn_text` **subscribes** it: that one line repaints
  when `count` changes — nothing else.
- `count.update(ui, { |c| c + 1 })` reads-modifies-writes in one call.

Tap the button → only the `count: …` line repaints.

---

## 6. Layout: rows and columns

`root` is a column. Nest with `row`/`col` builders. **Container builders use
`do … end` and you must type the param** `|c: &var Col|`:

```ruxen
var app = App.build({ |ui: &var Ui, root: &var Col|
  let count = ui.state(0)
  root.text("Counter")
  root.row do |c: &var Col|
    c.button("-", { |ui| count.update(ui, { |x| x - 1 }) })
    c.dyn_text({ |ui| " #{count.get(ui)} " })
    c.button("+", { |ui| count.update(ui, { |x| x + 1 }) })
  end
end)
```

> **`do…end` vs `{ }`** is the house rule:
> **`do…end`** = a container that runs *right now* while building (`row`/`col`/
> `list`). **`{ }`** = a callback you *store* for later (`dyn_text`, button
> handlers). Don't mix them.

---

## 7. More widgets: checkbox, slider, text

These bind directly to typed state:

```ruxen
var app = App.build({ |ui: &var Ui, root: &var Col|
  let dark = ui.state_bool(false)
  let vol  = ui.state(50)
  let name = ui.state_str("")

  root.checkbox(dark, "Dark mode")          # State[Bool]
  root.slider(vol, 0, 100, 200)             # State[Int], min, max, width-px
  root.textarea(name, 200, 1)               # State[String], width-px, rows

  root.dyn_text({ |ui| "volume: #{vol.get(ui)}" })
end)
```

Typing in the textarea / dragging the slider / toggling the checkbox updates the
bound state, and any `dyn_text` reading it repaints.

---

## 8. The rules (read this — it saves hours)

quiver is built on Ruxen v1, which has sharp edges. These are the ones a beginner
hits; following them avoids crashes/miscompiles:

1. **Outer `App.build` block keeps its types**: `{ |ui: &var Ui, root: &var Col| }`.
   **Stored inner closures drop types**: `{ |ui| … }`, `{ |c| … }`.
   **Container `do…end` blocks must type the param**: `row do |c: &var Col| … end`.
2. **`text` (static) and `dyn_text` (reactive) are different methods** — never
   overload one name on a string vs a closure.
3. **`do…end` = build-now container; `{ } = stored callback.**
4. **Read-modify-write state through ONE `update`** — `count.update(ui, { |c| c+1 })`,
   never a `peek` then `set` in the same frame (they'd self-deadlock and corrupt
   the value).
5. **`.size` returns `USize`** → write `(xs.size as Int)`.
6. **No `arr[i]`** for a non-literal index (parses as generics) → use `xs.get(i)`.
   There's no index assignment → push-only arrays or `Hash.insert`.
7. **String literals already ARE `String`** — write `"hi #{x}"` directly; only use
   `String.from(v)` to convert a `&str` *variable*.

(The full landmine list lives in `../CLAUDE.md` and `docs/DSL.md`.)

---

## 9. Run & build

```bash
ruxen run                 # build + open the window (debug)
ruxen build --release     # optimized binary in target/release/
ruxen check               # fast type-check, no codegen — your tightest loop
```

> **Web (wasm):** running quiver apps in the browser via CanvasKit is in
> progress (see `../../canvas/docs/WEB_BACKEND.md`). The *same* `App.build` block
> will run unchanged on the web — only the shell (Section 3) differs.

---

## 10. Where to go next

- **`examples/counter`** — the canonical app (the shell in Section 3 is its real
  shell). **`examples/todo`**, **`examples/settings`** — lists, checkboxes, sliders.
- **`docs/DSL.md`** — every builder method + why the API looks like it does.
- **`docs/REACTIVITY.md`** — how `State`/`Ui` tracking works.
- **`docs/LAYOUT.md`** — the layout model (sizes, rows/columns, lists).
- The Ruxen language tutorial: `../../ruxen/docs/tutorial/` (33 chapters).

Build something — copy the skeleton, edit the `App.build` block, `ruxen run`.
