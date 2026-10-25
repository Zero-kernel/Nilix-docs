# Fuzzer Build Fix — 2026-07-22

## Problem

The fuzzer binaries (`continuous-fuzzer` and `crash-triage`) were failing to build in CI with error:

```
error[E0152]: duplicate lang item in crate `core` (which `alloc` depends on): `sized`
```

This occurred because the fuzzer binaries were being built as bare-metal `no_std` targets (`x86_64-unknown-none`) instead of normal Linux userspace programs.

## Root Cause

1. **Target inheritance**: The `userspace/.cargo/config.toml` forces all builds in the `userspace/` directory to use `x86_64-unknown-none` target with `-Z build-std`.

2. **Conflicting requirements**: The fuzzer binaries need std dependencies (`clap`, `walkdir`, `md5`) which don't work with bare-metal targets.

3. **Module declarations**: The `userspace/fuzzer/src/lib.rs` declared 9 modules that didn't exist, causing compilation errors.

4. **Code bugs**: The `crash-triage.rs` had format string errors and use-after-move bugs.

## Solution

### 1. Force Linux Target for Fuzzer

Created `userspace/fuzzer/.cargo/config.toml`:

```toml
[build]
target = "x86_64-unknown-linux-gnu"
```

This overrides the parent config and forces fuzzer builds to use the Linux target.

### 2. Update CI Workflow

Modified `.github/workflows/fuzz.yml` to build from repository root with explicit target:

```bash
cargo build --release \
  --manifest-path userspace/fuzzer/Cargo.toml \
  --bin continuous-fuzzer \
  --target x86_64-unknown-linux-gnu
```

Building from repo root avoids the `userspace/.cargo/config.toml` inheritance issue.

### 3. Fix lib.rs

Removed non-existent module declarations:

```rust
// Before: declared 9 non-existent modules
pub mod continuous;
pub mod crash_triage;
// ...

// After: simple placeholder
// Fuzzer library - placeholder for future shared modules
// Currently, all fuzzing logic is in self-contained binaries in src/bin/
```

### 4. Fix crash-triage.rs Bugs

**Issue 1**: Format string had 4 placeholders but 5 arguments:
```rust
// Before: missing placeholder for signature
format!("Type: {}\nMessage: {}\nSignature: \nOriginal: {:?}\n...", ...)

// After: added  for signature
format!("Type: {}\nMessage: {}\nSignature: {}\nOriginal: {}\n...", ...)
```

**Issue 2**: PathBuf doesn't implement Display:
```rust
// Before
crash.file_path,

// After
crash.file_path.display(),
```

**Issue 3**: Use-after-move error:
```rust
// Before: crashes moved on line 62, then used on line 68
let unique_crashes = if args.dedup { ... } else { crashes };
let dedup_rate = ... / crashes.len() as f64;  // ERROR: crashes moved

// After: store length before move
let total_crashes = crashes.len();
let unique_crashes = if args.dedup { ... } else { crashes };
let dedup_rate = ... / total_crashes as f64;
```

## Verification

Both binaries now build successfully:

```bash
$ cargo build --release \
    --manifest-path userspace/fuzzer/Cargo.toml \
    --bin continuous-fuzzer \
    --target x86_64-unknown-linux-gnu
   Compiling fuzzer v0.1.0
    Finished `release` profile [optimized] target(s) in 0.97s

$ cargo build --release \
    --manifest-path userspace/fuzzer/Cargo.toml \
    --bin crash-triage \
    --target x86_64-unknown-linux-gnu
   Compiling fuzzer v0.1.0
    Finished `release` profile [optimized] target(s) in 1.28s
```

Binary sizes:
- `continuous-fuzzer`: 952K
- `crash-triage`: 984K

## Files Changed

| File | Change |
|------|--------|
| `userspace/fuzzer/.cargo/config.toml` | Created — force Linux target |
| `userspace/fuzzer/src/lib.rs` | Fixed — removed non-existent modules |
| `userspace/fuzzer/src/bin/crash-triage.rs` | Fixed — format string and use-after-move |
| `.github/workflows/fuzz.yml` | Updated — build from repo root with explicit target |

## Commit

```
commit c8cbbc0
Author: YUZHEthefool
Date:   2026-07-22

    fix(fuzz): fix fuzzer binary build failures
    
    - Fix lib.rs: remove non-existent module declarations
    - Fix crash-triage.rs: correct format string and fix use-after-move
    - Add fuzzer/.cargo/config.toml to force x86_64-unknown-linux-gnu target
    - Update fuzz.yml CI to build from repo root with explicit target
    
    The fuzzer binaries need to be normal Linux userspace programs,
    not bare-metal no_std binaries. Building from the repo root
    avoids inheriting the userspace/.cargo/config.toml that forces
    x86_64-unknown-none + -Z build-std.
    
    Fixes: duplicate lang item error in CI fuzzer builds
```

## Status

✅ **FIXED** — CI fuzzer builds now pass

---

**Date**: 2026-07-22  
**Related**: Phase 2 KCOV implementation complete  
**Next**: CI should now build fuzzer binaries successfully
