# CLAUDE.md

quiver — the L2 framework of the Ruxen GUI stack: fine-grained reactive
state + builder DSL + layout + paint + dispatch, in **100% safe Ruxen**
(no `unsafe`, no FFI — that's `canvas`, the L1 sibling at `../canvas`).
Written in Ruxen (`.rx`), built with the `ruxen` toolchain (`~/.ruxen/bin`).

## Commands

```bash
ruxen test                        # the 37-test headless suite (tests/*.rx)
ruxen test app                    # substring-filter on test FILE path
ruxen test --nocapture --format=documentation   # see per-test output
ruxen build                       # library rlib (compiles canvas dep piece too)
cd examples/counter && ruxen run  # the milestone demo binary
ruxen fmt <file>                  # canonical formatting
```

## Layout

- `src/lib.rx` — reactive core: `Ui` (subscription graph, passed as `&var Ui`)
  + `State[T]` (value behind `SharedSync[Mutex[T]]`)
- `src/dsl.rx` — `Col` builder: `text` / `dyn_text` / `button`, flat-arena tree
- `src/app.rx` — `App`: build/mount, targeted `flush`, `arrange` (layout),
  hit-test + `pointer_down`
- `src/paint.rx` — `PaintSurface` mixin, `RecordingSurface`, `paint_all`/`paint_dirty`
- `src/run.rx` — `first_frame` / `frame` drivers
- `examples/counter/` — separate **binary** package (the only place that may
  reference canvas symbols)
- `docs/` — design docs; DSL.md and REACTIVITY.md record every API deviation
  from the original sketch and why

## Invariants

- **The DSL rule is load-bearing**: builder block runs once; only `dyn_text`
  closures ever re-run. `tests/app.rx` + `tests/counter.rx` pin it — changing
  those tests is a design event, not a refactor.
- Tracking-scope ids ARE node ids; `flush` returns exactly the repaint set.
- No canvas symbols in `src/` or `tests/` (ruxen v1 library/test builds don't
  merge dependency sources; only binaries do).
- One `Mutex` lock per function frame — guards release at function exit only.
  **This spans a `State.peek` + `State.set` PAIR on the same handle:** a `peek`
  taken anywhere in a frame is still held when a later `set` runs in that same
  frame (even if each is in its own sub-method, because the guard releases at
  the *calling* frame's exit), and the two locks collide → the value silently
  corrupts/empties (verified 2026-06-08 building the text input). Read-modify
  -write must go through ONE `State.update(ui, { |cur| … })` call, not a
  peek-then-set; that locks exactly once. A lone `peek` (no `set` in the frame)
  is fine.

## Ruxen v1 landmines (all verified; violating them = crash or miscompile)

- No top-level globals. No `Signal`/`Runner`/… class names (flat namespace,
  std collision). No module-wrapped generic classes.
- No recursive class types (`Array[Self]` field ⇒ compiler stack overflow)
  → flat parallel arrays + `-1` sentinels.
- No `Option[any Fn]` fields (garbage values) → index arrays into closure pools.
- No overloading one method name on `&str` vs closure (heap corruption).
- `move` closures can't capture non-Copy class values; plain capture works
  (pointer-copy) — sound today because drops don't run yet.
- `arr[i]` with a non-literal index parses as generics → use `.get(i)`;
  there's no index assignment → push-only arrays or `Hash.insert`.
- A closure passing its `&var T` param as an argument more than once must
  reborrow: `f(&var *ui2)`. Hash tuple-iteration values don't type-resolve
  for method calls → iterate `.keys` + `.get`.
- `.size` returns `USize` → cast `as Int`. Multi-statement `match` arms need
  `if let` or a helper (braces in an arm parse as a closure). `layout` is a
  keyword. Doc comments (`##`) between mixin method signatures break parsing.
- Test files: synthesized into `def main` — no top-level `def`s (use
  `let`-closures inside `Tester.describe`), no `use` of this package, plain
  `#` comments at file top. Generic functions don't monomorphize for types
  defined in a *consuming* package (keep mixin impls used by quiver generics
  inside quiver).
- **Q25 (ruxen ledger) — RESOLVED 2026-06-08.** `Hash.key?`/`get` on an EMPTY
  hash used to SEGFAULT; **fixed and installed** — empty-hash lookups now return
  `false`/`nil` cleanly (verified Int + String value types). The defensive
  `if h.size as Int > 0` guards quiver added were removed; lookups are now plain
  `match h.get(i) { … nil -> sentinel }`. Repro kept for history in
  `tmp/test-cache/ruxen-empty-string-hash-segfault.md`.
- **`&Hash[...]` / `&Set[...]` parameter types are unsound** — `Hash`/`Set` are
  mixins without runtime dispatch. Both *free fns* and *methods* now reject a
  `&Hash`/`&Set` param at compile time (`E1118`; the silent-method-miscompile
  facet was fixed alongside Q25). Still: don't pass a collection by `&` — access
  the field directly (`self.<field>.get`).
- **Never pass a non-Copy class CALL-RESULT directly as a by-value METHOD
  argument — `let`-bind it first** (verified 2026-06-09). `b.take(make)` where
  `make -> SomeClass` and `take(t: SomeClass)` is a method SEGFAULTS; `let t =
  make; b.take(t)` works. Free fns are fine; Int/Copy args are fine; only a
  non-Copy class temporary into a *method* arg crashes (its storage is freed
  before the call). Repro: `tmp/test-cache/ruxen-direct-callresult-method-arg-segfault.md`.
- **No `State[Array[T]]` / `Mutex[Array[T]]` / `SharedSync[Array…]`** — the
  Send-ness of `Array`'s element type doesn't propagate, so these don't
  construct (`E1011`/`E1101`/`E1102`), even though `Array[Int]`/`String` are
  `Send`. Back a shared mutable collection with a plain class that `include
  Send` (holds the `Array` directly) and share it via `SharedSync` capture.
  Repro: `tmp/test-cache/ruxen-state-array-send.md`.
- **Q26 (ruxen ledger) — RESOLVED 2026-06-08.** A capturing closure stored from
  inside a self-reborrow build block (`def var container(build) … build.(&var
  *self)`) used to lose its captures (wrong Int; SEGFAULT for a captured class
  handle). **Fixed and installed** — reactive `dyn_text`/`button` children inside
  `row`/`col` now keep their captured `State` handles and work exactly like
  top-level reactive nodes (pinned by `tests/nesting.rx`: "a dyn_text child
  re-renders when a button child mutates its state"). Repro (now passing) in
  `tmp/test-cache/ruxen-closure-capture-reborrow.md`.
- **`yield` with TWO `&var` reference args miscompiles (ruxen ledger Q36).** A
  `yield(&var app.ui, &var app.root)` — two `&var` references to fields of the
  same object in one `yield` — builds into an EMPTY/wrong target (the block runs
  but its `&var` params don't bind). A single-`&var`-arg `yield(&var *self)` is
  fine, and the ordinary closure-call form `f.(&var app.ui, &var app.root)` is
  fine. **This is why `App.build` stays an explicit closure param** (`f: any
  Fn[Fn(&var Ui, &var Col) -> nil]`), not a `&block` + `yield`. The single-`&var`
  container builders (`row`/`col`/`list`/`row_styled`/`col_styled`) DID convert
  to `&block` + `yield`. Repro: `tmp/test-cache/ruxen-two-var-yield.md`.
- **Block params don't infer through the `&block`/`yield` seam.** A container
  builder block must TYPE its param: `root.row do |c: &var Col| … end`, never
  `do |c| … end` (the latter leaves `c` as `?T` → `no runtime symbol for ?T::text`
  at codegen). Typed works in both `do…end` and `{ }` forms.

## Blocks, alias & string literals (the current idiom)

- **`&block` + `yield` for immediately-invoked builders.** The container builders
  (`row`/`col`/`list`/`row_styled`/`col_styled`) declare a `&block: Fn[(&var Col)
  -> nil]` and `yield(&var *self)`; call sites read `root.row do |c: &var Col| …
  end`. `list`'s block is OPTIONAL via `block_defined?` (`root.list(h)` with no
  block builds an empty viewport). See the ruxen tutorial
  `../ruxen/docs/tutorial/09-closures-and-blocks.md` and the two ADRs
  `../ruxen/docs/decisions/{ruby-block-semantics,alias-keyword}.md`.
- **STORED callbacks stay `{ |x| … }` brace closures, not blocks** — a `&block`
  slot is for yield-time use; storing it is a different lifetime story.
  `dyn_text`, `button` handlers, `set_measure`, and `list_of`'s item builder
  (re-invoked on every rebuild) are stored ⇒ explicit closure params / brace
  literals. House rule: **do…end = immediately-invoked structure; { } = stored
  behaviour.** Documented in docs/DSL.md.
- **`alias new old`** is a pure synonym (one body, zero extra codegen): `Col`
  has `alias length size`; `ListModel` has `alias size count` / `alias length
  count`; `App` has `alias type_char text_input` / `alias key key_down` (these
  replaced hand-written delegating methods). Operator-spelled aliases are staged
  (E1123) — don't use them.
- **String literals ARE `String` — never write `String.from` on a literal.** A
  bare `"x"` (interpolated literals too: `"x #{y}"`) coerces to `String` in every
  position (args, fields, `push`, `Err("…")`, `to_eq`). `String.from(...)` is
  kept ONLY for a `&str` VARIABLE → `String` conversion (a real runtime op, e.g.
  `String.from(label)` where `label: &str`).

### Q18 — RESOLVED 2026-06-08: stale divergent install (NOT a master bug)

**Root cause was a stale, DIVERGENT toolchain install** (`local-48c51aa`), a
commit from an in-progress stdlib→`.rx` migration branch whose migrated prelude
fails its own borrow-check — so that build could not compile *anything*, not
even a bare `def main` (`ruxen build`/`ruxen test` failed at fixed std-internal
positions; `ruxen check` still passed). It was **never a quiver bug and never a
master bug**.

Fix: the toolchain was **reinstalled from master** (`35d80b7`). Verified — a
bare program builds, and the full quiver suite is green (`ruxen test` → 42
passed, 0 failed). No quiver source change was involved.

Lesson for next time: if `ruxen build` fails on a *bare* `def main` with
std-internal borrow errors, suspect the install, not your code — check
`ruxen --version` / the install commit before assuming a language bug, and
reinstall from master.

## Task tracking & keeping context current

- **Canonical task list: [`docs/ROADMAP.md`](docs/ROADMAP.md)** — milestones,
  resolved decisions, and the "later cycles" backlog. Single place open quiver
  work is tracked; the workspace umbrella (`../CLAUDE.md`) links it, not duplicates it.
- Every change updates docs in the same commit: tick/extend `docs/ROADMAP.md`,
  add a `CHANGELOG.md` entry, and record any API deviation from the original
  sketch in `docs/DSL.md` / `docs/REACTIVITY.md` (that's their stated job).
  Changing `tests/app.rx` or `tests/counter.rx` is a design event — note why.
- A new Ruxen v1 landmine is a **language task** *and* a landmine-list update: add
  a `Q##` entry to `../ruxen/docs/dev/gui-stack-v1-issues.md` (repro + severity),
  list it in `../ruxen/docs/TASKS.md`, and add it to the landmines section above.
- A `Stop` hook in `.claude/settings.json` reminds you of the above when a session
  touched source.
