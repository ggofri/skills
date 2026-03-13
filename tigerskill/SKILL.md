---
name: tigerskill
description: Apply TigerStyle development principles when writing or reviewing code.
  Use when the user mentions "tigerstyle", "tiger style", "TigerBeetle", is working
  in a TigerBeetle-style codebase, asks to review code for safety/performance/correctness
  using TigerStyle, or wants guidance on static allocation, assertions, zero-copy,
  mechanical sympathy, or the primary colors framework. Also trigger when the user
  wants naming conventions or is asking about TigerStyle-compliant code structure.
version: 0.1.0
---

# TigerStyle

TigerStyle is a development methodology created by TigerBeetle — a distributed financial database. Its central maxim is **"do the hard thing today to make tomorrow easy."** TigerStyle is not a style guide for aesthetics; it is a discipline for writing systems software that is correct, fast, and maintainable under adversarial conditions. Every rule exists because failure has a cost: lost money, lost data, lost time. The methodology organizes its principles around three core values — **Safety**, **Performance**, and **Experience** — treated as non-negotiable constraints, not tradeoffs.

---

## Safety

Safety means the code provably cannot do the wrong thing. The goal is to make bugs loud, early, and impossible to miss.

### Assertions
- Assert every argument, return value, and invariant — both what *should* occur and what *should not* occur.
- Assertions are not a debugging aid; they are a specification tool. A codebase without assertions is an unspecified codebase.
- Never disable assertions in production. If an assertion fires, it is cheaper to crash than to continue with corrupt state.
- Use assertions to check: preconditions, postconditions, loop invariants, and all state transitions.

### Static Allocation
- Allocate all memory statically at startup. Never allocate after initialization.
- Dynamic allocation hides unbounded resource usage and defers failures to unpredictable moments. Static allocation makes resource usage explicit and verifiable at startup.
- Size all buffers, pools, and queues explicitly at compile time or initialization time.

### Explicit Limits
- Bound every loop, queue, connection pool, concurrency degree, and retry count.
- An unbounded system cannot be reasoned about, tested exhaustively, or safely deployed.

### No Recursion
- Avoid recursion. Prefer iteration with explicit stacks if traversal depth must vary.
- Recursion hides stack consumption. Fixed-depth iteration keeps resource usage auditable.

### Minimal Interface Surface
- Keep interfaces small. Push control flow up, push data down.
- Prefer simple return types: `bool` over error-code integers. Avoid forcing callers to branch on complex return types.
- Do not expose implementation details that could cause caller-side state management.

### Zero Dependencies
- Avoid external dependencies. Every dependency is a supply chain risk and a safety risk.
- A dependency you don't own cannot be asserted, cannot be audited, and can change under you.

### Zero Technical Debt
- Fix the bug, the test gap, and the design flaw today. TigerStyle codebases start and stay clean.
- Technical debt is not deferred work — it is compounding interest on future failures.

---

## Performance

Performance means knowing the cost of everything before writing a line of code. TigerStyle treats performance analysis as a prerequisite, not a post-hoc optimization pass.

### Back-of-Envelope First
- Before writing code, estimate the performance envelope with back-of-envelope calculations.
- Know the order-of-magnitude costs: disk seek (~1 ms), SSD random read (~100 µs), network round trip (~0.5 ms LAN), memory access (~100 ns), L1 cache hit (~1 ns).
- If your design cannot fit within those constraints on paper, it cannot fit in production.

### Primary Colors Framework
Analyze every operation along two dimensions:

| Resource | Bandwidth | Latency |
|----------|-----------|---------|
| **Network** | throughput (bytes/sec) | round-trip time |
| **Storage** | IOPS / MB/s | seek + transfer time |
| **Memory** | cache line utilization | access time |
| **Compute** | instructions/cycle | pipeline stall rate |

- Identify which resource is the bottleneck for each operation.
- Design to saturate the right resource, not the wrong one.
- Do not fix latency problems with parallelism. Do not fix bandwidth problems with caching. Fix the right resource.

### Control Plane vs. Data Plane
- Separate **control plane** (low-frequency: configuration, coordination, recovery) from **data plane** (high-frequency: the hot path).
- Control plane code can afford assertions, allocations, and complex logic.
- Data plane code must be zero-overhead: no allocations, no locks, no branches on error paths.

### Batching
- Amortize fixed costs by batching. Larger buffers reduce per-operation overhead.
- Design for batch-first APIs. Single-item APIs are a performance anti-pattern in high-throughput systems.

### Zero Copy, Zero Deserialization
- In the hot path, avoid copying data. Operate on data in-place where possible.
- Avoid deserialization on the critical path. Use fixed-size binary structures that can be read directly from wire/disk.

### Cache-Friendly Data Layout
- Use fixed-size, cache-line-aligned structs (typically 64 bytes).
- Separate hot fields (frequently accessed) from cold fields (rarely accessed).
- Prefer arrays of structs or structs of arrays based on access patterns.

---

## Experience

Experience means the codebase reads like a precise technical document. Every name communicates intent without ambiguity.

### Naming Conventions

**Big-Endian Qualification** — qualify names from most significant to least significant, left to right:
- `connection_delay_ms_max` — not `max_connection_delay_ms`
- `replica_count_max` — not `max_replicas`
- `transfer_size_bytes` — not `bytes_transferred`

This ensures names sort and grep consistently, and the noun (the thing) comes before its qualifiers.

**Suffixes for Units and Bounds:**
- Append units: `_ms`, `_us`, `_ns`, `_bytes`, `_kb`, `_mb`
- Append bounds: `_max`, `_min`, `_count`, `_limit`
- Never omit units from names that represent a quantity with a unit

**snake_case everywhere.** No abbreviations. Write out full words.
- `timeout_ms` — not `tmo`
- `connection` — not `conn`
- `buffer` — not `buf`

**Descriptive nouns for state, verbs for actions:**
- `replica_count` (noun — state)
- `send_message()` (verb — action)

**Consistent character length for paired names:**
- Prefer `source`/`target` over `src`/`destination` (unequal length creates visual noise)
- Prefer `client`/`server` over `cli`/`server`

### Simplicity and Elegance
- Prefer simple, direct code over clever code. Clever code is a bug waiting to be found.
- If you cannot explain what a function does in one sentence, it is doing too much.
- Delete code ruthlessly. The best code is no code.
- A function that does one thing can be tested, asserted, and understood. A function that does many things cannot.

---

## Code Review Checklist

Use this checklist when reviewing a changeset against TigerStyle:

### Safety
- [ ] All arguments validated with assertions at function entry
- [ ] All return values checked; no silently ignored errors
- [ ] All memory allocated at initialization — no post-init allocations
- [ ] All loops, queues, and buffers have explicit upper bounds
- [ ] No recursion introduced
- [ ] No new external dependencies added
- [ ] Interface surface minimized — is anything exposed that callers don't need?
- [ ] No technical debt introduced (TODOs without a plan, known bugs deferred)

### Performance
- [ ] Back-of-envelope analysis documented for any performance-sensitive path
- [ ] Hot path is free of allocations, locks, and avoidable branches
- [ ] Data structures are cache-line aligned; hot/cold fields separated
- [ ] Batch APIs preferred over single-item APIs where throughput matters
- [ ] No unnecessary copies or deserialization in the critical path
- [ ] Primary colors bottleneck identified and addressed

### Experience
- [ ] Names use big-endian qualification (`noun_qualifier_unit_bound`)
- [ ] Units and bounds are suffixed on all quantitative names
- [ ] No abbreviations; full words used throughout
- [ ] snake_case used consistently
- [ ] Paired names have consistent visual weight
- [ ] Each function does one thing and can be described in one sentence
- [ ] Dead code removed; no commented-out code left behind

---

## Quick Reference

```
Safety:     assert everything · static alloc · explicit bounds · no recursion · no deps
Performance: back-of-envelope first · primary colors · control/data plane split · zero copy
Experience: big-endian names · suffix units · no abbrevs · one function one thing
```

For detailed rule explanations, see `references/rules.md`.
