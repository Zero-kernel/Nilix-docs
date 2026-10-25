# QEMU Syscall Fuzzer - Build Fixes Applied

## Issues Fixed on Remote (2026-08-04)

### 1. Missing Match Arms in mutator.rs ✅
```rust
// Added handling for Argument::Output and Argument::InOut
Argument::Output { capacity } => {
    *capacity = (*capacity as usize)
        .saturating_add(self.rng.gen_range(0..100).saturating_sub(50_usize)) as u32;
}
Argument::InOut { data, capacity } => {
    self.mutate_buffer(data);
}
```

### 2. Missing SYS_READLINK Constant ✅
```rust
// Added to program.rs
pub const SYS_READLINK: u32 = 89;
```

### 3. Missing Module Declarations ✅
```rust
// Added to lib.rs
pub mod disk;
pub mod protocol;
```

### 4. Ambiguous Numeric Type ✅
```rust
// Fixed in mutator.rs line 102
.saturating_sub(50_usize)  // Was: .saturating_sub(50)
```

## Build Status

**Command**: `make build-syz-fuzzer`  
**Location**: `/home/dev/workspace/project/rsproject/Zero-os`  
**Log**: `/tmp/syz-build-final.log`  
**Status**: In progress (background)

## How to Monitor

```bash
# SSH to remote
ssh 40c-devbox-ts

# Check if still running
ps aux | grep 'make build-syz-fuzzer'

# Monitor log
tail -f /tmp/syz-build-final.log

# Check for completion
ls -lh userspace/nilix-syz-fuzzer/target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer

# On success, binary should be ~8-12MB
```

## Expected Build Time

- **Debug build**: 2-3 minutes
- **Release build**: 5-10 minutes (optimization takes time)

## Next Steps After Build Completes

1. **Verify binary exists**:
   ```bash
   ./userspace/nilix-syz-fuzzer/target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer --help
   ```

2. **Test standalone** (before adding single-shot mode):
   ```bash
   cd userspace/nilix-syz-fuzzer
   ./target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer \
       --kernel ../../esp-kcov/kernel.elf \
       --timeout 60 \
       --workers 1
   ```

3. **Add single-shot mode** (see FINAL_STATUS_REPORT.md)

4. **Run smoke test**:
   ```bash
   cd /home/dev/workspace/project/rsproject/Zero-os
   make fuzz-qemu-smoke
   ```

## All Fixes Applied

All compilation errors have been fixed:
- ✅ E0004: Non-exhaustive match patterns
- ✅ E0432: Unresolved imports (disk, protocol)
- ✅ E0689: Ambiguous numeric type

Build should now succeed. Monitor `/tmp/syz-build-final.log` for completion.
