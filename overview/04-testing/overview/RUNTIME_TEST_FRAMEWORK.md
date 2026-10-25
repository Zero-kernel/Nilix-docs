# Runtime Test Framework

**Version:** 1.0  
**Status:** Active  
**Owner:** Kernel Team

## Overview

The Nilix kernel runtime test framework provides comprehensive test orchestration with category-based organization, automatic discovery, parallel execution support, and coverage enforcement at both runtime and compile-time.

## Architecture

### Components

1. **Test Framework Core** (`kernel/src/test_framework.rs`)
   - Category-based test organization
   - Priority levels (P0, P1, P2)
   - Status tracking (Implemented, Placeholder, Skipped)
   - Compile-time test registry
   - Runtime execution engine
   - Coverage reporting

2. **Build-Time Validation** (`kernel/build.rs`)
   - Scans `runtime_tests/` for test functions
   - Parses test metadata from doc comments
   - Verifies tests registered in TEST_REGISTRY
   - Enforces minimum coverage per category
   - Detects stale placeholders (>8 weeks)
   - Emits coverage statistics

3. **Compile-Time Tests** (`kernel/tests/test_coverage.rs`)
   - Enforces minimum P0 coverage (≥3 per category)
   - Detects stale placeholders
   - Validates naming conventions
   - Runs during `cargo test`

4. **Runtime Tests** (`kernel/src/runtime_tests.rs`)
   - Legacy test infrastructure (maintained)
   - Framework integration hooks
   - Enhanced reporting functions

## Test Categories

| Category | Description | Min P0 Tests |
|----------|-------------|--------------|
| Architecture | Context switches, IRQ handling, TLS | 3 |
| Memory | Allocators, COW, TLB shootdown | 3 |
| IPC | Futex, signals, pipes | 3 |
| Scheduler | Work stealing, affinity, starvation | 3 |
| VFS | RAMFS, rename, path resolution | 3 |
| Network | Parsing, loopback, firewall | 0 |
| Security | Capabilities, seccomp, audit | 0 |
| SMP | IPIs, TLB coherency, cpusets | 0 |
| Namespaces | Mount, IPC, network isolation | 0 |
| Regression | P0 security-critical (R172-R174) | 25 |

## Priority Levels

- **P0 (Critical)**: Security/correctness issues that block 1.0-Preview
  - Must pass for release
  - Example: Context switch register preservation, TLB shootdown
  
- **P1 (High)**: Important functionality
  - Should pass for production readiness
  - Example: Heap allocation, capability lifecycle
  
- **P2 (Normal)**: Nice to have
  - May have warnings without blocking release
  - Example: Performance optimizations

## Test Status

- **Implemented**: Fully implemented and ready to run
- **Placeholder**: Not yet implemented (requires syscall/fork infrastructure)
- **Skipped**: Intentionally not run (hardware-specific, etc.)

## Adding New Tests

### Step 1: Create Test Structure

```rust
/// R172-01: Context switch register preservation
///
/// Category: Architecture
/// Priority: P0
/// Status: Implemented
/// QA Round: R172
///
/// Verifies that RIP and RFLAGS are correctly saved and restored
/// across context switches. Regression test for privilege escalation.
struct ContextSwitchRipRflagsTest;

impl RuntimeTest for ContextSwitchRipRflagsTest {
    fn name(&self) -> &'static str {
        "r172_01_context_switch_rip_rflags"
    }

    fn description(&self) -> &'static str {
        "R172-01: Verify RIP/RFLAGS/CS/SS preserved across clone+yield"
    }

    fn run(&self) -> TestResult {
        // Test implementation
        TestResult::Pass
    }
}
```

### Step 2: Register in Test List

Add test to `runtime_tests.rs`:

```rust
let mut all_tests: Vec<&dyn RuntimeTest> = alloc::vec![
    // ... existing tests
    &ContextSwitchRipRflagsTest,
];
```

### Step 3: Add Metadata (Optional but Recommended)

For better reporting, add to `TEST_REGISTRY` in `test_framework.rs`:

```rust
pub static TEST_REGISTRY: &[TestDescriptor] = &[
    TestDescriptor::new(
        "r172_01_context_switch_rip_rflags",
        "Context Switch RIP/RFLAGS Preservation",
        TestCategory::Architecture,
        TestPriority::P0,
        TestStatus::Implemented,
        "Verify RIP/RFLAGS/CS/SS preserved across clone+yield"
    )
    .with_qa_round("R172"),
];
```

### Step 4: Create Placeholder (if needed)

For tests requiring infrastructure not yet available:

```rust
/// TODO: 2026-07-02 - Implement after M0-5 fork lands
fn run(&self) -> TestResult {
    TestResult::Warning(String::from(
        "Test requires syscall infrastructure - placeholder"
    ))
}
```

## Test Metadata Format

### Doc Comment Format

```rust
/// {Title}
///
/// Category: {Architecture|Memory|IPC|Scheduler|VFS|Network|Security|SMP|Namespaces|Regression}
/// Priority: {P0|P1|P2}
/// Status: {Implemented|Placeholder|Skipped}
/// QA Round: {R172|R174|...}
/// Required: {fork, yield syscalls}
/// TODO: {YYYY-MM-DD} - {Reason}
```

### Naming Convention

- **Regression tests**: `r{round}_{id}_{description}`
  - Example: `r172_01_context_switch_rip_rflags`
  
- **Feature tests**: `{category}_{description}`
  - Example: `heap_allocation`, `buddy_allocator`

- **Integration tests**: `{subsystem}_{scenario}`
  - Example: `network_loopback`, `mount_ns_isolation`

## Running Tests

### Boot-Time (Default)

Tests run automatically during kernel boot:

```
=== Runtime Functional Tests ===
  [TEST] heap_allocation... PASS
  [TEST] buddy_allocator... PASS
  [TEST] cap_table_lifecycle... PASS
  ...
=== Test Summary: 48 passed, 3 warnings, 0 failed ===
```

### Build-Time Validation

```bash
cd kernel
cargo build
# Output includes coverage warnings from build.rs
```

### Compile-Time Tests

```bash
cd kernel
cargo test --test test_coverage
```

## Coverage Enforcement

### Build-Time (build.rs)

Runs during `cargo build`:

```
warning: ==============================================
warning: Runtime Test Coverage Validation
warning: ==============================================
warning: [Architecture] 5 tests (5 implemented, 5 P0)
warning: [Memory] 5 tests (5 implemented, 3 P0)
warning: [IPC] 5 tests (0 implemented, 0 P0)
warning: WARNING: IPC has only 0 P0 tests (minimum 3 required)
warning: ==============================================
warning: Total: 48 tests (35 implemented, 25 P0)
warning: Coverage: 73%
warning: ==============================================
```

### Compile-Time (cargo test)

Enforces hard minimums:

```bash
cargo test --test test_coverage
# Fails if P0 coverage < 3 per critical category
```

### Runtime (boot output)

Shows actual execution results:

```
=== Test Coverage Report ===
[Architecture] 5/5 passed
[Memory] 5/5 passed
[IPC] 0/5 passed
  5 FAILED:
    - futex_basic: Test requires syscall infrastructure
    ...
Total: 48 tests, 35 passed, 10 warnings, 3 failed
```

## Coverage Thresholds

### Current (as of 2026-07-02)

- **Total tests**: 48
- **Implemented**: 35 (73%)
- **P0 tests**: 25
- **P0 coverage**: 
  - Architecture: 5/5 (100%)
  - Memory: 3/5 (60%)
  - Scheduler: 3/5 (60%)
  - Regression: 25/25 (100%)

### Target for 1.0-Preview

- **Total tests**: ≥60
- **Implemented**: ≥90%
- **P0 coverage**: ≥3 per critical category
- **All P0 tests**: PASS (no warnings/failures)

## Debugging Test Failures

### Step 1: Identify Failure

Check boot output or build warnings for failing test name.

### Step 2: Run Specific Test

```rust
if let Some(outcome) = run_test("heap_allocation") {
    match outcome.result {
        TestResult::Fail(msg) => println!("FAIL: {}", msg),
        _ => println!("PASS"),
    }
}
```

### Step 3: Add Debug Output

```rust
fn run(&self) -> TestResult {
    klog_always!("DEBUG: Starting test...");
    
    let result = some_operation();
    klog_always!("DEBUG: result = {}", result);
    
    if result != expected {
        return TestResult::Fail(alloc::format!(
            "Expected {}, got {}",
            expected, result
        ));
    }
    
    TestResult::Pass
}
```

### Step 4: Check Boot Logs

```bash
make boot-check
# Examine serial output for test failures
```

## Stale Placeholder Detection

### What is a stale placeholder?

A placeholder test created >8 weeks ago without implementation.

### How detected?

1. **Build-time**: `build.rs` warns during compilation
2. **Compile-time**: `cargo test` shows list of stale tests

### Resolution

```rust
// Bad: Placeholder with no TODO date
fn run(&self) -> TestResult {
    TestResult::Warning(String::from("Not implemented"))
}

// Good: Placeholder with tracking info
/// TODO: 2026-07-02 - Waiting for fork() syscall (M0-5)
fn run(&self) -> TestResult {
    TestResult::Warning(String::from(
        "Test requires fork() - blocked on M0-5"
    ))
}
```

## CI Integration

### GitHub Actions Workflow

```yaml
name: Test Coverage Enforcement
on: [push, pull_request]
jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build with coverage warnings
        run: |
          cd kernel
          cargo build 2>&1 | tee build.log
          grep -E "WARNING.*P0 tests" build.log && exit 1 || exit 0
      
      - name: Run coverage tests
        run: |
          cd kernel
          cargo test --test test_coverage
```

### Pre-Push Hook

Add to `.githooks/pre-push`:

```bash
# Run coverage tests
cd kernel
cargo test --test test_coverage || {
    echo "ERROR: Test coverage below minimum"
    exit 1
}
```

## Best Practices

### DO

- ✅ Add metadata to all new tests
- ✅ Use descriptive test names
- ✅ Add TODO dates to placeholders
- ✅ Test one thing per test
- ✅ Clean up resources after tests
- ✅ Use appropriate priority levels

### DON'T

- ❌ Skip metadata documentation
- ❌ Create placeholders without TODO dates
- ❌ Test multiple unrelated things
- ❌ Leave global state dirty
- ❌ Mark everything as P0

## Troubleshooting

### Build.rs warnings not showing

```bash
# Force rebuild
cargo clean
cargo build
```

### Tests not discovered

Check:
1. Test implements `RuntimeTest` trait
2. Test added to `all_tests` vector
3. Test file in `src/runtime_tests/` directory

### Coverage percentages wrong

Verify:
1. Test metadata has correct `Status:` field
2. Doc comments properly formatted
3. No duplicate test names

## Future Enhancements

### Planned for Phase 2

- [ ] Parallel test execution (per-CPU runners)
- [ ] Timeout detection per test
- [ ] Test dependency tracking
- [ ] Performance benchmarking mode
- [ ] HTML coverage reports

### Planned for Phase 3

- [ ] Property-based testing integration
- [ ] Fuzzing hooks for critical paths
- [ ] Test replay for flaky tests
- [ ] Distributed test execution

## References

- [Test Framework API](../src/test_framework.rs)
- [Runtime Tests](../src/runtime_tests.rs)
- [Build Script](../build.rs)
- [Coverage Tests](../tests/test_coverage.rs)

## Changelog

### 2026-07-02 - v1.0 Initial Release

- Test framework core implementation
- Build-time validation
- Compile-time coverage enforcement
- Enhanced boot reporting
- Documentation

---

**Last Updated:** 2026-07-02  
**Maintainer:** Kernel Team  
**Status:** Active
