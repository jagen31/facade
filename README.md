# facade

The fourth iteration of the **Art** language — rewritten in
[Rhombus](https://rhombus-lang.org).

Art (art3, in Racket) is a framework for extensible *Art languages*: a host of
art forms — objects, coordinates, rewriters, and embeddings — composed by the
`@`-form, with rewriting driven by context. Tonart (music), Spielart (scripts),
and Programmart (concert programs) are instantiations of it. `facade`
re-imagines that core in Rhombus.

## Design

`facade` is a faithful port of art3's core
([`art/private/core.rkt`](https://github.com/jagen31/art3)): a **compile-time
metaprogramming system over syntax objects**, not a runtime library. Art forms
are Rhombus *syntax*; the engine rewrites that syntax at expansion time, exactly
as art3 does.

| art3 (`core.rkt`, Racket) | facade (`core.rhm`, Rhombus) |
|---|---|
| `define-syntax name (object/s)` | a name bound in the `af` **space** to a compile-time kind |
| `syntax-local-value` | `syntax_meta.value(name, af_meta.space, …)` |
| identity context (syntax property) | a `group_property` on each form's syntax |
| `(@ [(coord …)] …)` | `at [coord, …]: …` (`@` is reserved in Rhombus) |
| coordinate merge / `within?` rules | attached to each coordinate's binding |
| `run-art-expr` / `rewrite` | same, as `meta` functions over syntax |
| realizers | `realize name: …` — rewrites, then runs the realizer |

Merge/`within?` rules travel with a coordinate's binding (rather than a global
mutable registry) so they are visible wherever the coordinate is — the
expansion-time analogue of art3's per-coordinate rules.

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

See `facade-lib/tests/core-demo.rhm` for a worked example covering coordinate
merging, nested-interval translation, a rewriter, an embedding, and a realizer.

## Layout

- `facade-lib/` — the library (collection `facade`)
  - `main.rhm` — the public entry point (re-exports the core)
  - `private/core.rhm` — the compile-time engine
  - `tests/core-demo.rhm` — the worked example
- `facade/` — the metapackage (pulls in `facade-lib`)

## Local build

You need `raco` on your path (https://download.racket-lang.org/), and the
`rhombus` package installed.

```
raco pkg install facade/ facade-lib/
raco make facade-lib/main.rhm
racket facade-lib/tests/core-demo.rhm
```
