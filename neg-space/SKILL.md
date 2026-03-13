---
name: neg-space
description: Apply Negative Space Programming when writing or reviewing code.
  Trigger on: "negative space programming", "negative space", invariant negation,
  forbidden states, rejection-first testing, NOT HANDLED annotations, non-goal
  documentation, boundary articulation, negative interface contracts, MUST NOT ASSUME,
  exclusion specification, "define what code doesn't do", "document what function
  doesn't handle", "enumerate invalid inputs".
version: 0.1.0
---

# Negative Space Programming

Negative Space Programming: a unit of code is specified as precisely by what it rejects, ignores, and forbids as by what it accepts. The invalid domain is a first-class design artifact — named, encoded, tested, and documented with the same rigor as the valid domain. Incomplete specification is not a gap to fill later; it is a defect that exists today.

---

## Specification

A complete specification names both the accepted domain and the rejected domain. Code that only describes what it handles is half-specified.

### NS1 — Exclusion Specification

- Enumerate rejected inputs **before** writing the implementation — as comments, docstrings, or type constraints.
- Every function answers two questions: "what do I accept?" **and** "what do I refuse?"
- Use `NOT HANDLED:` annotations in docstrings for explicitly excluded cases — cases the caller is responsible for excluding before calling.

**Anti-pattern:**
```typescript
function divide(a: number, b: number): number {
  return a / b; // silently returns NaN or Infinity for b=0
}
```

**Pattern:**
```typescript
/**
 * Divides a by b.
 *
 * NOT HANDLED: b === 0 — caller must validate denominator is non-zero.
 * NOT HANDLED: non-finite inputs — caller must validate a and b are finite.
 */
function divide(a: number, b: number): number {
  assert(Number.isFinite(a), `divide: a must be finite, got ${a}`);
  assert(Number.isFinite(b), `divide: b must be finite, got ${b}`);
  assert(b !== 0, `divide: b must be non-zero`);
  return a / b;
}
```

---

### NS2 — Non-Goal Documentation

- Every module or class with a `GOALS:` block requires a paired `NON-GOALS:` block.
- Non-goals are design decisions stated imperatively — they describe what this code deliberately does not do.
- Non-goals include explicit delegation: who is responsible for what this module omits.

**Pattern:**
```python
"""
GOALS:
  - Parse and validate incoming transfer payloads.
  - Produce typed TransferRequest objects for the domain layer.

NON-GOALS:
  - Does NOT authenticate the caller — authentication is enforced at the API gateway.
  - Does NOT persist or enqueue transfers — caller is responsible for storage.
  - Does NOT apply business rules (limits, fraud checks) — domain layer owns those.

Caller is responsible for:
  - Ensuring the request body has been authenticated before passing it here.
  - Handling persistence of the returned TransferRequest.
"""
```

---

### NS3 — Forbidden State Enumeration

- Name impossible or forbidden states as constants or enum variants, even when the type system prevents them.
- Document **why** a state is forbidden, not merely that it is.
- For state machines: enumerate non-transitions explicitly alongside valid transitions.

**Anti-pattern:**
```python
class ConnectionState(Enum):
    CONNECTING = "connecting"
    CONNECTED = "connected"
    DISCONNECTED = "disconnected"
    # No documentation of which transitions are forbidden or why
```

**Pattern:**
```python
class ConnectionState(Enum):
    CONNECTING = "connecting"
    CONNECTED = "connected"
    DISCONNECTED = "disconnected"

# Forbidden transitions: states that must never occur in sequence.
# DISCONNECTED → CONNECTED is forbidden because authentication must
# run during CONNECTING; skipping it would bypass credential checks.
FORBIDDEN_TRANSITIONS: frozenset[tuple[ConnectionState, ConnectionState]] = frozenset({
    (ConnectionState.DISCONNECTED, ConnectionState.CONNECTED),
    (ConnectionState.CONNECTED, ConnectionState.CONNECTING),
})

def transition(current: ConnectionState, next: ConnectionState) -> ConnectionState:
    assert (current, next) not in FORBIDDEN_TRANSITIONS, (
        f"Forbidden state transition: {current} → {next}"
    )
    return next
```

---

## Invariants

Invariants are not just positive constraints ("balance must be >= 0"). Every positive invariant has a negative form — a condition that must never be true. Both must be asserted.

### NS4 — Invariant Negation

- Every positive invariant has a paired negative (`NEVER`) form — both encoded as assertions.
- Assert `NEVER` conditions at **every state transition**, not just at function exit.
- Name invariant bounds as constants so assertions are self-documenting.

**Anti-pattern:**
```typescript
function applyDebit(account: Account, amount: number): void {
  account.balance -= amount;
  assert(account.balance >= 0); // only positive bound checked
}
```

**Pattern:**
```typescript
const BALANCE_MIN = 0;
const BALANCE_MAX = 1_000_000_00; // 1,000,000.00 in cents

function applyDebit(account: Account, amount: number): void {
  assert(amount > 0, `applyDebit: amount must be positive, got ${amount}`);
  assert(account.balance >= BALANCE_MIN, `applyDebit: pre-condition: balance below minimum`);
  assert(account.balance <= BALANCE_MAX, `applyDebit: pre-condition: balance above maximum`);

  account.balance -= amount;

  // NEVER: balance may not go below BALANCE_MIN (overdraft is a domain violation)
  assert(account.balance >= BALANCE_MIN, `applyDebit: post-condition: overdraft occurred`);
  // NEVER: balance may not exceed BALANCE_MAX (debit cannot increase balance)
  assert(account.balance <= BALANCE_MAX, `applyDebit: post-condition: balance exceeded maximum`);
}
```

---

### NS5 — Boundary Articulation

- Extract the boundary between valid and invalid into a **named predicate** (`is_valid_transfer_amount`).
- Document the boundary as a closed interval: what is included, what is excluded, and why each endpoint is what it is.
- Single authoritative definition — no duplication of boundary logic across callers or tests.

**Anti-pattern:**
```python
# Scattered inline boundary checks — duplicated across callers
def create_transfer(amount: int) -> Transfer:
    if amount <= 0 or amount > 100_000_00:
        raise ValueError("invalid amount")
    ...

def preview_transfer(amount: int) -> TransferPreview:
    if amount < 1 or amount > 100_000_00:  # ← off-by-one vs. create_transfer
        raise ValueError("bad amount")
    ...
```

**Pattern:**
```python
# Valid: [AMOUNT_MIN, AMOUNT_MAX] inclusive.
# Zero rejected: a zero-value transfer is a no-op and indicates a caller bug.
# Above AMOUNT_MAX rejected: exceeds single-transfer limit to prevent overflow.
AMOUNT_MIN = 1          # smallest valid transfer: 1 cent
AMOUNT_MAX = 100_000_00 # largest valid transfer: 100,000.00

def is_valid_transfer_amount(amount: int) -> bool:
    return AMOUNT_MIN <= amount <= AMOUNT_MAX

def create_transfer(amount: int) -> Transfer:
    assert is_valid_transfer_amount(amount), f"invalid transfer amount: {amount}"
    ...

def preview_transfer(amount: int) -> TransferPreview:
    assert is_valid_transfer_amount(amount), f"invalid transfer amount: {amount}"
    ...
```

---

## Testing and Contracts

Tests and interface contracts both describe behavior. Complete tests describe rejection behavior. Complete contracts describe what callers may not assume.

### NS6 — Rejection-First Testing

- Write rejection tests **before** happy-path tests; group them under a `describe("rejects invalid inputs")` block or equivalent.
- Every `NOT HANDLED:` annotation must have a corresponding rejection test.
- Test the boundary explicitly: `min − 1`, `min`, `max`, `max + 1`.
- Rejection tests assert three things: the error is raised, it is the correct type, and its message names the invalid value.

**Anti-pattern:**
```typescript
describe("createUser", () => {
  it("creates a user with valid inputs", () => {
    // only happy path — rejection behavior is unspecified and untested
    const user = createUser("alice", "alice@example.com");
    expect(user.name).toBe("alice");
  });
});
```

**Pattern:**
```typescript
describe("createUser", () => {
  describe("rejects invalid inputs", () => {
    it("rejects empty name", () => {
      expect(() => createUser("", "a@b.com")).toThrow(/name/);
    });
    it("rejects name longer than NAME_LENGTH_MAX", () => {
      expect(() => createUser("a".repeat(NAME_LENGTH_MAX + 1), "a@b.com")).toThrow(/name/);
    });
    it("accepts name at NAME_LENGTH_MAX (boundary)", () => {
      expect(() => createUser("a".repeat(NAME_LENGTH_MAX), "a@b.com")).not.toThrow();
    });
    it("rejects name at NAME_LENGTH_MAX - 1 (below lower bound is impossible; above tested above)", () => {
      // document: min is 1; empty string (length 0) is tested separately
    });
    it("rejects malformed email", () => {
      expect(() => createUser("alice", "not-an-email")).toThrow(/email/);
    });
    it("rejects null email (NOT HANDLED: null inputs)", () => {
      expect(() => createUser("alice", null as any)).toThrow(/email/);
    });
  });

  describe("creates user with valid inputs", () => {
    it("returns a user for valid name and email", () => {
      const user = createUser("alice", "alice@example.com");
      expect(user.name).toBe("alice");
    });
  });
});
```

---

### NS7 — Negative Interface Contracts

- Every public interface's docstring requires both a `GUARANTEES:` block and a `MUST NOT ASSUME:` block.
- `MUST NOT ASSUME:` covers what callers commonly but incorrectly assume: thread-safety, caching behavior, mutability, reference/lifetime validity, and ordering.
- Both blocks are required — a contract without its negative half is incomplete.

**Pattern:**
```python
class TokenCache:
    """
    GUARANTEES:
      - get(key) returns the token if it was set and has not expired.
      - set(key, token, ttl_s) stores the token for at least ttl_s seconds.
      - Expired tokens are never returned by get().

    MUST NOT ASSUME:
      - NOT thread-safe: concurrent get/set requires external locking.
      - Does NOT persist across process restarts: cache is in-memory only.
      - set() does NOT deduplicate: calling set() twice overwrites the first token silently.
      - Returned tokens are NOT copies: mutating the returned object mutates the cache entry.
      - Expiry is NOT exact: a token may be returned up to one eviction cycle after its TTL.
    """
```

---

## Code Review Checklist

### Specification
- [ ] Functions enumerate rejected inputs before implementation (in docstring or type annotations)
- [ ] `NOT HANDLED:` annotations present for all explicitly excluded cases
- [ ] Module has both `GOALS:` and `NON-GOALS:` blocks
- [ ] Non-goals include explicit delegation (caller is responsible for X)
- [ ] Forbidden states named as constants or enum variants, with rationale
- [ ] State machine non-transitions enumerated explicitly

### Invariants
- [ ] Every positive invariant has a paired `NEVER` assertion
- [ ] Invariant bounds are named constants, not magic numbers
- [ ] `NEVER` assertions fire at every state transition, not just function exit
- [ ] Valid/invalid boundary extracted into a named predicate
- [ ] Boundary documented as closed interval (min, max, zero-case, above-max-case)
- [ ] No duplication of boundary logic across callers

### Testing and Contracts
- [ ] Rejection tests appear before happy-path tests
- [ ] Every `NOT HANDLED:` annotation has a corresponding rejection test
- [ ] Boundary tested at: `min − 1`, `min`, `max`, `max + 1`
- [ ] Rejection tests assert: error raised + correct type + message names the invalid value
- [ ] Public interface has both `GUARANTEES:` and `MUST NOT ASSUME:` blocks
- [ ] `MUST NOT ASSUME:` covers: thread-safety, caching, mutability, lifetime, ordering

---

## Quick Reference

```
Specification:  NOT HANDLED: · NON-GOALS: · named forbidden states + rationale
Invariants:     paired NEVER assertions · named boundary predicates · closed-interval docs
Testing:        rejection-first · boundary explicit (min−1/min/max/max+1) · annotations tested
Contracts:      GUARANTEES: + MUST NOT ASSUME: on every public interface
```

For extended worked examples, see `references/patterns.md`.
