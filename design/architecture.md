# cp — architecture

**Wave:** R50 (Wave 2)
**Repo:** github.com/paideia-os/cp
**Upstream design:** `design/tooling/r49-r50-plan.md` §5.6 in
[paideia-os](https://github.com/paideia-os/paideia-os).

This document describes the internal shape of cp. It does not repeat
the wave-level rationale from the paideia-os plan doc; read that first
for the D3 audit-first upgrade, the D5 undo-first-order contract (I5
upgraded from "destructive ops are reversible" to "every operation is
discoverable"), and for the KIND_PDXFS_TXN transaction model M2 will
lift out of stub form.

## 1. Milestone position

M1 lands the frame: the argv surface, the positional-count validator,
the single-file copy body, the paideia-as build manifest, and the
caps.decl manifest. It does **not** land recursion, cap-tail
preservation, cross-subtree elevate, audit journaling, semantic-pipe
records, or the PdxFS undo record. Those all land at M2 or M3.

The M1 first-runnable shape is one command: `cp src dst` with exactly
two positional arguments and no `-r` flag. The invocation walks the
full dispatch chain (argv → parse → recognizer → dispatch → copy body
→ exit) end-to-end. The copy body opens the source read-only, opens
the destination with O_CREAT, wraps both opens between a
`pdxfs_txn_begin` / `pdxfs_txn_commit` pair, and streams bytes through
a static .bss buffer until source EOF. On success cp is silent (POSIX-
canonical); on error cp prints a one-line diagnostic on stderr and
returns a non-zero exit code.

The three M1 issues are #1 (scaffold + caps.decl + build manifest +
this architecture doc), #2 (argv surface + main entry + print helper +
argv-surface design doc), #3 (dispatch + copy body + pdxfs syscall
wrappers + first-runnable single-file copy).

## 2. Public surface

M1 has one public entry point, exported by `src/main.pdx`:

```
pub let cp_main : (u64, u64) -> u64 !{mem, sysreg} @{}
```

`cp_main(argc, argv) -> exit_code`. The loader-supplied `_start` stub
reads argc/argv from the stack per the SysV/x86_64 convention and
calls `cp_main` with them; `cp_main`'s return value is the process
exit code passed to sys_exit (paideia-os syscall #60). The rest of the
public surface is the flag + positional grammar described in
`design/argv-surface.md`.

The internal module layout is:

| Module      | File                | Responsibility                                                       |
|-------------|---------------------|----------------------------------------------------------------------|
| `CpMain`    | `src/main.pdx`      | `cp_main` entry; hands off to dispatch                               |
| `Dispatch`  | `src/dispatch.pdx`  | flag gate (`-r`, cross-subtree) + positional-count validate + route  |
| `CopyFile`  | `src/copy.pdx`      | single-file copy body (open + txn + read/write loop + close)         |
| `Pdxfs`     | `src/pdxfs.pdx`     | sys_open / sys_read / sys_write / sys_close + txn stubs              |
| `Print`     | `src/print.pdx`     | `sys_write(fd=1|2)` helper for stdout/stderr diagnostics             |

The build manifest at `manifest.pdxproj` names every source file
paideia-as compiles into the binary and sets the entry symbol to
`CpMain::cp_main`. The caps manifest at `caps.decl` declares the M1
baseline cap set the loader's InitCap sidecar must seed for cp to run.

## 3. Storage model (M1)

Every M1 module keeps its scratch state in `.bss` — the singleton
pattern from `src/user/tokenizer.pdx` and `src/user/dispatch.pdx` in
paideia-os. This is deliberate for bootstrap:

- One `cp_main` call per process. Every cp invocation is one copy;
  M1 does not need to build multiple parse contexts or multiple
  copy contexts.
- Zero heap dependency. cp predates any userspace allocator in the
  R50 wave; every buffer is a static array.
- Trivial reset. `Dispatch::dispatch_reset` clears the dispatch-
  chosen flag; `CopyFile::copy_reset` clears the two fd slots + the
  bytes-copied counter; the print helper is stateless.

The M1 copy body uses a 4096-byte read/write scratch buffer in
`CopyFile`'s `.bss` (`copy_buf`). One read populates it, one write
drains it. Each iteration copies at most 4 KiB — enough to make the
syscall count small (a 1 MiB file is 256 iterations) without pinning a
larger buffer that would waste .bss on a small-file-typical workload.

The M1 print helper writes bytes directly through `sys_write(1, buf,
len)` and `sys_write(2, buf, len)` (paideia-os syscall #1 fast-path
for fd 1 / 2 per `src/kernel/core/syscall/dispatch.pdx` L25). The
buffer is caller-owned; no `.bss` staging. This is enough for the M1
diagnostics; M3 adds a libpdx-semantic-pipe wrapper that writes typed
`CopyProgressRecord[]` in `--verbose` mode.

## 4. Dispatch shape

`cp_main(argc, argv)` walks argv via libpdx-argv (calls
`ParsedArgs::reset()` then `Parser::parse_argv(argv, argc)`), then
delegates to `Dispatch::dispatch_copy(argv, argc)`.

`Dispatch::dispatch_copy() -> exit_code`:

1. If libpdx-argv reported a parse error (via `Parser::parse_argv`
   return != 0), print `PARSE_FAIL_MSG` on stderr, return 2
   (`EXIT_USAGE`). This branch actually lives in `cp_main` because
   the parse call site is there; the dispatch is only reached on
   parse-OK.
2. Walk `ParsedArgs::flag_names[]`. If `-r` (short `r`) is present,
   print `RECURSIVE_NOT_YET_MSG` on stderr, return 3
   (`EXIT_NOT_YET_IMPLEMENTED`). Same for a positional count
   != 2 (recursive requires the "one source list + one directory
   destination" shape which M1 does not implement).
3. If `ParsedArgs::pos_count != 2`, print `USAGE_MSG` on stderr,
   return 2 (`EXIT_USAGE`).
4. Load `pos_ptrs[0]` (src path) and `pos_ptrs[1]` (dst path); call
   `CopyFile::copy_single_file(src_ptr, dst_ptr)`; return its exit
   code.

`CopyFile::copy_single_file(src_ptr, dst_ptr) -> exit_code`:

1. Reset the .bss slots (fd_src, fd_dst, bytes_copied).
2. `pdxfs_txn_begin()` — M1 stub returns 0 (see §6).
3. `pdxfs_open(src_ptr, O_RDONLY)` → fd_src (or negative errno).
4. On open failure: print `OPEN_SRC_FAIL_MSG`, tear down TXN, return 1.
5. `pdxfs_open(dst_ptr, O_CREAT | O_WRONLY)` → fd_dst.
6. On open failure: print `OPEN_DST_FAIL_MSG`, close src, tear down
   TXN, return 1.
7. Read/write loop:
   - `pdxfs_read(fd_src, copy_buf, COPY_BUF_LEN)` → n.
   - n == 0 → EOF, break.
   - n < 0 → read error; print `READ_FAIL_MSG`; goto teardown.
   - `pdxfs_write(fd_dst, copy_buf, n)` → w.
   - w < 0 → write error; print `WRITE_FAIL_MSG`; goto teardown.
   - w < n → short write (uncommon on the M1 substrate but possible
     if the destination fills a quota); print `SHORT_WRITE_MSG`;
     goto teardown.
   - Accumulate `bytes_copied += w`; loop.
8. `pdxfs_close(fd_dst)`, `pdxfs_close(fd_src)`.
9. `pdxfs_txn_commit()` — M1 stub returns 0.
10. Return 0.

The M2 upgrade replaces steps 2/9 with real TXN begin/commit against
the kernel's `KIND_PDXFS_TXN` cap; the rest of the body stays
unchanged. M3 adds a `libpdx-audit::audit_begin` before step 3 and a
`libpdx-audit::audit_commit` after step 10, and interleaves
`libpdx-semantic-pipe::emit(CopyProgressRecord{...})` inside the
loop when `-v` is set.

## 5. paideia-as compliance

Every module in this repo follows the constraints in
`design/kernel/paideia-as-conformance.md` (paideia-os) as they apply
to the userspace toolchain at v0.33+:

- Module names are PascalCase basename (`CpMain`, `Dispatch`,
  `CopyFile`, `Pdxfs`, `Print`) — no directory prefix.
- No `test` mnemonic; every zero-check uses `cmp reg, 0`.
- Every `cmp reg, imm` uses an immediate ≤ 0x7FFFFFFF (or sign-extends
  from a negative i32); larger immediates (e.g. the u64 sentinels for
  negative-errno returns from sys_read / sys_write) go via r11 staging.
- Register `r11` is scratch and is never assumed live across a call.
- Byte loads use `xor rax, rax; mov_b rax, [ptr]` per the paideia-as
  #1248 mitigation pattern.
- SysV push/pop parity: rsp % 16 == 0 at every nested call site.

## 6. What M1 explicitly does not do

Called out here so a reader of M1 code does not mistake absence for
bug:

- No recursion. `-r` is recognised at parse time (libpdx-argv accepts
  the short flag) but the dispatch layer exits 3
  (`EXIT_NOT_YET_IMPLEMENTED`) if it is present. M2 lands the
  recursive walk.
- No cap-tail preservation. The M1 copy body writes bytes only; the
  destination file's cap tail is whatever the kernel's default file
  creation policy yields. M2 re-signs at destination under invoker
  user_sk (per `design/user/model.md` §10.2 signed-inode field) if
  the key is unlocked, else graceful degrade with a `--verbose`
  diagnostic.
- No real TXN. `pdxfs_txn_begin` and `pdxfs_txn_commit` are stubs
  that return 0 unconditionally. The kernel's PdxFS v1 substrate at
  R42 landed the `KIND_PDXFS_TXN` derived-kind + witness only —
  `sys_pdxfs_txn_begin` and `sys_pdxfs_txn_commit` are not yet
  exposed. M2 lands the real begin/commit path; the M1 call sites
  are structured so only the two stub bodies change.
- No `--over-existing` semantics. The M1 copy uses O_CREAT and the
  kernel's default semantics — an existing target is overwritten in
  place. M2 adds the PdxFS v1 undo record before the write so
  `undo cp <same-args>` restores the pre-copy destination.
- No cross-subtree elevate. If the destination-parent path exceeds
  the invoker's cap scope, M1 exits 4 (`EXIT_CAP_DENIED`) with a
  diagnostic naming the missing cap. M3-003 wires libpdx-elevate.
- No semantic-pipe emission. `-v` is recognised but has no visible
  effect at M1. M3-001 emits `CopyProgressRecord[]` per file (or
  per N MiB block on large files) via libpdx-semantic-pipe.
- No audit journaling. M3-002 wires libpdx-audit's pre-output journal
  invariant. M1 exits do not touch `/system/audit/user-events/`.
- No `--help` renderer through `doc`. M1 hand-rolls the usage string
  inline; a later milestone wires `doc cp --help` once `doc` reaches
  a stable release (cross-repo dep in the plan doc §5.6 M5 line via
  the M5 `.pdxdoc` shipment).

## 7. Cross-repo dependencies

Per r49-r50-plan.md §5.6: **cp.M1 depends on §5.0 substrate + shell.M4**
in the wave-close sense; in the tool-code sense cp.M1 depends only on
libpdx-argv.M1 for the argv parse.

- libpdx-argv M1 landed (issues #1, #2, #3 closed 2026-08-21) — the
  argv parse dep is met.
- libpdx-cap M1 landed — indirect (loader-side manifest verify still
  uses the M1 skeleton; M2 flips to the real OK|MISSING|EXTRA compare).
- libpdx-audit / libpdx-elevate / libpdx-semantic-pipe M1 all landed —
  none are direct deps at cp.M1.
- paideia-os R48 substrate closed: KIND_USER = 0x190,
  KIND_ELEVATE_CHANNEL = 0x191, KIND_PDXFS_FILE = 0x195,
  KIND_PDXFS_TXN = 0x196 (commits 411ad0e, e56a95b, 2ff76d4).
- paideia-os kernel exposes sys_read (sysno 0), sys_write (sysno 1),
  sys_open (sysno 2), sys_close (sysno 3), sys_exit (sysno 60) at
  `src/kernel/core/syscall/dispatch.pdx`. These are the five syscalls
  cp M1 issues; no other syscall is required.
- The `KIND_PDXFS_TXN` begin/commit syscalls are NOT exposed at HEAD;
  see §6 for the M1 stub-and-swap-at-M2 approach.

paideia-as ≥ v0.33 is required by the module encoder (the byte-compare
idiom in `Dispatch::dispatch_copy` needs the mov_b narrow-load mnemonic
and the @align attribute on .bss slots).

At M2 the following cross-repo deps activate:
- libpdx-cap.M2 (signed-inode preservation helpers).
- paideia-os PdxFS v1 TXN begin/commit syscall exposure (blocker for
  the real single-TXN body).

At M3:
- libpdx-semantic-pipe.M2 (CopyProgressRecord[] emit).
- libpdx-audit.M2 (CopyRecord pre-output journal).
- libpdx-elevate.M3 (cross-subtree elevate retry).
