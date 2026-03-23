# FlowDB Page Manager Design

## Core Responsibilities
1. Track pin counts for allocated pages to prevent premature eviction from buffer pool
2. Initialize pin count to 0 when a new page is allocated by the disk manager
3. Provide thread-safe pin/unpin operations with overflow/underflow protection
4. Do NOT manage free lists or disk allocation (handled by disk manager)
5. Do NOT track dirty state (handled by buffer pool)

## Data Structures
### Page Descriptor Table
- `pin_counts: Vec<AtomicU32>` - Vector of atomic pin counters indexed by page number
- `length: AtomicUsize` - Current valid length of pin_counts vector
- `resize_mutex: Mutex<()>` - Mutex for vector resizing operations

### Per-Page State (stored in pin_counts[page_no])
- `pin_count: u32` - Number of times page is currently pinned (0 = not pinned)
  - Range: 0 to u32::MAX-1 (u32::MAX reserved for error detection)

## Memory Layout Decisions
- Each pin counter: 4 bytes (AtomicU32)
- Vector grows exponentially (doubling) when resizing to minimize amortized cost
- Cache alignment: Each AtomicU32 naturally aligned to 4-byte boundary
- False sharing mitigation: Pages accessed concurrently are likely spaced far apart in vector (page numbers differ significantly)
- No padding between elements - maximizes pin counts per cache line (typically 64 bytes = 16 counters)
- Tradeoff: Vector resizing requires mutex acquisition, potentially blocking concurrent allocations. Chosen because allocations are infrequent (<1% of operations) versus pin/unpin which are extremely frequent.

## Concurrency Model
### Pin Operation (thread-safe, lock-free for hot path)
1. Read current vector length via `length.load(Acquire)`
2. If page_no < length: proceed to step 4
3. Else: 
   - Acquire resize_mutex
   - Recheck length (double-checked locking)
   - If still insufficient: resize vector to max(page_no+1, length*2), new elements initialized to 0
   - Release resize_mutex
4. Load pin counter: `pin_counts[page_no].load(Acquire)`
5. If value == u32::MAX: return PinOverflow error
6. Attempt CAS: `compare_exchange_weak(old, old+1, Acquire, Relaxed)`
7. On failure: retry from step 4
8. On success: return Ok(())

### Unpin Operation (thread-safe, lock-free)
1. Read current vector length via `length.load(Acquire)`
2. If page_no >= length: return PageNotAllocated error (should not happen if caller validated)
3. Load pin counter: `pin_counts[page_no].load(Acquire)`
4. If value == 0: return NotPinned error
5. Attempt CAS: `compare_exchange_weak(old, old-1, Acquire, Relaxed)`
6. On failure: retry from step 3
7. On success: return Ok(())

### Vector Resizing (requires mutex)
- Only triggered when accessing page_no >= current length
- Multiple threads may contend for mutex during allocation bursts
- After mutex acquisition, recheck length to avoid redundant resizing
- New elements zero-initialized (valid initial state)

## Failure Scenarios
### Crash During Pin
- **Scenario**: Power failure after CAS succeeds but before instruction completes
- **Effect**: Pin count may be incremented in memory but not persisted
- **Recovery**: On restart, all pin counts reset to 0
- **Correctness**: Safe because no transactions survive crash; any pinned page at crash time belongs to aborted transaction
- **Tradeoff**: Loses pin count precision but maintains safety invariant (no false positives for pinned pages)

### Crash During Unpin
- **Scenario**: Power failure after CAS succeeds but before instruction completes
- **Effect**: Pin count may be decremented in memory but not persisted
- **Recovery**: On restart, all pin counts reset to 0
- **Correctness**: Safe because over-unpinning is impossible (we prevent underflow); any undercount at crash time belongs to aborted transaction's cleanup

### Vector Resize Failure
- **Scenario**: OOM during vector resize
- **Effect**: Pin operation returns error
- **Correctness**: Caller must handle as allocation failure (propagate up)
- **Tradeoff**: Fail-stop rather than corrupting state; simpler than trying to preserve partial state

### Pin Count Overflow
- **Scenario**: More than u32::MAX-1 concurrent pins on same page
- **Effect**: Pin operation returns PinOverflow error
- **Correctness**: Prevents undefined behavior from wraparound
- **Tradeoff**: Arbitrary limit (4B-1 pins) chosen as practically unlimited; real systems hit other limits first

## Interaction With Other Components
### Buffer Pool -> Page Manager
- **Calls**: `pin_page(page_no)` before reading/modifying page
          `unpin_page(page_no)` after finishing with page
- **Invariants**: 
  - Buffer pool must call unpin for every successful pin
  - Buffer pool must not access page after unpin without re-pinning
  - Page manager guarantees: if pin count > 0, page cannot be deallocated by disk manager

### Disk Manager -> Page Manager
- **Calls**: `init_page(page_no)` immediately after allocating new page
- **Invariants**:
  - Disk manager must call init_page before making page available for allocation
  - Page manager guarantees: init_page sets pin count to 0 (clean state)
  - Disk manager must not allocate page with non-zero pin count

### Recovery Manager -> Page Manager
- **Implicit**: On startup, recovery manager replays log
- **Effect**: Recovery manager's page accesses will trigger pin/unpin via buffer pool
- **Invariants**:
  - After crash, all pin counts are 0 (initialized by page manager on restart)
  - Recovery process correctly pins pages during log replay
  - No special handling needed in page manager for recovery

## Rejected Designs
### Combined Pin/Dirty/Free State in Single AtomicU64
- **Reason**: Required persisting state across crashes for correctness
- **Problem**: Page manager state (in memory) not recoverable after crash without complex logging
- **Tradeoff**: Simpler concurrent operations but unacceptable recovery complexity

### Separate Free List Tracking in Page Manager
- **Reason**: Duplicated responsibility with disk manager
- **Problem**: Required synchronizing two free lists (memory + disk) 
- **Tradeoff**: Slightly faster allocation avoidance but complex crash consistency

### Lock-Based Per-Page Pinning
- **Reason**: Mutex per page would cause excessive memory usage
- **Problem**: 100M pages would require ~800MB just for mutexes (assuming 8 bytes each)
- **Tradeoff**: Simpler locking but prohibitive memory overhead

### Hazard Pointers for Page Reclamation
- **Reason**: Overkill for pin counting use case
- **Problem**: Complexity not justified by access patterns
- **Tradeoff**: Wait-free pinning but significantly increased implementation complexity
