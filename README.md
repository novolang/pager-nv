# pager-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package
works; calling it panics with `not implemented`.

## What this is

The half that touches the file.  A database header, a write-ahead log
with SQLite's rolling checksum and its salts, one page in and one page
out, commit and checkpoint — and the loop that runs a `sql-engine-nv`
statement to completion by answering everything it asks for.

Three packages come out of novodb and this is the only one with `[fs]`
in it.  For an end user that means the effect rows are the map: a
signature here that does not say `[fs]` cannot reach a disk, and a
`core` package underneath cannot reach one at all.

## The one example that will work

```novo
use pager
use driver

fn main() [fs, io]
    match pager.open("app.db")
        Err(e) => println(e.message())
        Ok(p)  =>
            match driver.execute(p, "SELECT name FROM users", [])
                Ok(out) => println(string_of_int(out.rows_affected))
                Err(e)  => println(e.message())
```

## The layer, and why

`host`.  Two effects, and both are disclosures rather than
conveniences.

**`[fs]`** is on every function that reaches the file, and only on
those: `pagefmt` — the header, the frame, the checksum — is pure
arithmetic over byte lists and declares nothing.  It sits in a `host`
package because the FORMAT is the pager's, and it is written so that
moving it to `core` later costs a manifest edit and nothing else.
`tests/pagefmt_tests.nv` checks the whole format without touching a
disk, which is what that separation buys.

**`[time]`** is on exactly one function, `now_snapshot`, and it is here
because it could not be anywhere else.  novodb rewrites `DATE('now')`
and its family into literals inside its executor, and reaches for
`time.now()` to do it by DISCHARGING the effect — its own comment says
the discharge is what keeps the executor effect-free.  A `core`
package may not: `no-discharge-in-core` is a shard row, and
discharging an effect is a lie about a budget rather than a way to
meet one.  So the read moved to the only layer allowed to do it, and
`sql-engine-nv.substitute_now` takes the three formatted strings as an
argument.  One `[time]` function here replaces a discharge there.

## The load-bearing interface

```novo
pub fn answer(p: Pager, request: btree.PageRequest) -> Result<Answered, DriveError> [fs]
```

Four arms — `PrNeedPage`, `PrWritePage`, `PrAllocPage`, `PrFreePage` —
and that one function is the entire contract between the two `core`
packages and the machine.  `step_once` puts it beside a
`sqlengine.step`; `execute` puts `step_once` in a loop; and all three
are published, because a driver that is the only way in is a driver
that decides scheduling for its callers.

**btree-nv also publishes a second shape — `trait PageIo[e]` and
`pub fn run<S: PageIo[e]>(…) [e]` — and this package would supply
`impl PageIo[fs] for Pager`.**  That is SPEC § 5.6's design working
exactly as intended: a `core` walk charged `[fs]` because this impl
costs `[fs]`, and nothing in btree-nv's own budget touched.  The impl
is not in this release, and the reason is a compiler defect rather
than a design one — a generic function specialised across a module
boundary loses the bound it declared, and the clone is then checked
with `e` read as a typo'd effect label.  It is filed against novo-lang
as
*cross-module-specialisation-of-an-effect-polymorphic-function-drops-its-bound*.
When it is fixed the impl is four one-line methods and nothing in the
signatures below changes, which is the argument for having published
both shapes.

## The reference implementation

SQLite's pager and WAL as the design, and **novodb's `paged_file.nv`,
`paged_file_writer.nv`, `lazy_page_store.nv` and `pagefile.nv` as the
code** — for the read path.  The write path had nothing to port.

| novodb | here | why |
| --- | --- | --- |
| **no write-ahead log.**  `page_log_save` builds a whole in-memory image and serialises it; v2 appends a NEW image and moves an offset in an outer header, and recovery scans the file for valid image offsets | a WAL: frames, salts, a cumulative checksum, and a commit record that is a field on the last frame | novodb's is crash-safe and rewrites every page on every commit; its own read-side comment measures a 5.9 GB file costing ~47 GB of resident memory to open |
| the demand-paged reader keeps its cache in a runtime handle table behind `[ffi]` shims, so a caller need not thread the store | the pager is a value, threaded — `read_page` returns a `Pager` beside the page | more typing at every call, and `[ffi]` appears in no row in this package; that is the difference between a library a reader can check and one they cannot |
| `page_free_deferred` defers a free while a snapshot is pinned | `free_page` defers for the same reason | this one came across unchanged, because it is right: a freed id handed back to `alloc_page` would be overwritten under a reader still holding it |

The salts are the part of the WAL design most worth reading before
adding a body.  A checkpoint changes them rather than truncating the
log, so a frame whose salts do not match the header's is a frame from
before the last checkpoint — which is what makes a checkpoint O(1) at
the end and recovery a forward scan that stops by itself.

## Building it, and checking it

```bash
novo pkg add pager-nv       # add it to a package
novo pkg build              # type-check and effect-check every module
novo test tests/pagefmt_tests.nv
```

**`novo test` is red on every suite, and that is the published state.**
Each test calls a function whose body is `todo()`, so the first
assertion in each file panics:

```
$ novo test tests/pager_tests.nv
  ✗ test_opening_a_file_that_is_not_there_says_which_one
      not implemented: pager-nv.pager.open
  0 passed, 1 failed
```

The tests are the design under review, not a regression net.  Two of
the four files are worth reading for a second reason: every function
in `tests/pager_tests.nv` and `tests/driver_tests.nv` declares the
effect row a real consumer will carry, so the cost of depending on
this package is written out before anybody depends on it.

## Status

| function | implemented |
| --- | --- |
| `pagefmt.db_header_size`, `wal_header_size`, `wal_frame_header_size`, `default_page_size` | no |
| `pagefmt.new_db_header`, `encode_db_header`, `decode_db_header` | no |
| `pagefmt.encode_wal_header`, `decode_wal_header`, `encode_wal_frame`, `decode_wal_frame` | no |
| `pagefmt.checksum_seed`, `checksum`, `frame_is_current`, `frame_offset` | no |
| `pager.open`, `create`, `close` | no |
| `pager.read_page`, `read_page_at`, `write_page`, `alloc_page`, `free_page` | no |
| `pager.snapshot`, `release`, `begin`, `commit`, `rollback`, `checkpoint` | no |
| `pager.header`, `page_size`, `page_count`, `stats`, `set_cache_limit` | no |
| `pager.now_snapshot` | no |
| `driver.execute`, `execute_script`, `next_row`, `step_once`, `answer` | no |
| `driver.load_schema`, `store_schema` | no |
