# PostgreSQL Optimized Build Pipeline

## Overview

This document describes the optimized build pipeline that uses AutoFDO (Automatic Feedback-Directed Optimization), ThinLTO (Thin Link-Time Optimization), and BOLT (Binary Optimization and Layout Tool) to build high-performance PostgreSQL binaries.

## Optimization Techniques

### 1. ThinLTO (Thin Link-Time Optimization)

ThinLTO is a scalable link-time optimization technique that enables whole-program optimization with lower memory overhead than traditional LTO.

**Benefits:**
- Cross-module inlining and optimization
- Improved code generation through better global analysis
- Faster build times compared to full LTO
- Better parallelization during compilation

**Implementation:**
- Enabled via Clang with `-flto=thin` flag
- Uses LLD linker for best performance
- Applied during both initial and final builds

### 2. AutoFDO (Automatic Feedback-Directed Optimization)

AutoFDO uses hardware performance counters to collect runtime profiles without instrumentation overhead.

**Benefits:**
- No need for instrumented builds
- Profile data from real workloads
- Better branch prediction and code layout
- Improved inlining decisions based on actual usage

**Implementation:**
- Collects profiles using Linux `perf` tool
- Converts profiles using Google's `create_llvm_prof` tool
- Applies profiles during compilation with `-fprofile-sample-use` flag

**Workflow:**
1. Build initial binary with ThinLTO
2. Run representative workload while collecting perf data
3. Convert perf data to LLVM AutoFDO format
4. Rebuild with profile-guided optimizations

### 3. BOLT (Binary Optimization and Layout Tool)

BOLT is a post-link optimizer that rearranges code layout for better instruction cache performance.

**Benefits:**
- Improved instruction cache utilization
- Better branch prediction through layout optimization
- Function and basic block reordering
- Code splitting (hot/cold separation)

**Implementation:**
- Applied as final post-processing step
- Uses `llvm-bolt` tool on compiled binary
- Performs layout optimizations:
  - Function reordering with hfsort+
  - Basic block reordering with ext-tsp
  - Function splitting for hot/cold paths
  - Identical code folding (ICF)

## CI/CD Pipeline

The optimized build pipeline is implemented as a GitHub Actions workflow (`.github/workflows/optimized-build.yml`).

### Pipeline Stages

```
┌─────────────────────────────────────────────────────┐
│ Stage 1: Initial Build with ThinLTO                │
├─────────────────────────────────────────────────────┤
│ - Configure with Clang and ThinLTO enabled         │
│ - Build PostgreSQL with whole-program optimization │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Stage 2: Profile Collection                         │
├─────────────────────────────────────────────────────┤
│ - Initialize PostgreSQL database                    │
│ - Run representative workload                       │
│ - Collect performance profile with perf            │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Stage 3: AutoFDO Profile Conversion                │
├─────────────────────────────────────────────────────┤
│ - Convert perf.data to LLVM AutoFDO format         │
│ - Generate .afdo profile file                      │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Stage 4: Optimized Build with AutoFDO + ThinLTO   │
├─────────────────────────────────────────────────────┤
│ - Rebuild with AutoFDO profile                     │
│ - Apply ThinLTO for whole-program optimization     │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Stage 5: BOLT Post-Link Optimization               │
├─────────────────────────────────────────────────────┤
│ - Apply BOLT to final binary                       │
│ - Optimize code layout and basic blocks            │
│ - Split hot/cold code paths                        │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Stage 6: Verification                               │
├─────────────────────────────────────────────────────┤
│ - Test optimized binary                            │
│ - Run basic smoke tests                            │
│ - Generate optimization report                     │
└─────────────────────────────────────────────────────┘
```

## Expected Performance Improvements

Based on industry benchmarks for similar optimizations:

- **ThinLTO alone**: 5-10% performance improvement
- **AutoFDO on top of ThinLTO**: Additional 2-5% improvement
- **BOLT optimization**: Additional 2-5% improvement
- **Combined**: 10-20% overall performance improvement

Actual results may vary depending on workload characteristics.

## Limitations and Notes

### CI Environment Limitations

1. **Perf Access**: Hardware performance counters may have limited access in CI environments
   - The pipeline gracefully handles cases where perf is unavailable
   - In such cases, only ThinLTO and BOLT optimizations are applied

2. **Representative Workload**: The CI pipeline uses basic SQL operations for profiling
   - For production deployments, collect profiles from actual workloads
   - More representative profiles lead to better optimization results

3. **Build Time**: The optimized build takes longer than standard builds
   - Multiple compilation passes required
   - Profile collection and conversion add overhead
   - Consider running on dedicated optimization workflows

### Requirements

- **Compiler**: LLVM/Clang 18 or later
- **Linker**: LLD (LLVM linker)
- **Tools**: 
  - `perf` for profile collection
  - `create_llvm_prof` from google/autofdo
  - `llvm-bolt` for binary optimization
- **OS**: Linux (for perf support)

## Customization

### Adjusting Workload for Profiling

To use custom workloads for profiling, modify the "Collect performance profile" step:

```bash
# Replace basic SQL operations with your workload
./postgres -D /tmp/pgdata &
PG_PID=$!

# Run your application workload here
# Examples:
# - Import production-like data
# - Run typical queries
# - Execute stored procedures
# - Perform transactions

sudo perf record -e cycles:u -g -o /tmp/perf.data -p $PG_PID -- sleep 30
kill $PG_PID
```

### BOLT Optimization Options

The BOLT step can be tuned with different options:

```bash
llvm-bolt postgres \
  -o postgres.bolt \
  -reorder-blocks=ext-tsp \      # Basic block reordering algorithm
  -reorder-functions=hfsort+ \   # Function reordering algorithm
  -split-functions \             # Enable function splitting
  -split-all-cold \              # Aggressive cold code splitting
  -dyno-stats \                  # Print optimization statistics
  -icf=1                         # Identical code folding level
```

## Usage

### Automatic Trigger

The workflow runs automatically on:
- Push to `main` or `master` branches
- Pull requests to `main` or `master` branches

### Manual Trigger

You can manually trigger the workflow:

1. Go to GitHub Actions tab
2. Select "Optimized Build with AutoFDO/ThinLTO/BOLT" workflow
3. Click "Run workflow"
4. Select branch and click "Run workflow"

## References

- [Google AutoFDO](https://github.com/google/autofdo)
- [LLVM ThinLTO](https://clang.llvm.org/docs/ThinLTO.html)
- [LLVM BOLT](https://github.com/llvm/llvm-project/tree/main/bolt)
- [Profile-Guided Optimization Guide](https://clang.llvm.org/docs/UsersManual.html#profile-guided-optimization)

## Troubleshooting

### Issue: Perf cannot collect data in CI

**Solution**: This is expected in many CI environments due to security restrictions. The pipeline will skip AutoFDO optimization and continue with ThinLTO and BOLT.

### Issue: BOLT fails with relocation errors

**Solution**: Ensure the binary is built with relocations preserved. Add `-Wl,--emit-relocs` to LDFLAGS if needed.

### Issue: Build takes too long

**Solution**: 
- Reduce parallelism with `-j` flag
- Use smaller workload for profiling
- Consider caching build artifacts
- Run optimization workflow separately from regular builds

## Future Enhancements

Potential improvements to the optimization pipeline:

1. **Profile Collection from Production**: Integrate real production profiles
2. **Multi-workload Profiles**: Merge profiles from different workload types
3. **Iterative Optimization**: Multiple AutoFDO + BOLT passes
4. **Propeller Integration**: Add Google's Propeller for advanced layout optimization
5. **Architecture-specific Tuning**: Optimize for specific CPU microarchitectures
6. **Caching**: Cache intermediate artifacts to speed up rebuilds
