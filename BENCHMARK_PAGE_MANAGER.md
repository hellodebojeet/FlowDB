# Page Manager Benchmark Plan

## Benchmark Goals
Measure the performance characteristics of the PageManager under realistic database workloads, focusing on:
1. Throughput (operations per second) for pin/unpin operations
2. Latency distribution (P50, P95, P99) for pin and unpin operations
3. Memory usage and allocation overhead
4. Scalability with increasing concurrent threads

## Test Scenarios

### 1. Sequential Workload
**Description**: Simulates a table scan workload where pages are accessed in sequential order.
- Threads pin pages in increasing order (page 0, 1, 2, ...) then unpin in same order
- Represents OLAP scan workloads and sequential index traversals
- **Metrics**: 
  - Throughput: ops/sec (pin+unpin pairs)
  - Latency: average and tail latency per operation
  - Cache efficiency: measured via hardware counters (if available) or inferred from latency

### 2. Random Workload
**Description**: Simulates random index lookups or OLTP workloads.
- Each thread repeatedly:
  1. Selects a random page number within current allocated range
  2. Pins the page
  3. Performs a dummy operation (to simulate work)
  4. Unpins the page
- **Metrics**:
  - Throughput: ops/sec
  - Latency: P50, P95, P99 for pin and unpin separately
  - Contention: measured via thread stall time or retry rates in CAS loops

### 3. High-Concurrency Workload
**Description**: Stress test with many threads competing for the same pages.
- Fixed set of hot pages (e.g., 10 pages)
- Large number of threads (e.g., 2x, 4x, 8x hardware threads) repeatedly:
  1. Pin a random hot page
  2. Immediately unpin it
- **Metrics**:
  - Throughput: ops/sec as thread count increases
  - Latency: tail latency (P99) under contention
  - Scalability: throughput vs. thread count curve
  - Retry rate: percentage of CAS operations that fail and require retry

### 4. Allocation Workload
**Description**: Measures performance during active page allocation.
- Background thread continuously allocates new pages (calling EnsureCapacity/AllocPage)
- Foreground threads perform random pin/unpin on existing pages
- **Metrics**:
  - Allocation throughput: pages allocated/sec
  - Impact on foreground latency: compare to baseline without allocation
  - Memory usage: total memory consumed by pin count array

### 5. Long-Running Stability Test
**Description**: Tests for memory leaks and gradual performance degradation.
- Mixed workload of pin/unpin and allocation for extended period (e.g., 1 hour)
- **Metrics**:
  - Memory growth over time
  - Throughput stability (coefficient of variation)
  - Any increase in latency or retry rates

## Expected Bottlenecks

### 1. Seqlock Contention
- **Where**: During concurrent EnsureCapacity calls or when writers block readers
- **Impact**: Under high allocation rates, writers may cause readers to retry frequently
- **Mitigation Considered**: 
  - Using sequence locks minimizes writer-blocking-readers time
  - Allocation is infrequent (<1% of ops) so impact should be low
  - **Validation**: Measure retry rates in allocation-heavy scenarios

### 2. Cache Line Ping-Ponging
- **Where**: When multiple threads pin/unpin pages that share cache lines
- **Impact**: False sharing degrades performance significantly in random workloads
- **Mitigation**: 
  - Cache-line padding in paddedPin structure
  - **Validation**: Compare padded vs. unpadded versions under random workload

### 3. CAS Retry Loops
- **Where**: In Pin/Unpin operations under high contention
- **Impact**: High retry rates waste CPU cycles and increase latency
- **Mitigation**:
  - Exponential backoff in retry loops (not implemented; would add complexity)
  - **Validation**: Measure retry rates vs. thread count; if >10% consider backoff

### 4. Memory Allocation During Resize
- **Where**: During EnsureCapacity when array needs to grow
- **Impact**: Stop-the-world pause during copy; allocates large new array
- **Mitigation**:
  - Geometric growth minimizes frequency
  - Copy is O(n) but n is number of pages, not tuples
  - **Validation**: Measure 99th percentile latency spikes during allocation bursts

## Optimization Strategy Based on Results

### If False Sharing Dominates (evident in random workload):
- Consider adaptive padding: only pad regions with high access density
- Use page access histograms to identify hot regions and apply padding selectively
- Tradeoff: increased complexity for better memory utilization

### If Seqlock Contention High (allocation-heavy workloads):
- Batch allocations: allow disk manager to allocate multiple pages at once
- Use per-CPU or per-thread allocation buffers to reduce global allocation frequency
- Tradeoff: increased memory overhead but reduced contention

### If CAS Retry Rates High (>15%):
- Implement lightweight backoff in Pin/Unpin loops:
  - First retry: immediate
  - Second retry: yield
  - Third+: short sleep (futex wait)
- Tradeoff: slightly higher latency at low contention but much better scalability

### If Allocation Latency Spikes Noticeable:
- Pre-allocate page ranges based on workload forecasts
- Use huge pages for the pin count array to reduce TLB pressure
- Tradeoff: complexity vs. reduced latency variance

## Success Criteria
1. Sequential workload: >1M ops/sec with <10µs P99 latency
2. Random workload: >500K ops/sec with <50µs P99 latency at 64 threads
3. Allocation impact: <5% throughput degradation on foreground workload during allocation bursts
4. Memory overhead: <10 bytes per page (including padding) for typical page counts (<1M pages)
5. No memory leaks or growing latency in long-running tests

## Implementation Note
These benchmarks would be implemented using Go's testing/benchmark framework with:
- `testing.B` for microbenchmarks
- Custom harnesses for multi-threaded scenarios
- `runtime/pprof` for latency measurements
- `golang.org/x/exp/rand` for deterministic random workloads