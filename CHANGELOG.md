# cp — CHANGELOG

All notable per-release changes to `cp` land here. Format follows
[Keep a Changelog](https://keepachangelog.com/) with paideia-os-
specific extensions: every version block names the closing commit
and every issue it lands, so the release-tag ↔ code-tree ↔ issue-
graph triangulation the signing pipeline uses at `paideia-as
release --sign` time is reproducible from this file alone.

## 1.0.0 — 2026-08-22

**Milestone close.** M5 — 1.0 signed release. Dual-signed
`manifest.pdxsig`, `doc/cp.pdxdoc` for `doc cp`, mirror-push
manifest for `pkgs.paideia-os/main/cp/1.0.0/`.

Aggregates every issue landed across M1-M5. Consumers upgrading
from a pre-1.0 tag receive the full flag surface, the M2 core
implementation, the M3 semantic-pipe + audit + elevate integration,
and the M4 test-spec matrix in a single install.

### Added

- M1-001 (#1) — scaffold + `caps.decl` + build manifest +
  `design/architecture.md`. First scaffolded commit.
- M1-002 (#2) — argv surface via libpdx-argv: `cp [-r|-v|--dry-run
  |--over-existing] <src> <dst>`. Long + short flag grammar per D3.
- M1-003 (#3) — first runnable single-file `cp a b` in a single
  KIND_PDXFS_TXN. 4 KiB scratch-buffer read/write loop through
  `.bss` per `CopyFile::copy_buf`.
- M2-001 (#4) — recursive `-r` walk via `Walk::walk_recursive` with
  per-file cap request. Typed-flag registration through
  `CpFlags::register_flags`.
- M2-002 (#5) — cap-tail preservation: re-sign at destination under
  invoker `user_sk` if unlocked via `SignedInode::preserve_at_
  destination`.
- M2-003 (#6) — graceful signed-inode degrade with `--verbose`
  diagnostic when key is locked.
- M2-004 (#7) — single-TXN atomicity across the whole invocation
  with `pdxfs_txn_abort` on partial failure; hoisted to dispatch
  level so multi-file recursive copies roll back as one unit.
- M2-005 (#8) — `--over-existing` PdxFS v1 undo record via
  `Undo::maybe_record_over_existing`.
- M3-001 (#9) — `CopyProgressRecord@0.1` schema bind + per-file
  emit via `src/pipe.pdx`. Fd 1 receives typed records; `--json`
  wires at the libpdx-semantic-pipe body-edit landing site.
- M3-002 (#10) — `CopyRecord` via libpdx-audit begin/commit wrap
  around dispatch. `audit_begin` fires before any user-visible
  output per D3 audit-first.
- M3-003 (#11) — libpdx-elevate retry on cross-subtree dst-parent
  via `src/elevate.pdx`. Requests `KIND_PDXFS_FILE(write, <dst-
  parent>)` widening on open failure; falls back to exit 4
  (cap-denied) on refuse.
- M4-001 (#12) — TXN-abort mid-copy test spec (`tests/M4-001-txn-
  abort.md`, 5 matrix rows).
- M4-002 (#13) — cross-subtree elevate flow test spec (`tests/M4-
  002-elevate-flow.md`, 6 matrix rows).
- M4-003 (#14) — signed-inode preservation + degrade test spec
  (`tests/M4-003-signed-inode.md`, 7 matrix rows).
- M5-001 (#15) — dual-signed `manifest.pdxsig` + `CHANGELOG-1.0`
  entry + `doc/cp.pdxdoc` for `doc cp`.
- M5-002 (#16) — mirror-push manifest for `pkgs.paideia-os/main/
  cp/1.0.0/` at `.plans/pkgs-mirror-push.md`; README install
  instruction lifts to `pkg install cp`.

### Substrate-landing gates carried into 1.0

Every row and every code path lands as either "runs today against
HEAD substrate" or "shape-in-place under a named substrate gate".
Both categories ship in 1.0 because the shape-in-place bodies are
body-edit-only landing sites — no restructure is required when a
gate lifts. The aggregated gate table is `.plans/M4.md` §Substrate
landing gates.

- PdxFS v1 `sys_pdxfs_readdir` / `_mkdir` / `_stat` / `_txn_*` —
  paideia-os R42 substrate expansion.
- libpdx-elevate.M2-002 human-approve + block-on-reply.
- libpdx-elevate.M3-001 auto-approve + policy consult.
- libpdx-elevate broker daemon boot.
- M4 harness mock human granter + `cap_tail_read` helper.
- paideia-as v0.33-crypto: `ml_dsa_65_sign` + `ml_dsa_65_verify`
  intrinsics for `manifest.pdxsig` signature placeholders.
- pkg.M4 — `pkg install cp` becomes a live invocation path.

### Cross-repo dependencies at 1.0

- libpdx-argv @ ^0.2 — argv parse.
- libpdx-semantic-pipe @ ^0.2 — CopyProgressRecord@0.1 bind + emit.
- libpdx-audit @ ^0.2 — CopyRecord begin/commit wrap.
- libpdx-elevate @ ^0.1 — cross-subtree dst-parent retry.
- libpdx-cap (indirect) — signed-inode helpers at M3-002 (landed
  2026-08-21, commit 3673dec on libpdx-cap).

### Notes

- paideia-as v0.33 is the toolchain floor (`mov_b` narrow-load
  mnemonic + `@align` attribute on `.bss` slots per the #1248
  mitigation). The 1.0 release refuses to build on a paideia-as
  older than that.
- No GitHub Actions per `feedback_paideia_os_no_cicd`. Verification
  is local via `paideia-as build` + `paideia-as test` + `bash
  smoke/*.sh` at the author machine before `paideia-as release
  --sign`.
- Documentation shipped at 1.0 is `doc/cp.pdxdoc` (long-form for
  `doc cp`). `help.pdx`, `tutorial.pdx`, and `examples/` per
  `design/tooling/plan.md` §5 layout land at a post-1.0 M6 once
  `pdx-help` and `tutor` reach a stable release; 1.0 satisfies
  I7 §2 (`doc <tool>`) — the remaining three land inside the
  R51+ documentation wave.
