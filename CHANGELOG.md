# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## Unreleased

### Fixed

- Recovering a log in which many frames are commit records takes time linear in the log, and a checkpoint lists the distinct pages of the log in linear time.

## 0.1.0 — 2026-10-11

The bodies land.  Every function the interface release declared is
implemented, and 97 of the 98 tests pass.  No public signature, effect
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

- `tests/pagefmt_tests.nv test_a_corrupted_wal_header_is_refused` fails
  and cannot be made to pass without changing the format it tests.  The
  test zeroes byte 16 of an encoded WAL header and expects the checksum
  to stop decoding it; byte 16 is the HIGH byte of the big-endian salt
  1 that the same test wrote as 111, so zeroing it moves nothing, the
  bytes stay a header and the decode succeeds.  The only reading that
  would make the test pass is a little-endian salt field, which
  contradicts the one byte order `src/pagefmt.nv` and the README both
  state.  The test is left as it is rather than edited to go green.
- Line coverage of `src/` is 733 of 780, or 94.0%, as the per-line
  maximum over the four suites: `pagefmt.nv` 116 of 116, `pager.nv` 372
  of 398, `driver.nv` 245 of 266.  What is left is error arms a test
  cannot reach without a fault-injecting filesystem: a write the system
  takes and an fsync it then refuses, a file that exists and cannot be
  opened, a log truncated after a commit, and two arms that exist for a
  signature rather than for a caller (`recover_log`'s `Err`, which no
  path reaches, and `next_row`'s "stopped without a row and without
  finishing", which `drive_rows` cannot return).

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
