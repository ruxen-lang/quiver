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

## Ruxen v1 landmines (all verified; violating them = crash or miscompile)

- No top-level globals. No `Signal`/`Runner`/… class names (flat namespace,
  std collision). No module-wrapped generic classes.
- No recursive class types (`Array[Self]` field ⇒ compiler stack overflow)
  → flat parallel arrays + `-1` sentinels.
- No `Option[any Fn]` fields (garbage values) → index arrays into closure pools.
- No overloading one method name on `&str` vs closure (heap corruption).
- `do…end` blocks segfault when passed to *free functions* with explicit
  closure params; methods are fine. Stored callbacks use `{ |x| … }` braces.
- `move` closures can't capture non-Copy class values; plain capture works
  (pointer-copy) — sound today because drops don't run yet.
- `arr[i]` with a non-literal index parses as generics → use `.get(i)`;
  there's no index assignment → push-only arrays or `Hash.insert`.
- `.size` returns `USize` → cast `as Int`. Multi-statement `match` arms need
  `if let` or a helper (braces in an arm parse as a closure). `layout` is a
  keyword. Doc comments (`##`) between mixin method signatures break parsing.
- Test files: synthesized into `def main` — no top-level `def`s (use
  `let`-closures inside `Tester.describe`), no `use` of this package, plain
  `#` comments at file top. Generic functions don't monomorphize for types
  defined in a *consuming* package (keep mixin impls used by quiver generics
  inside quiver).
