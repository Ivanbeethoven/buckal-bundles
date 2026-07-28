# ScorpioFS, Buckal, and Buck2 Compatibility Notes

## Scope

This document records failures observed while building rk8s Mega changelist
`1XFJ4PGK` through a ScorpioFS Antares mount. The representative target is:

```text
buck2 build //:aardvark-dns
```

The tested runtime used ScorpioFS `0.2.4`, `libfuse-fs` `0.1.15`, and
`asyncfuse` `0.1.12`.

## Summary

There are three independent compatibility layers:

| Layer | Owner | Status |
| --- | --- | --- |
| Relative Buck-generated tool-shim paths in Rust build scripts | Buckal | Fixed by upstream PR #12 |
| Compiler execution from the bundled Buck2 Prelude | Buck2 | Requires an upstream Buck2 fix or a patched binary/prelude |
| Buck2 Prelude API compatibility with the selected Buckal cell | Buck2 + Buckal version pair | Must be validated for each Buck2 upgrade |

ScorpioFS itself successfully served the Antares mount. The remaining build
failures are build-rule and toolchain integration failures above the FUSE
filesystem layer.

## 1. Relative `AR`, `CC`, `CXX`, and `LD` shim paths

### Symptom

Rust `build.rs` actions receive tool variables such as:

```text
CC=buck-out/.../__cc_shim.sh
AR=buck-out/.../__ar_shim.sh
```

Build scripts can change their working directory before `cc-rs` invokes the
tool. A project-root-relative `buck-out/...` path is then resolved relative to
the build-script directory and cannot be found.

### Root cause

`tool/buildscript_run.py` received a relative Buck output path and passed it
into the build-script environment unchanged. This is independent of the real
C compiler and independent of ScorpioFS.

### Resolution

Upstream [buck2hub/buckal-bundles PR #12](https://github.com/buck2hub/buckal-bundles/pull/12)
is merged. It normalizes `AR`, `CC`, `CXX`, and `LD` values beginning with
`buck-out` to absolute paths before the build script is launched.

Mega must pin a Buckal external-cell commit that contains this change. A merge
in `buckal-bundles` alone does not update repositories still pinned to an
older `external_cell_buckal.commit_hash`.

## 2. Buck2 Prelude compiler-launch failure

### Symptom

After the Buckal shim-path fix, the build reaches the `ring` build script but
the generated C compiler shim exits with status 1:

```text
Action failed: root//third-party/rust/crates/ring/0.17.14:build-script-run
error occurred in cc-rs: command did not execute successfully
.../__cc_shim.sh ... sha256-x86_64-elf.S
```

The absolute `.../__cc_shim.sh` path in the `cc-rs` error is important: it
shows that the Buckal relative-path failure was already bypassed.

### Root cause

The shim is generated from Buck2's bundled Prelude helper:

```text
prelude/rust/tools/from_any_dir.py
```

The helper changes directory and then uses:

```python
os.execl(cc[0], cc[0], *cc[1:])
```

`execl` does not search `PATH`. When `cc[0]` is a compiler name rather than an
absolute executable path, the invocation can fail after the directory change.
This is a Buck2 Prelude issue; Buckal does not own this generated helper.

### Verified temporary workaround

Changing the helper to the following allowed the `2026-04-15` Buck2 build to
complete successfully through ScorpioFS:

```python
os.execvp(cc[0], cc)
```

The temporary, materialized copy can be found below the mounted project's
`buck-out`, for example:

```text
buck-out/buck-isolation-<id>/art/prelude/rust/tools/__from_any_dir__/*/from_any_dir.py
```

That file is regenerated and must not be treated as a permanent fix.

### Durable resolution

Submit the one-line change to `facebook/buck2` at:

```text
prelude/rust/tools/from_any_dir.py
```

Until an upstream release includes it, use a custom Buck2 binary built with
the patch. An alternative is vendoring the entire Buck2 Prelude in Mega and
removing `prelude = bundled`, but that creates a substantial maintenance
burden and is not recommended.

Changing Buckal again is only a fragile workaround: it would need to duplicate
Buck2's generated-shim behavior and still could not cover every Prelude tool
invocation.

## 3. Latest Buck2 and old Buckal API mismatch

### Symptom

The latest tested Buck2 binary was:

```text
buck2 2026-07-28-704fe0539afb8dd117f0e0ef5ad35cef8f45b0c8
```

With an older pinned Buckal cell, it failed during rule loading before any
compilation started:

```text
From load at toolchains/buckal-bundles/cargo_buildscript.bzl:37
File not found: prelude//rust/tools/buildscript_platform.bzl
```

### Root cause and action

The newer bundled Prelude no longer exposes the API expected by that older
Buckal revision. Update Mega to a Buckal commit compatible with the selected
Buck2 version, then rerun the build. Do not infer that a newer Buck2 release
solves the compiler-launch issue in section 2: the current Buck2 Prelude still
uses `os.execl` in `from_any_dir.py`.

## 4. ScorpioFS and Antares operational findings
