# ADR: Production text editing (F3) — selection, clipboard, multi-line, IME, undo

Status: **Accepted** (2026-06-11). Phase 2, F3. Extends the single-line `input`
(focus + caret + key editing) into a production editing surface, staying
canvas-free: every platform capability crosses through an **injected seam** or an
**App entry point**, exactly like `set_measure` (text metrics) and `set_clock`
(time).

## Context

quiver's `input` already has: a focusable node bound to `State[String]`, a
per-input `caret` index on `App`, `text_input(cp)` to insert, `key_down(code)`
for backspace/delete/arrows/home/end, and single-lock `edit_value` RMW (the
one-lock-per-frame landmine is why edits go through one `State.update`). Caret
paint is a thin rect at `x + pad_x + caret * char_width` (char-metric).

Canvas (audited) already provides everything F3 needs across the seam boundary:

- **Clipboard:** `Window.clipboard_text` / `Window.set_clipboard_text` (class
  methods, work headless).
- **IME:** `Event.TextEditing(start, len)` + `Window#composition_text` (the
  marked text as a side-channel read right after polling).
- **Measure:** the existing `set_measure` seam returns true advance widths.

**The one gap:** `Event.KeyDown(Int)` carries **no modifier state** — no shift,
ctrl, cmd. quiver cannot know that an arrow press was shift+arrow (selection
extend) vs a plain arrow (caret move). See "Canvas gap" below.

## 1. Selection (anchor + caret)

Per-input **anchor** alongside the existing **caret** (both flat-index char
positions). The selection is the range `[min(anchor,caret), max(anchor,caret)]`;
empty when `anchor == caret`.

```
anchor: Hash[Int, Int]     # selection anchor per input node (default = caret)
```

- **Pointer drag-select.** A `pointer_down` on an input sets `anchor = caret =`
  the char index under x (computed from measured prefix widths — see below) and
  **captures** the input (reuse the existing `captured` drag primitive — generalise
  it from sliders to inputs). `pointer_move` while captured moves `caret` to the
  char under x, leaving `anchor` fixed → a live selection. `pointer_up` releases.
- **Keyboard extend.** Shift+arrow / shift+home / shift+end move `caret` while
  keeping `anchor` (extend); a plain arrow collapses the selection (anchor =
  caret = the moved caret). This needs a shift flag the event does not carry —
  see the modifier decision.
- **Caret x from measured prefixes.** Replace the char-metric caret math
  (`caret * char_width`) with a `prefix_width(input, n)` helper: the measured
  width of the first `n` chars via the injected measure seam (char-metric
  fallback when unset — byte-identical to today). The inverse —
  `char_index_at_x(input, x)` — walks prefixes to find the click target. Both go
  through the one `text_width` decision point already in `App`.
- **Paint.** A selection highlight rect (one rect — single-line is one row)
  behind the text, from `prefix_width(sel_start)` to `prefix_width(sel_end)`,
  recorded BEFORE the text draw so the text sits on top.

## 2. Modifier decision (the canvas gap)

`Event.KeyDown(Int)` has no modifiers. **Decision:** add the modifier-aware App
API **now**, so quiver's contract is complete and shells opt in as canvas grows:

```
def var key_down_mod(code: Int, shift: Bool, ctrl: Bool) -> nil
alias key_down  →  key_down_mod(code, false, false)   # back-compat: old callers = no mods
```

- `key_down(code)` stays (and the `key` alias) — it forwards to
  `key_down_mod(code, false, false)`, so every existing call site and pin is
  unchanged.
- The windowed shells call `key_down_mod(k, false, false)` **today** (canvas
  delivers no modifiers), so shift-extend and ctrl-shortcuts are reachable via
  the App API and headless tests, but not yet wired from real key events.
- **Canvas gap filed for the coordinator:** `Event.KeyDown` needs a modifier
  bitmask (shift/ctrl/cmd/alt) — SDL has it (`SDL_Keymod`); it is not yet on the
  ring. Until then, real shift+arrow selection and ctrl+c/x/v/z from the keyboard
  are blocked at the shell; the App-level `copy`/`cut`/`paste`/`undo` +
  `key_down_mod` are fully functional and pinned headless. (Clipboard shortcuts
  can alternatively be driven from a menu/button in the shell, which needs no
  modifier — noted as the interim path.)

Ctrl is threaded through the same call (for a future ctrl+arrow word-move / the
ctrl+c/x/v/z path); today it is always `false` from shells and exercised only by
tests.

## 3. Clipboard (a seam, exactly like measure/clock)

```
def var set_clipboard(get: any Fn[Fn() -> String], set: any Fn[Fn(String) -> nil]) -> nil
```

- Two 0-or-1-entry pools + a presence flag (the `measure_fns` / `clock_fns`
  pattern — NOT `Option[any Fn]` fields). One seam call stores both closures.
- App-level operations on the focused input's selection:
  - `copy` — write the selected substring to the clipboard `set` closure.
  - `cut` — copy, then delete the selection (single-lock `edit_value` RMW).
  - `paste` — read the clipboard `get` closure, replace the selection with it
    (delete selection + insert at caret, one `edit_value` RMW), caret to end of
    inserted text.
- The shells wire canvas through the seam:
  `app.set_clipboard({ || match Window.clipboard_text { Ok(s) -> s, Err(_) -> "" } },
  { |s| let _ = Window.set_clipboard_text(s) })`.
- Headless tests inject a **fake clipboard** (a captured `State[String]` /
  `SharedSync[Mutex[String]]`) so copy/cut/paste round-trip deterministically
  with no SDL.
- With no clipboard seam set, `copy`/`cut`/`paste` are inert no-ops (no `Err`,
  no crash — the measure-seam default discipline).

## 4. Selection-aware editing

Inserting (`text_input`) or backspacing **with a non-empty selection replaces /
deletes the selection first**, then proceeds — the standard editor contract. All
through one `edit_value` RMW (the selection bounds are read from `anchor`/`caret`,
which are App-side Ints, not the `State[String]` — so no second lock). After any
edit the selection collapses (`anchor = caret`).

The selection-delete + insert reuses the existing `for c in cur.chars` rebuild
idiom (no slice/index-assign in ruxen v1): walk chars, emit those outside
`[sel_start, sel_end)`, splice the inserted text at `sel_start`.

## 5. Multi-line input (`textarea`)

```
root.textarea(state, width, rows)     # rows * row_height tall, kind_input + a multiline flag
```

- **Storage:** a single `String` with `\n` separators — caret is a **flat char
  index** (no recursive line types — the landmine). Line navigation scans for
  newlines:
  - up/down: find the current line's start (scan back to prev `\n`), the column
    (caret − line_start), the target line's start, clamp the column to the target
    line's length, set caret. Column-preservation via measured x is the precise
    version (target x = `prefix_width` of the column on the target line); v1 uses
    char-column (measured-x column is filed as the additive refinement).
  - home/end: to the current line's start / end (not the whole buffer).
- **Layout:** `rows * row_height` tall, fixed `width`. A new `multiline` flag on
  the input node (per-node Bool hash, the `horizontal_of` pattern) drives paint +
  navigation; a single-line `input` is `multiline = false` (unchanged).
- **Paint:** split the value on `\n`, draw each line at `y + line_index *
  row_height + baseline_drop`; the caret rect is at the caret's line + measured
  column x. Selection highlight spans may cross lines — v1 highlights the caret
  line's portion and files full multi-line selection rects as additive (honest
  scope: single-line selection rects are exact; multi-line is staged).
- **Scrolling within the viewport:** if the content exceeds `rows`, reuse the
  list clip machinery (scroll the textarea's text by a line offset). **Honest
  scope:** if the clip reuse is cheap it lands; otherwise the textarea is
  fixed-`rows` (no internal scroll) and scrolling is filed. **Wrap is deferred**
  (consistent with F1's deferred wrap) — long lines overflow the width.

## 6. IME composition

```
def var text_editing(start: Int, len: Int, text: String) -> nil
```

- Stores the **marked (composition) text** on the focused input (per-input
  `String` + `(start, len)` ints on `App`), distinct from the committed value.
- **Paint:** the marked text renders inline at the caret, distinguished by a
  thin underline rect beneath it (a `fill_rect` 1px tall — no new op). The
  committed value + marked text compose visually at the caret position.
- **Commit sequence (how SDL sequences it):** composition updates arrive as
  `text_editing(...)`; when the user commits, SDL sends the final `TextInput(cp)`
  per committed codepoint and an empty `text_editing("", 0, 0)` to clear the
  mark. So: a non-empty `text_editing` sets the mark; the subsequent `text_input`
  inserts the committed char(s) and clears the mark. quiver does not invent the
  sequencing — it mirrors what canvas/SDL delivers.
- The shells route `Event.TextEditing(s, l)` + `win.composition_text` through
  `app.text_editing(s, l, marked)`.
- Headless tests drive `text_editing` directly with a fake marked string and
  assert the marked-render + the commit-replaces-mark sequence.

## 7. Undo (per-input bounded history)

```
def var undo -> nil      # on the focused input
def var redo -> nil      # if cheap; else filed
```

- **Op model: snapshots** (simplest sound) — a per-input bounded ring of
  `(value, caret)` snapshots (cap N ≈ 50 entries; oldest dropped). Snapshots are
  plain `String`/`Int` parallel arrays per input (flat-arena discipline; no
  recursive history type).
- A snapshot is pushed **before** a mutating edit. `undo` pops to the previous
  snapshot (restores value via one `State.set` + restores caret); `redo` re-applies
  if a redo stack is kept.
- **Typing coalescing:** group rapid typing into one undo entry by **clock
  distance** (the clock seam again) — if the last edit was within
  `undo_group_ms` of `now`, extend the current entry instead of pushing a new
  one. **Honest scope:** if coalescing is cheap it lands; otherwise per-op
  entries land and coalescing is filed. The acceptance bar — *undo restores exact
  prior text + caret* — is met either way.
- `redo` is landed if the redo-stack bookkeeping is cheap; else filed.

## Pins (`tests/text_editing.rx`, all under fake measure + fake clipboard)

1. **Selection geometry** — anchor/caret set the highlight rect x-span from
   measured prefixes (fake `*7` measure); a pointer drag selects a char range.
2. **Clipboard round-trip** — copy → fake clipboard holds the selection; paste
   into another input inserts it; cut removes the selection and clipboards it.
3. **Selection-replace** — typing/backspace with a selection replaces it.
4. **Textarea navigation** — up/down/home/end move the caret across lines on a
   `\n`-separated buffer; column clamps on a short target line.
5. **IME** — `text_editing` sets the marked render; the committing `text_input`
   replaces the mark and inserts.
6. **Undo** — restores exact prior `(value, caret)`; coalesced typing is one
   undo (if landed) else per-char.

## Consequences

- `App` gains: `anchor` hash, clipboard seam pools, marked-text state, per-input
  undo history, `key_down_mod`, `copy`/`cut`/`paste`/`undo`(`/redo`),
  `text_editing`, prefix-width / char-at-x helpers. `key_down` becomes a
  back-compat alias → `key_down_mod(code, false, false)`.
- `Col` gains: `textarea` builder + a `multiline` flag (per-node Bool hash).
- Paint gains: selection highlight rect, measured-prefix caret x, multi-line text
  draw, marked-text underline. All default-inert: a plain single-line `input`
  with no selection / no marked text / no measure is byte-identical to today.
- The single-line `input` pins (`tests/input.rx`, `tests/caret_edit.rx`) stay
  green unchanged (selection collapsed = caret-only = old behaviour).

## Canvas gaps filed (for the coordinator to route)

1. **`Event.KeyDown` modifier bitmask** (shift/ctrl/cmd/alt) — REQUIRED for
   keyboard selection-extend and ctrl+c/x/v/z from real keys. SDL exposes
   `SDL_GetModState` / `SDL_Keymod`; the canvas event ring carries only the
   keycode today. Until added, the App API (`key_down_mod`) is complete and
   pinned headless, but shells pass `shift=false`/`ctrl=false`, so those
   keyboard paths are dormant on a real window. Severity: medium (App contract is
   ready; only the shell wiring is blocked).

## What this defers (revisit triggers)

- Real keyboard modifiers (blocked on the canvas gap above).
- Multi-line selection highlight rects across lines (single-line is exact).
- Measured-x column preservation on up/down (char-column lands first).
- Soft wrap (consistent with F1 deferred wrap).
- Textarea internal scroll if the list-clip reuse is not cheap.
- Redo / undo coalescing if not cheap (per-op undo lands regardless).
- Word-granularity (ctrl+arrow) move/delete, double-click word select.
