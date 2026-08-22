# cp — status

**Wave:** R50 (Wave 2)
**Current milestone:** M1 (design + skeleton) — complete

## Milestone map

- **M1 — design + skeleton (complete).** Scaffold (caps.decl +
  build manifest — issue #1), argv surface with `[-r|-v|--dry-run|
  --over-existing] <src> <dst>` via libpdx-argv (issue #2), first-
  runnable single-file `cp a b` (issue #3). First-runnable shape:
  `cp a b` walks the full dispatch chain (parse → flag gate →
  positional validate → copy body → exit) and copies bytes through
  a 4 KiB .bss scratch buffer via sys_open/sys_read/sys_write/
  sys_close. `KIND_PDXFS_TXN` begin/commit are M1 stubs (return 0)
  because the kernel does not yet expose txn syscalls; the M2 patch
  swaps only the two stub bodies.
- **M2 — core implementation (not started).** Recursive `-r`, cap-
  tail preservation (re-sign at destination under invoker's user_sk
  if unlocked; graceful `--verbose` degrade otherwise), real single-
  TXN atomicity across the whole invocation, `--over-existing` undo
  record on PdxFS v1.
- **M3 — semantic-pipe / audit integration (not started).**
  CopyProgressRecord[] per file via libpdx-semantic-pipe; CopyRecord
  pre-output journal via libpdx-audit; libpdx-elevate retry when
  the destination-parent crosses a per-user-subtree boundary.
- **M4 — tests + smoke (not started).** Single-file, multi-file,
  recursive, over-existing (undo replayed), TXN-abort mid-copy,
  cross-subtree elevate flow (auto-approve + human-approve paths),
  signed-inode preservation matrix.
- **M5 — 1.0 signed release (not started).** Dual-signed
  manifest.pdxsig for cp v1.0 + CHANGELOG entry, pkgs.paideia-os
  mirror push, .pdxdoc for `doc cp`.

See `design/tooling/r49-r50-plan.md` §5.6 in paideia-os for the full
breakdown and cross-repo dependencies.

## Cross-repo state at M1 close

- libpdx-argv M1 landed (issues #1, #2, #3 closed 2026-08-21) — the
  argv parse dep is met.
- libpdx-cap M1 landed — indirect (loader-side manifest verify still
  uses the M1 skeleton; M2 flips to the real OK|MISSING|EXTRA compare).
- libpdx-audit / libpdx-elevate / libpdx-semantic-pipe M1 all landed —
  none are direct deps at cp.M1.
- paideia-os R48 substrate closed: KIND_USER = 0x190,
  KIND_ELEVATE_CHANNEL = 0x191, KIND_PDXFS_FILE = 0x195,
  KIND_PDXFS_TXN = 0x196 (commits 411ad0e, e56a95b, 2ff76d4).
- paideia-os kernel exposes sys_read/sys_write/sys_open/sys_close/
  sys_exit at their standard Linux syscall numbers (0/1/2/3/60);
  the KIND_PDXFS_TXN begin/commit syscalls remain to land at R42
  substrate expansion.
