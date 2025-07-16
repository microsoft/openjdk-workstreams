# G1 Time-Based Heap Uncommit

A novel implementation of time-based heap memory uncommit for the G1 Garbage Collector in OpenJDK.

## Overview

This project introduces a time-based heap uncommit mechanism that operates independently of garbage collection cycles. Instead of relying solely on traditional GC-triggered heap shrinking, this implementation tracks region activity over time and proactively uncommits idle memory regions to reduce memory footprint during low-activity periods.

## Key Features

- **Independent Operation**: Works independently of GC cycles using a background `PeriodicTask`
- **Temporal Tracking**: Tracks region activity using timestamps rather than allocation-based metrics
- **Configurable Parameters**: Evaluation interval, uncommit delay, and minimum region thresholds
- **Non-Invasive**: Minimal impact on normal GC operations and application performance with defaults
- **Safepoint Safety**: Uses VM operations for safe memory uncommitting

## Performance Results

Based on SPECjbb2015 benchmarks, the implementation shows significant memory reclamation. Here's an example:

- **Initial Heap**: 7168MB
- **Peak Uncommitting**: 704MB in a single evaluation (88 regions)
- **Final Stable Size**: 4704MB (34% reduction from peak)
- **Evaluation Frequency**: Every 60 seconds
- **Uncommit Threshold**: 5-minute idle time

### Key Metrics

```
Time-based uncommit sequence (21:26-21:33):
├── 21:26:42 → 96MB  (12 regions from 49 inactive)
├── 21:27:42 → 704MB (88 regions from 376 inactive) 
├── 21:28:42 → 488MB (61 regions from 246 inactive)
├── 21:29:42 → 344MB (43 regions from 174 inactive)
├── 21:30:42 → 168MB (21 regions from 84 inactive)
├── 21:31:42 → 112MB (14 regions from 56 inactive)
├── 21:32:42 → 56MB  (7 regions from 30 inactive)
└── 21:33:42 → 24MB  (3 regions from 15 inactive)

Heap size progression:
7168MB → 7072MB → 6368MB → 5880MB → 5184MB → 5016MB → 4904MB → 4848MB → 4704MB
```

## Documentation Structure

```
G1-time-based-uncommit/
├── docs/
│   ├── technical-overview.md       # Detailed implementation explanation
│   └── tuning-guide.md            # Usage and tuning parameters
├── benchmarks/
│   └── specjbb-analysis.md        # Comprehensive SPECjbb2015 analysis
└── graphs/
    ├── heap evaluation timeline.png    # G1 heap evaluation process and decisions over time
    ├── region transitions.png          # Region state transitions and uncommit rate analysis
    └── sizing activity summary.png     # Memory uncommit patterns and evaluation summary
```

### Performance Analysis Charts

The `graphs/` folder contains detailed performance visualization charts from SPECjbb2015 testing:

- **[Heap Evaluation Timeline](graphs/heap%20evaluation%20timeline.png)** - Shows heap sizing evaluation decisions, shrink operations, and heap size progression over time
- **[Region Transitions Analysis](graphs/region%20transitions.png)** - Displays region state transitions, G1 uncommit rate analysis, and cumulative memory reclaimed
- **[Sizing Activity Summary](graphs/sizing%20activity%20summary.png)** - Memory uncommit patterns over time, reclaimed distribution, and evaluation summary statistics

These charts provide visual evidence of the time-based uncommit feature's effectiveness during comprehensive benchmark testing.

## System Requirements

- **GC**: G1 Garbage Collector required (`-XX:+UseG1GC`) with experimental features enabled
- **Platform**: Linux, Windows, macOS (all platforms supported)

## Quick Start

### Enabling Time-Based Uncommit

```bash
# Basic configuration
-XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -XX:+G1UseTimeBasedHeapSizing

# Custom configuration with parameter explanations
-XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -XX:+G1UseTimeBasedHeapSizing \
-XX:G1TimeBasedEvaluationIntervalMillis=60000 \      # Evaluate heap every 60 seconds
-XX:G1UncommitDelayMillis=300000 \                   # Wait 5 minutes before uncommitting idle regions
-XX:G1MinRegionsToUncommit=10                        # Only uncommit if ≥10 regions are eligible
```

### Parameter Guide

| Parameter | Default | Description | Tuning Guidance |
|-----------|---------|-------------|-----------------|
| `G1TimeBasedEvaluationIntervalMillis` | 60000ms | How often to check for uncommit opportunities | Increase for lower overhead, decrease for faster response |
| `G1UncommitDelayMillis` | 300000ms | Minimum idle time before region uncommit | Increase to avoid thrashing, decrease for faster reclamation |
| `G1MinRegionsToUncommit` | 10 | Minimum regions needed to trigger uncommit | Increase for stability, decrease for smaller workloads |

**Uncommit Algorithm Details:**
- Only empty regions are candidates for uncommit
- Maximum uncommit per evaluation: smaller of 25% of inactive regions or 10% of total committed regions
- Heap size never reduced below `InitialHeapSize` or `MinHeapSize`
- Evaluation skipped during stop-the-world GC operations

### Monitoring Output

Enable relevant logging to observe time-based uncommit activity:

```bash
-Xlog:gc,gc:sizing,gc:heap:gc.log:time,pid,tid
```

**Expected log patterns:**

*Initialization:*
```
[info][gc,init] G1 Time-Based Heap Sizing enabled (uncommit-only)
[info][gc,init]   evaluation_interval=60000ms, uncommit_delay=300000ms, min_regions_to_uncommit=10
```

*Task and evaluation lifecycle:*
```
[debug][gc,init] G1 Time-Based Heap Evaluation task created (PeriodicTask)
[debug][gc,init] G1 Time-Based Heap Evaluation task enrolled (PeriodicTask)
[debug][gc,sizing] Starting heap evaluation
```

*No action needed (info level - every 10th evaluation):*
```
[info][gc,sizing] Time-based evaluation: no heap uncommit needed (evaluation #10)
```

*No action needed (trace level - every evaluation with details):*
```
[trace][gc,sizing] Time-based heap evaluation: no uncommit needed (inactive=5 min_required=10 heap=4704MB min=256MB)
```

*Active uncommitting (with detailed logging sequence):*
```
[debug][gc,sizing] Full region scan: found 49 inactive regions out of 1750 total regions
[debug][gc,sizing] Uncommit candidates found: 49 inactive regions out of 1750 total regions
[debug][gc,sizing] Region state transition: 49 regions found eligible for uncommit after scan
[info][gc,sizing] Time-based uncommit: found 49 inactive regions, uncommitting 12 regions (96MB)
[debug][gc,sizing] Time-based heap uncommit evaluation: Found 49 inactive regions out of 1750 total regions, target shrink: 100663296B (max allowed: 7264362496B)
[debug][gc,sizing] Region state transition: 12 regions selected for uncommit
[info][gc,sizing] Time-based evaluation: shrinking heap by 96MB
[info][gc,heap] Heap shrink completed: uncommitted 12 regions (96MB), heap size now 7072MB
[debug][gc,heap] Heap shrink details: requested=100663296B attempted=100663296B actual=100663296B regions_removed=12 heap_capacity=7516192768B
```

*Individual region uncommit evaluation (trace level):*
```
[trace][gc,sizing] Region 1234 uncommit check: elapsed=350000ms threshold=300000ms last_access=1738876543210 now=1738876893210 empty=true
[debug][gc,sizing] Region state transition: Region 1234 transitioning from active to inactive after 350000ms idle
```

### What to Expect

**Timeline for results:**
- Initial evaluation begins after first `G1TimeBasedEvaluationIntervalMillis` (default: 60 seconds after JVM startup)
- First uncommit opportunities appear after `G1UncommitDelayMillis` (default: 5 minutes of region idle time)

**Success indicators:**
- Log messages showing evaluation activity (note: "no heap uncommit needed" only logged every 10th evaluation)
- Gradual heap size reduction during application idle periods  
- Stable heap size once optimal working set is reached
- No performance degradation in application metrics

### When to Use Time-Based Uncommit

**Ideal Workloads:**
- Long-running applications with variable memory usage patterns
- Batch processing jobs with idle periods between operations
- Container environments with memory constraints
- Multi-tenant applications with periodic activity cycles
- Applications with distinct load phases (startup, processing, idle)

## Troubleshooting

### Common Issues

**No uncommit activity observed:**
- Verify evaluation interval and uncommit delay settings
- Check that regions have sufficient idle time (default: 5 minutes)
- Ensure minimum threshold is appropriate for workload (`G1MinRegionsToUncommit`)
- Look for periodic log messages indicating evaluations are running:
  ```
  [debug][gc,sizing] Starting heap evaluation
  [debug][gc,sizing] Full region scan: found X inactive regions out of Y total regions
  ```
- Check for "no uncommit needed" messages (logged every 10th evaluation when no action taken):
  ```
  [info][gc,sizing] Time-based evaluation: no heap uncommit needed (evaluation #10)
  ```
- For detailed monitoring of every evaluation, enable trace-level logging to see:
  ```
  [trace][gc,sizing] Time-based heap evaluation: no uncommit needed (inactive=5 min_required=10 heap=4704MB min=256MB)
  ```
  This shows exactly how many inactive regions were found vs. the minimum threshold required
- If you see these messages, the evaluation period, uncommit delay, and minimum uncommit threshold may need adjustment for your specific workload

**Excessive uncommitting/thrashing:**
- Increase `G1MinRegionsToUncommit` threshold (default: 10)
- Extend `G1UncommitDelayMillis` for more conservative behavior
- Monitor application allocation patterns

**Performance impact concerns:**
- Reduce evaluation frequency with longer `G1TimeBasedEvaluationIntervalMillis`
- Use debug logging to verify overhead is minimal
- Monitor system memory pressure if uncommitting very frequently

### Debug Logging

For detailed analysis of time-based uncommit behavior:

```bash
# Comprehensive GC logging
-Xlog:gc*:gc.log:time,pid,tid

# Focus on sizing decisions (includes info-level periodic messages)
-Xlog:gc+sizing:gc-sizing.log:time,pid,tid

# Debug-level sizing information (shows candidate selection details)
-Xlog:gc+sizing:debug:gc-sizing-debug.log:time,pid,tid

# Detailed trace-level evaluation information (shows every evaluation)
-Xlog:gc+sizing:trace:gc-sizing-trace.log:time,pid,tid

# Heap state tracking
-Xlog:gc+heap:gc-heap.log:time,pid,tid
```

**Logging Levels:**
- **Info level**: Shows successful uncommit operations, periodic "no uncommit needed" messages (every 10th evaluation), and main uncommit activity
- **Debug level**: Shows detailed evaluation process including:
  - Task lifecycle (creation, enrollment)
  - Full region scanning results 
  - Candidate selection process
  - Region state transitions
  - Heap shrink details with byte-level accounting
  - Uncommit calculations and thresholds
- **Trace level**: Shows detailed evaluation information on every evaluation, including:
  - Individual region uncommit checks with timestamps
  - Per-region elapsed time calculations  
  - Empty region validation
  - Complete evaluation context when no uncommit is needed

## Status

**Current Scope and Requirements:**
- **Experimental Status**: Requires `-XX:+UnlockExperimentalVMOptions`
- **G1 Only**: Currently focused on G1 garbage collector
- **Uncommit Only**: Current implementation focuses on memory reclamation, not proactive expansion
- **Memory Overhead**: Per-region timestamp tracking (8 bytes × number of regions, e.g., ~384KB for 1.5TB heap with 32MB regions)
- **Platform Dependencies**: Uncommit effectiveness depends on OS memory management

## Key Implementation Components

1. **G1HeapEvaluationTask** - Background periodic task for heap evaluation
2. **G1HeapSizingPolicy** - Core logic for uncommit candidate identification
3. **G1HeapRegion** - Region-level activity timestamp tracking
4. **VM_G1ShrinkHeap** - Safepoint operation for safe memory uncommitting

## Related Work

- **[JEP 8360935](https://bugs.openjdk.org/browse/JDK-8360935)**: Time-based heap sizing proposal
- **[JDK-8357445](https://bugs.openjdk.org/browse/JDK-8357445)**: Enhancement tracking issue
- **Implementation Branch**: [GitHub comparison view](https://github.com/openjdk/jdk/compare/master...mo-beck:jdk:feature/JDK-8357445-time-based-heap-sizing) - Shows complete implementation diff (1500+ lines across 21 files)
- **[PR 26240](https://github.com/openjdk/jdk/pull/26240)**: Implementation pull request

## Contributing

This is an experimental feature under active development. See the implementation documentation for technical details and the current status of the feature.

## License

This implementation is part of the OpenJDK project and follows the same licensing terms.
