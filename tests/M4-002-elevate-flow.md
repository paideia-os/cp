# cp.M4-002 — cross-subtree elevate flow (auto-approve + human paths)

**Issue:** #13
**Wave:** R50  Milestone: M4
**Upstream:** `design/tooling/r49-r50-plan.md` §5.6 M4 line + §5.1
(elevate policy) in the
[paideia-os](https://github.com/paideia-os/paideia-os) repo.

## 1. Purpose

Verify the M3-003 elevate wire: when cp's first `pdxfs_open(dst,
O_WRONLY|O_CREAT)` fails because the invoker's cap set does not
reach the destination-parent subtree, cp calls
`CpElevate::cp_elevate_try_dst_parent` which asks the elevate broker
to widen the invoker's authority for a bounded window
(`CP_ELEVATE_DUR_NS` = 60s per design/user/model.md §5.1). On
approval cp retries the open once; on refusal cp propagates the
original open failure via the `copy_open_dst_fail` branch.

The plan §5.6 M4 line breaks the "cross-subtree elevate flow" into
two paths: **auto-approve** (elevate_policy pre-populated with a
rule granting the request without a human hop) and
**human-approve** (broker escalates to a human decision, which
resolves within the timeout window). Both paths must exercise the
same cp-side retry code (copy_bytes_only Step 3a), so the M4 test
matrix has four rows: auto-approve OK, human-approve OK,
human-approve refused, and human-approve timeout.

## 2. Test matrix

### Row 1 — dst-parent-in-scope (baseline; no elevate; runs today)

- **Setup.** `/tmp/cp-m4/src/a` = 256 bytes of `0x41`;
  `/tmp/cp-m4/dst/` empty. Caps: KIND_USER = invoker;
  KIND_PDXFS_FILE(read, `/tmp/cp-m4/src`);
  KIND_PDXFS_FILE(write, `/tmp/cp-m4/dst`); KIND_PDXFS_TXN;
  KIND_IPC_ENDPOINT. **No KIND_ELEVATE_CHANNEL cap** — verifies
  cp does not attempt elevate when the first open succeeds.
- **Invocation.** `cp /tmp/cp-m4/src/a /tmp/cp-m4/dst/a`.
- **Assertions.**
    - Exit == `EXIT_OK` (0).
    - `CpElevate::elevate_calls` == 0 (the first `pdxfs_open` on
      dst succeeded; `copy_bytes_only` fell through to
      `copy_open_dst_ok` without touching the elevate arm).
    - `CpElevate::elevate_ok` == 0.
    - `CpElevate::elevate_refused` == 0.

### Row 2 — auto-approve OK

- **Substrate gate.** Blocked on libpdx-elevate.M3 real block-on-
  reply path (libpdx-elevate.M1-002 returns `ELVC_STUB` today,
  which `cp_elevate_try_dst_parent` folds to `CP_ELEVATE_REFUSED`).
  Row is shape-in-place — the cp-side retry code (copy_bytes_only
  Step 3a) is wired; only the ELVC_OK path from the broker is not
  exercisable until libpdx-elevate.M3-002 lands the retry-with-
  backoff for transient broker unavailability + the daemon boot.
- **Setup.** `/tmp/cp-m4/src/a` = 256 bytes of `0x41`.
  Destination `/opt/shared/pkg/a` — a path OUTSIDE the invoker's
  seeded write cap scope. Caps: as Row 1 plus
  KIND_ELEVATE_CHANNEL(invoke, svc.elevate-broker). The elevate
  policy at paideia-os `src/kernel/core/user/elevate_policy.pdx`
  contains a pre-installed rule granting invoker
  `KIND_PDXFS_FILE(write, /opt/shared/pkg)` for 60s on request
  without a human hop.
- **Invocation.** `cp /tmp/cp-m4/src/a /opt/shared/pkg/a`.
- **Expected sequence.**
    - `copy_bytes_only` opens src OK.
    - First `pdxfs_open(dst, 0x41)` fails (invoker's caps do not
      reach `/opt/shared/pkg`).
    - `copy_bytes_only` jumps to Step 3a, calls
      `cp_elevate_try_dst_parent(dst_ptr)`.
    - `cp_elevate_try_dst_parent` bumps `elevate_calls`, marshals
      the REQ frame (`caps = CP_ELEVATE_CAPS = 1`, `dur = 60e9`),
      calls `ElevateClient::elevate_client_request`.
    - libpdx-elevate consults the elevate_policy, finds the auto-
      approve rule, returns `ELVC_OK`.
    - `cp_elevate_try_dst_parent` bumps `elevate_ok`, returns
      `CP_ELEVATE_OK`.
    - `copy_bytes_only` retries `pdxfs_open(dst, 0x41)`; the
      invoker's now-widened cap set covers the target; open
      succeeds. Retry succeeds — copy proceeds.
- **Assertions.**
    - Exit == 0.
    - `/opt/shared/pkg/a` exists, byte-identical to source.
    - `CpElevate::elevate_calls` == 1.
    - `CpElevate::elevate_ok` == 1.
    - `CpElevate::elevate_refused` == 0.
    - `CpPipe::pipe_emits` == 1.
    - Audit journal contains one UEJ_KIND_TOOL_INVOKE record
      naming cp and its argv, followed by one elevate REQ+APR
      pair in the elevate-broker journal, followed by one
      UEJ_KIND_TOOL_EXIT record with `exit_code == 0`.

### Row 3 — human-approve OK

- **Substrate gate.** Same as Row 2 plus a running human-approve
  mock in the test harness. libpdx-elevate.M2-002 lands the
  human-approve path with configurable timeout — this row
  requires that landing + a mock human granter integrated into
  the M4 harness.
- **Setup.** Same as Row 2 but the elevate_policy rule for
  `/opt/shared/pkg` requires human decision instead of auto-
  approve. The mock human granter is configured to APPROVE with
  a 500ms delay (well under `CP_ELEVATE_DUR_NS` = 60s).
- **Invocation.** `cp /tmp/cp-m4/src/a /opt/shared/pkg/a`.
- **Expected sequence.** As Row 2 with a broker-side stall while
  the human decision is pending. libpdx-elevate.M2-002's block-on-
  reply with timeout tolerates the 500ms wait, returns `ELVC_OK`
  on approval.
- **Assertions.**
    - Exit == 0.
    - `/opt/shared/pkg/a` exists, byte-identical to source.
    - `CpElevate::elevate_calls` == 1.
    - `CpElevate::elevate_ok` == 1.
    - Elevate REQ+APR pair in the broker journal names the human
      granter's identity in the APR frame's `granter_user_id`
      field (frame layout defined by libpdx-elevate.M1-001).
    - Elapsed time between REQ send and APR receive >= 500ms
      (harness observability via a wall-clock hook alongside the
      audit journal timestamps).

### Row 4 — human-approve refused

- **Substrate gate.** Same as Row 3.
- **Setup.** Same as Row 3 but the mock human granter is
  configured to REFUSE.
- **Invocation.** `cp /tmp/cp-m4/src/a /opt/shared/pkg/a`.
- **Expected sequence.**
    - As Row 3 until the broker returns `ELVC_ERR_HUMAN_REFUSED`
      (libpdx-elevate.M2-002 rc; exact sentinel TBD by that
      milestone — the current M1-002 vocabulary reserves the
      `ELVC_ERR_*` space at `0xFFFFEA0*`).
    - `cp_elevate_try_dst_parent` folds ELVC_ERR_HUMAN_REFUSED
      to `CP_ELEVATE_REFUSED`, bumps `elevate_refused`, returns
      `CP_ELEVATE_REFUSED`.
    - `copy_bytes_only` sees `rax != 0`, falls through to
      `copy_open_dst_fail` — prints `OPEN_DST_FAIL_MSG` on
      stderr, closes fd_src, returns 1.
    - `Dispatch::dispatch_copy` sees body-fail, calls
      `pdxfs_txn_abort` (silent), returns 1.
- **Assertions.**
    - Exit == `EXIT_OP_FAIL` (1).
    - `/opt/shared/pkg/a` does NOT exist (the retry never
      happened; nothing was written).
    - stderr contains "cp: cannot open destination file\n".
    - `CpElevate::elevate_calls` == 1.
    - `CpElevate::elevate_ok` == 0.
    - `CpElevate::elevate_refused` == 1.
    - Audit journal: TOOL_INVOKE + elevate REQ+APR(refused) +
      TOOL_EXIT(exit_code=1).

### Row 5 — human-approve timeout

- **Substrate gate.** Same as Row 3 plus libpdx-elevate.M2-002's
  timeout branch.
- **Setup.** Same as Row 3 but the mock human granter is
  configured to NEVER respond. libpdx-elevate.M2-002's
  configurable timeout is set to 2000ms for this row (the M4
  harness overrides the default `CP_ELEVATE_DUR_NS` for testing
  via a broker-side param; cp's `CP_ELEVATE_DUR_NS` = 60e9 is
  the REQ-frame value the broker uses as its ceiling, not the
  cp-side wall-clock limit).
- **Invocation.** `cp /tmp/cp-m4/src/a /opt/shared/pkg/a`.
- **Expected sequence.** As Row 4 with the broker returning
  `ELVC_ERR_TIMEOUT` after 2000ms wall-clock instead of
  `ELVC_ERR_HUMAN_REFUSED`. The fold-to-REFUSED is identical.
- **Assertions.**
    - Exit == 1.
    - Elapsed time on the cp invocation >= 2000ms (harness
      observability).
    - `CpElevate::elevate_refused` == 1.
    - Same destination-side assertions as Row 4.

### Row 6 — no KIND_ELEVATE_CHANNEL cap seeded

- **Substrate gate.** Runs today — libpdx-elevate's client-side
  cap check fails before any broker hop.
- **Setup.** Same as Row 2 but WITHOUT KIND_ELEVATE_CHANNEL in the
  seeded cap set. The elevate broker endpoint lookup fails.
- **Invocation.** `cp /tmp/cp-m4/src/a /opt/shared/pkg/a`.
- **Expected sequence.**
    - `cp_elevate_try_dst_parent` bumps `elevate_calls`, calls
      `ElevateClient::elevate_client_request`.
    - libpdx-elevate's endpoint resolve fails; returns
      `ELVC_ERR_LOOKUP_FAIL` (0xFFFFEA0F).
    - Fold to `CP_ELEVATE_REFUSED`, propagate to
      `copy_open_dst_fail`.
- **Assertions.**
    - Exit == 1.
    - `CpElevate::elevate_calls` == 1.
    - `CpElevate::elevate_refused` == 1.
    - stderr contains "cp: cannot open destination file\n" (the
      normal open-fail diagnostic; cp does not emit a distinct
      "elevate unavailable" line at M3 — see the
      `ELEVATE_REFUSED_MSG` .rodata slot in `src/elevate.pdx`
      reserved for the M4-verbose surface).

## 3. Fixtures

- **Cross-subtree scratch.** `/opt/shared/pkg/` is a canonical
  cross-subtree path. The M4 harness pre-creates the directory
  under a mount that the invoker's seeded caps do NOT cover — the
  elevate broker is the sole authority granting write access.
- **Elevate policy fixture.** `paideia-os
  src/kernel/core/user/elevate_policy.pdx` at R48.M7 exposes a
  rule-add hook the M4 harness uses to inject the per-row rules
  (Row 2's auto-approve rule; Row 3-5's human-approve rule with
  test-time granter routing).
- **Mock human granter.** A test-only paideia-os user (`test-
  granter`) receives the human-approve prompt via the elevate
  broker's out-of-band channel; a harness-side responder scripts
  APPROVE/REFUSE/TIMEOUT per row.
- **KIND_ELEVATE_CHANNEL seeding.** Rows 2-5 have the cap seeded
  in the InitCap sidecar (the placeholder line in cp's `caps.decl`
  becomes real at M4 — see caps.decl §"placeholder: elevate-
  broker binding"); Row 1 and Row 6 do not.

## 4. Observation hooks

| Symbol                              | File                | Semantic                              |
|-------------------------------------|---------------------|---------------------------------------|
| `CpElevate::elevate_calls`          | src/elevate.pdx     | total `cp_elevate_try_dst_parent` invocations |
| `CpElevate::elevate_ok`             | src/elevate.pdx     | ELVC_OK returns (cp retries the open) |
| `CpElevate::elevate_refused`        | src/elevate.pdx     | non-OK returns (ELVC_STUB + ELVC_ERR_*) |
| `CpElevate::cp_elevate_req_buf`     | src/elevate.pdx     | REQ frame bytes (harness parses via libpdx-elevate frame layout) |
| `CpElevate::cp_elevate_reply_buf`   | src/elevate.pdx     | APR frame bytes (harness parses same) |
| `CopyFile::bytes_copied`            | src/copy.pdx        | 0 on Row 4-6 (open never succeeded); N on Row 2-3 |
| `CpPipe::pipe_emits`                | src/pipe.pdx        | 1 on Row 2-3; 0 on Row 4-6 |
| `CpAudit::audit_commit_calls`       | src/audit.pdx       | 1 across every row |

## 5. Invariants

I1. **Every cp invocation with a non-in-scope dst-parent calls
    `cp_elevate_try_dst_parent` exactly once per failed open.** The
    M3 wire retries once — never twice. If elevate approves but
    the retry still fails, cp propagates the original fail; it
    does not re-elevate.

I2. **Elevate approval widens the invoker's cap set for the
    granted window.** After Row 2/3 the invoker holds
    `KIND_PDXFS_FILE(write, /opt/shared/pkg)` for 60s; a
    subsequent cp within the window does NOT elevate again
    (cp_elevate_try_dst_parent is not called because the first
    open succeeds).

I3. **Elevate refusal is silent on stdout.** The refusal
    diagnostic goes to stderr via `OPEN_DST_FAIL_MSG` (the
    generic open-fail line). cp does not emit a distinct
    "elevate refused" line on stdout at M4 — the
    `ELEVATE_REFUSED_MSG` .rodata slot is reserved for a
    future `--verbose` surface (see cp.M2-003 for the analogous
    verbose-on-degrade pattern).

I4. **Elevate REQ + APR are journaled in the elevate-broker
    audit stream, not cp's own audit stream.** cp's
    UEJ_KIND_TOOL_INVOKE record does not carry elevate metadata;
    the broker's REQ+APR pair is the authoritative record.
    cp's TOOL_EXIT record still fires with the final exit code
    (0 or 1 per row).

I5. **The elevate REQ frame carries the exact
    `CP_ELEVATE_DUR_NS = 60000000000` value cp compiles in.** No
    per-invocation override at M4; the value is a static
    constant. A future `--elevate-timeout` flag lands as a cp
    argv addition + a REQ-frame per-invocation duration
    override.

## 6. Pass criteria

Row 1 and Row 6 pass today. Rows 2-5 shape-lands; they lift to
executable rows when libpdx-elevate.M2-002 + M3-001 land the
block-on-reply + auto-approve paths, and when the M4 harness gains
its mock human granter integration.

M4-002 closes when Rows 1-6 all pass on a gate-lifted substrate,
with the two "OK" rows (2, 3) verifying byte-identical destination
content, and the three "refused/timeout" rows (4, 5, 6) verifying
zero destination artifact and non-zero exit.

## 7. Not in scope for M4-002

- **Per-invocation duration override.** cp compiles in
  `CP_ELEVATE_DUR_NS = 60e9`; a `--elevate-timeout` flag is a
  future addition.
- **Cap-tail signing on the widened window.** After elevate
  approval the widened cap tail carries a signature from the
  broker; verifying that signature at cp side is
  libpdx-elevate.M4's scope (not cp.M4).
- **Multiple concurrent elevate requests.** cp is single-file
  today (the dst-parent open is one call). When multi-file copies
  each need a separate elevate (a hypothetical
  `-r --cross-subtree-fan-out`), that batching lives at cp.M5 or
  later.
- **Delegation chains (invoker elevates via a granter who
  themself must re-elevate).** Broker-side concern; cp sees the
  final APR or the propagated ELVC_ERR only.
