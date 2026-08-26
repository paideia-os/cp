# cp — enhancement plan (post-1.0.0 audit)

**Date:** 2026-08-25
**Repo:** github.com/paideia-os/cp @ 32f9cc4 (tag v1.0.0)
**Scope:** combined osarch + softarch audit of what cp *claims* versus
what `src/` actually does, and against what a paideia-os user needs at
monorepo HEAD. Every claim below is grep-verified against this tree and
against `/home/snunez/Development/PaideiaOS` at HEAD. No issue in this
plan is speculative.

Companion doc-set: `README.md` (already refreshed by the R-wave
source-verified README pass and materially honest), `doc/cp.pdxdoc`
(**not** refreshed — the primary source of aspirational cruft),
`design/architecture.md` + `design/argv-surface.md` +
`design/pdxfs-notes.md` (M1-era, never revised).

---

## 1. Current state

cp is 12 `.pdx` modules / 3426 lines. The dispatch chain is real and
runs end to end for the single-file case:

```
Main::cp_main
  → FlagSpec/StdVocab/Flags::register_cp_flags   (libpdx-argv)
  → Parser::parse_argv                           (libpdx-argv)
  → Pipe::cp_pipe_bind_stdout                    (libpdx-semantic-pipe)
  → Dispatch::dispatch_copy
      → Audit::cp_audit_begin_wrap               (libpdx-audit)
      → Flags::flags_reset + Flags::populate
      → pos_count == 2 gate
      → Pdxfs::pdxfs_txn_begin                   (STUB — returns 0)
      → Copy::copy_bytes_only  |  Walk::walk_recursive
      → Pdxfs::pdxfs_txn_commit / _abort         (STUBS — return 0)
      → Audit::cp_audit_commit_wrap              (libpdx-audit)
```

`Copy::copy_bytes_only` genuinely opens, streams through a 4 KiB `.bss`
buffer, closes, and returns. That much works.

### 1.1 Library linkage — verified, not assumed

`manifest.pdxproj` `deps:` names four libraries. Grepping the actual
`call` targets in `src/*.pdx`:

| Library | Claimed | Real call sites in `src/` | Verdict |
|---|---|---|---|
| libpdx-argv | yes | `reset`, `register_all`, `register`, `parse_argv`, `find_flag_by_id` | **real** |
| libpdx-semantic-pipe | yes | `libpdx_semantic_pipe_bind`, `send_record` | **real** |
| libpdx-audit | yes | `audit_begin`, `audit_commit` | **real** |
| libpdx-elevate | yes | `elevate_client_request` | **real** |
| libpdx-cap | listed in `README.md` "See also" and `doc/cp.pdxdoc` `@xref` | **none** — `src/signed_inode.pdx` names `cap_sign_inode` only inside a comment | **overstated** |

cp is therefore one of the better-linked tools in the org: four of five
claimed libraries are genuinely called. The libpdx-cap claim is the same
overstatement pattern found org-wide and is doc-only (no manifest dep to
remove).

### 1.2 Stack discipline — audited, clean

Every function's push/pop parity and `rsp % 16` arithmetic was walked by
hand. `Copy::copy_bytes_only` (5-push/5-pop), `Walk::walk_recursive`
(5/5), `Dispatch::dispatch_copy` (2-push + `sub rsp,8`),
`Flags::populate` (2 + pad), `Undo::maybe_record_over_existing` (2 +
pad), `Pipe::cp_pipe_emit_progress` (6 + pad), `Elevate::
cp_elevate_try_dst_parent` (3, no pad — correct: entry `rsp%16==8`,
minus 24 → 0), `SignedInode::preserve_at_destination` (1/1) all balance
on every branch, and every error branch routes through the shared
epilogue. **cp has no analogue of the mv `move.pdx:741` stack
imbalance.** The one cosmetic defect is the garbled "Correction:" prose
inside `cp_pipe_emit_progress`'s justification string — the arithmetic it
finally states is right.

---

## 2. Gap: documentation vs source

### 2.1 `doc/cp.pdxdoc` — nine verified mismatches

The `.pdxdoc` was written at M5 against the *intended* 1.0 and never
reconciled with the shipped body. Each of these is grep-provable:

1. **§3 / §9** — "Without `--over-existing`, an existing destination is
   treated as an error (exit 1)". `copy.pdx:264` opens the destination
   `O_WRONLY | O_CREAT` (`0x41`) unconditionally. There is no
   existing-destination gate anywhere in `src/`.
2. **§4.1** — "`-v` Emit CopyProgressRecord[] per file on stdout."
   `Copy::copy_bytes_only` calls `cp_pipe_emit_progress`
   unconditionally; `flag_verbose_seen` gates only two *stderr* strings
   (`Walk::VERBOSE_ENTER_MSG`, `SignedInode::SI_DEGRADE_MSG`).
3. **§4.1** — "`--dry-run` Preview the copy without writing to dst."
   See §2.3 — the flag is parsed, mirrored, and read by nobody.
4. **§5** — "At 1.0, `-r` at pos_count > 2 is the last surviving exit-3
   site". No code path in `src/` writes `rax = 3`; the `-r` arm now
   calls `Walk::walk_recursive` and the `pos_count != 2` arm returns 2.
   Exit 3 is unreachable.
5. **§6** — "On refuse, cp exits 4 with a stderr diagnostic naming the
   missing cap." `copy.pdx:281` jumps a refusal to `copy_open_dst_fail`,
   which prints `OPEN_DST_FAIL_MSG` and returns **1**. `EXIT_CAP_DENIED`
   is defined and never assigned.
6. **§7** — the record "nam[es] the source path, destination path, byte
   count copied, elapsed wall-clock, and the cap-tail preserve outcome".
   `Pipe::cp_progress_rec` is 40 bytes / 5 `u64`s: src, dst,
   bytes_copied, bytes_total, elapsed_ns. There is no preserve-outcome
   field, and `elapsed_ns` is a hardcoded `0`.
7. **§9** — "`--quiet` suppresses the records". `Flags::populate` never
   queries `STD_ID_QUIET`; no module reads it.
8. **§10** — the `--dry-run` example prints `cp: would copy 214 files
   (127.4 MiB) into /dst`, and the elevate example prints `cp:
   requesting KIND_PDXFS_FILE(write, …)` / `cp: approved (auto-policy:
   …)`. None of these strings exist in any `.rodata` slot in `src/`.
   §10 also shows `cap-tail preserve DEGRADED_CROSS_USER`; the only
   degrade string in the tree is `SI_DEGRADE_MSG` = `"cp: destination
   cap-tail unsigned (key locked)\n"`.
9. **§4.2** — "`--version` Print `cp 1.0.0 (build <hash>) sig
   <fingerprint>`". No version string is compiled into the binary.

`README.md` already documents (4), (5) and part of (1) honestly. The
`.pdxdoc` — the artefact a *user* reads via `doc cp` — does not.

### 2.2 Stale module names after the PascalCase rename

Commit `0f6f8ff` ("M0305 module name must match file basename") renamed
every module to its file basename. `manifest.pdxproj` was not updated:

```
manifest.pdxproj:  entry = CpMain::cp_main
src/main.pdx:67:   module Main = structure {
```

`CpMain` no longer exists. The same stale names (`CpMain`, `CopyFile`,
`CpFlags`, `CpPipe`, `CpAudit`, `CpElevate`) persist throughout
`README.md`, `doc/cp.pdxdoc`, `design/architecture.md`, `STATUS.md`,
`CHANGELOG.md` and every in-source header comment, so the docs are not
greppable back to the code.

### 2.3 `--dry-run` is parsed and ignored

`Flags::populate` queries `STD_ID_DRY_RUN` (3) and writes
`flag_dry_run` / `flag_dry_run_seen`. `grep -rn flag_dry_run src/` finds
**zero readers outside `flags.pdx`**. `cp --dry-run a b` performs a
real, complete copy. Of every unimplemented flag in this repo this is the
only one whose failure mode is *doing the destructive thing the user
explicitly asked it not to do*.

---

## 3. Gap vs what real paideia-os users need at HEAD

This is where cp's 1.0 diverges most sharply. The repo's substrate
assumptions were frozen at 2026-08-21 and the monorepo has moved.

### 3.1 The "R42 substrate" cp is waiting for has landed

`src/pdxfs.pdx` carries four stubs, each justified as "the R42 substrate
expansion has not exposed this syscall":

| cp stub | returns | kernel reality at HEAD |
|---|---|---|
| `pdxfs_stat` | `-2` (ENOENT) always | **sysno 77 `sys_stat`** — `dispatch.pdx:511` |
| `pdxfs_readdir` | `0` (EOF) always | **sysno 78 `sys_getdents`** — `dispatch.pdx:528` |
| `pdxfs_mkdir` | `0` (success) always | **sysno 79 `sys_mkdir`** — `dispatch.pdx:355` |
| `pdxfs_txn_begin/commit/abort` | `0` always | **sysno 70 `sys_pdxfs_txn_open`** — `dispatch.pdx:472` (open only; no commit/abort ordinal yet) |

The R56.M3-001 VFS metadata block (#1790, sysnos 77–82) landed *after*
cp v1.0.0 shipped. cp is a body-edit away from three working features it
currently disables at the trampoline.

Consequences today:

- **`cp -r <src> <dst>` exits 0 having copied nothing.** `pdxfs_mkdir`
  fakes success, `pdxfs_readdir` returns EOF on the first call, so
  `walk_recursive` runs mkdir→open→readdir→close→`return 0`. The
  per-entry body (`walk_entry_todo`, `walk.pdx:235-264`) is a comment
  block. A user script shaped `cp -r a b && rm -r a` **destroys data
  silently** — cp reports complete success.
- **`--over-existing` never journals.** `pdxfs_stat` returns `-2`, so
  `Undo::maybe_record_over_existing` always takes the
  `undo_gate_noexist` arm. The undo record I5 promises does not exist,
  while `doc/cp.pdxdoc` §8 states flatly that `undo cp <same-args>`
  restores the pre-copy bytes.

### 3.2 The atomicity guarantee is not implemented at all

This is cp's headline claim and the reason it holds a `KIND_PDXFS_TXN`
cap. `doc/cp.pdxdoc` §3:

> every write is either fully visible at the destination or fully rolled
> back. Partial-copy states are not visible to any concurrent reader; a
> mid-copy crash … leaves the destination byte-identical to its pre-copy
> state.

`Pdxfs::pdxfs_txn_begin`, `_commit` and `_abort` each `xor rax, rax;
ret`. No syscall is issued. `Dispatch::dispatch_copy`'s
begin/commit/abort scaffolding is real and correctly placed, but it wraps
three no-ops. A mid-copy failure today leaves exactly the partial
destination the doc promises is impossible — and `copy_bytes_only`'s
write loop has already committed those bytes to the vnode.

### 3.3 Overwrite corrupts: no `O_TRUNC`

`copy.pdx:264` opens the destination with `0x41` = `O_WRONLY | O_CREAT`.
`O_TRUNC` (`0x80`) is defined in `src/pdxfs.pdx:103` and never passed.
Copying a 100-byte source over an existing 5000-byte destination writes
100 bytes and leaves 4900 bytes of the old file in place. The
destination is neither the source nor the original — it is corrupt, and
cp exits 0.

Kernel side: `vfs_open.pdx:16` and `:37` still mark `O_TRUNC` "deferred
to R16.M2". This needs a monorepo companion.

### 3.4 cwd-relative paths — the *source* works, the *destination* does not

R86.M1-005 (#1958) rewired `vfs_open` to resolve against the caller's
`TASK_OFF_CWD` (+160) instead of a hardcoded `cwd = 0`, so cp's plain
`pdxfs_open(argv_ptr, flags)` gets relative resolution for free. cp needs
no `sys_getcwd`/`sys_chdir` calls of its own for the read path, and
**`cp foo/bar.txt /tmp/x` resolves `foo/bar.txt` correctly today.** That
part of the R86 gap does not apply to cp.

But the *create* path does not have parity. `vfs_open.pdx` Phase 4
(O_CREAT, target does not exist) scans the path for `'/'` and:

```
vfs_open_creat_scanned:
        cmp r9, 0;                              // did we find a '/'?
        je  vfs_open_fail;                      // no '/' → no cwd support, fail
```

A destination with no slash has no parent-directory substring to resolve,
so the create is refused outright. cp always opens the destination
`O_CREAT`, therefore:

```
$ cd /home/alice
$ cp notes.md backup.md
cp: cannot open destination file
$ echo $?
1
```

This is the single most common cp invocation there is, and it fails. The
fix is kernel-side (when no slash is found, anchor the parent at
`TASK_OFF_CWD` rather than failing), but cp is the tool that surfaces it
and cp owns the fingerprint. This is the concrete form the "cwd
relative-path gap" takes for cp — narrower than for rm/mv, but real.

### 3.5 Missing safety gate: `cp a a`

`Dispatch::dispatch_copy` validates only `pos_count == 2`. Nothing
compares src against dst. `cp a a` opens `a` read-only, opens `a`
write-only-with-create against the same vnode, and enters the
read/write loop — with no `O_TRUNC` and two independent file offsets on
the same inode. There is no refusal and no diagnostic. POSIX cp refuses
this outright. The recursive analogue (`cp -r a a/sub`) becomes
unbounded once §3.1's walk body lands.

### 3.6 Uncalled resets and dead code

`Copy::copy_single_file` (the M1-compat forwarder) has no caller.
`Undo::undo_reset`, `SignedInode::signed_inode_reset` and
`Walk::walk_reset` are defined and never called from `main.pdx` or
`dispatch.pdx` — benign while `.bss` is load-zeroed and one process is
one copy, but they are exactly the guarantees a future multi-copy shape
would rely on. `Dispatch::RECURSIVE_NOT_YET_MSG`,
`Dispatch::COPY_STUB_MSG`, and `Copy::TXN_BEGIN_FAIL_MSG` /
`Copy::TXN_COMMIT_FAIL_MSG` are dead `.rodata` with no emitter.

### 3.7 `caps.decl` under-declares the elevate channel

`caps.decl:642` keeps `KIND_ELEVATE_CHANNEL (invoke, svc.elevate-broker)`
commented out with an M1-era note that "the cross-subtree path refuses at
the dispatch layer". That is no longer true: M3-003 landed
`src/elevate.pdx`, `manifest.pdxproj` carries `libpdx-elevate @ ^0.1` as
a hard dep, and `copy.pdx:279` calls `cp_elevate_try_dst_parent` on every
destination-open failure. Once libpdx-cap's `OK | MISSING | EXTRA`
manifest compare is enforced at exec, cp's declared cap set will not
cover the broker hop it actually attempts.

### 3.8 `call reset` is ambiguous in `main.pdx`

```
main.pdx:124:  call reset;          // comment: FlagSpec::reset
main.pdx:130:  call reset;          // comment: ParsedArgs::reset
```

Since `7d257b5` ("call statements can't take `Module::` qualified
paths"), `call` takes a bare label. Two distinct libpdx-argv functions
are both spelled `reset`, so at most one symbol can bind. Whichever loses
is never invoked: if `FlagSpec::reset` is dropped the spec table is not
cleared before three `register` calls; if `ParsedArgs::reset` is dropped
the parse state is not zeroed. Harmless on a load-zeroed first run,
latent otherwise, and unreviewable as written.

---

## 4. Issue plan

Filed into milestone **Enhancement v1.1 — cp**. Ordering is dependency-
first: the substrate flip (ENH-001) unblocks the two features that are
currently silent no-ops.

| # | GH | Title | Effort | Deps |
|---|---|---|---|---|
| ENH-001 | #17 | Flip pdxfs stat/getdents/mkdir stubs to real syscalls 77/78/79 | M | none |
| ENH-002 | #18 | `cp -r` fails loud instead of exiting 0 having copied nothing | XS | none |
| ENH-003 | #19 | Land the recursive per-entry walk body | L | #17, #18 |
| ENH-004 | #20 | Destination open lacks O_TRUNC — overwrite leaves stale tail bytes | S | none |
| ENH-005 | #21 | `cp <src> <bare-name>` fails: O_CREAT refuses slash-less paths | S | none |
| ENH-006 | #22 | `--dry-run` is parsed and ignored — performs a real copy | S | none |
| ENH-007 | #23 | TXN begin/commit/abort are no-ops while docs assert atomicity | M | none |
| ENH-008 | #24 | Refuse `cp a a` and dst-under-src | S | none |
| ENH-009 | #25 | manifest entry symbol `CpMain::cp_main` no longer exists | XS | none |
| ENH-010 | #26 | `call reset` binds one symbol for two libpdx-argv functions | XS | none |
| ENH-011 | #27 | `caps.decl` omits KIND_ELEVATE_CHANNEL though elevate.pdx calls it | XS | none |
| ENH-012 | #28 | Elevate refusal exits 1; exit 3/4 contract is fiction | S | #27 |
| ENH-013 | #29 | `doc/cp.pdxdoc` aspirational-cruft sweep (nine mismatches) | S | #20, #22, #23, #28 |
| ENH-014 | #30 | Dead code + uncalled resets sweep | XS | #18 |

Milestone: **Enhancement v1.1 — cp** (milestone 6).

### Monorepo companions (flagged, not filed here)

Per the coordinating pass, these belong to paideia-os and are **not**
filed by this repo:

- **`vfs_open` O_CREAT with no `'/'` must anchor the parent at
  `TASK_OFF_CWD`** rather than `je vfs_open_fail`
  (`src/kernel/core/fs/vfs_open.pdx`, the `vfs_open_creat_scanned`
  label). Blocks cp.ENH-005, and identically blocks any tool that
  creates a file by bare name.
- **Implement `O_TRUNC` (0x80) in `vfs_open`** — still marked "deferred
  to R16.M2" at `vfs_open.pdx:16`/`:37`. Blocks cp.ENH-004.
- **Expose `sys_pdxfs_txn_commit` / `sys_pdxfs_txn_abort`** — only
  `sys_pdxfs_txn_open` (sysno 70) exists. Blocks cp.ENH-007's real fix;
  ENH-007's honesty half can land without it.
- **`sys_pdxfs_undo_write`** — the PdxFS v1 undo-journal sink cp's
  `Undo::undo_journal_write_stub` is the single call site for. Blocks
  the I5 reversibility claim.

---

## 5. Verdict on the v1.0.0 tag

**Not defensible as shipped. Recommend walking the tag back to
0.9.0-rc or re-cutting 1.0 only after ENH-001/002/003/004/007 land.**

The single-file byte-copy path is genuine, the module decomposition is
clean, the stack discipline is correct throughout, and four of five
claimed libraries are really called — this is competent work. But a
tool tagged 1.0 should not have all three of:

- `cp -r` exiting **0** while copying nothing (§3.1) — an active
  data-loss hazard for the `cp -r … && rm -r …` idiom;
- an overwrite path that leaves the destination **corrupt** rather than
  either old or new (§3.3);
- an **atomicity guarantee stated as fact** in the user-facing man page
  and implemented as three `xor rax, rax; ret` (§3.2).

Any one of those is a 0.x defect. Together they mean the shipped
artefact does not do the thing its name and its documentation promise.
The honest framing is that cp reached a well-structured M3 and the M5
release milestone signed a version number onto it before the substrate
it depends on had landed. That substrate has since landed; the fix is
mostly body-edits at known call sites, exactly as the M1/M2 stub
discipline intended. The tag should follow the capability, not lead it.
