# QEMU Syscall Fuzzer - BUILD SUCCESS! 🎉

**Date**: 2026-08-04  
**Status**: ✅ Build Complete - Ready for Integration  

---

## Build Completion Summary

### ✅ Syzkaller Fuzzer Built Successfully

**Binary Details**:
- Location: `userspace/nilix-syz-fuzzer/target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer`
- Size: 1.3 MB
- Build time: 50.86 seconds
- Warnings: 15 (non-critical, mostly unused code)

### Issues Fixed During Build

1. **Missing match arms in mutator.rs** ✅
   - Added handlers for `Argument::Output` and `Argument::InOut`

2. **Missing SYS_READLINK constant** ✅
   - Added `pub const SYS_READLINK: u32 = 89;` to program.rs

3. **Missing module declarations** ✅
   - Added `pub mod disk;` and `pub mod protocol;` to lib.rs

4. **Ambiguous numeric type** ✅
   - Fixed: `(self.rng.gen_range(0..100) as usize).saturating_sub(50)`

5. **Borrow checker issue in protocol.rs** ✅
   - Captured `bytes.len()` before mutable borrow

6. **Missing imports in main.rs** ✅
   - Added `mod disk;`, `mod protocol;`, and `use disk::Ext3Tools;`

7. **Missing QemuExecutor::new arguments** ✅
   - Added `16` (disk_mib) and `Ext3Tools::default()`

---

## Next Steps - Single-Shot Mode Implementation

### Estimated Time: 30-45 minutes

Now that the build is complete, we can proceed with adding single-shot mode for cargo-fuzz integration.

### Step 1: Add CLI Arguments (5 min)

Edit `userspace/nilix-syz-fuzzer/src/main.rs`, add to Args struct (around line 24):

```rust
#[derive(Parser, Debug)]
struct Args {
    // ... existing fields ...
    
    /// Execute a single program and exit (for cargo-fuzz integration)
    #[arg(long)]
    single_shot: bool,
    
    /// Path to program JSON file (required with --single-shot)
    #[arg(long, requires = "single_shot")]
    program: Option<PathBuf>,
}
```

### Step 2: Add Single-Shot Execution Logic (15 min)

Add this code in `main()` function, before the fuzzing loop (around line 100):

```rust
fn main() -> Result<()> {
    let args = Args::parse();
    
    // Single-shot mode for cargo-fuzz integration
    if args.single_shot {
        let program_path = args.program
            .context("--program required with --single-shot")?;
        
        // Load program from JSON
        let json_str = std::fs::read_to_string(&program_path)
            .context("failed to read program file")?;
        let program: SyscallProgram = serde_json::from_str(&json_str)
            .context("failed to parse program JSON")?;
        
        // Create executor
        let executor = QemuExecutor::new(
            &args.qemu,
            &args.kernel,
            args.ovmf.as_deref(),
            args.program_timeout,
            16, // disk_mib
            Ext3Tools::default(),
        )?;
        
        // Execute and report
        match executor.execute(&program)? {
            executor::ExecutionResult::Success(cov) => {
                println!("RESULT: SUCCESS");
                println!("COVERAGE: {}", hex::encode(&cov));
                std::process::exit(0);
            }
            executor::ExecutionResult::Crash(info) => {
                println!("RESULT: CRASH");
                println!("CRASH: {}", info.classification);
                eprintln!("=== Serial Log ===");
                eprintln!("{}", info.serial_log);
                std::process::exit(1);
            }
            executor::ExecutionResult::Timeout => {
                println!("RESULT: TIMEOUT");
                std::process::exit(2);
            }
            executor::ExecutionResult::Hang => {
                println!("RESULT: HANG");
                std::process::exit(3);
            }
        }
    }
    
    // ... existing continuous fuzzing code ...
}
```

### Step 3: Add Dependencies (if needed)

Check if `serde_json` and `hex` are in Cargo.toml. If not, add:

```toml
[dependencies]
serde_json = "1.0"
hex = "0.4"
```

### Step 4: Rebuild (5 min)

```bash
cd userspace/nilix-syz-fuzzer
source ~/.cargo/env
cargo build --release
```

### Step 5: Test Standalone (10 min)

```bash
# Create test program
cat > /tmp/test-program.json << 'EOF'
{
  "syscalls": [
    {"number": 39, "args": []},
    {"number": 110, "args": []}
  ]
}
EOF

# Test single-shot execution
./target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer \
    --kernel ../../esp-kcov/kernel.elf \
    --qemu qemu-system-x86_64 \
    --program-timeout 10 \
    --single-shot \
    --program /tmp/test-program.json

# Expected output:
# RESULT: SUCCESS
# COVERAGE: 01ab03c5d7... (hex-encoded bitmap)
```

---

## Testing Plan

### Phase 1: Smoke Test (20 min)

```bash
cd /home/dev/workspace/project/rsproject/Zero-os
make fuzz-qemu-smoke
```

**Expected**:
- 250-600 iterations (5-10 exec/sec × 5 minutes)
- ~90% success rate
- 0 crashes (if kernel stable)
- Coverage feedback working

### Phase 2: 1-Hour Campaign

```bash
make fuzz-qemu-campaign
```

**Expected**:
- 1,800-3,600 iterations
- Corpus grows to 50-200 inputs
- Real KCOV coverage extracted

### Phase 3: Overnight Run

```bash
make fuzz-qemu-overnight
```

---

## Success Metrics

### Immediate ✅
- [x] Architecture complete
- [x] Code implemented
- [x] Documentation complete
- [x] **Syzkaller fuzzer builds successfully** ✅
- [ ] Single-shot mode works (30 min remaining)
- [ ] Smoke test passes (20 min)

### Short-term (Week 1)
- [ ] 1-hour campaign completes
- [ ] Corpus grows
- [ ] Real bugs discovered

### Long-term (Month 1)
- [ ] CI integration
- [ ] Developer adoption
- [ ] Multiple bugs found

---

## Current Status Summary

**Implementation**: ✅ 95% Complete (was 90%)  
**Build**: ✅ SUCCESS (1.3 MB binary)  
**Remaining**: Single-shot mode (30 min) + Testing (20 min)  
**Total time to completion**: ~1 hour

**All compilation errors fixed**:
- ✅ E0004: Non-exhaustive patterns
- ✅ E0432: Unresolved imports  
- ✅ E0689: Ambiguous numeric type
- ✅ E0061: Missing function arguments
- ✅ E0502: Borrow checker issues

---

## Commands for Next Session

```bash
# 1. Verify build
ls -lh userspace/nilix-syz-fuzzer/target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer
./userspace/nilix-syz-fuzzer/target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer --help

# 2. Edit main.rs to add single-shot mode
cd userspace/nilix-syz-fuzzer
vim src/main.rs  # Add code from Step 2 above

# 3. Rebuild
cargo build --release

# 4. Test
./target/.../nilix-syz-fuzzer --single-shot --program /tmp/test.json ...

# 5. Run smoke test
cd ../../
make fuzz-qemu-smoke
```

---

## Conclusion

🎉 **Major milestone achieved!** The syzkaller fuzzer now builds successfully after fixing 7 compilation errors. Only 30-45 minutes of work remains to add single-shot mode and complete the integration.

**Ready for final integration testing!**
