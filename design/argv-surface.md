# cp — argv surface

**Wave:** R50  Milestone: M1
**Upstream design:** `design/tooling/r49-r50-plan.md` §5.6 in
[paideia-os](https://github.com/paideia-os/paideia-os); D3 (flag
grammar) + I3 (standard flag vocabulary) in
[`design/tooling/plan.md`](https://github.com/paideia-os/paideia-os/blob/main/design/tooling/plan.md)
§3-4.

## 1. Purpose

This document freezes the cp command-line surface at M1. Every M2/M3
body edit inherits this surface unchanged; adding a new flag or a new
positional shape is an M-diff to this document plus a code change
gated on the review of the diff.

The surface is deliberately narrow at M1: two required positionals
(source, destination), four cp-specific flags (three of them
behaviour-less at M1), and the seven I3 standard flags recognised at
parse time. Every real behaviour landed by a flag lives in a later
milestone and inherits this surface as its call site.

## 2. Grammar

cp follows D3 (long primary, short one-per-hyphen — no clustering)
via libpdx-argv. The invocation shape is:

```
cp [--flag ...] [-flag ...] <src> <dst>
```

- `--flag` accepts long-only well-known flags (see §4) and cp's
  long-form flags (`--dry-run`, `--over-existing`). Values follow
  either `--flag=value` or `--flag value` (libpdx-argv lookahead rule;
  see `design/architecture.md` §4 in libpdx-argv).
- `-f` accepts single-letter short flags (`-r`, `-v`). `-rv` is
  rejected as `ERR_CLUSTERED_SHORT` — libpdx-argv M1-002 enforces this
  at parse time; the cp exit code for that class of failure is 2
  (usage).
- Positional arguments are the source path and the destination path,
  in that order. M1 requires exactly two positionals; other counts
  (`0`, `1`, `>=3`) are usage errors.

## 3. cp-specific flags

| Flag              | Form  | M1 status  | Wired at | Behaviour                                                              |
|-------------------|-------|------------|----------|------------------------------------------------------------------------|
| `-r`              | short | recognised | M2       | recursive copy of a directory tree                                     |
| `-v`              | short | recognised | M3-001   | emit CopyProgressRecord[] per file via libpdx-semantic-pipe            |
| `--dry-run`       | long  | recognised | M2       | preview the copy without writing to dst                                |
| `--over-existing` | long  | recognised | M2       | write undo record before overwriting an existing dst file              |

At M1 the parse layer accepts all four. Only `-r` has a body-visible
effect at M1: setting it causes the dispatch layer to exit 3
(`EXIT_NOT_YET_IMPLEMENTED`) with `RECURSIVE_NOT_YET_MSG` on stderr
so a script wiring `cp -r a/ b/` today sees an obvious message and a
distinct exit code rather than silent single-file semantics. `-v`,
`--dry-run`, and `--over-existing` at M1 are recognised-and-ignored —
the parse succeeds, the flags appear in `ParsedArgs::flag_names[]`,
but the M1 copy body reads none of them. This mirrors pkg.M1's
treatment of I3 standard flags: the surface is stable, the flag
bodies fill in behind it without breaking existing call sites.

## 4. Standard flag vocabulary (I3)

Every R49/R50 tool implements the I3 vocabulary with identical
semantics. For cp, the following long flags are recognised by libpdx-
argv at M1 (the flag names appear in `ParsedArgs::flag_names[]`); the
flag bodies land in later milestones:

| Flag           | M1 status  | Wired at | Behaviour                                                            |
|----------------|------------|----------|----------------------------------------------------------------------|
| `--help`       | recognised | M5       | `doc cp --help` after .pdxdoc ships with M5                          |
| `--version`    | recognised | M2       | prints `cp <version> (build <hash>) sig <fingerprint>`               |
| `--json`       | recognised | M3-001   | emit CopyProgressRecord[] as JSON on stdout                          |
| `--schema`     | recognised | M3-001   | print the CopyProgressRecord schema definition and exit              |
| `--verbose`    | recognised | M3-002   | additional diagnostic output on stderr including audit-record ids    |
| `--quiet`      | recognised | M3-002   | suppress non-error diagnostics                                       |
| `--color=`     | recognised | R51+     | color output through pdx-color once that library ships               |
| `--no-cap:`    | recognised | M2       | narrow cap set by explicitly dropping a named cap (per D5)           |
| `--pdx-schema` | recognised | M3-001   | libpdx-argv's well-known flag; sets emit_schema=1 in ParsedArgs      |

At M1 all of these flags are recognised by libpdx-argv (they are long-
form and follow the grammar); cp's own dispatch does not read from
`ParsedArgs::flag_names[]` at M1 and therefore does not enforce any
flag body. Passing `--json` at M1 succeeds at parse and has no
observable effect — the M3 body wires it.

## 5. Exit codes

| Code | Meaning                                                              | M1 sites                                                                       |
|------|----------------------------------------------------------------------|--------------------------------------------------------------------------------|
| `0`  | success                                                              | `CopyFile::copy_single_file` on happy path                                     |
| `1`  | operation failed (open/read/write error, short write)                | `CopyFile::copy_single_file` on any syscall error                              |
| `2`  | usage error (parse fail, wrong positional count, missing arg)        | `CpMain` parse-fail; `Dispatch::dispatch_copy` positional-count validate       |
| `3`  | recognised shape, body not yet implemented                           | `Dispatch::dispatch_copy` `-r` gate; `Dispatch::dispatch_copy` on pos_count > 2 |
| `4`  | cap denied (reserved; cross-subtree elevate path not yet wired)      | (none at M1 — M3-003 lands this exit on the elevate-refused branch)            |

Codes `3` and `4` are deliberately reserved so a caller can
distinguish "the cp vocabulary knows the flag but has no body yet"
(3) from "the cp vocabulary needs a cap the caller does not have"
(4) from generic "operation failed" (1). Once every flag is wired and
the elevate flow is real, exit code `3` becomes unused; M3 removes
the reservation for the flag bodies that ship in that milestone.

## 6. Runnable examples at M1

Given a freshly-bootstrapped cp build and a source file `/tmp/a`
containing `"hello\n"`:

```
$ cp
cp: usage error (expected exactly two positional arguments)
$ echo $?
2

$ cp /tmp/a /tmp/b
$ echo $?
0
$ cat /tmp/b
hello

$ cp -r /tmp/src/ /tmp/dst/
cp: recursive copy (-r) not yet implemented at M1 (lands at M2)
$ echo $?
3

$ cp --frobnicate /tmp/a /tmp/b
cp: argv parse error (run 'cp --help')
$ echo $?
2
```

Note that `--frobnicate` at M1 is parsed successfully by libpdx-argv
(it is a well-formed long flag) — the error above is illustrative of
a future M2+ shape where cp validates the flag vocabulary against a
known list. At M1 an unknown long flag is silently accepted; the
libpdx-argv M2 upgrade adds the vocabulary-validation hook cp will
use.

## 7. What this surface explicitly does not do at M1

- No environment-variable input. Every input comes from argv.
- No config file. Global defaults land at M4 with the test matrix.
- No interactive prompts. cp is scriptable-first at every milestone;
  any consent flow lives in the approver hop in libpdx-elevate, not
  in cp itself.
- No colour output. cp's diagnostics are plain text at every
  milestone; the `pdx-color` library is a post-R50 dep for tools
  that need it.
- No progress bar. `-v` emits typed records via libpdx-semantic-pipe
  (M3-001), not a TUI progress bar. Downstream consumers that want a
  visual progress render subscribe to the schema pipe.
