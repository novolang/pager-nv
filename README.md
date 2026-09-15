# pager-nv

A **pager** is the layer of a database that owns the file. It divides
the file into fixed-size pages, hands one page at a time to the layers
above it, and makes a set of writes either all appear or none of them.
A **write-ahead log** is how it does the last part: a change is
appended to a second file first, and copied back into the database
file later. SQLite's [write-ahead log](https://www.sqlite.org/wal.html)
is the design this package follows. It answers the page requests that
[btree-nv](https://novo-lang.org/packages/btree-nv) and
[sql-engine-nv](https://novo-lang.org/packages/sql-engine-nv) make, and
it is the only one of the three that opens a file.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

A database here is two files: the **database file**, which holds the
pages, and the **write-ahead log**, the WAL, which holds pages that
have been changed and not yet copied back. A **page** is a fixed-size
block of bytes, 4096 by default.

A write does not touch the database file. It appends a **frame** to
the WAL, and a frame is a 24-byte header and one page of content. A
**commit** is the last frame of a transaction, and it is marked by
that frame's header carrying the database's new size in pages. There
is no separate commit record. A **checkpoint** copies every committed
frame back into the database file and resets the log.

A read consults, in order, the frames of the write transaction that is
open, then the WAL's index of committed frames, then the database
file. That order is the isolation. A **snapshot** is a frame count: a
reader holding one consults the index only up to it, so it keeps
seeing the database as it was while writes carry on.

Every frame carries the two **salts** from the WAL's header, and a
**checkpoint changes the salts** rather than truncating the log. A
frame whose salts do not match the header's is left over from before
the last checkpoint. That is what makes a checkpoint cost nothing at
the end and makes recovery a forward scan that stops by itself.

Every frame also carries a **cumulative checksum**: each frame's
checksum folds in the frames before it. A frame lifted out of another
log and dropped into this one is therefore detected, where a
per-frame checksum would accept it.

**Recovery** happens on `open`. Frames are replayed into the index
while they pass their checksums and carry the header's salts, and the
first frame that fails either test ends the scan. A torn final frame
is a frame that was never committed, and dropping it is the whole of
crash recovery.

The file formats are this package's own, and they are given here in
full.

| Structure | Size |
| --- | --- |
| Database header, at the front of page 1 | 48 bytes |
| WAL header, at the front of the log | 32 bytes |
| WAL frame header, followed by one page | 24 bytes |
| Default page size | 4096 bytes |
| Page sizes accepted | a power of two from 512 to 65536 |

**The database header.**

| Bytes | Contents |
| --- | --- |
| 0 to 3 | The magic, `NVPG` |
| 4 to 7 | Format version |
| 8 to 11 | Page size in bytes |
| 12 to 15 | Page count |
| 16 to 19 | The first page of the free list, 0 for none |
| 20 to 23 | Free-list page count |
| 24 to 27 | Change counter, incremented at every commit |
| 28 to 31 | Schema cookie, incremented at every DDL statement |
| 32 to 35 | The catalog's root page |
| 36 to 47 | Reserved, zero |

**The WAL header.**

| Bytes | Contents |
| --- | --- |
| 0 to 3 | The magic, `NVWL` |
| 4 to 7 | Format version |
| 8 to 11 | Page size, which must equal the database's |
| 12 to 15 | Checkpoint sequence, incremented at every checkpoint |
| 16 to 19 | Salt 1, changed at every checkpoint |
| 20 to 23 | Salt 2, random at every checkpoint |
| 24 to 27 | Checksum 1 over bytes 0 to 23 |
| 28 to 31 | Checksum 2 over bytes 0 to 23 |

**A WAL frame header.**

| Bytes | Contents |
| --- | --- |
| 0 to 3 | The page id this frame holds |
| 4 to 7 | The database size in pages after this frame, or 0 when the frame is not a commit |
| 8 to 11 | Salt 1, copied from the WAL header |
| 12 to 15 | Salt 2, copied from the WAL header |
| 16 to 19 | Checksum 1, over the running total and this frame |
| 20 to 23 | Checksum 2 |

Every number in both formats is big-endian.

Only the functions that reach a file declare `[fs]`, and exactly one
function declares `[time]`. `pagefmt` declares no effect at all: it is
arithmetic over byte lists, so the whole format can be tested with no
disk in the test.

## Install

```
novo pkg add pager-nv
```

## Example

```novo
use driver
use pager

fn main() [io, fs]
    // Open the database and replay its write-ahead log. A torn final
    // frame is one that was never committed, and it is dropped here.
    match pager.open("app.db")
        Err(e) => println(e.message())
        Ok(p)  =>
            // Run one statement to the end. This call answers every
            // page the engine asks for, which is where the [fs] is.
            match driver.execute(p, "SELECT name FROM users", [])
                Err(e)  => println(e.message())
                Ok(out) => println("${out.rows_affected} rows")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: pager-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `pagefmt` | The two headers and the frame header, their encoders and decoders, the rolling checksum, and the salt comparison recovery stops on. It performs nothing. |
| `pager` | The file and the log: open, create and close, one page in and one page out, allocate and free, snapshot and release, begin, commit, rollback and checkpoint. |
| `driver` | The loop that answers a SQL statement's page requests to the end, and the three smaller pieces it is built from. |

## How to choose an entry point

**`driver.execute` runs one statement to the end.** It steps the SQL
engine, reads whatever page the engine asks for, feeds it back, and
steps again, until there are no more rows. It is the whole of this
package from an application's side.

**`driver.next_row` stops at each row.** Use it for a query whose
answer is larger than memory, or one the caller may abandon.

**`driver.step_once` is the same body without the loop**, for a caller
that has to interleave two statements, batch its reads, or put a
statement away and come back to it.

**`driver.answer` performs one page request.** It takes a
`btree.PageRequest` and does the read, the write, the allocation or
the free. A caller driving the B-tree itself, with no SQL in the
program, needs only this and `pager`.

**`pager` on its own is a page store with transactions.** A program
with its own structure over pages uses `read_page`, `write_page`,
`alloc_page`, `begin` and `commit`, and never names the SQL layer.

## The rules a user needs

1. **Every function that can touch a file says `[fs]` in its
   signature.** A signature here without it cannot reach a disk, and
   `pagefmt` has none at all.
2. **The pager is a value and it is threaded.** `read_page` answers
   the page and a `Pager` beside it, because the cache and the
   counters moved. Two `Pager` values over one file are two
   independent readers.
3. **A write needs a transaction.** `write_page` outside one is
   refused with `NoTransaction`, and a second `begin` on the same
   pager is refused the same way. Silently joining the outer
   transaction is how a rollback loses somebody else's work.
4. **A commit is one write.** `commit` writes the last frame with the
   database's new size in its header and then fsyncs. That non-zero
   size is the commit record.
5. **A rollback costs nothing on disk.** The frames stay in the log
   and the index that pointed at them is dropped. The next commit's
   checksum chain simply does not include them.
6. **A checkpoint waits for every snapshot to be released.** It
   rewrites pages a reader may still be reading out of the database
   file. `SnapshotHeld` is the refusal, carrying how many are
   outstanding.
7. **`close` does not checkpoint.** A log left behind is read on the
   next open. Checkpointing at close would turn a crash during close
   into a half-checkpointed file for no benefit.
8. **A free is deferred while any snapshot is outstanding.** A freed
   page id handed straight back to `alloc_page` would be overwritten
   under a reader still holding it. `release` is what lets the
   deferred frees complete.
9. **Read the change counter before trusting a cached page.** It is
   incremented at every commit, and comparing it is the cheapest
   staleness check there is.
10. **Re-prepare a statement when the schema cookie moves.** It is
    incremented at every DDL statement, and a plan that outlives its
    schema is how a query silently reads the wrong column.
11. **The WAL's page size must equal the database's.** Two files whose
    headers disagree are not a pair, and `PageSizeMismatch` says so
    rather than reading one as the other.
12. **The checksum is cumulative and cannot be computed frame by
    frame.** `pagefmt.checksum` folds a run of bytes into a running
    pair, and `pagefmt.checksum_seed` starts it from the header's
    salts. It is SQLite's rolling pair: `s1 += x + s2` then
    `s2 += x + s1` over 32-bit big-endian words.
13. **A frame whose salts do not match the header's is old.**
    `pagefmt.frame_is_current` is that comparison, and recovery stops
    at the first frame that fails it.
14. **`pager.now_snapshot` is the only function here that reads a
    clock.** It answers the three formatted strings a SQL engine needs
    to substitute for `DATE('now')` and its family.
    `sql-engine-nv.substitute_now` takes them as an argument, because
    a package that declares no effects may not read a clock.

## What is not included

- **A `PageIo` implementation.** btree-nv publishes
  `trait PageIo[e]` and `pageio.run`, and this package would supply
  `impl PageIo[fs] for Pager`, so that a walk in a package with no
  effects is charged `[fs]` because this implementation costs `[fs]`.
  It is not in this release. The driver below is written against the
  page-request enum instead, which is btree-nv's other published
  shape.
- **Concurrency between processes.** There is no file lock here. Two
  processes writing one database is not something this release
  defends against.
- **A background checkpointer.** `checkpoint` is a call the program
  makes. Deciding when is the program's.
- **Compression and encryption of pages.** A page is written as it was
  handed over.
- **A device build.** Every function that matters takes or returns a
  `[Int]` of page bytes, and the package's whole purpose is a file.

## Related packages

- [btree-nv](https://novo-lang.org/packages/btree-nv) is the ordered
  map over these pages. It never reads one: it answers
  `btree.PageRequest`, and `driver.answer` is what performs those
  requests.
- [sql-engine-nv](https://novo-lang.org/packages/sql-engine-nv) is the
  SQL lexer, parser, planner and executor. It relays the page requests
  its cursors make, so `driver.execute` drives one loop rather than
  two.
- [sqlite-nv](https://novo-lang.org/packages/sqlite-nv) reads and
  writes a real SQLite database file, in SQLite's own format. Take
  that package to open a file another program wrote. Take this one for
  a database in a format of your own.
- [lsm-nv](https://novo-lang.org/packages/lsm-nv) is the other storage
  shape, and its log makes the same three decisions as the one here: a
  generation rather than a truncation, a cumulative checksum, and the
  commit marker inside the record. It logs entries where this logs
  pages.
- `std.fs` in the standard library is where the bytes come from and
  where they go.

## Tests

```bash
novo test tests/pagefmt_tests.nv   # 14 tests: the two headers and the frame, by offset
novo test tests/pager_tests.nv     # 19 tests: the file, the log, and recovery
novo test tests/driver_tests.nv    # 12 tests: the page-request loop
```

The reference implementation is SQLite's pager and its write-ahead
log. The rolling checksum is SQLite's own, and the salt rule and the
commit-by-frame-size rule are its design.

`pagefmt_tests.nv` asserts the whole format with no disk in it, which
is what keeping `pagefmt` free of effects buys. Every function in
`pager_tests.nv` and `driver_tests.nv` declares the effect row a real
consumer will carry, so the cost of depending on this package is
written out before anything depends on it. The suite checks that a
torn final frame ends recovery and is dropped, that a frame from
before the last checkpoint is not replayed, that a checkpoint is
refused while a snapshot is held, that a free is deferred under one,
that a nested `begin` is refused, and that a WAL whose page size
differs from the database's is not read.

The tests compile today and fail at run, each on the
`not implemented: pager-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at
a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `pagefmt.db_header_size`, `.wal_header_size`, `.wal_frame_header_size`, `.default_page_size` | no |
| `pagefmt.new_db_header`, `.encode_db_header`, `.decode_db_header` | no |
| `pagefmt.encode_wal_header`, `.decode_wal_header`, `.encode_wal_frame`, `.decode_wal_frame` | no |
| `pagefmt.checksum_seed`, `.checksum`, `.frame_is_current`, `.frame_offset` | no |
| `pagefmt.FormatError.message` | no |
| `pager.open`, `.create`, `.close` | no |
| `pager.read_page`, `.read_page_at`, `.write_page`, `.alloc_page`, `.free_page` | no |
| `pager.snapshot`, `.release`, `.begin`, `.commit`, `.rollback`, `.checkpoint` | no |
| `pager.header`, `.page_size`, `.page_count`, `.stats`, `.set_cache_limit` | no |
| `pager.now_snapshot`, `PagerError.message` | no |
| `driver.execute`, `.execute_script`, `.next_row`, `.step_once`, `.answer` | no |
| `driver.load_schema`, `.store_schema`, `DriveError.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
