# quiver — Roadmap

`quiver` is **L2** of the GUI stack. The first milestone is the **desktop
counter app** — the thin vertical slice (with `canvas` L1) that proves the whole
stack and, crucially, de-risks the *language-level* reactive API.

## Milestone 0 — scaffold ✅ (current)

- `Ruxen.toml` (library + `canvas` path dependency).
- API skeleton: `Signal[T]`, `Widget`, `column`/`text`/`button`, `run`.
- `examples/counter.rx`.
- Design docs (`DESIGN`, `ARCHITECTURE`, `REACTIVITY`, `DSL`, `ROADMAP`).

## Milestone 1 — the counter app (L2 core slice)

The minimum that proves L2 end-to-end on top of `canvas` Milestone 1:

1. **Signal arena runtime** + `Signal[T]` (`Copy` handles; deterministic drop).
2. **Block DSL** for `column` / `text` / `button`, with the
   `{}`-reactive / value-static rule.
3. **One simple layout pass** (enough for a column of three nodes).
4. **Paint** onto `canvas`'s `Canvas`.
5. **Event dispatch**: click → `count.update` → **targeted repaint** of only the
   count node.
6. Tests pinning the static-vs-reactive boundary.

**Demo:** tap a button, the number increments — the "hello world" of reactive
GUI.

**Explicitly out of this slice:** mobile/web, the full widget library,
accessibility, text i18n/shaping beyond basic, packaging.

## Open decisions (resolved in the core-slice spec, not here)

- **Signal API shape** — `count.get`/`count.set` vs call-style `count()`.
- **Layout engine** — own simple flex vs bind Yoga (C, would live in `canvas`).
- **First-slice widget set** — bounded by the counter app (`column`, `text`,
  `button`).

## Later cycles

- **Widget library** — buttons, text, lists, inputs, containers, …
- **Text / i18n / accessibility** — via `canvas`'s Skia paragraph/HarfBuzz/ICU
  and platform accessibility trees.
- **Platform matrix** — desktop → mobile → web (tracks `canvas`).
