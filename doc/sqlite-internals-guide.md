# SQLite Internals Guide

A technical reference for the SQLite register-based virtual machine (VM) architecture.
Each section identifies the primary C functions and explains the relevant logic as found
in the SQLite source tree.

---

## Table of Contents

1. [Compilation – From SQL Text to VDBE Opcodes](#1-compilation)
2. [The VDBE Execution Loop](#2-the-vdbe-execution-loop)
3. [B-Tree Storage Engine](#3-b-tree-storage-engine)
4. [The Pager and VFS Layer](#4-the-pager-and-vfs-layer)
5. [Trace Example – `SELECT name FROM users WHERE id = 5;`](#5-trace-example)

---

## 1. Compilation

**Relevant sources:** `src/tokenize.c`, `src/parse.y`, `src/prepare.c`,
`src/select.c`, `src/insert.c`, `src/update.c`, `src/delete.c`, `src/where.c`,
`src/expr.c`, `src/vdbeaux.c`

### Pipeline Overview

```
SQL text
   │
   ▼
sqlite3RunParser()       ← tokenize.c: feeds tokens to the LALR(1) parser
   │
   ▼
sqlite3Parser()          ← generated from parse.y by the Lemon parser generator
   │  (builds an AST of Select / Expr / SrcList nodes inside Parse*)
   ▼
Code generators          ← select.c / insert.c / update.c / delete.c / where.c
   │  (walk the AST, call sqlite3VdbeAddOp*() to emit opcodes)
   ▼
Vdbe program (aOp[])    ← array of Op structs ready for sqlite3VdbeExec()
```

### Key Functions

| Function | Source file | Role |
|---|---|---|
| `sqlite3Prepare()` | `prepare.c:682` | Entry point for `sqlite3_prepare_v2()`; allocates a `Vdbe`, initializes a `Parse` context, calls `sqlite3RunParser()`. |
| `sqlite3RunParser()` | `tokenize.c:600` | Tokenises the SQL string one character at a time and feeds each token to the Lemon-generated `sqlite3Parser()`. |
| `sqlite3Parser()` | generated `parse.c` | LALR(1) reduce actions; builds `Select`, `Expr`, `SrcList`, etc. nodes and eventually calls the code-generator entry points. |
| `sqlite3Select()` | `select.c` | Walks a `Select` AST and emits `OP_OpenRead`, `OP_Column`, `OP_ResultRow`, etc. |
| `sqlite3WhereBegin()` | `where.c` | Chooses the query plan (table scan, index scan, or index seek) and emits the loop setup opcodes. |
| `sqlite3VdbeAddOp2()` / `sqlite3VdbeAddOp3()` | `vdbeaux.c` | Append one `Op` record to the growing `aOp[]` array inside the `Vdbe` object. |

### How `parse.y` Works

`src/parse.y` is a **Lemon grammar** file.  Running the `tool/lemon` parser
generator against it produces `src/parse.c` (the state-machine tables and reduce
actions) and `src/parse.h` (token constants).

Each grammar rule whose action calls a code-generator function causes code
emission:

```
cmd ::= SELECT selcollist FROM seltablist WHERE expr .
{
    /* Lemon reduce action */
    sqlite3Select(pParse, p, 0);   /* → select.c emits VDBE opcodes */
}
```

Because Lemon is re-entrant and operates entirely through the opaque `Parse*`
pointer (`pParse`), the same parser can be used recursively for sub-queries.

---

## 2. The VDBE Execution Loop

**Relevant sources:** `src/vdbe.c`, `src/vdbeInt.h`, `src/vdbemem.c`

### The `sqlite3VdbeExec()` Loop

```c
/* src/vdbe.c:846 */
int sqlite3VdbeExec(Vdbe *p)
{
    Op    *aOp = p->aOp;   /* immutable opcode array */
    Op    *pOp = aOp;      /* program counter – pointer into aOp[] */
    Mem   *aMem = p->aMem; /* register file */
    Mem   *pIn1, *pIn2, *pIn3, *pOut;
    int    rc   = SQLITE_OK;

    for(;;){
        switch( pOp->opcode ){
            case OP_Goto:      ...
            case OP_Integer:   ...
            case OP_Column:    ...
            /* ~180 cases total */
        }
        pOp++;   /* advance program counter unless a jump was taken */
    }
}
```

The function is a single `for(;;)` loop over a `switch` on `pOp->opcode`.
Each opcode reads its operands from `pOp->p1`/`p2`/`p3`/`p4`/`p5` and from
the `Mem` register array `aMem[]`.  When `OP_ResultRow` fires, the loop
returns `SQLITE_ROW` to the caller; when `OP_Halt` fires it returns
`SQLITE_DONE` (or an error code).

### The `Mem` Structure

```c
/* src/vdbeInt.h */
struct Mem {
    union {
        double r;          /* MEM_Real  */
        i64    i;          /* MEM_Int   */
        int    nZero;      /* MEM_Zero | MEM_Blob – extra zero bytes */
        const char *zPType;/* pointer type tag */
    } u;
    u16   flags;   /* MEM_Null | MEM_Str | MEM_Int | MEM_Real | MEM_Blob … */
    u8    enc;     /* SQLITE_UTF8, SQLITE_UTF16LE, SQLITE_UTF16BE */
    int   n;       /* byte length of z (for MEM_Str / MEM_Blob) */
    char *z;       /* string or blob data */
    char *zMalloc; /* heap-allocated buffer if szMalloc > 0 */
    …
};
```

`flags` is a bitmask that encodes the active type:

| Flag | Meaning |
|---|---|
| `MEM_Null` (0x0001) | SQL NULL |
| `MEM_Str`  (0x0002) | String stored in `Mem.z` |
| `MEM_Int`  (0x0004) | 64-bit integer in `Mem.u.i` |
| `MEM_Real` (0x0008) | IEEE-754 double in `Mem.u.r` |
| `MEM_Blob` (0x0010) | Raw bytes stored in `Mem.z` |

### Five Common Opcodes

#### `OP_Integer` (vdbe.c:1371)
```
Opcode: Integer P1 P2 * * *
Synopsis: r[P2]=P1
```
Writes the small signed integer `P1` directly into register `P2`.
The handler sets `pOut->u.i = pOp->p1` and `pOut->flags = MEM_Int`.

---

#### `OP_String` / `OP_String8` (vdbe.c:1414, 1458)
```
Opcode: String P1 P2 P3 P4 P5
Synopsis: r[P2]=P4
```
Places the string constant pointed to by `pOp->p4.z` into register `P2`.
`OP_String8` is the initial form; at first execution it is re-written to
`OP_String` (UTF-8, pre-measured length) to avoid re-encoding on every call.
The handler sets `pOut->z`, `pOut->n`, and `pOut->flags = MEM_Str|MEM_Static`.

---

#### `OP_Column` (vdbe.c:2975)
```
Opcode: Column P1 P2 P3 P4 P5
Synopsis: r[P3]=PX
```
Decodes column `P2` of the record pointed to by cursor `P1` and stores the
result in register `P3`.  Internally it calls `sqlite3VdbeSerialGet()` to
parse the record's type/data header and set the appropriate `MEM_*` flag in
the output `Mem`.  This is the workhorse for projecting column values out of
table and index rows.

---

#### `OP_MakeRecord` (vdbe.c:3469)
```
Opcode: MakeRecord P1 P2 P3 P4 *
Synopsis: r[P3]=mkrec(r[P1@P2])
```
Serialises the `P2` consecutive registers starting at `P1` into a single
packed record blob and writes it to register `P3`.  The format is SQLite's
own record format: a header of varint type codes followed by the data.
This record can later be inserted into a B-tree or decoded by `OP_Column`.

---

#### `OP_ResultRow` (vdbe.c:1746)
```
Opcode: ResultRow P1 P2 * * *
Synopsis: output=r[P1@P2]
```
Marks registers `P1` through `P1+P2-1` as the current result row.
Sets `p->pResultRow = &aMem[pOp->p1]` and returns `SQLITE_ROW` to the
`sqlite3_step()` caller.  The calling application then uses `sqlite3_column_*()`
APIs to read values directly from those `Mem` cells.

---

### Other Frequently Seen Opcodes

| Opcode | Brief description |
|---|---|
| `OP_Init` | Always instruction 0; initialises the program; may jump to the body. |
| `OP_Goto` | Unconditional jump; sets `pOp` to `&aOp[pOp->p2]`. |
| `OP_Halt` | Ends execution; stores error code/message in the `Vdbe`. |
| `OP_OpenRead` | Opens a B-tree cursor on table/index `P2` for reading. |
| `OP_Next` | Advances a cursor to the next row; jumps to `P2` if a row exists. |
| `OP_SeekRowid` | Seeks a table cursor to the row with integer key `r[P3]`. |
| `OP_Transaction` | Begins a read or write transaction on database `P1`. |

---

## 3. B-Tree Storage Engine

**Relevant sources:** `src/btree.c`, `src/btreeInt.h`

### Internal Page Structure

Every SQLite database file is divided into fixed-size pages (512–65 536 bytes,
always a power of two). Each page is either:

- **B-tree table page** – stores rows indexed by integer rowid.
- **B-tree index page** – stores records indexed by arbitrary key columns.
- **Overflow page** – holds payload that overflows a single page.
- **Freelist page** – tracks unallocated pages.

The in-memory representation of a page is `struct MemPage` (`btreeInt.h:273`):

```c
struct MemPage {
    Pgno  pgno;        /* page number */
    u16   nCell;       /* number of cells (key+data pairs) on this page */
    u16   cellOffset;  /* offset in aData[] to the first cell pointer */
    u8   *aData;       /* pointer to raw 4096-byte (or chosen page-size) buffer */
    u8   *aDataEnd;    /* one byte past the last byte of aData */
    DbPage *pDbPage;   /* handle to the underlying pager page */
    …
};
```

An interior B-tree page holds `N` keys and `N+1` child page pointers, laid out
as described at the top of `src/btreeInt.h`:

```
| Ptr(0) | Key(0) | Ptr(1) | Key(1) | … | Key(N-1) | Ptr(N) |
```

Leaf pages hold only cells (key + data payload) with no child pointers.
Finding a key therefore takes O(log M) page reads, where M is the number of
entries.

### `btreeCursor()` and Key Seeks

#### Opening a cursor

```c
/* src/btree.c:4685 */
static int btreeCursor(
    Btree     *p,          /* B-tree handle */
    Pgno       iTable,     /* root page number of the table/index */
    int        wrFlag,     /* 1 = writable cursor */
    struct KeyInfo *pKeyInfo,
    BtCursor  *pCur        /* caller-supplied BtCursor struct to fill */
)
```

`btreeCursor()` validates locking and transaction state, records `iTable` as
`pCur->pgnoRoot`, and sets `pCur->eState = CURSOR_INVALID` (no position yet).
The public wrapper `sqlite3BtreeCursor()` (`btree.c:4765`) acquires the shared
B-tree mutex before calling `btreeCursor()`.

The `BtCursor` struct (`btreeInt.h:531`) tracks the cursor's position as a
stack of `(MemPage*, index)` pairs:

```c
struct BtCursor {
    u8        eState;      /* CURSOR_INVALID / CURSOR_VALID / CURSOR_FAULT … */
    Pgno      pgnoRoot;    /* root page of this B-tree */
    i8        iPage;       /* depth in apPage[] stack */
    u16       ix;          /* cell index on the current page */
    MemPage  *pPage;       /* current page */
    MemPage  *apPage[…];   /* parent pages (path from root to current page) */
    CellInfo  info;        /* parsed cell at current position */
    i64       nKey;        /* integer key (for intkey tables) */
    …
};
```

#### Seeking to a specific key

```c
/* src/btree.c:860 */
static int btreeMoveto(
    BtCursor *pCur,
    const void *pKey, i64 nKey,
    int bias,
    int *pRes     /* out: 0=exact match, <0=pCur<key, >0=pCur>key */
)
```

1. `moveToRoot()` (`btree.c:5542`) reads the root page via the pager and
   positions the cursor at the root.
2. `btreeMoveto()` descends the tree: at each interior page it performs a
   binary search over the `nCell` cell pointers to find the child subtree that
   may contain the target key, then calls `moveToChild()` to load that page.
3. On a leaf page, a binary search sets `pCur->ix` to the matching cell (or
   the insertion point if no exact match).
4. The public `sqlite3BtreeMovetoUnpacked()` variant accepts an `UnpackedRecord`
   (column values already decoded) for index-key comparisons.

At the VDBE level, `OP_SeekRowid` and the `OP_SeekXX` family call through to
these functions to position a cursor before `OP_Column` reads payload or
`OP_Next` / `OP_Prev` iterates rows.

---

## 4. The Pager and VFS Layer

**Relevant sources:** `src/pager.c`, `src/wal.c`, `src/os.c`,
`src/os_unix.c:3643`, `src/os_win.c:2849`

### Architecture

```
SQL layer (vdbe.c)
     │ sqlite3PagerWrite() / sqlite3PagerGet()
     ▼
  Pager  (pager.c)          ← page cache, journalling, transaction state machine
     │ WAL reads/writes
     ▼
  WAL   (wal.c)             ← write-ahead log (when journal_mode=WAL)
     │ sqlite3OsWrite() / sqlite3OsSync()
     ▼
  VFS   (os.c / os_unix.c / os_win.c)   ← OS abstraction layer
```

### Tracing a `COMMIT` from SQL to the Kernel

#### Step 1 — VDBE emits `OP_Transaction` (write mode)

The code generator emits `OP_Transaction P1=0 P2=1` (write transaction on
database 0).  In `sqlite3VdbeExec()` this calls `sqlite3BtreeBeginTrans()` which
calls `sqlite3PagerBegin()` to transition the pager to `WRITER_LOCKED` state.

#### Step 2 — Dirty pages are written during DML

Each time a DML opcode modifies a B-tree page, `sqlite3PagerWrite(PgHdr*)` is
called (`pager.c:6234`).  This:
- Copies the original page image to the rollback journal (`pager-journal` file)
  **before** the page is modified (so a crash can roll back).
- Marks the page dirty in the page cache.
- Transitions the pager to `WRITER_DBMOD` state.

#### Step 3 — `OP_Halt` triggers commit

When execution reaches `OP_Halt` with no error, `sqlite3VdbeHalt()` is called.
For an implicit auto-commit or an explicit `COMMIT` statement, it calls
`sqlite3BtreeCommitPhaseOne()` → `sqlite3PagerCommitPhaseOne()`.

#### Step 4 — `sqlite3PagerCommitPhaseOne()` (`pager.c:6465`)

```c
int sqlite3PagerCommitPhaseOne(
    Pager      *pPager,
    const char *zSuper,   /* super-journal name, or NULL */
    int         noSync
)
```

1. Flushes the page cache (dirty pages) to the database file via
   `sqlite3OsWrite()`.
2. Calls `sqlite3OsSync()` on the journal file (fdatasync / FlushFileBuffers)
   to make the journal durable before touching the database file.
3. Updates the database file header page.
4. Transitions the pager to `WRITER_FINISHED`.

#### Step 5 — `sqlite3PagerCommitPhaseTwo()` (`pager.c:6702`)

Deletes or zeroes the rollback journal file (making the transaction permanent),
then transitions the pager back to `READER` state.

#### Step 6 — VFS `xWrite` called

All actual I/O goes through the `sqlite3_vfs` / `sqlite3_file` vtable.
`sqlite3OsWrite()` calls `pFile->pMethods->xWrite()`:

- **POSIX** (`src/os_unix.c:3643`) — `unixWrite()`:
  ```c
  static int unixWrite(sqlite3_file *id, const void *pBuf,
                       int amt, sqlite3_int64 offset)
  ```
  Uses `pwrite()` (or `write()` after `lseek()` as a fallback) to write
  `amt` bytes from `pBuf` at `offset` in the database file.

- **Windows** (`src/os_win.c:2849`) — `winWrite()`:
  ```c
  static int winWrite(sqlite3_file *id, const void *pBuf,
                      int amt, sqlite3_int64 offset)
  ```
  Calls `WriteFile()` after setting the file pointer via
  `SetFilePointerEx()`.  Retries on `ERROR_ACCESS_DENIED` with an
  exponential back-off.

### WAL Mode Difference

When `journal_mode=WAL`, modified pages are appended to the WAL file instead of
a rollback journal.  A `COMMIT` still eventually calls `unixWrite`/`winWrite`
via `walWriteLock()` → `sqlite3OsWrite()`, but the write target is `<db>-wal`
rather than the database file itself.  A subsequent `CHECKPOINT` operation
copies WAL frames back to the database file.

---

## 5. Trace Example

### Query: `SELECT name FROM users WHERE id = 5;`

Assuming `users` is a plain rowid table with at least two columns (`id`, `name`)
and an integer primary key, the VDBE program generated by SQLite looks roughly
like this (output from `EXPLAIN SELECT name FROM users WHERE id = 5;`):

```
addr  opcode         p1    p2    p3    p4             p5  comment
----  -------------  ----  ----  ----  -------------  --  -------
0     Init           0     8     0                    0   Start
1     Integer        5     1     0                    0   r[1]=5
2     OpenRead       0     2     0     2              0   root=2 iDb=0; users
3     SeekRowid      0     7     1                    0   intkey=r[1]
4     Column         0     1     2                    0   r[2]=users.name
5     ResultRow      2     1     0                    0   output=r[2]
6     Goto           0     3     0                    0   next row
7     Halt           0     0     0                    0   End
8     Transaction    0     0     1     0              1   usesStmtJournal=0
9     TableLock      0     2     0     users          0   iDb=0 root=2 write=0
10    Goto           0     1     0                    0   (back to Init body)
```

### Step-by-step walkthrough

| Addr | Opcode | What happens |
|------|--------|--------------|
| 0 | `OP_Init` | Initialises the VM; may invoke progress callbacks; jumps to addr 8 to set up transaction, then falls to addr 1. |
| 8 | `OP_Transaction` | Opens a read transaction on database 0. |
| 9 | `OP_TableLock` | Acquires a shared lock on the `users` table (root page 2). |
| 1 | `OP_Integer` | Loads the constant `5` into register `r[1]`.  `r[1].flags = MEM_Int`, `r[1].u.i = 5`. |
| 2 | `OP_OpenRead` | Opens B-tree cursor 0 on root page 2 (the `users` table) for reading. |
| 3 | `OP_SeekRowid` | Calls `sqlite3BtreeMovetoUnpacked()` to seek cursor 0 to the row with integer key `r[1]=5`.  If no such row exists, jumps to addr 7 (`OP_Halt`). |
| 4 | `OP_Column` | Reads column index 1 (`name`) from the record at cursor 0; stores the result in `r[2]`.  Internally calls `sqlite3VdbeSerialGet()` to decode the stored record. |
| 5 | `OP_ResultRow` | Marks `r[2]` as the output row; returns `SQLITE_ROW` to `sqlite3_step()`.  The application reads `r[2]` via `sqlite3_column_text()`. |
| 6 | `OP_Goto` | Jumps back to addr 3 to seek the next row (in practice `OP_SeekRowid` on a unique key will find at most one row, so the second seek falls through to `OP_Halt`). |
| 7 | `OP_Halt` | Closes cursors, ends the statement, returns `SQLITE_DONE`. |

### Data flow through `Mem` registers

```
r[1]  MEM_Int  i=5          ← set by OP_Integer at addr 1
r[2]  MEM_Str  z="Alice"    ← set by OP_Column at addr 4
             │
             └─► returned to caller by OP_ResultRow at addr 5
```

---

*Guide generated from SQLite source revision as found in this repository.
Primary source files consulted: `src/vdbe.c`, `src/vdbeInt.h`, `src/btree.c`,
`src/btreeInt.h`, `src/pager.c`, `src/os_unix.c`, `src/os_win.c`,
`src/tokenize.c`, `src/prepare.c`, `src/parse.y`.*
