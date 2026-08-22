# cp — pkgs.paideia-os mirror-push manifest

**Wave:** R50  Milestone: M5-002 (mirror push)
**Upstream:** `design/tooling/r49-r50-plan.md` §5.6 M5 line + §9.3
signing pipeline in paideia-os.

## 1. Purpose

Freeze the mirror-push contract for cp v1.0.0. This document names
the canonical URL layout at `pkgs.paideia-os`, the payload files the
mirror serves, the promotion path from `staging/` to `main/`, the
sha256 gate the fetcher verifies before invoking `ml_dsa_65_verify`,
and the human-in-the-loop review step §9.3 mandates for every
first-party tool release.

`pkg install cp` on a user machine consults this URL layout via the
mirror list `/system/packages/mirrors.pdx` (seeded by pkg.M1). The
fetcher pulls `manifest.pdxsig` first, verifies both signatures
against the locally-trusted `paideia_root_pk` + a
first-run-TOFU'd `author_pk`, then pulls each `payload:` line in
the manifest and verifies each against its named sha256.

## 2. Canonical URL layout

```
https://pkgs.paideia-os/
├── staging/
│   └── cp/
│       └── 1.0.0-rc<N>/           # N=1 after first sign; increments per re-sign
│           ├── manifest.pdxsig
│           ├── cp.tar             # tarball of the payload lines below
│           └── cp.tar.sha256      # sha256 of cp.tar (for pre-verify fetch check)
├── main/
│   └── cp/
│       ├── latest -> 1.0.0/       # symlink; updated by promotion step §5
│       ├── 1.0.0/
│       │   ├── manifest.pdxsig
│       │   ├── cp.tar
│       │   └── cp.tar.sha256
│       └── 1.0.x/                 # bugfix stream once 1.0.1 lands
└── keys/
    ├── paideia_root_pk            # ML-DSA-65 public key (1952 bytes)
    ├── paideia_root_pk.fp         # SHA-256 fingerprint (32 hex chars)
    └── authors/
        └── snunez.pk              # cp author public key
```

Every `manifest.pdxsig` at any URL above is byte-identical to the
in-tree `manifest.pdxsig` at the corresponding release tag with the
three `<pending-*>` fields replaced by their signed values. The
release-flow tooling never rewrites any other line — the payload
manifest, the release_gates block, and every metadata field survive
the sign step unchanged so diff-inspecting an in-tree manifest
against a mirror manifest yields exactly three modified lines.

## 3. Payload tarball contents

`cp.tar` at any URL above unpacks to a `cp/` directory containing
exactly the paths named in `manifest.pdxsig`'s `payload:` lines.
The tarball ordering is deterministic (paths sorted lexically) so
two independent signers producing tarballs from the same source
tree yield byte-identical archives — sha256 across the two must
match for the reviewer to countersign at §5.

```
cp/
├── build-out/
│   ├── cp                     # the elaborated ELF-like Paideia binary
│   └── cp.caps                # caps-manifest copy for InitCap sidecar
├── caps.decl
├── manifest.pdxproj
├── doc/
│   └── cp.pdxdoc
├── CHANGELOG.md
├── LICENSE
├── README.md
└── STATUS.md
```

Source `.pdx` files, test specs, `.plans/`, and design docs are
NOT included — they live at the source repo (`github.com/paideia-
os/cp` at tag `v1.0.0`) and are fetched by `pkg install cp --from-
source` when a user wants to rebuild locally. The default `pkg
install cp` pulls only the binary + doc + manifest.

## 4. Release-flow steps (§9.3 concrete)

Each step below names the machine it runs on and the artefact it
produces. The steps are strictly sequential — no step begins
before its predecessor's artefact is fully written and the reviewer
(where mentioned) has explicit approval.

### Step 1 — Author machine: tag + build

The author runs `git tag v1.0.0` on the `cp` repo at the M5-close
commit, then invokes `paideia-as build` to produce the `build-out/`
tree. Every `payload:` line's `<pending-build>` in
`manifest.pdxsig` is replaced by the on-disk sha256 of the named
file. `manifest_hash` is computed over the canonical concatenation
of payload lines (per manifest.pdxsig section header comment).

### Step 2 — Author machine: sign

The author unlocks `user_sk` via the Argon2id-KDF passphrase path
(model.md §11.2), then runs `paideia-as release --sign`. This
computes ML-DSA-65 signature over `manifest_hash`, then rewrites
`author_sig` + `author_pk_fp` in `manifest.pdxsig`. The
`paideia_root_sig` line remains `<pending-root-sign>`.

The author then packs `cp.tar` deterministically (paths sorted,
timestamps zeroed, uid/gid = 0/0, mode preserved) and computes
`cp.tar.sha256`.

### Step 3 — Author machine: push to staging

The author pushes `manifest.pdxsig` + `cp.tar` + `cp.tar.sha256` to
`https://pkgs.paideia-os/staging/cp/1.0.0-rc1/` via the mirror-
push endpoint. The staging URL is publicly readable but
`pkg install cp` refuses staging URLs by default — only
`pkg install --staging cp` fetches from there, and only if the
paideia_root_sig field verifies (i.e. never at rc1, but at rc2+
after a re-sign following a review-requested change).

### Step 4 — Reviewer + signing bot: review + re-sign

A human reviewer inspects three things at the staging URL:
- The source repo at tag `v1.0.0` is byte-identical to the tarball
  under `cp/` (unpack cp.tar, sha256 every file, compare against
  the payload manifest — this is the same computation the reviewer
  would run on their own machine from the source tag).
- The CHANGELOG entry accurately enumerates every issue in the
  release; every issue is closed and merged; the M4 test-spec
  matrix is present and lints clean.
- The author_pk_fp matches the value on file at `pkgs.paideia-
  os/keys/authors/snunez.pk` (i.e. the author has not been
  swapped).

On review pass, the reviewer approves the release and the
paideia signing bot (running on the root-key-holding machine)
invokes `ml_dsa_65_sign` with `paideia_root_sk` over the same
`manifest_hash` and rewrites `paideia_root_sig` +
`paideia_root_pk_fp`. The now-fully-signed `manifest.pdxsig` is
copied into place at `pkgs.paideia-os/staging/cp/1.0.0-rc1/`
(overwriting the rc1 manifest — the reviewer's approval turns
rc1 into a re-signed rc1 rather than rc2).

### Step 5 — Signing bot: promote to main

After a 24-hour bake window (during which the community can
inspect the staged release with `pkg install --staging cp`
against the fully-signed rc1), the signing bot copies the three
files from `staging/cp/1.0.0-rc1/` to `main/cp/1.0.0/` and
updates the `main/cp/latest` symlink to point at `1.0.0/`. This
is the moment `pkg install cp` on a stock user machine picks up
the release on next `pkg upgrade` or explicit install.

The staging URL remains readable for one 1.0.x cycle (for
diff-inspection against later releases); it is garbage-collected
when 1.0.1 promotes to main.

## 5. Fetcher-side install path

`pkg install cp` on a user machine (once pkg.M4 lands the install
path — see plan doc §5.1) walks:

1. Consult `/system/packages/mirrors.pdx` for a live mirror URL.
2. Fetch `<mirror>/main/cp/latest/manifest.pdxsig`.
3. Verify `author_sig` against the locally-trusted
   `author_pk_fp` (TOFU'd on first install; explicit on
   `pkg install cp --author-pk=<fp>`).
4. Verify `paideia_root_sig` against
   `/system/keys/paideia_root_pk` (installed at OS-image-build
   time; rotatable via a signed `KeyRotationRecord`).
5. Fetch `<mirror>/main/cp/latest/cp.tar` + `.sha256`; verify
   the sha256 against the fetched tarball; verify each unpacked
   file's sha256 against the payload manifest.
6. Write the tarball contents to `/system/packages/cp/1.0.0/`;
   update `/system/packages/latest/cp` symlink; run cp's
   `caps.decl` through the loader-side manifest verify
   (libpdx-cap.M2 OK|MISSING|EXTRA compare).
7. Journal an `InstallRecord` to `/system/audit/user-events/`
   naming the version, both signature fingerprints, and the
   consenting-user id (D3 audit-first).

Any failure at any step aborts the install with a specific
diagnostic; nothing rolls forward to the next step on a soft-fail.
`pkg install cp` is atomic — either the install lands fully or the
filesystem is byte-identical to its pre-install state (I5 undo
invariant applies to installs, not just user-visible ops).

## 6. Mirror SLA

The paideia signing bot runs the promotion pipeline within 72
hours of a signed rc landing at staging. `pkgs.paideia-os/main/`
availability target is 99.9% (three-9s) measured over rolling 30
days; the mirror-list in `/system/packages/mirrors.pdx` includes
two backup mirrors so a primary outage does not block installs.

Mirror rotation: every 1.0.x release stays at
`main/cp/1.0.x-latest` for 90 days after supersession; deep
archive is at `pkgs.paideia-os/archive/cp/` and remains fetchable
via `pkg install cp@1.0.0` (explicit version pin).

## 7. Substrate-landing gates (M5-002-specific)

| Gate                                    | Blocks                                                              |
|-----------------------------------------|----------------------------------------------------------------------|
| pkg.M4 install path                    | Every fetcher-side step in §5.                                       |
| paideia signing bot                     | Every root-signature step in §4 (step 4).                            |
| `pkgs.paideia-os` staging + main hosts | Every URL under §2.                                                  |
| paideia-as v0.33-crypto                | `manifest.pdxsig` populate at §4 (step 2) + verify at §5 (steps 3, 4). |
| M4 harness + reviewer discipline        | §4 step 4 review checklist.                                          |

Every gate lifts under body-edit of the release-flow tooling; no
in-tree cp change is required to lift a gate. The mirror-push
manifest is stable at 1.0 and inherited unchanged by 1.0.x
bugfixes.

## 8. What this manifest explicitly does not do

- **Does not commit signed bytes.** The signing bot produces
  those; this manifest ships the layout + the URLs + the review
  discipline that the bot follows. Diff-inspection against a live
  mirror at `pkgs.paideia-os/main/cp/1.0.0/manifest.pdxsig` is the
  runtime check.
- **Does not host the mirror.** `pkgs.paideia-os` infrastructure
  is stood up separately (per §9.3); the manifest names the URL
  layout the fetcher expects, not the hosting substrate.
- **Does not bootstrap the trust root.** The `paideia_root_pk` at
  `<mirror>/keys/paideia_root_pk` is installed at OS-image-build
  time (see model.md §2); this manifest assumes it is present.
- **Does not describe the mirror-list rotation policy.** Additions
  + removals to `/system/packages/mirrors.pdx` are a pkg concern;
  see `paideia-os/pkg` at pkg.M5 (`pkg keys` documentation of the
  paideia_root_pk fingerprint) for the operator-side rotation
  discipline.
