# cp.M4-001 — TXN-abort mid-copy: destination bytes fully unwound

**Issue:** #12
**Wave:** R50  Milestone: M4
**Upstream:** `design/tooling/r49-r50-plan.md` §5.6 M4 line in the
[paideia-os](https://github.com/paideia-os/paideia-os) repo.

## 1. Purpose

Verify the M2-004 single-TXN atomicity claim: every cp invocation
(single-file, multi-file, recursive, `--over-existing`) runs inside
one outer `KIND_PDXFS_TXN` opened at `Dispatch::dispatch_copy`, and
any per-file failure inside that TXN causes `pdxfs_txn_abort` to
unwind every already-written destination byte before cp returns to
the shell. When cp exits with `EXIT_OP_FAIL (1)`, the destination
subtree state must equal the pre-invocation state — no partial
files, no truncated files, no new inode entries.

The four driver cases (single-file, multi-file, recursive,
over-existing) are also test rows in this matrix: each stresses a
different code path that must respect the outer TXN wrap. The
recursive and over-existing rows are additionally the drivers that
create the "already-written bytes" the abort must unwind — a
single-file abort has no prior-file state to preserve, so the
recursive rows carry the strongest invariant.

## 2. Test matrix

### Row 1 — single-file happy path (baseline; runs today)

- **Setup.** Scratch subtree `/tmp/cp-m4/`; source file
  `/tmp/cp-m4/src/a` = 4096 bytes of `0x41` ("A"). Destination
  directory `/tmp/cp-m4/dst/` exists and is empty. Caps: KIND_USER
  = invoker; KIND_PDXFS_FILE(read, `/tmp/cp-m4/src`);
  KIND_PDXFS_FILE(write, `/tmp/cp-m4/dst`); KIND_PDXFS_TXN;
  KIND_IPC_ENDPOINT.
- **Invocation.** `cp /tmp/cp-m4/src/a /tmp/cp-m4/dst/a`.
- **Assertions.**
    - Exit code == `EXIT_OK` (0).
    - `/tmp/cp-m4/dst/a` exists, size == 4096, byte-identical to
      source (SHA-256 or a straight byte-compare).
    - `CopyFile::bytes_copied` == 4096.
    - `Dispatch::chosen_arm` == `ARM_SINGLE_FILE` (1).
    - `CpPipe::pipe_emits` == 1 (one CopyProgressRecord emitted).
    - `CpAudit::audit_id` != 0 (audit_begin_wrap returned a valid id
      OR bumped `audit_begin_errors` — the M3 log-and-continue path
      lets the wrap fail silently; either observable state is OK
      at M4 while the broker daemon has not booted).
    - `Undo::undo_records_written` == 0 (no --over-existing).
    - `CpElevate::elevate_calls` == 0 (dst-parent open succeeded on
      first try).

### Row 2 — multi-file (via recursive walk over N regular files)

- **Substrate gate.** Blocked on R42 `sys_pdxfs_readdir` +
  `sys_pdxfs_mkdir` + real per-entry parse in
  `Walk::walk_entry_todo`. Row is shape-in-place; harness skips
  until the gate lifts and cp starts returning non-zero
  `Walk::entries_visited`.
- **Setup.** `/tmp/cp-m4/src/multi/` contains three files: `f1`
  (1024 bytes of `0x31`), `f2` (2048 bytes of `0x32`), `f3` (3072
  bytes of `0x33`). Destination `/tmp/cp-m4/dst/multi/` does not
  exist. Caps identical to Row 1 modulo the widened src/dst
  subtree scope.
- **Invocation.** `cp -r /tmp/cp-m4/src/multi /tmp/cp-m4/dst/multi`.
- **Assertions.**
    - Exit code == 0.
    - `/tmp/cp-m4/dst/multi/` exists (directory).
    - `/tmp/cp-m4/dst/multi/{f1,f2,f3}` exist, sizes and contents
      byte-identical to sources.
    - `Walk::entries_visited` == 3.
    - `CopyFile::bytes_copied` last-value == 3072 (per-file counter
      is reset by `copy_reset` at each entry; the M4 harness
      captures the sum via the semantic-pipe stream).
    - `CpPipe::pipe_emits` == 3 (one CopyProgressRecord per file).
    - `Dispatch::chosen_arm` == `ARM_RECURSIVE_WALK` (4).
    - `CpAudit::audit_commit_calls` == 1 (audit wraps the whole
      recursive invocation once, not per file — verifies D3
      audit-first ordering holds under recursion).

### Row 3 — --over-existing with undo record replay

- **Substrate gate.** Blocked on R42 `sys_pdxfs_stat` and PdxFS v1
  undo journal write. Row is shape-in-place; the M2-005
  `Undo::maybe_record_over_existing` stub currently returns
  `UNDO_OK` without journaling — the pass-criteria "replay
  restores original" edge lands with the substrate.
- **Setup.** `/tmp/cp-m4/src/a` = 512 bytes of `0x41`;
  `/tmp/cp-m4/dst/a` pre-exists = 1024 bytes of `0x42`. Caps
  as Row 1.
- **Invocation.** `cp --over-existing /tmp/cp-m4/src/a
  /tmp/cp-m4/dst/a`.
- **Assertions (post-copy, pre-replay).**
    - Exit == 0.
    - `/tmp/cp-m4/dst/a` size == 512, contents == 0x41 * 512.
    - `Undo::undo_records_written` == 1.
    - `Undo::undo_reason` == `UNDO_REASON_OVER_EXISTING`.
- **Replay step.** Harness invokes the PdxFS v1 undo replay
  (interface TBD by R42; the plan §5.6 rubric names "PdxFS v1
  undo record" as the replayable artifact — no cp-side entry
  point).
- **Assertions (post-replay).**
    - `/tmp/cp-m4/dst/a` size == 1024, contents == 0x42 * 1024
      (byte-identical to pre-copy destination).

### Row 4 — TXN-abort mid-copy (the M4-001 core case)

- **Substrate gate.** Blocked on R42 `sys_pdxfs_txn_abort` real
  body (the M2 stub returns 0 without touching the substrate).
  Row is shape-in-place — the abort call path is wired at
  `Dispatch::dispatch_copy` (`dispatch_body_fail_abort` and
  `dispatch_txn_commit_fail` both jump through the abort call)
  and only the semantic content of the abort ("unwind everything
  written under this txn handle") lives in the substrate. When
  the substrate lands, this row's byte-assertion moves from
  "no assertion possible today" to "must equal pre-copy state".
- **Setup.** `/tmp/cp-m4/src/tree/` contains `a` (4096 bytes of
  `0x41`), `b` (4096 bytes of `0x42`), and `c` — a file the src
  cap does not cover (e.g. a symlink to a non-readable target,
  or the cap-scope excludes it explicitly). Destination
  `/tmp/cp-m4/dst/tree/` does not exist. Caps as Row 2 but with
  src cap narrowed so `c` reads fail with `-EACCES` (or the
  substrate-equivalent negative errno).
- **Invocation.** `cp -r /tmp/cp-m4/src/tree /tmp/cp-m4/dst/tree`.
- **Expected sequence (per dispatch flow at src/dispatch.pdx).**
    - `dispatch_copy` opens the outer TXN (`pdxfs_txn_begin`).
    - Walk descends into `/tmp/cp-m4/src/tree`.
    - `a` copies successfully — 4096 bytes appended under txn.
    - `b` copies successfully — 4096 bytes appended under txn.
    - `c` open fails (negative errno from `pdxfs_open`).
    - `CopyFile::copy_bytes_only` returns 1 to
      `Walk::walk_recursive`; walk propagates 1 upward.
    - `Dispatch::dispatch_copy` sees `rbx = 1` on body return and
      jumps to `dispatch_body_fail_abort` which calls
      `pdxfs_txn_abort` (silent; return ignored) then returns 1.
- **Assertions.**
    - Exit == `EXIT_OP_FAIL` (1).
    - `/tmp/cp-m4/dst/tree/` does NOT exist (abort unwound the
      mkdir). If it exists, verify it's empty AND every entry
      has been unwound — an empty dst-tree directory is
      acceptable only if `sys_pdxfs_mkdir` under the aborted
      TXN legally survives (implementation-defined for PdxFS v1;
      the spec assertion here follows what the R42 substrate
      documents).
    - `/tmp/cp-m4/dst/tree/a` does NOT exist.
    - `/tmp/cp-m4/dst/tree/b` does NOT exist.
    - `Walk::entries_visited` == 2 (the two files copied before
      the failure; `c` fails at open so it does not increment
      the counter — the counter bump lives inside the copy body).
    - `CopyFile::bytes_copied` last-value == 0 (the `c` failure
      reset counter before it opened src; the two successful
      copies incremented `CpPipe::pipe_emits` instead).
    - `CpPipe::pipe_emits` == 2 (the two successful pre-abort
      emits are wire-observable — cp cannot un-send a semantic-
      pipe record; the receiver sees the two records but the
      abort is signalled to the receiver via the TOOL_EXIT audit
      record's non-zero exit code, per D3 audit-first).
    - `CpAudit::audit_commit_calls` == 1 with `exit_code == 1`
      arg — verifies audit_commit_wrap fires on the failure
      path (dispatch_epilogue is the sole exit route).

### Row 5 — TXN-commit fail (the sibling of Row 4)

- **Substrate gate.** Blocked on R42 `sys_pdxfs_txn_commit` real
  body with a failure-return path. The M2 stub returns 0
  unconditionally.
- **Setup.** Same as Row 4 minus the failing `c` file. The
  failure is induced at commit time via a substrate hook —
  simulating a quota exhaustion revealed only at commit — TBD by
  the R42 substrate's test-hook surface.
- **Invocation.** `cp /tmp/cp-m4/src/a /tmp/cp-m4/dst/a`.
- **Expected sequence.** copy_bytes_only returns 0; dispatch calls
  `pdxfs_txn_commit`; commit returns negative errno; dispatch
  jumps to `dispatch_txn_commit_fail` which calls
  `pdxfs_txn_abort` then prints `TXN_COMMIT_FAIL_MSG` and returns
  `EXIT_OP_FAIL`.
- **Assertions.**
    - Exit == 1.
    - Destination byte-identical to Row 4 unwind state.
    - stderr contains "cp: transaction commit failed\n".
    - `CpAudit::audit_commit_calls` == 1 with `exit_code == 1`.

## 3. Fixtures

The harness pre-creates a scratch subtree at `/tmp/cp-m4/` for every
run. Each row has its own subdirectory under `/tmp/cp-m4/src/` and
`/tmp/cp-m4/dst/` to avoid cross-row contamination. Fixture-creation
uses the same PdxFS v1 API cp itself uses (`pdxfs_mkdir`,
`pdxfs_open`, `pdxfs_write`) so the fixtures are self-hosting once
the R42 substrate lands.

**Byte-pattern convention.** Every fixture file is filled with a
single-byte repetition (`0x41` = 'A', `0x42` = 'B', etc.) so the
assertion can verify by name-of-fill without carrying a large
expected-content buffer. The chosen fill byte matches the file name
lexicographically: `a` → 0x41, `b` → 0x42, `f1` → 0x31, and so on.

**Cap-narrowing convention.** Cross-subtree caps (Row 4's failing
`c`) are set up by having the loader's InitCap sidecar seed a
narrower read cap than the src subtree covers. cp's request for
`pdxfs_open(c, O_RDONLY)` fails with negative-errno because the
substrate's cap-check rejects the open — no explicit cap-mismatch
diagnostic is emitted (that surface lands as cp.M4-005+ in a
future round; M4-001 sees only the negative-errno return path).

## 4. Observation hooks (cp .bss counters)

Every counter used above is exposed as a `pub let mut` in cp's
module namespace so the harness reads it via loader symbol lookup:

| Symbol                              | File                | Semantic                        |
|-------------------------------------|---------------------|---------------------------------|
| `CopyFile::bytes_copied`            | src/copy.pdx        | last successful copy body's byte count |
| `Dispatch::chosen_arm`              | src/dispatch.pdx    | ARM_SINGLE_FILE / _USAGE / _RECURSIVE_WALK |
| `Walk::entries_visited`             | src/walk.pdx        | recursive-walk per-entry increments |
| `CpPipe::pipe_emits`                | src/pipe.pdx        | successful CopyProgressRecord sends |
| `CpPipe::pipe_emit_errors`          | src/pipe.pdx        | send failures (broker unavailable) |
| `CpAudit::audit_commit_calls`       | src/audit.pdx       | audit_commit_wrap fires (1 per invocation) |
| `Undo::undo_records_written`        | src/undo.pdx        | --over-existing pre-copy captures |
| `CpElevate::elevate_calls`          | src/elevate.pdx     | dst-parent elevate attempts |

The harness reads a counter by resolving the symbol via the same
loader manifest cp itself uses (paideia-os
`src/kernel/core/loader/init_caps.pdx` R20b.M4 sidecar). No cp-side
API is needed — every counter is already `pub`.

## 5. Invariants (cross-row)

I1. **Every exit path fires `CpAudit::cp_audit_commit_wrap` exactly
    once.** No dispatch exit bypasses `dispatch_epilogue`. Verified
    by asserting `CpAudit::audit_commit_calls` increments by exactly
    1 across each invocation.

I2. **Every failed copy body triggers `pdxfs_txn_abort` at the
    outer TXN boundary.** No M4 test row asserts on abort return
    value (dispatch ignores it), but every failure row asserts on
    the resulting destination-side byte state — "no partial file
    survives".

I3. **The single-TXN wrap covers both `-r` and single-file arms
    identically.** Row 4 and Row 5 assert on abort semantics in
    both arms so the harness can distinguish "abort works in
    single-file, not recursive" from "abort works uniformly".

I4. **CopyProgressRecord@0.1 emits before the abort call.** Row 4's
    two pre-abort emits are wire-observable; a receiver sees them
    even though the TXN is later aborted. This is the intended
    semantics — semantic-pipe records are unwrappable telemetry,
    not part of the transactional commit.

I5. **The undo record precedes the destructive write.** Row 3's
    `Undo::undo_records_written` == 1 assertion at pre-replay time
    verifies the journal write happened BEFORE the copy body
    overwrote the destination. This is I5-descendant from the
    D5 undo-first-order contract in paideia-os
    `design/tooling/r49-r50-plan.md` §3.

## 6. Pass criteria

The M4-001 row passes if every assertion in the row's Assertions
list evaluates true. A row is `SKIP` if its Substrate gate is not
yet lifted (the harness still runs the row; the shape-in-place
assertions that don't need the gate remain in the pass rubric). A
row is `FAIL` if any gated assertion evaluates false OR any
counter drifts from the expected value under a gate-lifted run.

The M4-001 issue closes when Rows 1, 2, 3, 4, 5 all pass on a
gate-lifted substrate. The M4-001 issue is _shape-lands_ (this
milestone) when Rows 1's assertions all pass today and Rows 2-5's
harness rows compile against the cp .bss symbol table above.

## 7. Not in scope for M4-001

- **Concurrent cp invocations.** Row-level assertions assume one cp
  at a time — the outer TXN is per-process. A future "two cps
  writing to the same dst-parent" case lands with the PdxFS v1
  contention model (R42-PREP-004).
- **Signature verification of the abort.** Row 4/5 do not assert
  on the cap-tail state of the (non-existent) destination files.
  Signed-inode preservation is M4-003's scope.
- **Quota exhaustion during copy.** Row 5 simulates commit-time
  quota failure but the mid-copy quota-exceed path (write returns
  `-EDQUOT`) is a future M4.5 addition; today it degenerates into
  Row 5's shape via a substrate-side hook.
