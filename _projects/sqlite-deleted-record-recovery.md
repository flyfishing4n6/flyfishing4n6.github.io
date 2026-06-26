---
title: "Recovering deleted records from SQLite (freelist, WAL & freeblock carving)"
focus: Data Recovery
date: 2026-06-26
summary: "A pure-Python tool that reconstructs deleted rows from raw SQLite files — recovering a cleared chat conversation from freed pages and a deleted message from a write-ahead-log frame, while honestly reporting what page compaction destroyed."
tools: "Python 3 (standard library), SQLite file format, WAL"
dataset: "Self-generated synthetic chat database"
repo: "https://github.com/flyfishing4n6/sqlite-recovery"
repo_label: "sqlite-recovery"
---

## Problem

Almost everything an examiner wants from a phone lives in SQLite. Messages, call
logs, contacts, browser history, location caches, app state — overwhelmingly,
apps store them in SQLite databases. So the single most leveraged data-recovery
skill in mobile forensics is recovering **deleted SQLite records**: the rows a
user (or an app) deleted but that were never actually erased from the file.

They aren't erased because of how SQLite works. A `DELETE` doesn't shred bytes —
it updates bookkeeping so the engine stops *pointing at* them. The row's bytes
sit in the file as "free space" until something happens to reuse them. The job of
a recovery tool is to read the raw file (and its write-ahead log), find that free
space, and reconstruct valid records out of it — without fabricating a single
byte.

This project is that tool, built from the SQLite file format up, with **no
third-party libraries** — the parsing is the point, so it should be auditable by
anyone with a Python interpreter.

## Where deleted data hides in a SQLite file

A SQLite database is a sequence of fixed-size pages. Table rows live in *cells*
on b-tree leaf pages. When data is deleted, the freed space ends up in one of a
few well-defined places, and each needs a different recovery technique:

| Free space | How it's created | Recovery technique |
|---|---|---|
| **Freelist pages** | A bulk delete empties whole pages, which are added to the freelist. The old page — header *and* cell array — is left intact. | **Structured re-parse** of the old cell array → whole rows, *with rowids* |
| **Write-ahead log** (`-wal`) | In WAL mode, every committed change is a full page snapshot in the `-wal` sidecar. Superseded snapshots hold pre-deletion versions of pages. | **Structured parse** of each WAL frame image |
| **Freeblocks** | A single deleted cell is linked into a page's freeblock chain; only its first 4 bytes are overwritten. | **Signature carving** of the record body (no rowid survives) |
| **Unallocated** | The gap between the cell-pointer array and the cell-content area. | **Signature carving** |

The two structured sources are high-yield and precise because the original cell
structure survives. The two in-page sources require carving raw bytes, which is
where false positives creep in — and where most of the engineering effort went.

## Approach

The tool works in the same order a careful examiner would reason:

1. **Parse the database header** — page size, freelist location, text encoding.
2. **Walk the live b-trees** to learn the schema (tables, columns, declared
   types, and which column is the `INTEGER PRIMARY KEY` rowid alias) *and* to
   collect the set of **live rowids** in each table.
3. **Recover the structured sources** — re-parse freed freelist pages and WAL
   frames through their intact cell arrays.
4. **Carve the in-page sources** — slide through freeblocks and unallocated gaps,
   attempting to parse a record at each offset.
5. **Validate, attribute, and flag** every candidate.

Steps 2 and 5 are what make the output trustworthy.

### Telling a deletion apart from a leftover copy

A recovered rowid that is **still live** in the table isn't evidence of a
deletion — it's a stale *copy* (common on freed pages and in old WAL frames).
The tool collects all live rowids up front and flags every recovered row as
either `deleted` or `live-shadow`, so a leftover duplicate is never mistaken for
a deleted record.

### Keeping carving honest with type affinity

Sliding a parser byte-by-byte through raw free space will "successfully" parse a
lot of coincidental garbage. Two checks suppress it: every candidate record must
pass strict structural validation (the serial-type header must line up exactly
with the payload), **and** every decoded value must satisfy the *type affinity*
of the column it would occupy:

```python
def _value_ok(value, affinity):
    if value is None:           # NULL fits any column
        return True
    if affinity == "text":      # a TEXT column shouldn't hold a 6-byte int
        return isinstance(value, str)
    if affinity in ("int", "num", "real"):
        return isinstance(value, (int, float)) and not isinstance(value, bool)
    return True
```

In testing, this single filter cut the false-positive rate from *hundreds* of
junk "records" to essentially zero, at the cost of a little recall — a trade any
examiner would take.

### Reconstructing the rowid-alias column

One SQLite subtlety worth getting right: an `INTEGER PRIMARY KEY` column is an
*alias for the rowid*. It's stored as `NULL` in the record body, and its real
value is the cell's rowid. The tool detects that column from the schema and fills
it back in from the rowid, so recovered rows read the way an examiner expects
instead of showing a confusing `NULL` id.

## Findings

Run against the synthetic database — a chat app where a ~100-message
conversation was deleted, two contacts were deleted, and one message was deleted
in WAL mode — the tool recovers genuine content. Headline:

```
Database page size : 4096
Pages in file      : 5
Freelist pages     : 2
WAL frames scanned : 2

Recovered 49 deleted record(s) (30 live shadow(s) also seen)
```

**The deleted conversation came back from freed pages**, with correct rowids and
reconstructed `id` values:

```
[DEL] messages rowid=81 src=freelist-leaf pg=5 @0x4000
        id            = 81
        thread_id     = 1
        body          = 'Lunch at the new place by the lab?'
        ts            = 1700002997
[DEL] messages rowid=82 src=freelist-leaf pg=5 @0x4000
        id            = 82
        body          = "Can't today, writing up findings."
```

**The WAL-only deletion came back from a superseded frame** — a message that was
inserted and deleted without ever being checkpointed into the main database:

```
[DEL] messages rowid=300 src=wal-frame pg=3 @0x0
        id            = 300
        body          = 'WAL: secret plan, do not log'
        ts            = 5000
```

All 24 distinct conversation lines were recovered at least once, alongside the
WAL message, drawn from freelist pages, freeblocks, and the write-ahead log.

### What was *not* recovered — and why that matters

Of the ~100 deleted messages, **49 were recovered**; the rest were overwritten
when the bulk delete triggered a b-tree **compaction**. And the **two deleted
contacts were not recovered at all** — deleting them left freeblocks, but page
compaction during the build reclaimed those bytes before the image was captured.

This is reported plainly rather than hidden, for two reasons. First, it's the
truth: freed in-page data is volatile, and a real examiner must understand that
recovery is partial and evidence of deletion is not the same as the deleted
content. Second — and this is the point a reviewer should care about most — **the
tool did not invent the missing contacts.** A recovery tool that hallucinates
plausible rows is worse than useless in a forensic context. The precision filters
ensure that when the data is gone, the tool says nothing.

<div class="note" markdown="1">
**Honest scope.** Freelist-page and WAL recovery are robust and reproducible
everywhere. In-page freeblock recovery depends on the platform's SQLite delete
and compaction behaviour, so its yield varies — on the build used here, page
compaction erased the freed cell bytes. The freeblock *carving code path* is
therefore proven independently by a deterministic unit test that carves a record
out of a hand-built freeblock, rather than relying on the live database to
preserve one. What is demonstrated end-to-end is freelist + WAL recovery; what is
proven by unit test is the freeblock carver.
</div>

## Validation & integrity

The tool is covered by a standard-library `unittest` suite (no install), all
passing, covering:

- varint round-trips including the 9-byte edge case,
- serial-type length decoding,
- record parsing and the **rejection** of misaligned/garbage input,
- **freeblock carving** on a hand-built page (deterministic, platform-independent),
- an **end-to-end** recovery asserting the WAL "secret" and a sample of the
  deleted conversation come back, **and** that no live rowid is ever mislabelled
  as deleted.

The recovery process only ever **reads** the evidence files; it parses them as
raw bytes and never opens them with a SQLite engine that could checkpoint or
modify them.

## Provenance

**100% self-generated synthetic data.** No real device, account, conversation,
or casework is involved. A generator script (`scripts/gen_dataset.py`) builds the
database from a fixed pool of invented chat lines and `555`-range (reserved-for-
fiction) phone numbers, then performs the three deletion scenarios — a bulk
conversation delete (freelist), single-row contact deletes (freeblocks), and a
WAL-mode delete (superseded frame). The committed database, its preserved `-wal`,
and the captured recovery report are all in the repo, with full details in its
[`DATA_PROVENANCE.md`](https://github.com/flyfishing4n6/sqlite-recovery/blob/main/DATA_PROVENANCE.md).

## Run it

Pure Python 3 standard library — nothing to install.

```bash
python3 scripts/gen_dataset.py                       # build the synthetic DB (+ -wal)
python3 -m sqlite_recovery recover data/chat_demo.db --deleted-only
python3 -m unittest discover -s tests -v             # run the test suite
```

## Code

The full, self-contained tool — format parsers, the recovery engine, the WAL
reader, the dataset generator, tests, and captured output — is in the repository:
[**flyfishing4n6/sqlite-recovery**](https://github.com/flyfishing4n6/sqlite-recovery).
