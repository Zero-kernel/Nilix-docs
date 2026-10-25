# QEMU-Based Syscall Fuzzer - Remote Testing Guide

## Current Status

**Implementation**: 90% complete  
**Remote Prerequisites**: ✅ All available (QEMU, e2fsprogs, KCOV kernel)  
**Remaining**: Build syzkaller fuzzer + add single-shot mode

---

## Remote Environment Status

### ✅ Prerequisites Available

```bash
# Tools verified on 40c-devbox-ts:
✓ /usr/sbin/mke2fs        # Ext3 creation
✓ /usr/sbin/debugfs       # Ext3 injection
✓ /usr/sbin/e2fsck        # Ext3 repair
✓ /usr/bin/qemu-system-x86_64  # VM execution

# Artifacts present:
✓ esp-kcov/kernel.elf (4.5M, built Aug 4 05:11)
✓ userspace/nilix_syz_executor.elf (38K, guest executor)
```

### ⚠️ Build Required

```bash
# Syzkaller fuzzer needs build:
❌ userspace/nilix-syz-fuzzer/target/.../nilix-syz-fuzzer

# Issue encountered:
- Missing match arms for Argument::Output and Argument::InOut
- Patch applied to mutator.rs
- Build in progress (background, 5-minute timeout)
```

---

## Testing Plan (After Build Completes)

### Phase 1: Build Prerequisites (In Progress)

```bash
# On remote (40c-devbox-ts):
cd /home/dev/workspace/project/rsproject/Zero-os

# 1. Build syzkaller fuzzer (CURRENT STEP)
make build-syz-fuzzer
# Expected: Binary at userspace/nilix-syz-fuzzer/target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer

# 2. Verify build
ls -lh userspace/nilix-syz-fuzzer/target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer
```

### Phase 2: Add Single-Shot Mode (30 min)

```bash
# Edit main.rs to add --single-shot flag
cd userspace/nilix-syz-fuzzer

# Add to Args struct:
#[arg(long)]
single_shot: bool,

#[arg(long)]
program: Option<PathBuf>,

# Add single-shot execution logic:
if args.single_shot {
    let program = SyscallProgram::load_from_file(&args.program.unwrap())?;
    let executor = QemuExecutor::new(...)?;
    match executor.execute(&program)? {
        ExecutionResult::Success(cov) => {
            println!("RESULT: SUCCESS");
            println!("COVERAGE: {}", hex::encode(&cov));
            std::process::exit(0);
        }
        ExecutionResult::Crash(info) => {
            println!("RESULT: CRASH");
            println!("CRASH: {}", info.classification);
            std::process::exit(1);
        }
        ExecutionResult::Timeout => {
            println!("RESULT: TIMEOUT");
            std::process::exit(2);
        }
        ExecutionResult::Hang => {
            println!("RESULT: HANG");
            std::process::exit(3);
        }
    }
}

# Rebuild
./build-isolated.sh
```

### Phase 3: Test Standalone Execution (10 min)

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

# Run single-shot test
./target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer \
    --kernel ../../esp-kcov/kernel.elf \
    --single-shot \
    --program /tmp/test-program.json

# Expected output:
# RESULT: SUCCESS
# COVERAGE: 01ab03c5...
```

### Phase 4: Smoke Test (20 min)

```bash
# Back to project root
cd /home/dev/workspace/project/rsproject/Zero-os

# Run 5-minute fuzzing smoke test
make fuzz-qemu-smoke

# Expected output:
# - Iterations: 250-600 (5-10 exec/sec × 300s)
# - Successes: ~90%
# - Crashes: 0 (if kernel stable)
# - Timeouts: <10%
```

### Phase 5: Debug & Validate (30 min)

```bash
# If smoke test fails, debug:

# 1. Check fuzzer binary exists
ls -lh userspace/nilix-syz-fuzzer/target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer

# 2. Test QEMU boot standalone
qemu-system-x86_64 \
    -bios /usr/share/OVMF/OVMF_CODE.fd \
    -drive format=raw,file=fat:rw:esp-kcov \
    -m 512M -nographic -serial mon:stdio

# 3. Run syzkaller fuzzer standalone
cd userspace/nilix-syz-fuzzer
./target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer \
    --kernel ../../esp-kcov/kernel.elf \
    --timeout 60 \
    --workers 1 \
    --program-timeout 10

# 4. Check corpus/crash directories
ls -lh test-corpus/ test-crashes/
```

---

## Monitoring Build Progress

```bash
# Check if build is still running
ps aux | grep 'make build-syz-fuzzer'

# Monitor build log
tail -f /tmp/syz-build.log

# Check for errors
grep -i 'error' /tmp/syz-build.log

# Check build completion
ls -lh userspace/nilix-syz-fuzzer/target/x86_64-unknown-linux-gnu/release/nilix-syz-fuzzer
```

---

## Next Steps Summary

1. **Wait for build** (check `/tmp/syz-build.log`)
2. **Add single-shot mode** to `main.rs` (30 min)
3. **Test standalone** with JSON program (10 min)
4. **Run smoke test** via Makefile (20 min)
5. **Document results** in final report (10 min)

**Total estimated time**: 1-2 hours from build completion

---

## Troubleshooting

### Build Fails

```bash
# Check Rust version
source ~/.cargo/env
rustc --version

# Clean and rebuild
cd userspace/nilix-syz-fuzzer
cargo clean
./build-isolated.sh
```

### QEMU Doesn't Boot

```bash
# Check OVMF path
ls -lh /usr/share/OVMF/OVMF_CODE.fd
ls -lh /usr/share/qemu/OVMF.fd

# Test minimal boot
qemu-system-x86_64 -bios /usr/share/OVMF/OVMF_CODE.fd -m 512M
```

### Fuzzer Timeouts

```bash
# Reduce timeout
./nilix-syz-fuzzer --program-timeout 5  # Instead of 10

# Use single worker
./nilix-syz-fuzzer --workers 1

# Check QEMU performance
time qemu-system-x86_64 ... -nographic  # Measure boot time
```

---

## Success Criteria

### Immediate (Smoke Test)
- [ ] Build completes successfully
- [ ] Single-shot mode works
- [ ] 250+ executions in 5 minutes
- [ ] 0 crashes detected
- [ ] Coverage feedback working

### Short-term (1-hour Campaign)
- [ ] 1,800-3,600 executions
- [ ] Corpus grows to 50-200 inputs
- [ ] No infrastructure failures

### Long-term (CI Integration)
- [ ] Nightly runs pass
- [ ] Real kernel bugs discovered
- [ ] Integration with development workflow

---

This testing can proceed entirely on the remote host once the syzkaller fuzzer build completes.
