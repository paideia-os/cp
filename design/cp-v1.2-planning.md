# cp — v1.2 planning: `-p` (permission preserve) + `-r` (recursive)

**Wave:** R90-XREPO  **Issue:** paideia-os/cp#37 (planning only)
**Status:** PLANNING ONLY. No implementation in this landing.
Implementation is tracked for a future v1.2.0-A-shaped release, after
paideia-os/cp#35's outer TXN wrap (v1.3.0, `src/pdxfs_txn.pdx` +
`src/copy.pdx` Step 4.5) has soaked and, ideally, after a real
`parent_slot` is discoverable for `pdxfs_txn_begin` — a recursive
multi-file copy is exactly the case where per-file TXN atomicity
matters most, so `-r` should not land ahead of a working TXN wrap.

## 1. Scope

cp v1.3.0 copies exactly one file per invocation and preserves only
the source's POSIX mode bits at create time (`copy_bytes_only`'s
Step 3/4 `src_mode` -> `pdxfs_open`'s mode arg). It has no `-p`, no
`-r`, and `src/dispatch.pdx` does not recognise either flag as
anything other than an unconsumed libpdx-argv token. This document
plans both without touching a single source file.

## 2. `-p` semantics (permission preserve)

### 2.1 Behaviour

After the existing copy body completes successfully (`cp_eof`, before
the `pipe_emit_copy_record` tap), `-p` runs three additional steps
against the resolved destination path:

1. `stat(src)` — re-read (or reuse the Step 2 `copy_src_stat` buffer,
   still valid at `cp_eof` since nothing overwrites it after Step 2)
   for `mode`, `uid`, `gid`, `atime`, `mtime`.
2. `chmod(dst, src_mode)` — **hard failure**. If this fails, `-p`'s
   contract is broken in a way a caller cannot silently ignore (the
   file exists with a wrong, possibly more permissive, mode), so
   `copy_bytes_only` returns the chmod's raw negative-errno u64
   verbatim, matching every other hard-failure branch in this
   function. This is the ONE new hard-failure branch `-p` introduces.
3. `chown(dst, src_uid, src_gid)` then `utimensat(dst, src_atime,
   src_mtime)` — **fail-open (best-effort)**, same discipline as this
   codebase's existing "return value discarded, not a per-run
   condition the caller can act on" convention (`pipe_emit_copy_
   record`, `pdxfs_close` in the success path). Rationale: a copy
   performed by a non-privileged user routinely cannot `chown` to an
   arbitrary source uid/gid (EPERM is the expected common case, not
   an anomaly), and a clock/timestamp-adapter gap should not fail an
   otherwise-successful, correctly-permissioned copy.

### 2.2 New kernel syscalls needed

Neither `sys_chown` nor `sys_utimensat`/`sys_utimes` exists in
`src/kernel/core/syscall/dispatch.pdx` at this commit (verified via
the same grep sweep this planning doc's sibling, cp#35, used against
sysnos 70/104/105/107 — no `chown`/`utime*` hits anywhere in
`src/kernel/core/syscall/handlers/`). `chmod` similarly has no
dedicated syscall; `sys_open`'s `mode` argument is the only mode-set
path that exists today (per `design/pdxfs-notes.md` §2, "mode
argument to sys_open is IGNORED at HEAD" for at least one backend).
`-p` is therefore blocked on THREE kernel primitives landing, not
zero — this is the primary reason implementation defers past this
planning issue.

### 2.3 New trampolines (future `src/pdxfs.pdx` additions)

```
pdxfs_chmod(path_ptr, mode)                    -> 0_or_errno
pdxfs_chown(path_ptr, uid, gid)                -> 0_or_errno
pdxfs_utimens(path_ptr, atime_ns, mtime_ns)    -> 0_or_errno
```

All three are path-based (matching `pdxfs_stat`'s existing shape,
not an fd-based `fchmod`/`fchown`/`futimens` — cp already has the
resolved absolute dst path in hand at `cp_eof` and no open fd by that
point, since `pdxfs_close(fd_dst)` runs first in Step 6).

## 3. `-r` semantics (recursive)

### 3.1 Behaviour

```
cp -r SRC DST:
  if SRC is not a directory: identical to today's copy_bytes_only.
  else:
    mkdir(DST) if it does not exist (mode from stat(SRC) if -p, else default).
    readdir(SRC):
      for each entry NOT "." and NOT "..":
        child_src = SRC + "/" + entry.name
        child_dst = DST + "/" + entry.name
        if entry is a directory: recurse (this same procedure).
        elif entry is a symlink: REFUSE (see 3.2) unless -L given.
        else: copy_bytes_only(child_src, child_dst).
      any per-entry failure: print a diagnostic naming the entry,
      continue to the next entry (matches GNU cp's default -- one
      bad file does not abort the whole tree), and remember that at
      least one failure occurred so the overall exit code is non-zero.
```

### 3.2 Symlink refusal

No `-L` flag exists at this planning stage and none is scoped for the
v1.2.0-A landing. `-r` refuses to descend into or copy a symlink
entry (diagnostic: `cp: -r does not follow symlinks (no -L): <path>`,
per-entry failure per 3.1) rather than silently copying the link's
target's content under the link's name (a correctness trap) or the
link's target as a literal file (also wrong). This mirrors GNU cp's
own posture that `-r` without `-L`/`-P`/`-H` needs an explicit
disambiguation the argv surface does not yet offer.

### 3.3 New kernel syscalls needed

`readdir`-shape access: `sys_pdxfs_dir_readnext` (sysno confirmed
landed — see `src/kernel/core/syscall/handlers/sys_pdxfs_dir_
readnext.pdx`, R42-PREP-008 #1630 + R90-XREPO.010.M1-006 #2114) is
already real, plus `sys_open`'s existing directory-open path and
`sys_stat`'s existing `S_IFDIR` bit (both of which `copy_bytes_only`
already uses for the DST-is-a-directory probe). `-r` is therefore
LESS blocked than `-p`: the missing piece is a `pdxfs_mkdir`
trampoline (mkdir(1) already ships this exact syscall in this repo
tree — `tools/user/mkdir/src/*.pdx` is the shape reference) plus a
`pdxfs_readdir_open`/`pdxfs_readdir_next` trampoline pair wrapping
the landed `sys_pdxfs_dir_readnext` handler, neither of which touches
unlanded kernel surface.

### 3.4 Symlink-type detection

`sys_stat`'s frozen 64-byte layout (mode `u32@+24`) already carries
enough to distinguish a symlink (`S_IFLNK` bits, mirroring the
`S_IFDIR` check `copy_bytes_only::S_IFDIR_BITS` already performs) from
a regular file or directory, so 3.2's refusal gate needs no new
kernel surface beyond the existing `pdxfs_stat` trampoline.

## 4. Argv parsing

Both flags are single-character, no-argument boolean switches --
same shape as the already-recognised `--dry-run` / `--verbose` /
`--help` / `--version` StdVocab entries. Extends `libpdx-argv`'s
`FlagSpec` table (consumed by `src/dispatch.pdx`, mirroring `mv`'s
`-p`-adjacent flag wiring where it exists) with two new short-flag
rows:

| Flag | Long form (future) | ParsedArgs bit  |
|------|---------------------|-----------------|
| `-p` | (none planned)      | `CP_FLAG_PRESERVE` |
| `-r` | (none planned)      | `CP_FLAG_RECURSIVE` |

No libpdx-argv version bump is anticipated (the Wave 6 v1.1.3
StdVocab contract already supports arbitrary short-flag rows); this
is a `caps.decl`-neutral change (no new capability surface).

## 5. Test matrix (placeholder shapes only)

`tests/cp_p.pdx` (`TestCpP::run`, M4-driver convention, return codes
0/1/2/3):

1. Setup: create `/tmp/cp_p_src` with mode `0640`, unlink any stale
   `/tmp/cp_p_dst`.
2. `cp -p /tmp/cp_p_src /tmp/cp_p_dst`.
3. `sys_stat(/tmp/cp_p_dst)` and assert `mode & 0xFFF == 0640`.
4. (Once `pdxfs_chown`/`pdxfs_utimens` land) assert `uid`/`gid`/
   `mtime` match source within the backend's timestamp granularity.

`tests/cp_r.pdx` (`TestCpR::run`, same convention):

1. Setup: `/tmp/cp_r_src/{a.txt, sub/b.txt}`, unlink any stale
   `/tmp/cp_r_dst` tree.
2. `cp -r /tmp/cp_r_src /tmp/cp_r_dst`.
3. Assert `/tmp/cp_r_dst/a.txt` and `/tmp/cp_r_dst/sub/b.txt` both
   exist via `sys_stat` and their sizes match the sources.
4. Symlink-refusal case: add `/tmp/cp_r_src/link -> a.txt`, re-run,
   assert the tree copy still succeeds overall (partial-failure
   tolerance per §3.1) but `/tmp/cp_r_dst/link` does NOT exist and
   the process exit code is non-zero.

Both files are placeholders at this landing -- no `.pdx` test source
exists yet; they are named here so the v1.2.0-A implementation lands
tests in the same commit rather than as a follow-up gap.

## 6. Explicit non-scope of this document

This is a PLANNING issue. No `src/*.pdx`, `caps.decl`, `manifest.
pdxproj`, or `tests/*.pdx` file changes accompany it. Implementation
lands in a future v1.2.0-A release once (a) the `-p` kernel
primitives (§2.2) and the `-r` mkdir/readdir trampolines (§3.3) are
either landed or explicitly re-scoped, and (b) paideia-os/cp#35's
outer TXN wrap has a real `parent_slot` (recursive multi-file copies
are the strongest case for wanting per-file TXN atomicity).
