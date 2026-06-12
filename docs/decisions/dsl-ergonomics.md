# DSL ergonomics — audit + the in-language wins (2026-06-09)

quiver's DSL is **Ruby-flavoured builder blocks**, not a declarative tree of
widgets: `App.build({ |ui, root| … })`, `root.row({ |c| … })`,
`root.dyn_text({ |ui| … })`. That block idiom is the intended aesthetic — this
audit refines *within* it (less ceremony, more consistency), it does **not**
redesign toward a Flutter-style widget API.

The authoring surface a real app developer sees is the three examples
(`examples/counter`, `examples/todo`, `examples/settings`) plus the worked
snippets in `docs/DSL.md`. This doc is an honest critique of the friction there
and a record of what was fixed in-language vs. what is gated on a ruxen change.

## The pain points (from the example apps)

Ranked by how often a developer hits them.

### P1 — closure parameter type annotations on every stored block *(FIXED)*

Every reactive closure and every container block spelled out the full type of
its parameter:

```ruxen
root.dyn_text({ |ui2: &var Ui| "count: #{count.get(ui2)}" })
root.button("tap", { |ui2: &var Ui| count.update(ui2, { |c| c + 1 }) })
root.row({ |c: &var Col| c.text("x") })
root.list_of(todos, h, { |c: &var Col, m: ListModel, i: Int| c.text(m.row_text(i)) })
```

The `: &var Ui` / `: &var Col` / `: ListModel` / `: Int` annotations are pure
noise — the builder method's signature already fixes them. They read like Rust,
not like a Ruby block. This is the single most repeated piece of ceremony in
every app.

**Fix (verified, no language change):** the inner block param types can simply
be omitted — ruxen infers them from the builder method's `Fn(...)` parameter:

```ruxen
root.dyn_text({ |ui2| "count: #{count.get(ui2)}" })
root.button("tap", { |ui2| count.update(ui2, { |c| c + 1 }) })
root.row({ |c| c.text("x") })
root.list_of(todos, h, { |c, m, i| c.text(m.row_text(i)) })
```

This is **additive and non-breaking** — the annotated form still compiles
everywhere (every existing test proves it); the terse form is an equivalent,
cleaner spelling. It now reads as an ordinary Ruby block: `{ |ui| … }`. Applied
across all three examples and the `docs/DSL.md` worked snippets.

### P2 — the outer `App.build` block still needs its param types *(LANGUAGE-GATED → Q-candidate)*

The one place the terse form does **not** work is the outermost build block:

```ruxen
# works:
App.build({ |ui: &var Ui, root: &var Col| … })
# does NOT compile — `root.dyn_text` fails to resolve (?T unresolved):
App.build({ |ui, root| … })
```

`App.build` takes `f: any Fn[Fn(&var Ui, &var Col) -> nil]`, so in principle the
param types are just as inferable as the inner blocks'. In practice this build's
inference does not flow the `Fn(...)` element types into the *top-level* closure
literal's parameters the way it does for a closure passed to a regular method
arg, so `root`'s type stays `?T` and method dispatch (`root.dyn_text`) fails at
codegen (`no runtime symbol for ?T::dyn_text`). This is a real, narrow inference
gap, not a quiver design choice. **Filed as a Q-candidate below.** Until it's
fixed, the build block keeps `{ |ui: &var Ui, root: &var Col| … }` — the one
annotated block per app, which is a fair place to be explicit anyway (it's the
app's entry point).

### P3 — `&var *root` reborrow at every view call site

The settings app threads `root` into view functions:

```ruxen
header(&var *root, &name, &dark)
name_field(&var *root, &name)
toggles(&var *root, &notify, &dark)
```

The `&var *root` (reborrow) is required because `root` is itself a `&var Col`
and passing it to more than one call would move the reference (landmine:
"a closure passing its `&var T` param as an argument more than once must
reborrow"). This is correct and unavoidable *today*; it is ceremony forced by
the borrow model, not by quiver. It reads acceptably once you know the rule, and
there is no in-language way to drop it without either (a) making views methods on
a wrapper that owns the `Col` (a bigger redesign that fights the "views are free
functions" model the tutorial teaches) or (b) a language change to auto-reborrow
a `&var` argument used N>1 times. **Left as-is; noted as a Q-candidate (low
priority).**

### P4 — `let style = card_style` let-bind before a method arg

```ruxen
let style = card_style
root.row_styled(style, { |c| … })
```

You cannot write `root.row_styled(card_style, …)` — handing a non-Copy class
CALL-RESULT directly as a by-value *method* argument segfaults in this ruxen
build (landmine, verified 2026-06-09). The `let`-bind is the documented fix. It
is one extra line per styled container. **Language-gated** (the same root cause
as the call-result landmine); not hacked around. Noted as a Q-candidate.

### P5 — three separate state constructors

`ui.state(0)` / `ui.state_str("")` / `ui.state_bool(false)` are distinct names
because ruxen v1 has no method overloading on the argument type
(`&str`-vs-other overloads miscompile — landmine). This is *fine* and arguably
clearer than an overloaded `state(...)`; it reads well as a Ruby-ish builder.
**No change** — calling it out only so the next reader knows it is deliberate,
not an oversight.

## What changed in this pass

- **P1 applied everywhere.** All inner stored-closure / container-block params
  dropped their type annotations across `examples/counter`, `examples/todo`,
  `examples/settings` (views + main), and the worked snippets in `docs/DSL.md`
  and `docs/REACTIVITY.md`. The annotated form remains valid; nothing breaks.
- Everything else (P2–P5) is **language-gated** and filed below rather than
  worked around — per the brief, an ergonomic win that needs a ruxen change is a
  Q-candidate, not a hack.

## Q-candidates for the ruxen ledger (triage upstream)

These are the in-language ergonomic wins that need a compiler change. Listed
here for the user to triage into `../ruxen/docs/dev/gui-stack-v1-issues.md` +
`../ruxen/docs/TASKS.md`.

- **Qxx (P2) — infer param types for a closure literal passed to a method whose
  `Fn` type is known, at the TOP level too.** Today inner blocks
  (`root.row({ |c| … })`) infer `c: &var Col`, but the outermost
  `App.build({ |ui, root| … })` does not — `root` stays `?T` and `root.dyn_text`
  fails to resolve at codegen. Closing this would let the entry-point block drop
  its annotations like every other block. Severity: low (one annotated block per
  app), but it is the most visible remaining annotation. Repro: omit the types on
  the `App.build` block param of `examples/counter` → `no runtime symbol for
  ?T::dyn_text`.

- **Qxx (P4) — non-Copy class call-result as a by-value method argument.**
  Already a known landmine (segfault; the `let`-bind workaround). Fixing it would
  remove the `let style = card_style` line before every `row_styled`/`col_styled`
  and similar. Severity: medium (it is a *crash*, not just ergonomics — the
  ergonomic cost is the visible symptom). Cross-link the existing landmine repro
  `tmp/test-cache/ruxen-direct-callresult-method-arg-segfault.md`.

- **Qxx (P3) — auto-reborrow a `&var T` argument used more than once.** Would
  drop the `&var *root` ceremony when threading one `&var Col` into several view
  calls. Severity: low (the explicit reborrow is teachable and correct).
```
