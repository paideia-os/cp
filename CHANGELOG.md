# cp — CHANGELOG

All notable per-release changes to `cp` land here. Format follows
[Keep a Changelog](https://keepachangelog.com/) with paideia-os-
specific extensions: every version block names the closing commit
and every issue it lands, so the release-tag ↔ code-tree ↔ issue-
graph triangulation the signing pipeline uses at `paideia-as
release --sign` time is reproducible from this file alone.

## 1.1.0-B — 2026-09-12 (userspace cwd-relative dst resolution)

Closes paideia-os/cp#21 (`cp.ENH-005 cp <src> <bare-name> fails:
O_CREAT refuses slash-less destinations`).

Under v1.1-A cp passed the destination argv verbatim to `sys_open`
with `O_WRONLY|O_CREAT|O_TRUNC`. The kernel's `vfs_open` O_CREAT
parent-scan (`src/kernel/core/fs/vfs_open.pdx`, label
`vfs_open_creat_scanned`) refuses a slash-less create outright, so
`cd /home/alice && cp notes.md backup.md` printed
`cp: cannot open destination file` and exited 1 — the most common
`cp` invocation there is. v1.1-B closes that gap in userspace by
resolving the destination against the task cwd BEFORE the O_CREAT
`sys_open` call, matching the mkdir v1.1-B precedent
(paideia-os/mkdir#25). Absolute destinations pass through unchanged.

The read side is unchanged: R86.M1-005 (paideia-os #1958) rewired
`vfs_open`'s main `path_resolve` to anchor relatives against
`TASK_OFF_CWD`, so `cp foo/bar.txt /tmp/x` already worked under
v1.1-A. Only the O_CREAT parent-scan lacked cwd parity — this
release lands that parity from the userspace side.

### Added

- `src/pdxfs.pdx::pdxfs_getcwd(buf_va, buf_len) -> bytes_or_errno`
  — arity-2 leaf trampoline for paideia-os sysno 86 (`sys_getcwd`,
  R86.M1-003 #1956). Returns bytes written INCLUDING the trailing
  NUL on success (POSIX `getcwd(3)` convention), or a negative-
  errno u64 (`-EFAULT`/`-ERANGE`/`-ENOENT`) on failure.
- `src/copy.pdx::copy_resolve_dst_cwd(dst_ptr) -> resolved_ptr_or_errno`
  — leaf helper that classifies `dst_ptr[0]`. `'/'` -> return
  `dst_ptr` unchanged; anything else (including `.`, `..`, `./x`,
  `foo`, `a/b`) -> call `pdxfs_getcwd` into `copy_cwd_scratch`,
  join `cwd + '/' + dst` into `copy_dst_resolved`, return the
  resolved pointer. Separator is elided when cwd is exactly `"/"`
  (no double slash on `cp foo bar` under root). Overflow returns
  `-ENAMETOOLONG` (-36) when the joined length would exceed the
  255-byte-plus-NUL scratch slot.
- `src/copy.pdx` — two 256-byte `.bss` scratches:
  - `copy_cwd_scratch` — destination for `sys_getcwd`; 256 bytes
    matches `SYS_GETCWD_PATH_MAX` in the kernel's own header, so
    any path the kernel can compose fits without a `-ERANGE`
    round-trip.
  - `copy_dst_resolved` — staging for the joined absolute path
    handed to `sys_open`; 256 bytes matches `SYS_MKDIR_PATH_MAX`
    per the mkdir v1.1-B precedent.
- `src/copy.pdx::copy_bytes_only` — new Step 0.5 between
  `copy_reset` and the `sys_stat(DST)` probe: call
  `copy_resolve_dst_cwd` and update `r12` in place before every
  subsequent step (stat, basename-join, open) sees the pointer.
- Two new diagnostic strings on stderr:
  - `cp: cannot read cwd for dst path` — a `sys_getcwd` -errno
    passthrough.
  - `cp: resolved destination path too long` — the join-length
    overflow gate (`cwd_len + sep + user_len > 255`).

### Changed

- `src/pdxfs.pdx` — module header expanded to describe the sixth
  syscall (`pdxfs_getcwd`, sysno 86). `SYSCALL_GETCWD : u64 = 86`
  added to the sysno constant block.

### Behavioural contract

- `cp SRC DST` where DST starts with `'/'` (absolute) — behaviour
  unchanged from v1.1-A.
- `cp SRC DST` where DST starts with anything else — v1.1-B calls
  `sys_getcwd`, joins `cwd + '/' + DST` (separator elided when
  cwd is `"/"`), and passes the resolved absolute path to
  `sys_open`. Downstream steps (stat, basename-join, open) see
  the resolved pointer. `cp foo .` still copies foo into the cwd
  directory (the stat probe finds `"/cwd/."` is a directory and
  `copy_join_dir_basename` writes `"/cwd/./foo"` — the kernel's
  `path_resolve` normalises the `.` component).
- New exit sentinels:
  - Any `sys_getcwd` -errno passthrough (`-EFAULT`=-14,
    `-ERANGE`=-34, `-ENOENT`=-2) surfaces as the tool's exit
    code with the `cp: cannot read cwd for dst path` diagnostic.
    Not reachable in practice at R86 (a task's cwd is always
    valid; a full 256-byte buffer never triggers `-ERANGE`).
  - `-ENAMETOOLONG` (-36) surfaces with the
    `cp: resolved destination path too long` diagnostic when
    the joined absolute path would exceed 255 bytes.

### Discipline (paideia-as v0.36+ / feedback_pdx_encoder_pitfalls)

- No `test rN, rN` — every zero-check via `cmp reg, 0`.
- No `and rN, imm64` on r8-r15 (no ANDs in the new helper at all).
- No 2-op `imul r, imm`.
- Every `cmp reg, imm` immediate fits `imm16` (largest is 255).
- `-ENAMETOOLONG` (`0xFFFFFFFFFFFFFFDC`) encoded as `mov r64, imm64`
  (movabs) into both `rax` (helper return) and `r11` (sentinel
  compare in the caller-side error branch).
- Byte reads: `xor rax, rax; mov_b rax, [reg + reg*1]` per the
  #1248 mitigation pattern.
- Reserved-label discipline: every helper label prefixed `crd_`
  (avoids `loop`/`done`/`top`/`if` keyword collisions); caller-
  side branches prefixed `cp_`.
- 1-push prologue for `copy_resolve_dst_cwd` (rbx) keeps
  `rsp % 16 == 0` at the nested `pdxfs_getcwd` call site.

### Kernel-side companion (unchanged from v1.1-A)

The issue body notes that the correct final resting place for the
O_CREAT parent-scan cwd anchor is inside the kernel's `vfs_open`
(anchor the parent at `TASK_OFF_CWD` when no `/` is found rather
than failing). v1.1-B's userspace resolve is compatible with that
future kernel fix: when the kernel learns to anchor, the resolved
absolute path cp already produces continues to work unchanged, and
the two arguments' semantics stay in lockstep.

## 1.1.0-A — 2026-09-07

**Real body extraction.** Retires the M1-001 STUB shape shipped by
v1.0 (the KIND_PDXFS_TXN begin/commit/abort trio, libpdx-audit +
libpdx-elevate + libpdx-semantic-pipe wire-ins, the recursive
walker, --over-existing undo journalling, and the cap-tail signed-
inode preservation — every one of them a shape-in-place stub
against a substrate that has not landed). What ships in v1.1-A is
a five-syscall real body: sys_open, sys_read, sys_write, sys_close,
sys_stat.

### Added

- `src/copy.pdx` — v1.1-A `copy_bytes_only(src_ptr, dst_ptr)` body:
  1. sys_stat(DST) — if DST resolves to a directory (POSIX mode with
     S_IFDIR bits), construct effective DST = DST + "/" + basename(SRC)
     into `copy_dst_path` (.bss, 512 bytes).
  2. sys_stat(SRC) — extract POSIX mode word (u32 @ statbuf+24);
     fall back to 0644 when the backend fills mode=0 (tmpfs today).
  3. sys_open(SRC, O_RDONLY, 0).
  4. sys_open(effective_DST, O_WRONLY|O_CREAT|O_TRUNC=0xC1, src_mode).
  5. Read/write loop over the 4 KiB `copy_buf` .bss scratch until EOF.
  6. sys_close(dst); sys_close(src).
  7. Return 0 on success or the raw negative-errno u64 from the
     failing syscall on any failure.
- `src/copy.pdx::copy_join_dir_basename` — leaf helper that builds
  the effective DST path when DST resolves to a directory. Caps the
  combined length at 511 bytes (+ NUL) to fit the 512-byte .bss slot.
- `src/pdxfs.pdx::pdxfs_open` widens to arity 3 — `(path, flags, mode)`
  — so the caller preserves the source's POSIX mode word when
  creating the destination. v1.0 hard-wired mode=0.

### Removed

- `src/audit.pdx` — libpdx-audit begin/commit wrap.
- `src/elevate.pdx` — libpdx-elevate cross-subtree retry.
- `src/undo.pdx` — --over-existing undo record path.
- `src/pipe.pdx` — CopyProgressRecord schema bind + emit.
- `src/signed_inode.pdx` — cap-tail preservation.
- `src/walk.pdx` — -r recursive walker.
- `src/flags.pdx` — cp-owned flag registration (no cp-owned flags
  at v1.1-A; libpdx-argv's StdVocab still parses --help / --version /
  --verbose / --dry-run so the recognisers do not error, but no arm
  consumes them).
- `src/pdxfs.pdx::pdxfs_txn_begin` / `pdxfs_txn_commit` /
  `pdxfs_txn_abort` — KIND_PDXFS_TXN stubs (returned 0).
- `src/pdxfs.pdx::pdxfs_readdir` / `pdxfs_mkdir` — walker-only
  entry points (no walker in v1.1-A).
- Manifest deps: libpdx-semantic-pipe, libpdx-audit, libpdx-elevate.
- Caps decl entries: KIND_IPC_ENDPOINT, KIND_PDXFS_TXN,
  KIND_ELEVATE_CHANNEL.

### Changed

- `src/dispatch.pdx::dispatch_copy` collapses from the v1.0
  audit-wrap + TXN-wrap + flags-populate + dry-run + paths-conflict +
  recursive-route shell to a two-step body: pos_count==2 gate then
  `copy_bytes_only` call. Return propagates copy body's rax verbatim.
- `src/main.pdx::cp_main` drops the seven counter-resets
  (cp_pipe_reset / cp_audit_reset / cp_elevate_reset / copy_reset /
  walk_reset / undo_reset / signed_inode_reset), the
  `register_cp_flags` call, and the `cp_pipe_bind_stdout` call. The
  argv-parse pipeline (FlagSpec::reset -> StdVocab::register_all ->
  ParsedArgs::reset -> Parser::parse_argv) plus `dispatch_copy` is
  the whole body.
- Exit codes: 0 on success, 2 on usage / parse error, or the raw
  negative-errno u64 from the failing syscall on copy failure. v1.0's
  EXIT_OP_FAIL (1), EXIT_NOT_YET_IMPL (3), EXIT_CAP_DENIED (4) all
  retire with the M1-001 STUB shape.

### Deferred to v1.2

- `-p` (preserve permissions beyond the source mode word — atime,
  mtime, uid, gid) once `sys_utimes` and `sys_chown` land.
- `-r` (recursive walk) once `sys_getdents` sits on a real,
  non-terminator-stub backend contract.
- `-i` (interactive overwrite) is out of scope for a non-interactive
  R50 tool.

### Notes

- paideia-as v0.36+ is the toolchain floor (`mov_b` + `mov_d`
  narrow-load mnemonics + `@align` attribute on `.bss` slots).
- No GitHub Actions per `feedback_paideia_os_no_cicd`. Verification
  is local via `bash tools/build.sh` + `bash tools/run-qemu.sh`.

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
