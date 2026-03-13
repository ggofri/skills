# TigerStyle Rules Reference

Extended rule details for the TigerStyle skill. This document provides deeper rationale and concrete examples for each rule category.

---

## Safety Rules — Extended

### S1: Assert Everything

**Rule:** Assert the full contract of every function — preconditions, postconditions, invariants. Assert that impossible states never occur, not just that expected states do occur.

**Rationale:** An unasserted assumption is an undocumented assumption. When conditions change (new caller, new input range, refactor), silent violations corrupt state without a clear crash signal. TigerBeetle's philosophy: it is always safer to crash than to continue with unknown state.

**Example:**
```zig
fn transfer_create(amount: u64, account_id: u128) void {
    assert(amount > 0);
    assert(amount <= constants.transfer_amount_max);
    assert(account_id != 0);
    // ... implementation ...
    assert(ledger.balance >= 0); // postcondition
}
```

**Anti-pattern:**
```zig
fn transfer_create(amount: u64, account_id: u128) void {
    if (amount == 0) return; // silently ignoring invalid input
    // ... implementation ...
}
```

---

### S2: Static Allocation

**Rule:** All memory is allocated at startup. The running system never calls malloc/new/allocator after initialization completes.

**Rationale:** Dynamic allocation at runtime can fail, fragment, and produce unpredictable latency spikes (GC pauses, allocator contention). Static allocation means: if the system starts, it will run. Resource exhaustion is detected at startup, not mid-operation.

**Pattern:**
```zig
// All pools sized at init time
const MessagePool = struct {
    messages: [constants.messages_max]Message,
    // ...
};

pub fn init(pool: *MessagePool) void {
    // No allocation — just initialization of already-allocated memory
    for (&pool.messages) |*msg| msg.* = std.mem.zeroes(Message);
}
```

**Anti-pattern:**
```zig
fn handle_request(req: Request) !void {
    var buf = try allocator.alloc(u8, req.size); // ❌ runtime allocation
    defer allocator.free(buf);
}
```

---

### S3: Explicit Bounds

**Rule:** Every resource has an explicit upper bound. Loops cannot run forever. Queues cannot grow unbounded. Connection pools have a fixed size.

**Rationale:** Unbounded growth is a denial-of-service waiting to happen. Explicit bounds make capacity planning possible and prevent runaway resource consumption.

**Pattern:**
```zig
const pipeline_requests_max = 256; // explicit constant

var pipeline: [pipeline_requests_max]Request = undefined;
var pipeline_count: u32 = 0;

fn pipeline_push(req: Request) void {
    assert(pipeline_count < pipeline_requests_max); // assert, don't silently drop
    pipeline[pipeline_count] = req;
    pipeline_count += 1;
}
```

---

### S4: No Recursion

**Rule:** Do not use recursion. If variable-depth traversal is required, use an explicit stack with a fixed maximum depth.

**Rationale:** Recursion hides stack depth. A deep call tree under adversarial input produces a stack overflow with no clear error boundary. An explicit stack with a size assertion is bounded and auditable.

**Pattern:**
```zig
// Tree traversal with explicit stack
const stack_depth_max = 64;
var stack: [stack_depth_max]*Node = undefined;
var stack_top: usize = 0;

stack[stack_top] = root;
stack_top += 1;

while (stack_top > 0) {
    stack_top -= 1;
    const node = stack[stack_top];
    // process node...
    if (node.right) |right| {
        assert(stack_top < stack_depth_max);
        stack[stack_top] = right;
        stack_top += 1;
    }
    // ...
}
```

---

### S5: Minimal Interface Surface

**Rule:** Expose the minimum interface needed. Push control flow decisions up to callers. Push raw data down to callees.

**Rationale:** Every exposed function or field is a contract. Contracts have a maintenance cost. Functions that accept complex inputs and make many decisions internally are hard to test and easy to misuse.

**Corollary — simple return types:** Prefer `bool` (did it work?) over error codes (which error?). Callers that branch on specific error codes are tightly coupled to implementation details. Use error codes only when the caller genuinely needs to distinguish failure modes.

---

### S6: Zero External Dependencies

**Rule:** No third-party libraries. Build what you need.

**Rationale:** External dependencies introduce: (1) supply chain risk — malicious or vulnerable packages; (2) safety risk — code you don't own cannot be asserted or audited; (3) stability risk — APIs change, maintainers abandon projects. TigerBeetle's standard library is TigerBeetle.

**Exception:** Operating system and hardware interfaces are unavoidable. The standard library of the implementation language (if carefully chosen) may be acceptable.

---

## Performance Rules — Extended

### P1: Back-of-Envelope Before Code

**Rule:** Before writing any performance-sensitive code, write down the performance target and a calculation showing the design can meet it.

**Reference numbers (approximate, 2024):**
| Operation | Latency |
|-----------|---------|
| L1 cache hit | ~1 ns |
| L2 cache hit | ~5 ns |
| L3 cache hit | ~40 ns |
| Main memory (DRAM) | ~100 ns |
| NVMe SSD random read | ~100 µs |
| Network LAN round trip | ~0.5 ms |
| Network WAN round trip | ~50 ms |
| HDD seek + read | ~10 ms |

**Usage:** If your design requires 10,000 random SSD reads per transaction and your target is 1 ms latency — that's 10,000 × 100 µs = 1 second. The design is wrong before a line of code is written.

---

### P2: Primary Colors Framework

**Rule:** Categorize every operation by its resource (network, storage, memory, compute) and texture (bandwidth, latency). Design to the right constraint.

**Resource definitions:**
- **Network:** data transferred between machines
- **Storage:** data read from or written to persistent media
- **Memory:** data accessed in RAM, registers, or cache
- **Compute:** CPU instructions executed

**Texture definitions:**
- **Bandwidth:** how much data can be moved per second (throughput)
- **Latency:** how long a single operation takes (response time)

**Analysis pattern:**
```
Operation: replicate a journal entry to 5 nodes
- Network bandwidth: 5 × message size — is this < NIC capacity?
- Network latency: 1 round-trip (parallel, not sequential) — is this < timeout budget?
- Storage bandwidth: 1 fsync per node — is this < disk write throughput?
- Storage latency: fsync latency — is this on the critical path?
```

---

### P3: Control Plane vs. Data Plane

**Control plane** — handles configuration, coordination, membership changes, recovery, error paths. Characteristics:
- Low-frequency (happens rarely or on startup)
- Correctness is paramount — use assertions liberally
- Allocations acceptable
- Complex logic acceptable

**Data plane** — handles the hot path: processing transactions, forwarding messages, writing journal entries. Characteristics:
- High-frequency (millions of times per second)
- Performance is paramount
- Zero allocations
- Zero locks on the critical path
- Branches minimized

**Pattern:** Structure your code so that data plane functions are pure and allocation-free. Any setup, configuration, or error recovery happens in control plane code that runs before or outside the hot path.

---

### P4: Zero Copy

**Rule:** On the hot path, data should not be copied between buffers. Operate on data in-place. Pass slices/pointers, not values.

**Anti-pattern:**
```zig
fn process_message(msg: Message) void { // copies Message onto stack
    var buf: [4096]u8 = undefined;
    @memcpy(&buf, msg.body); // second copy into local buffer
    // process buf...
}
```

**Pattern:**
```zig
fn process_message(msg: *const Message) void { // pointer, no copy
    // operate directly on msg.body slice — no copy
    parse_body(msg.body);
}
```

---

### P5: Cache-Friendly Layout

**Rule:** Struct fields that are accessed together should be stored together. Align structs to cache line boundaries. Separate hot and cold fields into separate structs.

**Cache line size:** 64 bytes on x86/ARM.

**Pattern:**
```zig
// Hot struct: fields accessed on every operation
const TransferHot = extern struct {
    amount: u64,           // 8 bytes
    debit_account_id: u128, // 16 bytes
    credit_account_id: u128, // 16 bytes
    ledger: u32,           // 4 bytes
    code: u16,             // 2 bytes
    _padding: [18]u8,      // pad to 64 bytes
    comptime { assert(@sizeOf(TransferHot) == 64); }
};

// Cold struct: fields accessed only on lookup/display
const TransferCold = extern struct {
    timestamp: u64,
    user_data: u128,
    flags: TransferFlags,
    // ...
};
```

---

## Experience Rules — Extended

### E1: Big-Endian Qualification

**Rule:** Name variables, functions, and types with the most significant qualifier first (left to right), ending with unit and bound suffixes.

**Structure:** `<noun>_<qualifier...>_<unit?>_<bound?>`

**Examples:**
| Good | Bad | Reason |
|------|-----|--------|
| `transfer_amount_max` | `max_transfer_amount` | noun first |
| `replica_count` | `num_replicas` | no `num_` prefix |
| `connection_timeout_ms` | `timeout_connection` | noun before qualifier |
| `message_size_bytes_max` | `MAX_MSG_SZ` | full words, no abbreviations |
| `pipeline_requests_in_flight_max` | `in_flight_max_pipeline_req` | big-endian order |

**Why big-endian?** Names sort alphabetically by domain. All `transfer_*` fields cluster together. All `connection_*` settings cluster together. Searching for `connection` finds all connection-related names.

---

### E2: Units as Suffixes

**Rule:** Any variable representing a quantity with a physical unit must include that unit as a suffix.

| Unit | Suffix |
|------|--------|
| Milliseconds | `_ms` |
| Microseconds | `_us` |
| Nanoseconds | `_ns` |
| Bytes | `_bytes` |
| Kilobytes | `_kb` |
| Megabytes | `_mb` |
| Gigabytes | `_gb` |
| Seconds | `_s` |

**Anti-pattern:** `var timeout = 5000;` — 5000 what? Seconds? Milliseconds?
**Pattern:** `var request_timeout_ms: u32 = 5000;` — unambiguous.

---

### E3: No Abbreviations

**Rule:** Write full words in all identifiers. No abbreviations, no acronyms (unless universally understood, e.g., `id`, `ip`, `url`).

| Abbreviation | Full word |
|-------------|-----------|
| `buf` | `buffer` |
| `conn` | `connection` |
| `msg` | `message` |
| `req` / `res` | `request` / `response` |
| `src` / `dst` | `source` / `destination` |
| `len` | `length` |
| `idx` | `index` |
| `cnt` | `count` |
| `cfg` | `config` |
| `addr` | `address` |

**Exception:** Well-known domain acronyms used consistently: `id`, `ip`, `url`, `tcp`, `udp`, `lsn` (log sequence number in database contexts).

---

### E4: Consistent Visual Weight for Paired Names

**Rule:** When two names appear together as a pair (source/destination, client/server, read/write), they should have equal or near-equal character length to maintain visual alignment.

| Preferred pair | Avoid |
|---------------|-------|
| `source` / `target` | `src` / `destination` |
| `client` / `server` | — |
| `reader` / `writer` | `read` / `writer` |
| `encode` / `decode` | — |
| `push` / `pull` | — |

---

### E5: One Function, One Thing

**Rule:** A function should do exactly one thing and be describable in a single, simple sentence. If you need "and" to describe it, split it.

**How to check:** Write a one-sentence description of the function. If it contains "and", "also", "then", or multiple verbs, the function is doing too much.

**Anti-pattern:**
```zig
// "validates the request, updates the state, and writes to the log"
fn process_request(req: Request) void { ... }
```

**Pattern:**
```zig
fn request_validate(req: Request) bool { ... }
fn state_apply(req: Request) void { ... }
fn log_append(entry: LogEntry) void { ... }
```

---

## Summary Table

| Value | Top Rules |
|-------|-----------|
| **Safety** | Assert everything · Static alloc · Explicit bounds · No recursion · Zero deps |
| **Performance** | Back-of-envelope · Primary colors · Control/data plane · Zero copy · Cache-aligned |
| **Experience** | Big-endian names · Unit suffixes · No abbreviations · One function one thing |
