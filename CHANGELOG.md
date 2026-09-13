# cp — CHANGELOG

All notable per-release changes to `cp` land here. Format follows
[Keep a Changelog](https://keepachangelog.com/) with paideia-os-
specific extensions: every version block names the closing commit
and every issue it lands, so the release-tag ↔ code-tree ↔ issue-
graph triangulation the signing pipeline uses at `paideia-as
release --sign` time is reproducible from this file alone.

## 1.3.0 — 2026-09-13 (outer TXN wrap + v1.2 planning)

Closes paideia-os/cp#35 (`cp.ENH-011` — outer TXN re-wire against the
R90 kernel substrate) and paideia-os/cp#37 (v1.2 planning trackers
for `-p` / `-r`, planning-only).

### Added

- `src/pdxfs_txn.pdx` — new module `PdxfsTxn`. Five trampolines
  against the landed R90 KIND_PDXFS_TXN substrate: `pdxfs_txn_begin`
  (sysno 70, arity 4), `pdxfs_txn_bind` (pure `.bss` context store,
  no syscall), `pdxfs_txn_add_write` (sysno 107 = sys_pdxfs_undo_
  write), `pdxfs_txn_commit` (sysno 104), `pdxfs_txn_abort` (sysno
  105), plus `pdxfs_txn_alloc_id` (monotonic txn_id counter). The
  module header documents a sysno reconciliation: the original
  dispatch template assumed 106/108/109/110 for add_write/abort/
  status/free, but the actual landed kernel (dispatch.pdx +
  handlers/sys_pdxfs_txn_*.pdx, R90-XREPO.010 #2111/#2112) uses
  104=commit, 105=abort, 107=undo_write, and has no status/free
  syscall yet — this module wires only what is real. `add_write`'s
  5-field real syscall shape is split across `pdxfs_txn_bind`
  (cap_slot, inode_no — once per file) + `pdxfs_txn_add_write`
  (offset, len, kbuf_ptr — once per block) so no function in the
  module exceeds 4 curried args.
- `src/copy.pdx::copy_bytes_only` — new Step 4.5 best-effort outer
  TXN wrap. After both fds open, stats the effective dst path for its
  inode number and calls `pdxfs_txn_begin` with a stub `parent_slot
  = 0` (no loader-side KIND_PDXFS_VOL slot-discovery syscall exists
  yet — see `pdxfs_txn.pdx` header). ANY negative return sets `r12`
  (dead after open-dst) to the `TXN_INACTIVE` (-1) sentinel and every
  downstream TXN call site gates on `cmp r12, 0; jl`, so this landing
  degrades cleanly to the pre-#35 body with zero behavioural change
  until a real parent_slot is discoverable. When a TXN opens, every
  write block calls `pdxfs_txn_add_write` before its byte count is
  accumulated, the `cp_eof` success tail calls `pdxfs_txn_commit`,
  and the `cp_read_fail` / `cp_write_fail` / `cp_short_write` error
  branches call `pdxfs_txn_abort` before their existing close +
  diagnostic logic.
- `design/cp-v1.2-planning.md` — new planning-only design doc for
  `-p` (permission preserve: chmod/chown/utimensat, fail-open except
  chmod) and `-r` (recursive: readdir + per-entry recurse-or-copy, no
  symlink descent without a future `-L`), an argv-parsing note, and a
  test-matrix placeholder (`tests/cp_p.pdx` / `tests/cp_r.pdx`
  shapes). Implementation is explicitly deferred to a future
  v1.2.0-A-shaped release; paideia-os/cp#37 is planning only.

### Changed

- `manifest.pdxproj` — `version = 1.2.0` -> `1.3.0`; source list adds
  `src/pdxfs_txn.pdx` (8 sources total).
- `manifest.pdxsig` — `version = 1.2.0` -> `1.3.0`.
- `caps.decl` — `KIND_PDXFS_VOL (invoke, <mount>)` row's comment
  updated: the invoke call site now exists (`pdxfs_txn_begin`) but
  runs against a stub slot pending the loader-side discovery syscall.

### Not changed

- `src/pdxfs.pdx`, `src/dispatch.pdx`, `src/main.pdx`, `src/pipe.pdx`,
  `src/tool_ident.pdx`, `src/print.pdx` — byte-identical to v1.2.0.

### Follow-up

- Real `parent_slot` resolution for `pdxfs_txn_begin` once a loader-
  side KIND_PDXFS_VOL slot-discovery syscall lands — only the stub
  constant in `src/pdxfs_txn.pdx` needs to change.
- True per-write pre-image capture for `pdxfs_txn_add_write` (today's
  `kbuf_ptr` is the block just written, a best-effort WAL record, not
  a genuine pre-overwrite snapshot) plus the commit-and-reopen
  batching a large-file copy needs against the 32-records/4KiB row
  cap — carried over from the v1.2.0 Follow-up note.
- `sys_pdxfs_txn_status` / `sys_pdxfs_txn_close` trampolines once the
  kernel lands those syscalls (currently no sysno assigned).
- paideia-os/cp#37's `-p` / `-r` implementation, tracked for a future
  v1.2.0-A-shaped release per `design/cp-v1.2-planning.md`.

## 1.2.0 — 2026-09-13 (Wave D drain close-out)

Closes paideia-os/cp#31 (`R90-XREPO.013.M3-003 cp — caps.decl +
adoption`), paideia-os/cp#32 (`v1.1-A real-body extraction` --
verified already landed at beecb0c, closed with cite),
paideia-os/cp#33 (`v1.1-B Semantic-pipe emission wire`),
paideia-os/cp#34 (`v1.1-C Release closer + tag`), and
paideia-os/cp#36 (`regression test: dst-basename-only path relative
to cwd`).

Consolidates the Wave D drain into a single v1.2.0 release rather
than the four sub-tags (v1.1-A/B/C) originally scoped for Track C.
Grounds: v1.1-A landed at beecb0c and v1.1-B / v1.1-C landed at
successive commits during the R90 sweep; the semantic-pipe wire
(this release) is functionally additive over that stack, and the
tool-identity externs (Wave 6 libpdx-argv v1.1.3 contract) plus
the KIND_PDXFS_VOL declaration (R90-XREPO.013.M3-003) constitute a
minor-version bump under semver rather than three chained patch
releases.

### Added

- `src/pipe.pdx` -- new REAL body semantic-pipe emit module.
  `Pipe::pipe_emit_copy_record(bytes_copied, src_mode)` marshals a
  24-byte CopyRecord@0.1 (bytes_copied u64@+0, src_mode u64@+8,
  tsc_ticks u64@+16) into a static `.bss` scratch buffer and
  invokes `sys_semantic_send` (sysno 115, R107-M0-001 paideia-os
  #2350) with schema tag `0x436F707952656301` (LE-packed
  "CopyRec\x01"; low byte is the version marker). Consumer decoders
  live in libpdx-semantic-pipe once the recv side lands (post-R107).
  Best-effort emission: return value from `sys_semantic_send` is
  discarded (both -EFAULT and -EINVAL are marshalling bugs in this
  file, not per-run conditions the caller can act on -- pdxsock
  v1.1-B precedent at `tools/user/pdxsock/src/main.pdx` L1929).
- `src/tool_ident.pdx` -- new module. `PDX_TOOL_NAME : [u8; 3] =
  "cp\0"` and `PDX_TOOL_VERSION : [u8; 6] = "1.2.0\0"` externs per
  the Wave 6 libpdx-argv v1.1.3 contract (ENH-032, closes the UND
  symbol requirement that landed with libpdx-argv 1.1.3). Resolved
  at final-link time by `VersionBackend::emit_default`; `cp
  --version` now prints `cp 1.2.0\nCP VERSION OK\n` once the
  libpdx-argv dep bumps to >= 1.1.3.
- `src/copy.pdx::copy_bytes_only` -- new emit block at the
  `cp_eof` success arm: `lea r11, [rip + bytes_copied]; mov rdi,
  [r11]; mov rsi, r13; call pipe_emit_copy_record;` sits between
  the two `sys_close` calls and the `xor rax, rax; jmp cp_epilogue`
  tail. `r13` still carries `src_mode` from the `cp_mode_ready`
  phase (callee-save preserved through every intervening call);
  `bytes_copied` reads its accumulated total from the module `.bss`
  slot the read/write loop stamped into.
- `caps.decl` -- new `KIND_PDXFS_VOL (invoke, <mount>)` row per
  R90-XREPO.013.M3-003 (paideia-os/cp#31). DECLARATION + ADOPTION
  NOTE only; the invoke arm lands under paideia-os/cp#35 body
  edit (outer TXN re-wire against the R90 kernel substrate
  sysnos 70/104/105/107). Landing the declaration now (rather
  than at cp#35 landing) means the manifest reconciler will not
  fail-fast with `CAP_MANIFEST_MISSING` when cp#35's first
  `pdxfs_txn_open` call site lands -- the cap is already declared;
  only the invoke happens later.
- `caps.decl declares_output_schemas:` -- new
  `CopyRecord@0.1 (schema=0x436F707952656301, len=24,
  via=sys_semantic_send)` entry. Consumed by the InitCap sidecar
  packaging step for the loader-side output-schema advertisement.
- `tests/dst_basename_smoke.pdx` -- new regression driver
  (`TestDstBasenameSmoke::run`). Closes paideia-os/cp#36 (the
  missing test for the v1.1-B cwd-relative dst contract). Three
  phases: (1) setup -- unlink + create source at
  `/tmp/dst_basename_src`, chdir into `/tmp`; (2) call
  `copy_bytes_only("dst_basename_src", "dst_basename_dst")` --
  both basenames, no leading slash; (3) `sys_stat` on
  `/tmp/dst_basename_dst` asserts the copy landed at the
  cwd-relative absolute path. Return codes 0/1/2/3 per this org's
  M4-driver convention.

### Changed

- `manifest.pdxproj` -- `version = 1.1.0-C` -> `version = 1.2.0`;
  source list expanded from 5 to 7 (adds `src/pipe.pdx` +
  `src/tool_ident.pdx`); header comment describes the Wave D drain
  scope.
- `manifest.pdxsig` -- `version = 1.0.0` -> `version = 1.2.0`;
  `release_date` -> `2026-09-13`; `paideia_as_min` bumped
  `0.33 -> 0.36` to match the encoder-discipline floor the current
  source tree targets (mov_b + mov_d + @align).

### Behavioural contract

- Every completed `cp SRC DST` invocation (regardless of whether
  DST was absolute, relative, or resolved to a directory) now emits
  ONE `CopyRecord@0.1` (24 bytes) to the kernel-side
  `sys_semantic_send` ring at the successful-exit tail of
  `copy_bytes_only`. The record shape is stable across future
  layout extensions -- consumers match on the leading 56 bits of
  the schema tag and use the ring slot's `len` header to detect
  layout-version drift.
- Failure branches (parse error, wrong pos_count, stat/open/read/
  write failure, dst-path overflow, cwd-resolve overflow, short
  write) emit NO record -- there is no "session" to describe when
  the copy did not complete. This mirrors the pdxsock v1.1-B
  precedent: emit only on the completed-path tail.
- `--version` continues to route through libpdx-argv's `StdVocab`
  (as it has since v1.1-A). Once the tool's deps bump to
  libpdx-argv >= 1.1.3, `VersionBackend::emit_default` reads the
  new `PDX_TOOL_NAME` + `PDX_TOOL_VERSION` externs and emits
  `cp 1.2.0\nCP VERSION OK\n` on stdout.

### Discipline (paideia-as v0.36+ / feedback_pdx_encoder_pitfalls)

- No `test rN, rN` -- every zero-check via `cmp reg, 0`.
- No `and rN, imm64` on r8-r15 (no ANDs in the new modules at all).
- No 2-op `imul r, imm`.
- Schema tag `0x436F707952656301` uses `mov r64, imm64` (movabs;
  paideia-as auto-selects the REX+imm64 encoding for immediates >
  0x7FFFFFFF).
- `rdtsc` reconstruction pattern (`shl rdx, 32; or rax, rdx`) per
  src/kernel/core/fs/pdxfs_lite/uuid.pdx.
- Module basenames PascalCase (`Pipe`, `ToolIdent`,
  `TestDstBasenameSmoke`) match file basename.
- Reserved-label discipline: `cp_test_` prefix throughout
  `dst_basename_smoke.pdx`; no branch labels in `pipe.pdx` /
  `tool_ident.pdx`.
- Pipe emit block in `copy.pdx` is a bare 3-instruction sequence
  ahead of `xor rax, rax`, no new labels introduced -- keeps the
  `cp_` label discipline of the enclosing function intact.

### Not changed

- `src/print.pdx`, `src/pdxfs.pdx`, `src/dispatch.pdx`,
  `src/main.pdx` -- byte-identical to v1.1.0-C. The
  semantic-pipe emit adds one bare `call pipe_emit_copy_record`
  inside `copy.pdx::copy_bytes_only::cp_eof` and one comment
  block; no other source file is touched.
- v1.1.0-C's atomicity-claim retirement (README.md,
  doc/cp.pdxdoc, design/pdxfs-notes.md) stands. `cp` is still NOT
  transactionally atomic at v1.2.0; the outer TXN re-wire against
  the R90 substrate (sysnos 70/104/105/107) remains tracked at
  paideia-os/cp#35 alongside the KIND_PDXFS_VOL invoke arm this
  release only declares.

### Follow-up

- paideia-os/cp#35 (`cp.ENH-011`) -- outer TXN re-wire against the
  R90 substrate. Now unblocked on the cap-manifest side: the
  `KIND_PDXFS_VOL (invoke, <mount>)` row this release declares
  is exactly the cap `pdxfs_txn_open(vol_slot, flags)` will
  consume.
- Consumer decoders for `CopyRecord@0.1` -- deferred to whichever
  R107 wave lands the sys_semantic_recv counterpart. The record
  shape declared at this release is the wire contract those
  decoders must match.

## 1.1.0-C — 2026-09-12 (atomicity-claim retirement, doc-only)

Closes paideia-os/cp#23 (`cp.ENH-007 TXN begin/commit/abort are
no-ops while the docs assert atomicity as fact`).

**Doc-only release.** No source file changes, no manifest surface
changes beyond the version bump, no build reshape. v1.1-A retired
the `pdxfs_txn_begin` / `pdxfs_txn_commit` / `pdxfs_txn_abort`
trampolines out of `src/pdxfs.pdx` and dropped the outer
begin/commit/abort wrap out of `src/dispatch.pdx::dispatch_copy`;
`caps.decl` dropped `KIND_PDXFS_TXN (invoke)` in the same release.
The tagline, the README Description section, the `doc/cp.pdxdoc`
§3 / §6 / §8 renders, and the design/pdxfs-notes.md substrate
audit did not get updated at the time and continued to assert
single-invocation transactional atomicity as a v1.1 property. This
release brings all four documents into line with what v1.1-A
actually shipped.

### Changed

- `README.md` — tagline drops "atomic single-TXN" (v1.1 is not
  atomic); the Description section drops the `dispatch_copy calls
  pdxfs_txn_begin/commit/abort` paragraph and adds a new
  "Atomicity status (v1.1)" section that names the current
  behaviour honestly ("cp is NOT atomic at v1.1"), catalogues the
  kernel substrate that has since landed (sysnos 70 / 104 / 105 /
  107), and points at paideia-os/cp#35 for the re-wire; the exit-
  code table row for `EXIT_OK` drops the `pdxfs_txn_commit
  succeeded` clause and the `EXIT_OP_FAIL` row is marked as no
  longer emitted at v1.1 (`copy_bytes_only` returns raw negative-
  errno u64 instead); the Capabilities section drops the
  `KIND_PDXFS_TXN (invoke)` row from the `caps.decl` example (it
  is not in `caps.decl` at v1.1).
- `doc/cp.pdxdoc` — `@version` bumped to 1.1.0-C; the `@one-liner`
  drops "with per-file cap re-signing" (retired at v1.1-A); §1
  Name loses the "WIRED for single-invocation transactional
  atomicity" claim; §3 Description drops the "wraps the whole
  invocation in one pdxfs_txn_begin/commit/abort triple" paragraph
  and adds an "atomicity status" paragraph that names the current
  behaviour honestly and points at paideia-os/cp#35; §6
  Capabilities drops the `KIND_IPC_ENDPOINT (invoke)` and
  `KIND_PDXFS_TXN (invoke)` lines and the elevate-retry paragraph;
  §8 Audit + undo drops the libpdx-audit and undo-record
  paragraphs and states plainly that cp v1.1 writes no audit
  record and persists no undo record.
- `design/pdxfs-notes.md` — new §5c v1.1-C addendum records the
  full R90 kernel substrate landing (sysnos 70 / 104 / 105 / 107
  with body-handler paths), states plainly that v1.1-C is doc-only
  and does not wire cp onto the substrate, and points at paideia-
  os/cp#35 (cp.ENH-011) for the substrate half of paideia-os/cp#23.
- `manifest.pdxproj` — `version = 1.1.0-B` -> `version = 1.1.0-C`;
  header comment describes the doc-only shape.

### Not changed

- `src/pdxfs.pdx` — six real trampolines only (open / read / write
  / close / stat / getcwd). No TXN trampolines were added at
  v1.1-C; they land under paideia-os/cp#35 alongside the
  substrate re-wire.
- `src/dispatch.pdx` — v1.1-A's `dispatch_copy` two-step body (pos-
  count gate + `copy_bytes_only` call) is unchanged. No outer TXN
  wrap was re-introduced at v1.1-C.
- `caps.decl` — v1.1-A's three-cap `requires:` block (`KIND_USER`,
  `KIND_PDXFS_FILE (read, <src>)`, `KIND_PDXFS_FILE (write,
  <dst-parent>)`) is unchanged. `KIND_PDXFS_TXN (invoke)` and a
  new `KIND_PDXFS_VOL` for `pdxfs_txn_open`'s volume argument land
  under paideia-os/cp#35.
- All source `.pdx` files, `manifest.pdxsig`, `tests/*.md`, and
  every other file not named in "Changed" are byte-identical to
  v1.1-B.

### Follow-up

- paideia-os/cp#35 (cp.ENH-011) — re-wire the outer TXN against
  the R90 kernel substrate that has since landed (sysnos 70 / 104
  / 105 / 107). Requires: (a) a `KIND_PDXFS_VOL` cap grant added
  to `caps.decl` and threaded through the InitCap sidecar for
  `pdxfs_txn_open(vol_slot, flags)`; (b) per-write pre-image
  snapshot marshaling through `pdxfs_undo_write(cap_slot, inode_no,
  offset, len, kbuf_ptr)` with per-row cap awareness (32 records /
  4 KiB per the R90 substrate — a large-file copy needs commit-
  and-reopen batching); (c) commit-on-success and abort-on-error
  plumbing on every exit path out of `copy_bytes_only`; (d) a new
  test row in `tests/M4-001-txn-abort.md` shape that exercises
  the abort path under a forced mid-copy failure.

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
