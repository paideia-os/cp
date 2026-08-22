# cp.M4-003 — signed-inode preservation (same-user) + degrade (cross-user)

**Issue:** #14
**Wave:** R50  Milestone: M4
**Upstream:** `design/tooling/r49-r50-plan.md` §5.6 M4 line +
`design/user/model.md` §10.2 (signed-inode field) in the
[paideia-os](https://github.com/paideia-os/paideia-os) repo.

## 1. Purpose

Verify the M2-002 + M2-003 cap-tail preservation contract: every
successful cp copy attempts to re-sign the destination inode under
the invoker's `user_sk`. When the invoker's key is unlocked and
the copy is same-user (destination inode's owner equals invoker),
the re-sign succeeds (`SI_OK`) and the destination cap-tail carries
a signature verifiable against the invoker's public key. When the
copy is cross-user OR the invoker's key is locked, the re-sign
degrades to `SI_DEGRADED_KEY_LOCKED` — the file bytes are copied,
but the destination inode's cap-tail is unsigned. Only `--verbose`
(or `-v`) emits a stderr diagnostic on degrade; silent-by-default
otherwise.

## 2. Test matrix

### Row 1 — same-user preservation OK (with unlocked key)

- **Substrate gate.** Blocked on libpdx-cap.M3-002 signed-inode
  helpers landed (2026-08-21, commit 3673dec on libpdx-cap). cp's
  `SignedInode::preserve_at_destination` still runs its M2 stub
  body — the row lifts when the M2-stub body is edited to call
  `cap_sign_inode` before falling through to the degrade branch.
- **Setup.** Invoker `alice` with unlocked `user_sk`. Source
  `/home/alice/src/a` = 256 bytes of `0x41`; destination
  `/home/alice/dst/a` does not exist. Caps: KIND_USER(alice, self);
  read cap on `/home/alice/src`; write cap on `/home/alice/dst`;
  KIND_PDXFS_TXN; KIND_IPC_ENDPOINT.
- **Invocation.** `cp /home/alice/src/a /home/alice/dst/a`.
- **Expected sequence.**
    - `copy_bytes_only` completes the read/write loop successfully.
    - Step 7 calls `SignedInode::preserve_at_destination(dst_ptr)`.
    - `preserve_at_destination` bumps `preserve_calls`, invokes
      the libpdx-cap.M3-002 signer against alice's `user_sk`
      (unlocked), returns `SI_OK`.
    - `bumps` NOT applied to `preserve_degrades` (SI_OK short-
      circuits the degrade branch — the M4 body-edit rearranges
      the M2 stub so `preserve_degrades` is bumped only on the
      degrade path, not unconditionally as the M2 stub does).
- **Assertions.**
    - Exit == 0.
    - `/home/alice/dst/a` exists, byte-identical to source.
    - `SignedInode::preserve_calls` == 1.
    - `SignedInode::preserve_degrades` == 0.
    - Destination inode's PdxFS cap-tail: `owner_user_id ==
      alice`, `signature` verifiable against alice's public key.
      (Harness reads the cap-tail via a libpdx-cap
      `cap_tail_read` helper landing at M4 — the tail is
      normally opaque to cp itself.)
    - stderr empty.

### Row 2 — same-user with locked key → degrade (runs today)

- **Substrate gate.** Runs today — the M2 stub's default return
  is exactly `SI_DEGRADED_KEY_LOCKED`.
- **Setup.** Invoker `alice` with LOCKED `user_sk` (passphrase not
  entered this session). Files/caps as Row 1.
- **Invocation.** `cp /home/alice/src/a /home/alice/dst/a`.
- **Expected sequence.**
    - Copy completes.
    - `preserve_at_destination` bumps `preserve_calls`. The M2
      stub's re-sign attempt (M4 body-edit calls
      `cap_sign_inode`) returns "key locked". Falls into the
      degrade branch: bumps `preserve_degrades`, gates on
      `CpFlags::flag_verbose_seen` — 0 today → skips the
      diagnostic, returns `SI_DEGRADED_KEY_LOCKED`.
    - `copy_bytes_only` treats `SI_DEGRADED_KEY_LOCKED` as a
      success-continuation code (per src/copy.pdx Step 7 comment)
      — the copy exit code stays 0.
- **Assertions.**
    - Exit == 0.
    - `/home/alice/dst/a` exists, byte-identical to source.
    - `SignedInode::preserve_calls` == 1.
    - `SignedInode::preserve_degrades` == 1.
    - stderr empty (verbose not set).
    - Destination inode's cap-tail: `owner_user_id == alice`,
      `signature` field zeroed / absent (harness reads via
      libpdx-cap `cap_tail_read`; the unsigned state is the M4
      pass criterion for this row).

### Row 3 — same-user, locked key, --verbose → degrade with diagnostic (runs today)

- **Substrate gate.** Runs today — same as Row 2 with the
  verbose gate flipped.
- **Setup.** Same as Row 2.
- **Invocation.** `cp -v /home/alice/src/a /home/alice/dst/a`.
- **Expected sequence.** Same as Row 2 with
  `CpFlags::flag_verbose_seen == 1`; `preserve_at_destination`
  emits `SI_DEGRADE_MSG` on stderr before returning.
- **Assertions.**
    - Exit == 0.
    - `/home/alice/dst/a` exists, byte-identical to source.
    - `SignedInode::preserve_degrades` == 1.
    - stderr contains "cp: destination cap-tail unsigned (key
      locked)\n" (SI_DEGRADE_MSG_LEN = 46 bytes verbatim).

### Row 4 — cross-user degrade (bob copies into shared subtree owned by alice)

- **Substrate gate.** Blocked on libpdx-cap.M3-002 signed-inode
  helpers producing the cross-user detection. The M2 stub returns
  `SI_DEGRADED_KEY_LOCKED` unconditionally; the M4 body-edit
  distinguishes "key locked" from "cross-user" via a new return
  code `SI_DEGRADED_CROSS_USER` (proposed at libpdx-cap.M3+; if
  the substrate collapses both into `SI_DEGRADED_KEY_LOCKED`, the
  M4 test row still passes on the byte + counter criteria; the
  differentiated-return assertion downgrades to "collapsed").
- **Setup.** Invoker `bob` with unlocked `user_sk`. Source
  `/home/bob/src/a` = 256 bytes of `0x41`; destination
  `/shared/a` — a subtree whose write cap is seeded for bob but
  whose signed-inode owner_user_id policy resolves to alice (a
  shared-writable subtree). Caps: KIND_USER(bob, self); read cap
  on `/home/bob/src`; write cap on `/shared`; KIND_PDXFS_TXN;
  KIND_IPC_ENDPOINT.
- **Invocation.** `cp /home/bob/src/a /shared/a`.
- **Expected sequence.**
    - Copy completes (bob's write cap covers `/shared`).
    - `preserve_at_destination` attempts to re-sign under
      bob's `user_sk`; libpdx-cap detects that the destination
      inode's owner_user_id resolves to alice (not bob); refuses
      to sign under bob's key (cross-user); returns
      `SI_DEGRADED_CROSS_USER` (or `SI_DEGRADED_KEY_LOCKED` if
      the substrate collapses the two).
    - Degrade branch: bumps `preserve_degrades`, gates on
      verbose (default off — silent), returns.
- **Assertions.**
    - Exit == 0.
    - `/shared/a` exists, byte-identical to source.
    - `SignedInode::preserve_degrades` == 1.
    - stderr empty.
    - Destination inode's cap-tail: `owner_user_id == alice` (the
      pre-existing owner if the inode existed, OR the shared-
      subtree default owner if newly created); `signature` field
      zeroed / absent.

### Row 5 — cross-user degrade with --verbose

- **Substrate gate.** Same as Row 4.
- **Setup.** Same as Row 4.
- **Invocation.** `cp -v /home/bob/src/a /shared/a`.
- **Assertions.** Same as Row 4 plus stderr contains SI_DEGRADE_MSG.

### Row 6 — recursive same-user preservation OK across N files

- **Substrate gate.** Row 1's gate + Row 2 of M4-001 (recursive
  walk substrate).
- **Setup.** Invoker `alice` with unlocked `user_sk`. Source
  `/home/alice/src/tree/` contains 3 files (`f1`, `f2`, `f3`, each
  1024 bytes). Destination `/home/alice/dst/tree/` does not exist.
- **Invocation.** `cp -r /home/alice/src/tree /home/alice/dst/tree`.
- **Assertions.**
    - Exit == 0.
    - 3 files under `/home/alice/dst/tree/`, byte-identical to
      sources.
    - `SignedInode::preserve_calls` == 3 (per-file invocation
      via walk_entry_todo's future body-edit).
    - `SignedInode::preserve_degrades` == 0.
    - Every destination inode carries a valid alice-signed
      cap-tail.

### Row 7 — recursive cross-user degrade across N files (bob → alice's tree)

- **Substrate gate.** Row 4's gate + M4-001 Row 2's gate.
- **Setup.** Invoker `bob` copying his own 3-file tree into
  `/shared/tree/`.
- **Invocation.** `cp -r -v /home/bob/src/tree /shared/tree`.
- **Assertions.**
    - Exit == 0.
    - 3 files under `/shared/tree/`, byte-identical.
    - `SignedInode::preserve_calls` == 3.
    - `SignedInode::preserve_degrades` == 3.
    - stderr contains SI_DEGRADE_MSG repeated 3 times (one per
      degraded entry; walk's per-entry `preserve_at_destination`
      call emits under `-v` — verify the diagnostic is per-file
      rather than once-per-invocation).

## 3. Fixtures

- **Two invoker identities.** `alice` (KIND_USER = 0x190 record
  at paideia-os R48.M1-001) and `bob` — both registered users
  with distinct `user_sk` values. `alice` is the default
  invoker for Rows 1-3, 6; `bob` for Rows 4-5, 7.
- **Key-unlocked / key-locked state.** Test harness controls the
  `user_sk` unlock state via a paideia-os user-events-journal
  hook (R48.M7) — Row 1 unlocks alice's key before cp
  invocation; Row 2 leaves it locked.
- **Shared subtree.** `/shared/` — a directory whose inode
  `owner_user_id` resolves to a shared / system owner (not
  alice, not bob). Both invokers have write caps but neither
  can sign the destination inodes under their own keys because
  the ownership policy locks the tail-signing identity to
  `alice`.

## 4. Observation hooks

| Symbol                                  | File                       | Semantic                                                |
|-----------------------------------------|----------------------------|---------------------------------------------------------|
| `SignedInode::preserve_calls`           | src/signed_inode.pdx       | total `preserve_at_destination` invocations             |
| `SignedInode::preserve_degrades`        | src/signed_inode.pdx       | degraded returns (SI_DEGRADED_KEY_LOCKED + future SI_DEGRADED_CROSS_USER) |
| `CpFlags::flag_verbose_seen`            | src/flags.pdx              | 0 or 1 mirror of the -v / --verbose parse              |
| `CopyFile::bytes_copied`                | src/copy.pdx               | per-file copy body byte count                           |
| `Walk::entries_visited`                 | src/walk.pdx               | per-entry increment (== preserve_calls under `-r`)      |
| Destination inode cap-tail (via libpdx-cap `cap_tail_read`) | libpdx-cap.M4 | owner_user_id + signature status of destination inode |

## 5. Invariants

I1. **Bytes always copy, even on degrade.** Every M4-003 row
    asserts destination byte-identical to source regardless of
    the signature outcome. cap-tail preservation is best-effort;
    the transactional guarantee is on file bytes, not on cap-
    tails (per design/user/model.md §10.2).

I2. **`preserve_calls` == number of successful copies.** Row 1-5:
    1. Row 6-7: 3. Under recursive walk, the per-entry
    `preserve_at_destination` call site lives inside
    `walk_entry_todo` and increments per entry.

I3. **`preserve_degrades` == number of non-`SI_OK` returns.** No
    row asserts both `preserve_calls > 0` AND `preserve_calls ==
    preserve_degrades` for a Row 1 shape — Row 1's landing at
    libpdx-cap.M3-002 gate flips it from "preserve_degrades =
    preserve_calls" (M2 stub) to "preserve_degrades = 0" (real
    signer).

I4. **Verbose diagnostic is per-entry under recursion.** Row 7
    asserts the SI_DEGRADE_MSG appears 3 times on stderr — the
    diagnostic emit lives inside `preserve_at_destination` which
    fires once per successful copy, not once per invocation.

I5. **Silent-by-default on degrade.** Rows 2, 4 assert stderr
    empty; only Rows 3, 5, 7 (verbose set) assert stderr contains
    SI_DEGRADE_MSG. This encodes the "silent-by-default,
    visible-on-request" discipline for auth-tail preservation
    events documented in design/user/model.md §10.2.

I6. **Copy exit code is always 0 on a degrade path.** No row
    asserts non-zero exit under a degrade — the degrade IS the
    success-continuation path. Only `SI_ERROR` (0xFFFFFFFF) would
    propagate to a non-zero exit; M4 does not exercise that
    branch (libpdx-cap.M3+ real signer is the only substrate
    that can return it; the M4 harness does not construct a
    scenario producing `SI_ERROR`).

## 6. Pass criteria

Rows 2, 3 pass today (M2-002/M2-003 stub returns
`SI_DEGRADED_KEY_LOCKED` unconditionally; verbose gate wired).
Rows 1, 4, 5, 6, 7 shape-lands and lift when
`SignedInode::preserve_at_destination`'s M2 stub is body-edited to
call libpdx-cap.M3-002's `cap_sign_inode` and the M4 harness gains
the cap-tail-read helper.

M4-003 closes when Rows 1-7 all pass on a gate-lifted substrate.

## 7. Not in scope for M4-003

- **Key-rotation mid-copy.** If alice's `user_sk` is rotated
  between the two files in Row 6, half the destinations carry
  the old-signed tail and half the new. Verifying that shape is
  out of M4 scope; the key-rotation contract lives in
  libpdx-cap.M5 or later.
- **Signature verification of source cap-tail.** cp does NOT
  verify the source inode's cap-tail before copying — cp
  respects any read cap it holds regardless of tail-signature
  validity. Verification is a filesystem-scan concern (e.g. `ls
  --verify-tails`), not cp's.
- **Cross-user resign under elevate.** If the elevate broker
  granted alice's authority to bob for the copy window, bob
  COULD in principle re-sign under alice's key. That path lands
  when libpdx-elevate.M4 grows a `sign-as-granter` capability;
  M4-003 does not exercise it.
- **Symlink cap-tail handling.** Symlinks are not in cp's M4
  scope (Walk skips non-regular entries with a --verbose
  diagnostic per src/walk.pdx walk_entry_todo comment); their
  cap-tail semantics land with the R42 symlink substrate.
