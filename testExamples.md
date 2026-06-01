# SQLite-Inspired Testing Examples

SQLite public-suite examples. Each: weak approach, misses, stronger reusable pattern.

---

## 1. Turn Every Bug Into a Regression Test

### Example: Empty Identifiers

**Bug Report:** "Verify that zero-length table, column, and index names work correctly."

SQLite keeps permanent regression in `test/tkt-78e04e52ea.test`.

**What Agents Do Wrong (Patch Without a Reproducer)**

```text
Fixed identifier validation to allow empty names.
Ran the existing unit tests.
```

**Problems:**
- No proof original bug existed.
- No protection for unusual identifier post-fix.
- Bug may return during parser, schema, index, query-planner changes.

**What Should Happen (Keep the Reproducer)**

```tcl
do_test tkt-78e04-1.0 {
  execsql {
    CREATE TABLE ""("" UNIQUE, x CHAR(100));
    INSERT INTO ""("") VALUES(1);
    SELECT * FROM "";
  }
} {1 {}}

do_test tkt-78e04-1.1 {
  catchsql {
    INSERT INTO ""("") VALUES(1);
  }
} {1 {UNIQUE constraint failed: .}}
```

Continue same edge case through:
- Schema inspection.
- Index creation.
- Query planning.
- Drop operations.

**Why Better:** Preserves exact strange input; verifies related interfaces.

**Agent Rule:** Bug not fixed until smallest useful reproducer stays in suite.

---

## 2. Sweep I/O Failures Across the Whole Operation

### Example: `VACUUM` Under Disk Errors

**Feature:** "`VACUUM` rewrites persistent data safely."

SQLite tests via `do_ioerr_test` in `test/ioerr.test`.

**What Agents Do Wrong (One Convenient Mock)**

```python
def test_vacuum_reports_write_error(db, disk):
    disk.fail_next_write()
    with pytest.raises(DiskError):
        db.vacuum()
```

**Problems:**
- Only one write fails.
- Read, sync, open, later-write failures untested.
- Database validity unproven.
- Cleanup/retry unknown.

**What Should Happen (Advance the Failure Point)**

```tcl
do_ioerr_test ioerr-2 -cksum true -ckrefcount true -sqlprep {
  CREATE TABLE t1(a, b, c);
  INSERT INTO t1 VALUES(1, randstr(50,50), randstr(50,50));
  CREATE TABLE t2 AS SELECT * FROM t1;
} -sqlbody {
  VACUUM;
}
```

Harness injects I/O error at successive operations until body completes without injected failure.

After each failure verify:
- Error controlled.
- DB checksum/integrity valid.
- No page-reference/file-descriptor leaks.
- Recovery works.

**Why Better:** Searches full failure surface, not one hand-picked point.

**Agent Rule:** Persistent-state operation: fail each external step in turn.

---

## 3. Test Transient and Persistent Faults

### Example: Allocation and I/O Failures

**Feature:** "Operation handles temporary and sustained resource failures."

SQLite defines both in `test/malloc_common.tcl`.

**What Agents Do Wrong (Assume All Failures Behave the Same)**

```python
service.fail_next_call()
assert operation() == RETRYABLE_ERROR
```

**Problems:**
- One-time fault may recover; sustained fault may loop forever.
- Cleanup may allocate/perform I/O after first error.
- Retry behavior untested.

**What Should Happen (Run Both Modes)**

```tcl
set FAULTSIM(oom-transient) [list
  -injectstart {oom_injectstart 0}
  -injectstop oom_injectstop
]

set FAULTSIM(oom-persistent) [list
  -injectstart {oom_injectstart 1000000}
  -injectstop oom_injectstop
]
```

Same distinction for:
- I/O errors.
- Disk-full errors.
- Shared-memory errors.
- Open failures.
- Interruptions.

**Why Better:** Tests recovery paths when environment recovers and stays broken.

**Agent Rule:** Run fault tests transient, then persistent.

---

## 4. Prove Crash Recovery With State Signatures

### Example: Crash During Commit

**Feature:** "Power failure during commit does not corrupt durable state."

SQLite tests in `test/crash.test`.

**What Agents Do Wrong (Restart Without Checking State)**

```python
process.kill_during_commit()
app.restart()
assert app.is_running()
```

**Problems:**
- Startup success != data integrity.
- Partial transaction may remain.
- Old vs new state undistinguished.

**What Should Happen (Capture and Compare State)**

```tcl
proc signature {} {
  return [db eval {
    SELECT count(*), md5sum(a), md5sum(b), md5sum(c) FROM abc
  }]
}

set ::sig [signature]

crashsql -delay 1 -file test.db-journal {
  DELETE FROM abc WHERE a = 1;
}

do_test crash-recovery {
  signature
} $::sig
```

Crash at:
- Journal writes.
- Journal syncs.
- Database writes.
- Database syncs.
- Multi-file transaction steps.

Post-restart verify:
- Transaction fully committed or fully rolled back.
- State signature matches allowed result.
- Integrity checks pass.

**Why Better:** Proves atomicity, not process survival only.

**Agent Rule:** Crash-safe update leaves old/new state, never unexplained middle.

---

## 5. Corrupt Stored Data Deliberately

### Example: Mutate Database Segments

**Feature:** "Malformed persistent data fails safely."

SQLite builds valid DB, overwrites segments with junk, probes behavior in `test/corrupt.test`.

**What Agents Do Wrong (Only Test a Missing File)**

```python
def test_missing_file():
    with pytest.raises(FileNotFoundError):
        load_database("missing.db")
```

**Problems:**
- Missing data simpler than malformed data.
- Structural corruption untested.
- Parser crashes, runaway allocation, resource leaks remain possible.

**What Should Happen (Mutate Valid Data)**

```tcl
for {set i 256} {$i<$fsize-256} {incr i 256} {
  forcecopy test.bu test.db
  set fd [open test.db r+]
  seek $fd $i
  puts -nonewline $fd $junk
  close $fd

  catchsql {SELECT count(*) FROM sqlite_master}
  catchsql {SELECT count(*) FROM t1}
  catchsql {PRAGMA integrity_check}
}
```

Verify:
- Reads fail with controlled errors where appropriate.
- No crash, null dereference, infinite loop.
- Read-only inspection does not worsen corruption.
- Page references and other resources are released.

**Why Better:** Real corruption often starts mostly valid, few bytes damaged.

**Agent Rule:** Treat files, cache entries, stored records as hostile input.

---

## 6. Preserve Fuzz Discoveries as Deterministic Tests

### Example: Parser and Query Regressions

**Feature:** "Generated inputs once exposing bugs never regress."

SQLite keeps fuzz-found SQL in `test/fuzz.test`.

**What Agents Do Wrong (Fix the Crash and Discard the Seed)**

```text
Fuzzer found a parser crash.
Patched the null check.
Restarted the fuzzer.
```

**Problems:**
- Exact failure absent from normal CI.
- Bug may return while fuzzer idle.
- Future developers cannot see key edge case.

**What Should Happen (Retain the Interesting Input)**

```tcl
do_test fuzz-1.9 {
  # This was causing a NULL pointer dereference of Expr.pList.
  execsql {
    SELECT 1 FROM (SELECT * FROM sqlite_master WHERE random())
  }
} {}
```

Another retained SQLite case:

```tcl
do_test fuzz-1.13 {
  execsql {
    SELECT 'abcd' UNION SELECT 'efgh' ORDER BY 1 ASC, 1 ASC;
  }
} {abcd efgh}
```

For each useful fuzz find:
- Minimize input when practical.
- Save seed/corpus entry.
- Add deterministic regression test.
- Keep original-failure comment.
- Rerun retained corpus in normal CI.

**Why Better:** Fuzzing becomes cumulative knowledge, not temporary activity.

**Agent Rule:** Every useful fuzz find outlives fuzz process.

---

## 7. Compare Optimized and Unoptimized Behavior

### Example: Query Optimization

**Feature:** "Optimization changes speed, not answers."

SQLite has `no_optimization` permutation in `test/permutations.test` + comparison tool `test/optfuzz.c`.

**What Agents Do Wrong (Test Only the Fast Path)**

```python
def test_cached_search():
    assert search("sqlite", cache_enabled=True) == ["SQLite"]
```

**Problems:**
- Slow path may already be wrong.
- Cache invalidation bugs hidden.
- Optimization may change meaning, not speed only.

**What Should Happen (Compare Both Paths)**

```python
optimized = search("sqlite", cache_enabled=True)
baseline = search("sqlite", cache_enabled=False)

assert optimized == baseline
```

SQLite's equivalent idea:

```tcl
optimization_control $::dbhandle all 0
```

Run correctness tests with optimizations disabled; compare normal config.

Apply to:
- Query planners.
- Caches.
- Indexes.
- Memoization.
- Compiler passes.
- Batch fast paths.

**Why Better:** Baseline path becomes independent oracle for optimized path.

**Agent Rule:** When possible, verify fast path against simpler path.

---

## 8. Push Limits From Both Sides

### Example: Runtime Limits

**Feature:** "Configured limits enforce without overflow/silent expansion."

SQLite checks runtime + compile-time limits in `test/sqllimits1.test`.

**What Agents Do Wrong (Test One Typical Value)**

```python
def test_username_limit():
    assert validate_username("alice")
```

**Problems:**
- Boundary untested.
- First rejected value untested.
- Clamping/overflow bugs missed.

**What Should Happen (Probe the Edge)**

```python
assert validate_username("a" * 29)
assert validate_username("a" * 30)
assert not validate_username("a" * 31)
```

SQLite also verifies excessive requested limits clamp:

```tcl
do_test sqllimits1-4.1.1 {
  sqlite3_limit db SQLITE_LIMIT_LENGTH 0x7fffffff
  sqlite3_limit db SQLITE_LIMIT_LENGTH -1
} $SQLITE_MAX_LENGTH
```

Test:
- Minimum allowed.
- Just above minimum.
- Just below maximum.
- Maximum allowed.
- First invalid.
- Huge values possibly overflowing/bypassing checks.

**Why Better:** Most limit bugs live at edge, not middle.

**Agent Rule:** Test the last valid value and the first invalid value.

---

## 9. Verify Cleanup After Failure

### Example: No Leaked File Descriptors

**Feature:** "Out-of-memory failures do not leak resources."

SQLite repeatedly checks cleanup in `test/malloc.test`.

**What Agents Do Wrong (Assert Only the Error)**

```python
with pytest.raises(OutOfMemoryError):
    open_database()
```

**Problems:**
- Operation may leak file descriptor.
- Repeated failures may exhaust resources.
- Error path may leave hidden state.

**What Should Happen (Assert Cleanup Too)**

```tcl
do_test malloc-1.X {
  catch {db close}
  set sqlite_open_file_count
} {0}
```

Track:
- Memory.
- File descriptors.
- Sockets.
- Threads.
- Locks.
- Transactions.
- Temporary files.

**Why Better:** Controlled failure includes cleanup, not error message only.

**Agent Rule:** Every failure-path test asks what must release.

---

## Source Notes

SQLite examples adapted from:
- [`test/tkt-78e04e52ea.test`](https://sqlite.org/src/file?name=test/tkt-78e04e52ea.test&ci=trunk)
- [`test/ioerr.test`](https://sqlite.org/src/file?name=test/ioerr.test&ci=trunk)
- [`test/malloc_common.tcl`](https://sqlite.org/src/file?name=test/malloc_common.tcl&ci=trunk)
- [`test/crash.test`](https://sqlite.org/src/file?name=test/crash.test&ci=trunk)
- [`test/corrupt.test`](https://sqlite.org/src/file?name=test/corrupt.test&ci=trunk)
- [`test/fuzz.test`](https://sqlite.org/src/file?name=test/fuzz.test&ci=trunk)
- [`test/permutations.test`](https://sqlite.org/src/file?name=test/permutations.test&ci=trunk)
- [`test/optfuzz.c`](https://sqlite.org/src/file?name=test/optfuzz.c&ci=trunk)
- [`test/sqllimits1.test`](https://sqlite.org/src/file?name=test/sqllimits1.test&ci=trunk)
- [`test/malloc.test`](https://sqlite.org/src/file?name=test/malloc.test&ci=trunk)
