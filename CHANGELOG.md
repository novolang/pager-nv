# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.0 — 2026-11-03

### Fixed

- Recovering a log in which many frames are commit records takes time linear in the log, and a checkpoint lists the distinct pages of the log in linear time.

This release requires novo 0.23.0 or later.

## 0.1.0 — 2026-10-11

The bodies land.  Every function the interface release declared is
implemented, and every test passes.  No public signature, effect
row or doc comment changed from 0.0.1, and the module headers still
state the design they stated then.

- The toolchain floor is `novo = ">= 0.20.2"`, the release the bodies
  were built and measured on.
- `sql-engine-nv` is wired by path, `{ path = "../sql-engine-nv" }`,
  while the two packages are developed together: the registry's 0.0.6
  is an interface release whose bodies are `todo()`, so `driver` cannot
  run against it.  The line becomes the registry range `^0.1.0`, the
  release that carries those bodies, before this package is published.
  `novo pkg publish` refuses a manifest with a path dependency until
  then, which is that rule working and not a defect here.
- `tests/pager_edge_tests.nv` is new: 53 tests over the wrong files and
  the wrong devices, from a header with the wrong magic and a frame
  with one byte moved to a full device in the log's place and a catalog
  page holding a line that is not a table.

### Known

- Line coverage of `src/` is 97.4% as the per-line maximum over the seven
  suites, and every line below that is inside a `// cov: skip` region
  with its reason at the site.  The regions are error arms that need the
  system to take one call and refuse the next, or a file to shrink under
  an open descriptor: no file a suite can name does either, because the
  package calls `std.fs` directly.  The ones a file can provoke are
  tested in `tests/pager_failure_tests.nv` and
  `tests/driver_failure_tests.nv`.  `recover_log` now answers a `Pager`
  and not a `Result`, since it had no path to an error.

## 0.0.6 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- btree-nv: `^0.0.3` to `^0.1.0`.
- sql-engine-nv: `^0.0.4` to `^0.0.6`.

## 0.0.5 — 2026-09-15

- README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.4 — 2026-09-12

- **BREAKING — `cell.Cell` is `btcell.Cell`.**  btree-nv's value module was named `cell`, which is a standard library module name and no longer builds, so btree-nv 0.0.3 renamed it to `btcell`.  Nothing in this package's own surface changed: the type is the same type, and a consumer that names it replaces `use cell` with `use btcell`.
- Depends on `btree-nv = "^0.0.3"` and `sql-engine-nv = "^0.0.4"`.  The versions below those do not build.

## 0.0.3 — 2026-09-10

- **Toolchain floor is 0.8.9**: the bodies and signatures use what 0.8.9 added (`todo()`, a bound effect parameter, the four layers), and the manifest says so instead of letting an older toolchain fail on an undefined function.  No signature changed.

## 0.0.2 — 2026-09-09

- **Dependencies are registry ranges**, not paths: the interface release 0.0.1 shipped a manifest whose dependencies pointed at sibling directories that exist only in the monorepo, so a consumer resolved the closure and then could not load the dependency.  No signature changed.

## 0.0.1 — 2026-09-09

- First interface release: every public signature and effect row, every body a `todo()`.  **NOT IMPLEMENTED — interface only.**
