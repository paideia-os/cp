# cp — status

**Wave:** R50 (Wave 2)
**Current milestone:** M5 (1.0 signed release) — M5-001 landed
**Version:** 1.0.0 (release-tag pending signing pipeline —
`manifest.pdxsig` author + paideia_root fields are `<pending-*>`
placeholders until paideia-as v0.33-crypto exposes `ml_dsa_65_sign`)

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
- **M2 — core implementation (complete).** Recursive `-r` walk via
  `Walk::walk_recursive` gated behind libpdx-argv typed-flag
  registration (issue #4). Cap-tail preservation via
  `SignedInode::preserve_at_destination` (issue #5). Graceful
  signed-inode degrade with `--verbose` diagnostic (issue #6). Single-
  TXN atomicity hoisted to dispatch level with `pdxfs_txn_abort` on
  partial failure (issue #7). `--over-existing` PdxFS v1 undo record
  via `Undo::maybe_record_over_existing` (issue #8). Multiple
  substrate-blocked call sites (readdir/mkdir/stat/undo-write) ship
  as stubs matching M1's txn-begin/commit stub discipline — R42
  substrate expansion is a body-edit landing site with no
  restructure required in cp.
- **M3 — semantic-pipe / audit integration (complete).**
  New modules `src/pipe.pdx` (CpPipe: bind fd 1 to
  CopyProgressRecord@0.1 schema hash + per-file emit), `src/audit.pdx`
  (CpAudit: begin/commit wrap around dispatch — audit_begin fires
  before any user-visible output per D3), and `src/elevate.pdx`
  (CpElevate: libpdx-elevate retry on dst-parent open failure).
  Wire-in edits: `src/main.pdx` (three resets + pipe_bind_stdout),
  `src/dispatch.pdx` (audit_begin_wrap at top + audit_commit_wrap at
  epilogue), `src/copy.pdx` (emit progress after successful copy +
  elevate retry on dst-open fail). walk.pdx documents the per-entry
  emit site inside walk_entry_todo (lands with R42 readdir substrate).
- **M4 — tests + smoke (complete).** Three test-spec markdown
  files under `tests/` — one per issue. 18 total matrix rows
  across `M4-001-txn-abort.md` (5 rows: single-file happy path
  runs today; multi-file + over-existing undo replay + TXN-abort
  + TXN-commit-fail shape-land under R42 substrate gates),
  `M4-002-elevate-flow.md` (6 rows: in-scope baseline + no-cap
  ELVC_ERR_LOOKUP_FAIL run today; auto-approve OK + human-
  approve OK/refused/timeout shape-land under libpdx-elevate.
  M2-002/M3-001 + broker daemon boot + M4 harness mock granter),
  `M4-003-signed-inode.md` (7 rows: key-locked degrade + verbose
  degrade run today; same-user preserve + cross-user degrade +
  recursive variants shape-land under libpdx-cap.M3-002 body-
  edit + M4 harness `cap_tail_read`). Every row's observation-
  hook assertions compile against the current cp .bss symbol
  table; each spec's substrate-landing-gate table names the
  body-edit landing sites for the shape-in-place rows.
- **M5 — 1.0 signed release (M5-001 landed).** Dual-signed
  `manifest.pdxsig` for cp v1.0 (signature bytes are
  `<pending-author-sign>` / `<pending-root-sign>` placeholders,
  ready for the signing pipeline to line-edit once paideia-as
  v0.33-crypto lands `ml_dsa_65_sign`); `CHANGELOG.md` 1.0 entry
  aggregating every M1-M5 issue; `doc/cp.pdxdoc` per I7 §2 for
  `doc cp`. Version bumped 0.4.0-m4 → 1.0.0 in
  `manifest.pdxproj`. M5-002 mirror push follows in the next
  commit — `.plans/pkgs-mirror-push.md` names the URL layout at
  `pkgs.paideia-os/main/cp/1.0.0/` and the staging → main
  promotion path.

See `design/tooling/r49-r50-plan.md` §5.6 in paideia-os for the full
breakdown and cross-repo dependencies.

## Cross-repo state at M1 close

- libpdx-argv M1 landed (issues #1, #2, #3 closed 2026-08-21) — the
  argv parse dep is met.
- libpdx-cap M1 landed — indirect (loader-side manifest verify still
  uses the M1 skeleton; M2 flips to the real OK|MISSING|EXTRA compare).
- libpdx-audit / libpdx-elevate / libpdx-semantic-pipe M1 all landed —
  none are direct deps at cp.M1. M3 upgrades these to direct deps
  (libpdx-semantic-pipe @ ^0.2, libpdx-audit @ ^0.2, libpdx-elevate
  @ ^0.1) via manifest.pdxproj.
- paideia-os R48 substrate closed: KIND_USER = 0x190,
  KIND_ELEVATE_CHANNEL = 0x191, KIND_PDXFS_FILE = 0x195,
  KIND_PDXFS_TXN = 0x196 (commits 411ad0e, e56a95b, 2ff76d4).
- paideia-os kernel exposes sys_read/sys_write/sys_open/sys_close/
  sys_exit at their standard Linux syscall numbers (0/1/2/3/60);
  the KIND_PDXFS_TXN begin/commit syscalls remain to land at R42
  substrate expansion.
