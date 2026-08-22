# cp

paideia-os copy — atomic single-TXN file / directory-tree copy
with cap-tail re-signing, audit-first journaling, semantic-pipe
progress records, PdxFS undo on overwrite, and cross-subtree
elevate for widening destination-parent writes.

## Install

```
pkg install cp
```

Pulls the dual-signed 1.0.0 release from `pkgs.paideia-os/main/
cp/1.0.0/`. `pkg` verifies both the author signature and the
paideia_root signature (ML-DSA-65, per `design/user/model.md` §2)
before writing anything to `/system/packages/cp/`. See
`.plans/pkgs-mirror-push.md` for the mirror layout and the
release-flow discipline.

## Documentation

```
doc cp
```

Renders `doc/cp.pdxdoc` — synopsis, description, options, exit
codes, capabilities, output schemas, audit + undo semantics,
POSIX differences, and a concrete example gallery.

## Status

Version 1.0.0 (M5 landed; signing pipeline pending paideia-as
v0.33-crypto). See `STATUS.md` for the per-milestone map,
`CHANGELOG.md` for the per-issue release notes, and
`design/tooling/r49-r50-plan.md` §5.6 in the
[paideia-os](https://github.com/paideia-os/paideia-os) repo for
the wave-level breakdown, KIND allocations, and cross-repo
dependencies.

## Layout

- `caps.decl` — capability manifest (baseline cap set: KIND_USER,
  KIND_IPC_ENDPOINT, KIND_PDXFS_FILE ×2, KIND_PDXFS_TXN).
- `manifest.pdxproj` — paideia-as build manifest (version 1.0.0).
- `manifest.pdxsig` — dual-signature block (author + paideia_root)
  over the release payload.
- `CHANGELOG.md` — per-version change log; 1.0.0 aggregates every
  M1-M5 issue.
- `doc/cp.pdxdoc` — long-form documentation for `doc cp`.
- `design/` — per-milestone design docs (architecture, argv
  surface, pdxfs stub notes).
- `src/` — .pdx sources (12 modules; one responsibility each).
- `tests/` — test-spec markdown files (18 matrix rows across
  three specs).
- `.plans/` — per-milestone rollup notes + mirror-push manifest.

## License

MIT — see LICENSE.
