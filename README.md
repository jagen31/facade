# facade

The fourth iteration of the **Art** language — rewritten in
[Rhombus](https://rhombus-lang.org).

Art (art3, in Racket) is a framework for extensible *Art languages*: a host of
art forms — objects, coordinates, rewriters, and embeddings — composed by the
`@`-form, with rewriting driven by context. Tonart (music), Spielart (scripts),
and Programmart (concert programs) are instantiations of it. `facade`
re-imagines that core in Rhombus.

## Design

`facade` is a faithful **runtime** port of art3's core
([`art/private/core.rkt`](https://github.com/jagen31/art3)). Where art3 does
everything at compile time over Racket *syntax*, facade models art forms as
runtime `Art` values and runs the same engine:

- **objects / coordinates / rewriters / embeddings** — registered by head symbol
- **identity context** — the list of coordinates each form carries
- **the `@`-form** (`at`) — attaches coordinates, which merge *downward* into
  the forms they wrap
- **coordinate merge / `within?` rules** — per-coordinate, e.g. interval
  translation or voice shadowing
- **current / lookup contexts** — the two dynamic contexts rewriters see
- **realizers** — compile a rewritten art into an ordinary value

A "context" is just a `List` of `Art`. (A compile-time macro *surface* on top of
this engine — so programs read like art3's `(@ [...] ...)` syntax — is the
natural next step.)

## Layout

- `facade-lib/` — the library (collection `facade`)
  - `main.rhm` — the public entry point (re-exports the core)
  - `private/core.rhm` — the engine
  - `tests/core-demo.rhm` — a worked example (objects, coordinate merging,
    a rewriter, an embedding, `context_ref`, a realizer)
- `facade/` — the metapackage (pulls in `facade-lib`)

## Usage

```
#lang rhombus
import: facade open

define_object(#'note)
define_coordinate(#'interval)

def ctxt = rewrite([at([o(#'interval, 0, 4)], o(#'note, #'c), o(#'note, #'d))])
// -> two `note` forms, each carrying an `interval` coordinate
```

## Local build

You need `raco` on your path (https://download.racket-lang.org/), and the
`rhombus` package installed.

```
raco pkg install facade/ facade-lib/
```

Recompile / run the demo after edits:

```
raco make facade-lib/main.rhm
racket facade-lib/tests/core-demo.rhm
```
