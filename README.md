# cp

paideia-os copy with PdxFS undo record (I5 invariant)

## Status

M1 in progress. See `STATUS.md` for the milestone map, and
`design/tooling/r49-r50-plan.md` §5.6 in the
[paideia-os](https://github.com/paideia-os/paideia-os) repo for the
wave-level breakdown, KIND allocations, cross-repo dependencies, and
per-milestone issue set.

## Layout

- `caps.decl` — capability manifest for exec-time verification.
- `manifest.pdxproj` — paideia-as build manifest.
- `design/` — per-milestone design docs (architecture, argv surface,
  pdxfs stub notes).
- `src/` — .pdx sources.
- `tests/` — placeholder; the M4 test suite lands here.
- `.plans/` — per-milestone rollup notes.

## License

MIT — see LICENSE.
