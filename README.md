# facade

The fourth iteration of the **Art** language — rewritten in
[Rhombus](https://rhombus-lang.org).

Art (art3, in Racket) is a framework for extensible *Art languages*: a host of
art forms — objects, coordinates, rewriters, and embeddings — composed by the
`@`-form, with rewriting driven by context. Tonart (music), Spielart (scripts),
and Programmart (concert programs) are instantiations of it. `facade`
re-imagines that core in Rhombus.

## Layout

This repository holds multiple packages, mirroring the `art3` layout:

- `facade-lib/` — the library (collection `facade`)
  - `main.rhm` — the public entry point
  - `private/core.rhm` — the core: objects, coordinates, the `@`-form, rewriting
- `facade/` — the metapackage (pulls in `facade-lib`)

## Local build

You need `raco` on your path (https://download.racket-lang.org/), and the
`rhombus` package installed.

```
raco pkg install facade/ facade-lib/
```

Then, from a Rhombus module:

```
#lang rhombus/static
import: facade
block:
  println(facade.facade_version)
```

To recompile after edits:

```
raco make facade-lib/main.rhm
```
