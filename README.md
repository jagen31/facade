# facade

The fourth iteration of the **Art** language — rewritten in
[Rhombus](https://rhombus-lang.org).

Art (art3, in Racket) is a framework for extensible *Art languages*: a host of
art forms — objects, coordinates, rewriters, and embeddings — composed by the
`@`-form, with rewriting driven by context. Tonart (music), Spielart (scripts),
and Programmart (concert programs) are instantiations of it. `facade`
re-imagines that core in Rhombus.

## Design

`facade` is a port of art3's core
([`art/private/core.rkt`](https://github.com/jagen31/art3)): a metaprogramming
system over syntax objects. Art forms are Rhombus *syntax*; the engine rewrites
that syntax, exactly as art3 does.

Unlike art3 (and unlike facade's own first cut), **the engine lives at phase 0**.
`private/interp.rhm` is ordinary `#lang rhombus` runtime code — the syntax
utilities, coordinate merge / `within?`, the art-id machinery, and
`rewrite` / `run_art_expr` — all manipulating syntax objects as plain values. So
the same engine can run at *either* phase. `private/core.rhm` is the phase-1 skin
that imports the engine `for-syntax`, re-exports it `for-syntax`, and wraps it in
the surface macros (`define_object`, `realize`, …).

The one inherently phase-specific thing an interpreter needs is *"what art form
is this name bound to?"*. art3 answered with `syntax-local-value` (phase 1 only);
facade abstracts that into a dynamic **parameter**, `kind_lookup` (consulted by
`kind_of`). Its *default* is art3's `syntax-local-value` over the `af` space:
`core.rhm` — the only module that can name `syntax_meta.value` — fills in that
default at phase 1 (via `install_default_kind_lookup`). So once facade is loaded,
the lookup **defaults to `syntax_meta.value`** and the surface macros need no
wrapping. A phase-0 client overrides it for a dynamic extent with
`with_kind_lookup(lookup, thunk)` (or `parameterize { kind_lookup: … }`) — e.g. a
runtime registry — and runs the very same `rewrite`. See
`facade-lib/tests/phase0-demo.rhm`.

| art3 (`core.rkt`, Racket) | facade (Rhombus) |
|---|---|
| `define-syntax name (object/s)` | a name bound in the `af` **space** to a kind |
| `syntax-local-value` | the `kind_lookup` **parameter** (`kind_of`); defaults to `syntax_meta.value` over the `af` space, overridable per extent |
| identity context (syntax property) | a `group_property` on each form's syntax |
| `(@ [(coord …)] …)` | `at [coord, …]: …` (`@` is reserved in Rhombus) |
| coordinate merge / `within?` rules | attached to each coordinate's binding |
| `run-art-expr` / `rewrite` | same, as **phase-0** functions over syntax |
| realizers | `realize name: …` — rewrites, then runs the realizer |

Merge/`within?` rules travel with a coordinate's binding (rather than a global
mutable registry) so they are visible wherever the coordinate is — the analogue
of art3's per-coordinate rules.

## Usage

```
#lang rhombus/and_meta
import: facade open

define_object note
define_coordinate interval ~merge:
  fun (l_star, r_star, l, r, ctxt):     // nesting translates the start
    cond
    | l_star && r_star:
        def s = args_of(l_star)[0].unwrap()
        'interval $(Syntax.make(s + args_of(r_star)[0].unwrap())) $(args_of(r_star)[1])'
    | ~else: (r_star || l_star)

define_realizer pitches:
  fun (ctxt): context_ref_all(ctxt, #'note).map(fun (n): args_of(n)[0].unwrap())

realize pitches:
  note c 0 5
  at [interval 0 4]:
    note c 0 5
    note d 0 5
// => [c, c, d]  (each nested note carries the merged interval coordinate)
```

See `facade-lib/tests/core-demo.rhm` and `facade-lib/tests/extras-demo.rhm` for
worked examples.

## Surface forms

| form | art3 analogue |
|---|---|
| `define_object name` | `define-art-object` |
| `define_coordinate name:` with `merge:` / `within:` / `nonhom:` clauses | `define-coordinate` + `define-hom-…` / `define-nonhom-…` rules |
| `define_rewriter name: fun (expr): …` | `define-art-rewriter` |
| `define_embedding name: fun (expr): …` | `define-art-embedding` |
| `define_realizer name: fun (ctxt): …` | `define-art-realizer` |
| `define_art name: <program>` | `define-art` (a variable, spliced on reference) |
| `at [coord, …]: <body>` | `(@ [(coord …)] …)` |
| `realize name: <program>` | `(realize (name …) …)` |
| `delete_by_id id` / `replace_full_context: …` | same (engine special forms) |

Context helpers available to rewriters and realizers (all `meta`): `context_ref`,
`context_ref_all`, `context_ref_surrounding`, `context_ref_within`,
`merge_coordinates`, `context_within`, `current_ctxt`, `lookup_ctxt`, `get_ctxt`,
`head_of`, `args_of`, `body_of`, `art_id_of`, `delete_expr`.

## Coordinates

`facade-lib/coordinates.rhm` ports art3's standard coordinates
(`art/coordinate/*`), written flat (`interval 0 4`, `index 1 2`, `name a b`):

| coordinate | merge | `within?` | accessors |
|---|---|---|---|
| `interval start end` | translate (`outer.start + inner`) | `l.start ∈ [r.start, r.end)` | `expr_interval`, `expr_start`, `expr_end` |
| `index i …` | append | `r` is a prefix of `l` | `expr_index` |
| `name a …` | append | `r` is a prefix of `l` | `expr_names` |
| `instant t` | — (can't merge) | equal `t` | `expr_instant` |
| `subset x …` | union | `r ⊆ l` | `expr_subset` |

They're re-exported from `main.rhm`, so `import: facade open` brings them in.

### Sequencing (`--`)

`coordinates.rhm` also ports art3's `--`: it lays `[len, body …]` boxes end
to end on the interval timeline, so box *i* covers `[t, t+len)` (as
`at [interval t (t+len)]: body; …`) and `t` advances by `len`. An optional
leading number sets the start (default 0). Because `--` is an *operator* token
in Rhombus rather than an identifier, it's written `#{--}` (which prints as
`--`):

```
#{--} [1, note a 0 4] [2, note d 0 5] [1, note c 1 5]   // 0..1, 1..3, 3..4
#{--} 4 [1, note a 0 4] [1, note b 0 4]                 // 4..5, 5..6
```

## Layout

- `facade-lib/` — the library (collection `facade`)
  - `main.rhm` — the public entry point (re-exports core + coordinates)
  - `private/interp.rhm` — the phase-0 engine (utils + rewrite, lookup abstracted)
  - `private/core.rhm` — the phase-1 skin: the `af` space + surface macros
  - `coordinates.rhm` — the standard coordinate library
  - `tests/` — `core-demo`, `extras-demo`, `coords-demo`, `phase0-demo`
- `facade/` — the metapackage (pulls in `facade-lib`)

## Local build

You need `raco` on your path (https://download.racket-lang.org/), and the
`rhombus` package installed.

```
raco pkg install facade/ facade-lib/
raco make facade-lib/main.rhm
racket facade-lib/tests/core-demo.rhm
```
