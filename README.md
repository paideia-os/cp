# cp

paideia-os copy — atomic single-TXN file / directory-tree copy with
cap-tail re-signing, audit-first journaling, semantic-pipe progress
records, PdxFS undo on overwrite, and cross-subtree elevate for
widening destination-parent writes.

Install with `pkg install cp` (pulls the dual-signed 1.0.0 release from
`pkgs.paideia-os/main/cp/1.0.0/`; `pkg` verifies both the author and the
`paideia_root` signature before writing to `/system/packages/cp/`).
Long-form documentation renders with `doc cp` from `doc/cp.pdxdoc`.

## Synopsis

```
cp [--flag ...] [-flag ...] <src> <dst>
cp -r <src-dir> <dst-dir>
```

Exactly two positional arguments are required. `Dispatch::dispatch_copy`
gates on `ParsedArgs::pos_count == 2`; every other count (0, 1, ≥3) is a
usage error. Short flags are one-per-hyphen — `-rv` is rejected by
libpdx-argv as `ERR_CLUSTERED_SHORT` and surfaces as a parse error.

## Description

cp streams bytes from `<src>` to `<dst>` through a 4096-byte `.bss`
scratch buffer (`CopyFile::copy_buf`), one `pdxfs_read` and one
`pdxfs_write` per iteration until source EOF. The whole invocation —
single-file or recursive — runs inside **one** `KIND_PDXFS_TXN`. Since
M2-004 the transaction is opened at dispatch level, not per file:
`dispatch_copy` calls `pdxfs_txn_begin` before routing, `pdxfs_txn_commit`
when the body returns 0, and `pdxfs_txn_abort` on any non-zero body
return, so a multi-file `-r` copy rolls back as a single unit. After a
successful copy the body calls `SignedInode::preserve_at_destination` to
re-sign the destination inode's cap tail; `SI_OK` and
`SI_DEGRADED_KEY_LOCKED` are both success-continuation codes — the bytes
land either way, only the signature differs.

Every invocation runs under the baseline capability row declared in
`caps.decl`: `KIND_USER (self)`, `KIND_IPC_ENDPOINT (invoke)`,
`KIND_PDXFS_FILE (read, <src>)`, `KIND_PDXFS_FILE (write, <dst-parent>)`,
`KIND_PDXFS_TXN (invoke)`. The loader's InitCap sidecar narrows the two
file caps to the argument path prefixes at exec, so cp cannot read or
write outside the source and destination-parent subtrees named on argv.
`KIND_ELEVATE_CHANNEL` is deliberately **not** ambient — it is a
commented placeholder in `caps.decl`. When the destination-parent open
fails, `CopyFile::copy_bytes_only` calls
`CpElevate::cp_elevate_try_dst_parent`, which requests a 60-second
widening window (`CP_ELEVATE_DUR_NS = 60000000000`) through
`ElevateClient::elevate_client_request` and retries the open exactly once
on approval.

**Undo record (I5).** The invariant cited by `src/undo.pdx` is
`design/tooling/plan.md` I5: *destructive operations are reversible* —
`undo cp <same-args>` restores the pre-copy destination.
`design/architecture.md` records that D5 later widened I5 from "destructive
ops are reversible" to "every operation is discoverable". cp's mechanism
is `Undo::maybe_record_over_existing(dst_ptr)`, called once per
destination from `copy_bytes_only` between the source open and the
destination open — i.e. strictly *before* any byte can be overwritten. It
short-circuits on two gates: `CpFlags::flag_over_existing_seen == 0` (the
flag was not passed) or `Pdxfs::pdxfs_stat(dst) != 0` (nothing exists to
journal). Only when both pass does it call the PdxFS v1 journal write.

**Destructive-op audit status.** cp is audit-first (D3): `dispatch_copy`
calls `CpAudit::cp_audit_begin_wrap` at its very top, before the
positional-count gate that can print to stderr, so the invoke record
lands before any user-visible output; `cp_audit_commit_wrap(exit_code)`
fires at the shared epilogue on every exit path. Two substrate gates are
open at 1.0 and are worth knowing before relying on reversibility:
`sys_pdxfs_undo_write` is not yet exposed, so `undo_journal_write_stub`
returns `UNDO_OK` without writing; and `pdxfs_stat` is an M2 stub that
returns `-2` (ENOENT) unconditionally, so the second undo gate never
passes today and `--over-existing` performs the copy but journals no undo
record. Both are body-edit landing sites awaiting the paideia-os R42
substrate expansion — the call shapes are wired end to end. Likewise the
libpdx-audit broker is a stub, so audit records are shaped but not yet
durably journaled.

## Options

All five flags cp reads are `FKIND_BOOL` (no argument). `-r`, `-v` and
`--over-existing` are registered by `CpFlags::register_cp_flags`;
`--dry-run` and `--verbose` come from libpdx-argv's `StdVocab`.
`CpFlags::populate` mirrors exactly these five IDs into `.bss` slots that
every downstream module reads.

| Flag | ID | Arg | Default | Description |
|------|----|-----|---------|-------------|
| `-r` | `CP_ID_RECURSIVE` = 100 | none | off | Recursively copy a directory tree. Routes `dispatch_copy` to `Walk::walk_recursive` instead of `CopyFile::copy_bytes_only`; both run inside the same outer TXN. |
| `-v` | `CP_ID_VERBOSE` = 101 | none | off | Enable the two stderr diagnostics gated on `flag_verbose_seen`: `cp: -r walk entering` at walk entry, and `cp: destination cap-tail unsigned (key locked)` on signed-inode degrade. Does **not** gate the progress records — those emit unconditionally. |
| `--verbose` | `STD_ID_VERBOSE` = 6 | none | off | Long alias for `-v`. `populate` ORs both seen-bits into `flag_verbose_seen`, so the two forms are indistinguishable downstream. |
| `--over-existing` | `CP_ID_OVER_EXISTING` = 102 | none | off | Arm the undo-record gate: write a PdxFS v1 undo record before overwriting an existing destination. See [Audit records](#audit-records). |
| `--dry-run` | `STD_ID_DRY_RUN` = 3 | none | off | *(implemented: parser only — `CpFlags::populate` fills `flag_dry_run` / `flag_dry_run_seen`, but no module in `src/` reads either slot.)* |

The remaining `StdVocab` flags (`--help`, `--version`, `--json`,
`--schema`, `--quiet`, `--color`, `--no-cap`, `--pdx-schema`) are
registered by `StdVocab::register_all` and therefore parse successfully,
but `CpFlags::populate` queries only the five IDs above — no cp module
reads them at 1.0.

Note on overwrite: `doc/cp.pdxdoc` §9 describes refusing an existing
destination unless `--over-existing` is present. The current body does
not implement that gate — `copy_bytes_only` opens the destination with
`O_WRONLY | O_CREAT` (`0x41`) on every invocation.

## Semantic pipe output

`CpMain::cp_main` binds fd 1 to `CopyProgressRecord@0.1` via
`Binding::libpdx_semantic_pipe_bind` before dispatching. The bind is
idempotent for the same `(fd, hash)` pair, and its return code is
deliberately ignored: a failed bind bumps `pipe_bind_errors` and the emit
path degrades to `SP_SEND_ERR_NOT_BOUND`, so cp still copies bytes.

`CpPipe::cp_pipe_emit_progress` marshals a fixed 40-byte record into
`cp_progress_rec` and sends it with `Send::send_record(1, &rec, 40)` after
each successful copy.

| Offset | Field | Type | Notes |
|--------|-------|------|-------|
| 0 | `src_path` | `u64` | VA of the NUL-terminated source path |
| 8 | `dst_path` | `u64` | VA of the NUL-terminated destination path |
| 16 | `bytes_copied` | `u64` | Bytes actually written to the destination |
| 24 | `bytes_total` | `u64` | Expected byte count; equals `bytes_copied` at 1.0 |
| 32 | `elapsed_ns` | `u64` | Placeholder `0` until a ring-3 `KIND_TIMER` read lands |

`COPY_PROGRESS_SCHEMA_HASH` is a 32-byte slot holding the deterministic
placeholder `"CopyProgressRecord@0.1"` followed by 10 NUL bytes; the
`svc.schema-registry` lookup replaces it with the canonical
BLAKE3-truncated digest once that service lands. Emit success bumps
`pipe_emits`; failure bumps `pipe_emit_errors` and never fails the copy.

## Exit codes

Defined as `Dispatch::EXIT_*` in `src/dispatch.pdx`.

| Code | Name | Meaning | Emitted today? |
|------|------|---------|----------------|
| 0 | `EXIT_OK` | Copy committed. | Yes — body returned 0 and `pdxfs_txn_commit` succeeded. |
| 1 | `EXIT_OP_FAIL` | Operation failed: open-src, open-dst, read, write, short write, `mkdir`/`opendir`/`readdir` under `-r`, TXN begin failure, TXN commit failure. | Yes. |
| 2 | `EXIT_USAGE` | argv parse error (from `cp_main`) or `pos_count != 2` (from `dispatch_copy`). | Yes. |
| 3 | `EXIT_NOT_YET_IMPL` | Reserved: "vocabulary recognised, body not wired". | No — no 1.0 code path returns it. |
| 4 | `EXIT_CAP_DENIED` | Reserved: cap denied / elevate refused. | No — an elevate refusal currently falls through to the ordinary open-fail path and exits 1. |

Codes 3 and 4 are kept distinct so a script can tell "not implemented"
and "missing capability" apart from a generic operation failure.

## Capabilities

The entry point's effect and capability annotation, verbatim from
`src/main.pdx`:

```
pub let cp_main : (u64, u64) -> u64 !{mem, sysreg} @{}
```

The widest annotations reached inside the process are the libpdx-audit
wrappers and the semantic-pipe emit:

```
pub let cp_audit_begin_wrap   : ()    -> u64 !{mem, sysreg} @{cap, sched}
pub let cp_audit_commit_wrap  : (u64) -> u64 !{mem, sysreg} @{cap, sched}
pub let cp_pipe_emit_progress : (u64, u64, u64, u64) -> u64 !{mem, sysreg} @{cap}
```

The `requires:` block of `caps.decl`:

```
- KIND_USER (self)
- KIND_IPC_ENDPOINT (invoke)
- KIND_PDXFS_FILE (read, <src>)
- KIND_PDXFS_FILE (write, <dst-parent>)
- KIND_PDXFS_TXN (invoke)
```

## Examples

Single file. Silent on success; one `CopyProgressRecord` goes to fd 1.

```
$ cp /home/alice/notes.md /tmp/notes.md
$ echo $?
0
```

Wrong positional count — the pre-TXN usage gate, so no transaction is
opened.

```
$ cp
cp: usage error (expected exactly two positional arguments)
$ echo $?
2
```

Clustered shorts are rejected by libpdx-argv before dispatch runs.

```
$ cp -rv /src /dst
cp: argv parse error (run 'cp --help')
$ echo $?
2
```

Verbose surfaces the cap-tail degrade. The bytes still land; only the
destination inode's signature is missing.

```
$ cp -v /home/alice/report.pdf /home/bob/inbox/report.pdf
cp: destination cap-tail unsigned (key locked)
$ echo $?
0
```

Arm the undo gate before an overwrite. Once the R42 `sys_pdxfs_stat` /
`sys_pdxfs_undo_write` pair lands, this is the invocation `undo cp` can
reverse.

```
$ cp --over-existing /new/config /etc/paideia/config
$ echo $?
0
```

## Audit records

**CopyRecord (libpdx-audit).** `CpAudit` wraps the whole dispatch. Fields
cp supplies:

| Field | Type | Source | Value |
|-------|------|--------|-------|
| `op_name` | `[u8; 3]` | `CP_OP_NAME_STR` | `"cp\0"` |
| `op_args` | `[u8; 15]` | `CP_OP_ARGS_STR` | `"cp <src> <dst>\0"` — a fixed canonical shape, not a per-invocation argv render |
| `audit_id` | `u64` | `cp_audit_id` (`.bss` singleton) | Returned by `AuditClient::audit_begin`; `0` means broker-unavailable |
| `exit_code` | `u64` | `cp_audit_commit_wrap` argument | cp's own exit code, 0–4 |

`cp_audit_begin_wrap` fires `UEJ_KIND_TOOL_INVOKE` (130) at dispatch
entry; `cp_audit_commit_wrap` fires `UEJ_KIND_TOOL_EXIT` (133) at the
epilogue, and is a no-op when `cp_audit_id == 0`. cp never branches on the
commit return code — a broker failure does not change the exit code.
`AUDIT_BEGIN_FAIL_MSG` and `AUDIT_COMMIT_FAIL_MSG` are reserved `.rodata`
slots, unused at 1.0.

**Undo record (PdxFS v1).** `Undo::maybe_record_over_existing` is the
gate; `Undo::undo_journal_write_stub(dst_ptr)` is the single sink that
flips to `sys_pdxfs_undo_write` when the substrate lands. Observable
state, all `u64` unless noted:

| Symbol | Type | Meaning |
|--------|------|---------|
| `UNDO_OK` | `u64` = 0 | No-op *or* undo captured — a success-continuation code |
| `UNDO_WRITE_ERR` | `u64` = `0xFFFFFFFE` | Journal write failed; reserved, never returned by the current stub |
| `undo_stat_buf` | `[u8; 144]` | Statbuf destination for `pdxfs_stat` |
| `undo_gate_calls` | `u64` | Total gate invocations (one per destination file) |
| `undo_gate_noflag` | `u64` | Skipped: `--over-existing` absent |
| `undo_gate_noexist` | `u64` | Skipped: `pdxfs_stat` reported the destination absent |
| `undo_records_wrote` | `u64` | Journal writes attempted |

## See also

- [libpdx-argv](https://github.com/paideia-os/libpdx-argv) — flag
  registration and argv parse (`FlagSpec`, `StdVocab`, `Parser`,
  `ParsedArgs`).
- [libpdx-audit](https://github.com/paideia-os/libpdx-audit) —
  `audit_begin` / `audit_commit` behind `CpAudit`.
- [libpdx-semantic-pipe](https://github.com/paideia-os/libpdx-semantic-pipe)
  — schema bind and `send_record` behind `CpPipe`.
- [libpdx-elevate](https://github.com/paideia-os/libpdx-elevate) — the
  cross-subtree cap-widening hop behind `CpElevate`.
- [mv](https://github.com/paideia-os/mv) — move / rename; shares the
  signed-inode logic.
- [rm](https://github.com/paideia-os/rm) — remove; shares the
  undo-record discipline.
- [undo](https://github.com/paideia-os/undo) — replays the PdxFS undo
  graph that `--over-existing` feeds.

Per-milestone state lives in `STATUS.md`, per-issue release notes in
`CHANGELOG.md`, and the internal module map in `design/architecture.md`.

## License

MIT — see LICENSE.
