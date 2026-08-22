# tests/ — cp M4 test + smoke matrix

**Wave:** R50  Milestone: M4 (tests / smoke matrix)
**Upstream:** `design/tooling/r49-r50-plan.md` §5.6 M4 line in the
[paideia-os](https://github.com/paideia-os/paideia-os) repo.

## Purpose

M4 lands the test + smoke matrix for every M2/M3 code path that cp
committed to. The three per-issue specs below enumerate concrete
fixtures, observation hooks (cp .bss counter slots), invariants, and
pass criteria for the M4 test runner. Every spec is written as a
shape-in-place document today: the pass-criteria assertions are named
and grep-able so the paideia-as test-harness landing at a later round
can lift them mechanically.

## Per-issue specs

| Issue | Spec                                | Covers                                                            |
|-------|-------------------------------------|-------------------------------------------------------------------|
| #12   | [M4-001-txn-abort.md](M4-001-txn-abort.md) | Single-file, multi-file, recursive, over-existing (undo replayed), TXN-abort mid-copy (destination bytes fully unwound) |
| #13   | [M4-002-elevate-flow.md](M4-002-elevate-flow.md) | Cross-subtree elevate flow — auto-approve + human-approve paths, refused + timeout branches |
| #14   | [M4-003-signed-inode.md](M4-003-signed-inode.md) | Signed-inode preservation across same-user copy; graceful degrade across cross-user copy; verbose diagnostic on degrade |

## Runner shape (universal across the three specs)

A cp M4 test case is one row of the matrix inside a per-issue spec.
Each row names:

1. **Setup** — fixture files/directories the harness pre-creates
   under a scratch subtree, and the InitCap sidecar the harness
   passes to cp (which caps are seeded + what subtrees they cover).
2. **Invocation** — the argv the harness constructs (`cp [-r|-v|
   --over-existing] <src> <dst>`) and the exit-code contract.
3. **Assertions** — one or more of:
     - *Byte assertions* against destination file(s) or absence of
       a destination file (TXN-abort unwind check).
     - *Counter assertions* against cp .bss slots — the four M3
       modules (`CpPipe`, `CpAudit`, `CpElevate`) plus M2 modules
       (`SignedInode`, `Undo`, `CopyFile`, `Walk`) all expose
       observable counters (see the per-spec "Observation hooks"
       section).
     - *Journal assertions* against the audit journal
       (UEJ_KIND_TOOL_INVOKE / UEJ_KIND_TOOL_EXIT records) and the
       PdxFS undo journal (--over-existing record replay).
     - *Wire assertions* against the semantic-pipe stream
       (CopyProgressRecord@0.1 record body byte-identical to the
       expected 40-byte layout).
4. **Cleanup** — teardown of scratch subtree (the outer TXN abort
   on the TXN-abort matrix is itself the "cleanup" for that row).

## Substrate-landing gates

The M4 test matrix has three substrate gates. Each row of the matrix
either runs today (M4-observable) or is blocked on one of these gates
(M4-shape-in-place, executes when the gate lifts). Both classes are
in-tree at M4 so the harness landing later flips only the
"blocked" rows from skip to run.

| Gate                                    | Blocked matrix rows                                                       | Landing round |
|-----------------------------------------|---------------------------------------------------------------------------|---------------|
| PdxFS v1 `sys_pdxfs_readdir` / `_mkdir` / `_stat` / `_txn_*` | Multi-file, recursive, over-existing, TXN-abort mid-copy | R42 substrate expansion (paideia-os) |
| libpdx-elevate broker daemon boot       | Elevate auto-approve OK, human-approve OK, human-approve timeout          | libpdx-elevate.M4 + paideia-os R48.M7 daemon integration |
| libpdx-cap.M3 signed-inode helpers signing under `user_sk` | Signed-inode preservation same-user OK path      | libpdx-cap.M3 (landed 2026-08-21, commit 3673dec on libpdx-cap) — SI hooks are available but cp-side wiring to `cap_sign_inode` remains a body-edit to `SignedInode::preserve_at_destination` |

Rows that run today: every single-file (M4-001), every "refused"
elevate row (M4-002 exercises `CP_ELEVATE_REFUSED` via ELVC_STUB
today), every degrade row (M4-003 exercises `SI_DEGRADED_KEY_LOCKED`
via the M2-002 stub).

## paideia-as compliance surface (test-runner-facing)

The M4 test harness runs cp as an ordinary paideia-as tool. All test
observations that read a cp .bss counter do so via the standard
loader-level symbol table lookup — cp exposes every counter via a
`pub let mut` slot in its module namespace (see the per-spec
"Observation hooks" sections for the full symbol list). No test-only
build variant is needed; production and test builds are byte-
identical.

## Not in scope for M4

- **No fuzzers.** Plan §5 rubric assigns fuzzers to the general M4
  milestone rubric, but cp's M4 issues do not include a fuzz target.
  A future M4.5 or M5 landing can add one against the argv surface
  (three flags, two positionals — a narrow surface).
- **No pkg.M4 integration.** cp's M5 depends on pkg.M4 for the
  packages install path; the "installed cp survives a `pkg
  install cp`" test is deferred to M5.
- **No cross-tool pipeline test.** `cp | cat` or `ls | grep | cp`
  scenarios depend on shell.M4 + cat.M4 landing; they live in the
  shell M4 matrix (`shell.M4-003 QEMU smoke: login → prompt →
  ls | cat → history persists`).
