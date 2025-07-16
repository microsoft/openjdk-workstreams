# Technical Overview: G1 Time-Based Heap Uncommit

## Abstract

The G1 Time-Based Heap Uncommit feature introduces a temporal approach to memory uncommit that operates independently of garbage collection cycles. This implementation tracks region activity using timestamps and proactively uncommits idle memory regions through a background periodic task.

## Architecture Overview

### Core Components

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ G1HeapRegion    │    │ G1HeapSizing    │    │ G1HeapEvaluation│
│                 │    │ Policy          │    │ Task            │
│ • _last_access_ │◄──►│                 │◄──►│                 │
│   timestamp     │    │ • evaluate_     │    │ • PeriodicTask  │
│ • record_       │    │   heap_resize() │    │ • WatcherThread │
│   activity()    │    │ • get_uncommit_ │    │ • configurable  │
│ • should_       │    │   candidates()  │    │   interval      │
│   uncommit()    │    │ • should_       │    │ • background    │
│                 │    │   uncommit_     │    │ • task() impl   │
│                 │    │   region()      │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
          │                       │                       │
          └─────────────────────┬─┴───────────────────────┘
                                │
                    ┌─────────────────┐
                    │request_heap_    │
                    │shrink()         │
                    │                 │
                    │ • Fast path:    │
                    │   call shrink() │
                    │ • Slow path:    │
                    │   schedule VM   │
                    │   operation     │
                    └─────────────────┘
                                │
                    ┌─────────────────┐
                    │ VM_G1ShrinkHeap │
                    │                 │
                    │ • extends       │
                    │   VM_Operation  │
                    │ • doit() calls  │
                    │   shrink()      │
                    └─────────────────┘
```

### Key Design Principles

1. **Temporal Independence**: Operations are time-based rather than allocation-based
2. **Background Execution**: Non-blocking periodic evaluation separate from GC threads
3. **Conservative Approach**: Multiple safety checks before uncommitting memory
4. **Configurable Thresholds**: Tunable parameters for different workload characteristics

**The real implementation is in the heap management layer:**

1. **Fast Path Optimization**: `request_heap_shrink()` checks if already at safepoint and executes directly, otherwise schedules VM operation

2. **Region Activity Tracking**: Regions get `record_activity()` calls when cleared/retired, updating `_last_access_timestamp`

3. **Safepoint Coordination**: The allocator code now accepts operations at safepoint OR with heap lock:
   ```cpp
   assert(Heap_lock->owner() != nullptr || SafepointSynchronize::is_at_safepoint(),
          "Should be owned on this thread's behalf or at safepoint.");
   ```

4. **Time-Based Evaluation**: `G1HeapEvaluationTask` runs as `PeriodicTask` on WatcherThread, calling `evaluate_heap_resize()` which implements the core logic

## Implementation Details

### 1. Region Activity Tracking

The core innovation is adding timestamp tracking to `G1HeapRegion`:

```cpp
class G1HeapRegion {
private:
  jlong _last_access_timestamp;  // NEW: tracks last allocation/access time
  
public:
  void record_activity() {
    _last_access_timestamp = os::javaTimeMillis();
  }
  
  jlong last_access_time() const {
    return _last_access_timestamp;
  }
  
  bool should_uncommit(uint64_t delay) const {
    if (!is_empty()) return false;
    jlong elapsed = os::javaTimeMillis() - _last_access_timestamp;
    return elapsed > (jlong)delay;
  }
};
```

**Activity Recording Points**: Regions call `record_activity()` when:
- Retired as mutator allocation region (`retire_mutator_alloc_region`)
- Retired as GC allocation region (`retire_gc_alloc_region`) 
- Initialized/cleared (`hr_clear`)

**Key Insight**: Tracking happens at region lifecycle events, not per-allocation, providing efficient temporal tracking without per-object overhead.

### 2. Periodic Evaluation Task

`G1HeapEvaluationTask` extends `PeriodicTask` to run on the WatcherThread:

```cpp
class G1HeapEvaluationTask : public PeriodicTask {
  G1CollectedHeap* _g1h;
  G1HeapSizingPolicy* _heap_sizing_policy;
  
public:
  G1HeapEvaluationTask(G1CollectedHeap* g1h, G1HeapSizingPolicy* heap_sizing_policy)
    : PeriodicTask(G1TimeBasedEvaluationIntervalMillis), _g1h(g1h), _heap_sizing_policy(heap_sizing_policy) {}
  
  virtual void task() override {
    if (!G1UseTimeBasedHeapSizing || _g1h->is_stw_gc_active()) return;
    
    ResourceMark rm;
    bool should_expand = false;
    size_t resize_amount = _heap_sizing_policy->evaluate_heap_resize(should_expand);
    
    if (resize_amount > 0 && !should_expand) {
      _g1h->request_heap_shrink(resize_amount);
    }
  }
};
```

**Lifecycle**: Created during G1 initialization, enrolled in `post_initialize()` to start periodic execution.

### 3. Heap Sizing Policy - The Core Logic

The real implementation is in `G1HeapSizingPolicy::evaluate_heap_resize()`:

```cpp
size_t G1HeapSizingPolicy::evaluate_heap_resize(bool& expand) {
  expand = false; // Time-based sizing only handles uncommit
  
  if (!G1UseTimeBasedHeapSizing || _g1h->is_stw_gc_active()) return 0;
  
  MutexLocker ml(Heap_lock); // Coordinate with allocations
  ResourceMark rm;
  
  // Scan all regions for uncommit candidates
  GrowableArray<G1HeapRegion*> candidates;
  get_uncommit_candidates(&candidates);
  
  uint inactive_count = candidates.length();
  
  if (inactive_count >= G1MinRegionsToUncommit) {
    size_t current_heap = _g1h->capacity();
    size_t min_heap = MAX2((size_t)InitialHeapSize, MinHeapSize);
    size_t max_shrink_bytes = current_heap > min_heap ? current_heap - min_heap : 0;
    
    if (max_shrink_bytes > 0) {
      // Conservative limits: max 25% of inactive OR 10% of total
      size_t by_inactive = inactive_count / 4;  
      size_t by_total = (_g1h->capacity() / G1HeapRegion::GrainBytes) / 10;
      size_t regions_to_uncommit = MIN2(by_inactive, by_total);
      
      size_t shrink_bytes = MIN2(regions_to_uncommit * G1HeapRegion::GrainBytes, max_shrink_bytes);
      
      // Never reduce below InitialHeapSize
      if (current_heap - shrink_bytes >= InitialHeapSize) {
        return shrink_bytes;
      }
    }
  }
  return 0;
}
```

**Key Algorithm Elements**:
- Only empty regions eligible (prevents corruption)
- Conservative limits prevent thrashing  
- Respects minimum heap size constraints
- Heap_lock synchronization during evaluation

### 4. Fast Path Heap Shrinking

The `request_heap_shrink()` method provides optimization for safepoint scenarios:

```cpp
bool G1CollectedHeap::request_heap_shrink(size_t shrink_bytes) {
  if (shrink_bytes == 0) return false;
  
  // Fast path: if already at safepoint, do work directly
  if (SafepointSynchronize::is_at_safepoint()) {
    shrink(shrink_bytes);
    return true;
  }
  
  // Otherwise, schedule VM operation
  VM_G1ShrinkHeap op(this, shrink_bytes);
  VMThread::execute(&op);
  return true;
}
```

**Thread Safety Updates**: The allocator now accepts operations at safepoint OR with heap lock:
```cpp
// Before: assert(Heap_lock->owner() != nullptr, "Should be owned on this thread's behalf.");
// After:
assert(Heap_lock->owner() != nullptr || SafepointSynchronize::is_at_safepoint(),
       "Should be owned on this thread's behalf or at safepoint.");
```

This allows the fast path to work correctly when already at a safepoint (e.g., during GC operations).

## Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `G1TimeBasedEvaluationIntervalMillis` | 60000ms | Frequency of heap evaluation |
| `G1UncommitDelayMillis` | 300000ms | Idle time before region becomes uncommit candidate |
| `G1MinRegionsToUncommit` | 10 | Minimum regions required to trigger uncommit |

### Conservative Algorithm Implementation

The uncommit algorithm follows a conservative approach:
- **Maximum uncommit per evaluation**: smaller of 25% of inactive regions OR 10% of total committed regions
- **Heap size constraints**: Never reduce below `InitialHeapSize` or `MinHeapSize`
- **Evaluation timing**: Skipped during stop-the-world GC operations
- **Region selection**: Only completely empty regions are eligible

## Performance Characteristics

The implementation is designed for minimal performance impact:

- **Execution**: Runs during application idle periods on WatcherThread
- **Memory Overhead**: 8 bytes per region for timestamp storage
- **Conservative Limits**: Maximum 25% of inactive regions OR 10% of total regions per evaluation
- **Batch Operations**: Multiple regions uncommitted in single VM operation to amortize overhead

The conservative approach prevents thrashing while allowing gradual memory reclamation over multiple evaluation cycles.

## Integration Points

### With G1 Garbage Collector

1. **Region Lifecycle**: Timestamps updated during state transitions
2. **Heap Management**: Integrates with existing heap sizing infrastructure
3. **Memory Operations**: Uses established uncommit mechanisms
4. **Logging**: Extends existing GC logging framework

### With VM Operations

1. **Safepoint Integration**: Leverages VM operation infrastructure
2. **Thread Safety**: Ensures proper synchronization
3. **Error Handling**: Maintains VM stability during failures

## Monitoring and Observability

### Log Output Analysis

The implementation provides detailed logging at multiple levels for analysis:

**Initialization Messages**:
```
[info][gc,init] G1 Time-Based Heap Sizing enabled (uncommit-only)
[info][gc,init]   evaluation_interval=60000ms, uncommit_delay=300000ms, min_regions_to_uncommit=10
```

**Task Lifecycle (debug level)**:
```
[debug][gc,init] G1 Time-Based Heap Evaluation task created (PeriodicTask)
[debug][gc,init] G1 Time-Based Heap Evaluation task enrolled (PeriodicTask)
[debug][gc,sizing] Starting heap evaluation
```

**Evaluation Results (info level - every 10th evaluation)**:
```
[info][gc,sizing] Time-based evaluation: no heap uncommit needed (evaluation #10)
```

**Detailed evaluation context (trace level - every evaluation)**:
```
[trace][gc,sizing] Time-based heap evaluation: no uncommit needed (inactive=5 min_required=10 heap=4704MB min=256MB)
```

**Active uncommitting sequence**:
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

**Individual region evaluation (trace level)**:
```
[trace][gc,sizing] Region 1234 uncommit check: elapsed=350000ms threshold=300000ms last_access=1738876543210 now=1738876893210 empty=true
[debug][gc,sizing] Region state transition: Region 1234 transitioning from active to inactive after 350000ms idle
```

## Monitoring and Error Handling

### Observability

The implementation provides logging at multiple levels (info, debug, trace) to track:
- Initialization and configuration
- Evaluation cycles and results  
- Memory reclamation activity
- Error conditions

### Error Handling

The system handles failures gracefully:
- **VM Operation failures**: No heap corruption, continues normal operation
- **Concurrent access**: Protected by safepoint coordination and heap locks
- **Configuration errors**: Validated at startup

All failures are logged and don't prevent future uncommit attempts.

## Conclusion

The G1 Time-Based Heap Uncommit implementation represents a significant advancement in JVM memory management, providing memory reclamation independent of garbage collection cycles. The design balances efficiency, safety, and configurability while maintaining compatibility with existing G1 operations.

The temporal approach opens new possibilities for memory management optimization and provides a foundation for future enhancements in adaptive heap sizing.
