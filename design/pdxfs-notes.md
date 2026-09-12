# cp — PdxFS syscall + TXN notes (M1)

**Wave:** R50  Milestone: M1-003
**Upstream:** `design/tooling/r49-r50-plan.md` §5.6 + §5.0 in
[paideia-os](https://github.com/paideia-os/paideia-os).

## 1. Purpose

This document records the M1-003 decisions about how cp talks to the
filesystem substrate before every piece of that substrate has landed.
Two axes matter:

- Which paideia-os syscalls cp can already issue at HEAD (2026-08-21).
- Which primitives cp needs at v1.0 but the substrate does not yet
  expose (KIND_PDXFS_TXN begin/commit) — and how the M1 code accounts
  for them so the M2 upgrade is a bounded body-edit.

## 2. Kernel substrate at HEAD

The paideia-os kernel exposes these syscalls at HEAD, verified against
`src/kernel/core/syscall/dispatch.pdx` L256-309:

| sysno | Name         | Body handler at                                             |
|-------|--------------|-------------------------------------------------------------|
| `0`   | `sys_read`   | `src/kernel/core/syscall/handlers/sys_read.pdx` (R16-M3-003) |
| `1`   | `sys_write`  | `src/kernel/core/syscall/handlers/sys_write.pdx`             |
| `2`   | `sys_open`   | `src/kernel/core/syscall/handlers/sys_open.pdx`  (R16-M3-001) |
| `3`   | `sys_close`  | `src/kernel/core/syscall/handlers/sys_close.pdx`             |
| `60`  | `sys_exit`   | `src/kernel/core/syscall/handlers/sys_exit.pdx`              |

Additional related state:

- `KIND_PDXFS_FILE = 0x195` and `KIND_PDXFS_TXN = 0x196` derived kinds
  landed at commit `411ad0e` — the descriptors + witnesses exist, but
  the *begin/commit syscalls* for KIND_PDXFS_TXN are not yet wired in
  dispatch.pdx.
- `sys_open` accepts `O_CREAT = 0x40` (bit 6) per
  `src/kernel/core/fs/vfs_open.pdx` L36. Access mode bits are accepted
  but not enforced at the vnode layer at HEAD.
- `mode` argument to `sys_open` is IGNORED per the sys_open handler
  comment (L18); `pdxfs_open` passes 0.

## 3. The five real trampolines

`src/pdxfs.pdx` wraps four ring-3 syscalls behind named entry points:

| Entry           | Arity | SysV → SYSCALL shuffle   |
|-----------------|-------|--------------------------|
| `pdxfs_open`    | 2 in  | none; writes rdx=0 (mode) |
| `pdxfs_read`    | 3 in  | none                     |
| `pdxfs_write`   | 3 in  | none                     |
| `pdxfs_close`   | 1 in  | none                     |

None of the four crosses arity 4, so no `mov r10, rcx` prefix is
needed (the shuffle template lives in
`src/user/syscall_shim.pdx` in paideia-os and in libpdx-audit's
`src/syscall_shim.pdx` for the arity-4 `sys_ipc_send`).

Every trampoline is a leaf `mov rax, N; syscall; ret` sequence and
returns the syscall's raw u64 in rax; callers gate errors with
`cmp rax, 0; jl err` (signed compare — SF derives from bit 63 of rax
so any negative-errno sentinel branches without immediate staging).

## 4. The two TXN stubs

`pdxfs_txn_begin` and `pdxfs_txn_commit` at M1 both return 0
(`PDXFS_TXN_OK`) unconditionally. This is a deliberate short-term
placeholder for two reasons:

1. **Substrate not yet exposed.** The `KIND_PDXFS_TXN` derived kind
   exists but no `sys_pdxfs_txn_begin` / `sys_pdxfs_txn_commit`
   dispatch entry has been added. Landing those syscalls is a
   paideia-os concern (§5.0 substrate gate in the plan doc calls out
   the general R42 PdxFS v1 substrate landing as a wave blocker;
   TXN begin/commit is the specific piece cp needs).
2. **Call-site stability.** cp's `CopyFile::copy_single_file` wraps
   the entire read/write loop between a `pdxfs_txn_begin` and a
   `pdxfs_txn_commit`. Keeping those calls at M1 (even as no-ops)
   means the M2 upgrade is a body-edit of the two Pdxfs entry points
   only. No line of `copy.pdx` moves at M2.

At M2 both stubs become real:

- `pdxfs_txn_begin` binds a `KIND_PDXFS_TXN` cap from the InitCap
  sidecar and issues the new `sys_pdxfs_txn_begin` syscall; the
  returned txn handle is stashed in a .bss slot (or in the returned
  register — final ABI is a paideia-os call).
- `pdxfs_txn_commit` issues the matching `sys_pdxfs_txn_commit`
  against the stashed handle; failure (e.g. quota exhaustion revealed
  at commit) returns a negative errno and the caller aborts.

At M2, cp's error branches inside `copy_single_file` also gain a
`pdxfs_txn_abort()` call before their print + return. That entry
point is not yet in `pdxfs.pdx` because the abort semantics would be
a no-op at M1 (nothing to roll back when begin is a no-op).

## 5. Error-code sentinel map

The kernel's error-return convention is Linux-signed u64 (negative
errno with bit 63 set). Recorded here so cp's error diagnostics can
stay uniform:

| Errno name | u64 sentinel        | Emitted by                            |
|------------|---------------------|---------------------------------------|
| `-ENOENT`  | `0xFFFFFFFFFFFFFFFE` | `sys_open` (path not found)          |
| `-EMFILE`  | `0xFFFFFFFFFFFFFFE8` | `sys_open` (no free fd slot)         |
| `-EBADF`   | `0xFFFFFFFFFFFFFFF7` | `sys_read`, `sys_write`, `sys_close` |
| `-EIO`     | `0xFFFFFFFFFFFFFFFB` | `sys_read`, `sys_write`              |
| `-EFAULT`  | `0xFFFFFFFFFFFFFFF2` | any (user pointer walker fail)       |

cp M1 collapses every non-zero negative return to `EXIT_OP_FAIL` (1)
with a two-word "what step failed" diagnostic (open source / open
dest / read / write / short write). M3 replaces these with structured
`CopyRecord` fields via libpdx-audit that carry the exact errno as a
first-class value.

## 5b. v1.1-B addendum — sys_getcwd for O_CREAT parent-resolve (cp#21)

**Wave:** R50  Milestone: v1.1-B (2026-09-12).

v1.1-A shipped exactly the five trampolines above and relied on the
kernel's `vfs_open` main `path_resolve` (R86.M1-005, paideia-os
#1958) to anchor slash-bearing relative paths against
`TASK_OFF_CWD` at every syscall boundary — free cwd resolution for
`sys_stat`/`sys_open`. That covered every cp code path *except*
the O_CREAT parent-scan.

**The gap.** `src/kernel/core/fs/vfs_open.pdx` label
`vfs_open_creat_scanned` scans the path for the last `/` to
determine the parent-directory to create the new inode inside.
When the create path has no `/` at all (`cp notes.md backup.md`
from `/home/alice`), the scan falls off the front, `r9 == 0`, and
the arm jumps straight to `vfs_open_fail`:

```
cmp r9, 0
je  vfs_open_fail   // no '/' -> no cwd support, fail
```

So a slash-less O_CREAT destination was refused before cp's own
error branches ever ran. See paideia-os/cp#21 for the full
teardown.

**The v1.1-B fix.** cp adds `pdxfs_getcwd` (sysno 86, R86.M1-003,
paideia-os #1956) as a sixth trampoline and resolves the
destination path against the task cwd in userspace BEFORE the
O_CREAT `sys_open` call. Slash-less destinations become absolute
paths (`/home/alice/backup.md`), the vfs_open parent-scan then
finds a `/` at the expected position, and the create succeeds
through the main `path_resolve` arm the read side already uses.
Absolute destinations pass through unchanged.

This lands the mkdir v1.1-B userspace-resolve precedent
(paideia-os/mkdir#25) at the cp side too, and keeps cp forward-
compatible with a future kernel-side fix that anchors the O_CREAT
parent at `TASK_OFF_CWD` when no `/` is found: whichever fix the
substrate lands, cp continues to hand it an absolute path.

| Entry            | Arity | SysV → SYSCALL shuffle        | Since   |
|------------------|-------|-------------------------------|---------|
| `pdxfs_getcwd`   | 2 in  | none                          | v1.1-B  |

Two 256-byte `.bss` slots in `Copy` back the resolve:

- `copy_cwd_scratch`   — destination for `sys_getcwd`; 256 bytes
  matches `SYS_GETCWD_PATH_MAX` in the kernel's header so any
  path the kernel can compose fits without a `-ERANGE` round-trip.
- `copy_dst_resolved`  — staging for the joined absolute path
  handed to `sys_open`; 256 bytes matches `SYS_MKDIR_PATH_MAX`.
  Overflow (`cwd_len + sep + user_len > 255`) returns
  `-ENAMETOOLONG` (-36) as the resolve helper's rax.

Table 5b: v1.1-B sysno additions.

| sysno | Name          | Body handler at                                            |
|-------|---------------|------------------------------------------------------------|
| `77`  | `sys_stat`    | `src/kernel/core/syscall/handlers/sys_stat.pdx`  (R56.M3-002) |
| `86`  | `sys_getcwd`  | `src/kernel/core/syscall/sys_getcwd.pdx`         (R86.M1-003) |

## 6. What M1 does NOT talk to yet

- No `sys_svc_lookup` (sysno 43). cp does not need the elevate-broker
  binding at M1 (the elevate path is M3-003, per the plan doc §5.6
  M3 line).
- No `sys_ipc_send` (sysno 42). cp does not emit typed records at M1;
  M3-001 wires libpdx-semantic-pipe over sys_ipc_send.
- No `sys_dup2` (sysno 32). cp does not manipulate fd 1/2 or splice
  its output into a downstream pipe at M1; the shell wires whatever
  redirection the user asked for before it invokes cp.
- No `sys_fork` / `sys_execve` / `sys_wait4`. cp is a leaf tool —
  never spawns anything at any milestone.
