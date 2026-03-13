# Negative Space Programming — Patterns Reference

Extended worked examples for each principle. Use these as templates when applying Negative Space Programming to real code.

---

## NS1 — Exclusion Specification: Three Stages of a Parser

The same Python CSV parser written at three levels of negative space specification.

**Stage 1: No negative space** — the invalid domain is invisible.
```python
def parse_csv_row(line: str) -> list[str]:
    return line.split(",")
```

**Stage 2: Comments only** — the invalid domain is documented but not enforced.
```python
def parse_csv_row(line: str) -> list[str]:
    # NOT HANDLED: empty string — caller must ensure line is non-empty.
    # NOT HANDLED: quoted fields containing commas — use a full CSV parser.
    # NOT HANDLED: lines with trailing newlines — caller must strip first.
    # NOT HANDLED: non-UTF-8 bytes — caller must decode before calling.
    return line.split(",")
```

**Stage 3: Encoded in types and assertions** — the invalid domain is enforced and documented.
```python
def parse_csv_row(line: str) -> list[str]:
    """
    Parse a single unquoted CSV row into a list of field strings.

    NOT HANDLED: empty string — caller must ensure line is non-empty.
    NOT HANDLED: quoted fields containing commas — use csv.reader for those.
    NOT HANDLED: lines with trailing newlines — caller must call strip() first.
    NOT HANDLED: non-UTF-8 bytes — caller must decode to str before calling.
    """
    assert isinstance(line, str), f"parse_csv_row: expected str, got {type(line)}"
    assert len(line) > 0, "parse_csv_row: line must be non-empty"
    assert "\n" not in line, "parse_csv_row: line must not contain newlines; call strip() first"
    assert '"' not in line, (
        'parse_csv_row: quoted fields not supported; use csv.reader for quoted CSV'
    )

    fields = line.split(",")

    assert len(fields) >= 1, "parse_csv_row: post-condition: must return at least one field"
    return fields
```

The progression shows that Stage 2 is better than Stage 1 but still incomplete: comments that are not enforced will drift. Stage 3 makes the exclusion specification executable — violations crash immediately at the call site.

---

## NS3 — Forbidden State Enumeration: TypeScript State Machine with Transition Matrix

A WebSocket connection state machine where forbidden transitions are encoded as a lookup table.

```typescript
enum ConnectionState {
  Idle = "idle",
  Connecting = "connecting",
  Authenticating = "authenticating",
  Connected = "connected",
  Draining = "draining",
  Closed = "closed",
}

/**
 * Forbidden transition matrix.
 * Key: `${from}:${to}` — value: reason the transition is forbidden.
 *
 * Forbidden because:
 * - Idle → Connected: authentication phase must not be skipped.
 * - Idle → Draining: cannot drain a connection that was never opened.
 * - Connecting → Connected: authentication phase must not be skipped.
 * - Connecting → Draining: cannot drain before handshake completes.
 * - Authenticating → Connecting: re-entry into connecting from auth is a logic error.
 * - Connected → Connecting: reconnect must go through Closed first.
 * - Connected → Idle: Idle is only a starting state.
 * - Draining → Connecting: cannot reconnect while draining.
 * - Draining → Authenticating: cannot re-authenticate while draining.
 * - Closed → * (any): Closed is terminal; no transitions out.
 */
const FORBIDDEN_TRANSITIONS: ReadonlyMap<string, string> = new Map([
  ["idle:connected",          "authentication phase must not be skipped"],
  ["idle:draining",           "cannot drain a connection that was never opened"],
  ["connecting:connected",    "authentication phase must not be skipped"],
  ["connecting:draining",     "cannot drain before handshake completes"],
  ["authenticating:connecting","re-entry into connecting from authenticating is a logic error"],
  ["connected:connecting",    "reconnect must go through Closed first"],
  ["connected:idle",          "Idle is only a starting state"],
  ["draining:connecting",     "cannot reconnect while draining"],
  ["draining:authenticating", "cannot re-authenticate while draining"],
  ["closed:connecting",       "Closed is terminal"],
  ["closed:authenticating",   "Closed is terminal"],
  ["closed:connected",        "Closed is terminal"],
  ["closed:draining",         "Closed is terminal"],
  ["closed:idle",             "Closed is terminal"],
]);

function transition(
  current: ConnectionState,
  next: ConnectionState,
): ConnectionState {
  const key = `${current}:${next}`;
  const reason = FORBIDDEN_TRANSITIONS.get(key);
  if (reason !== undefined) {
    throw new Error(
      `Forbidden state transition ${current} → ${next}: ${reason}`
    );
  }
  return next;
}
```

The matrix serves as executable documentation: anyone reading the code sees not just what transitions exist, but exactly which are forbidden and why. Adding a new state requires explicitly reasoning about every new transition pair.

---

## NS5 — Boundary Articulation: Shared Boundary Predicate Library

A single module that defines all transfer validation boundaries. Both production code and tests import from this module — there is one authoritative definition.

```python
# transfer_boundaries.py
#
# Single authoritative source for transfer amount and account validity boundaries.
# Both production code and tests must import from here.
# Do NOT duplicate these constants or predicates elsewhere.

# Transfer amount bounds
# Valid: [AMOUNT_MIN, AMOUNT_MAX] inclusive.
# AMOUNT_MIN = 1:        zero-value transfers are no-ops and indicate a caller bug.
# AMOUNT_MAX = 10_000_00: single-transfer cap set by compliance policy (10,000.00).
AMOUNT_MIN = 1
AMOUNT_MAX = 10_000_00

def is_valid_transfer_amount(amount: int) -> bool:
    """
    Returns True iff amount is a valid transfer amount.
    Valid: [AMOUNT_MIN, AMOUNT_MAX] inclusive.
    Zero rejected: no-op transfers indicate a caller bug.
    Above AMOUNT_MAX rejected: exceeds compliance single-transfer limit.
    """
    return AMOUNT_MIN <= amount <= AMOUNT_MAX

# Account ID bounds
# Valid: any positive 64-bit integer.
# Zero rejected: zero is used as a sentinel for "no account" in the storage layer.
# Negative rejected: IDs are unsigned; negative values indicate integer overflow.
ACCOUNT_ID_MIN = 1
ACCOUNT_ID_MAX = 2**63 - 1  # max positive int64

def is_valid_account_id(account_id: int) -> bool:
    """
    Returns True iff account_id is a valid account identifier.
    Valid: [ACCOUNT_ID_MIN, ACCOUNT_ID_MAX] inclusive.
    Zero rejected: sentinel value for "no account".
    Negative rejected: IDs are unsigned; negatives indicate overflow.
    """
    return ACCOUNT_ID_MIN <= account_id <= ACCOUNT_ID_MAX
```

```python
# In production code:
from transfer_boundaries import is_valid_transfer_amount, is_valid_account_id

def create_transfer(amount: int, from_account: int, to_account: int) -> Transfer:
    assert is_valid_transfer_amount(amount), f"invalid amount: {amount}"
    assert is_valid_account_id(from_account), f"invalid from_account: {from_account}"
    assert is_valid_account_id(to_account), f"invalid to_account: {to_account}"
    ...
```

```python
# In tests:
from transfer_boundaries import (
    AMOUNT_MIN, AMOUNT_MAX,
    ACCOUNT_ID_MIN, ACCOUNT_ID_MAX,
    is_valid_transfer_amount,
)

def test_create_transfer_rejects_below_minimum():
    with pytest.raises(AssertionError, match="invalid amount"):
        create_transfer(AMOUNT_MIN - 1, valid_account, valid_account)

def test_create_transfer_accepts_minimum():
    transfer = create_transfer(AMOUNT_MIN, valid_account, valid_account)
    assert transfer is not None

def test_create_transfer_accepts_maximum():
    transfer = create_transfer(AMOUNT_MAX, valid_account, valid_account)
    assert transfer is not None

def test_create_transfer_rejects_above_maximum():
    with pytest.raises(AssertionError, match="invalid amount"):
        create_transfer(AMOUNT_MAX + 1, valid_account, valid_account)
```

---

## NS6 — Rejection-First Testing: HTTP Request Validator

A full rejection test suite for an HTTP request validator, written before any acceptance test.

```typescript
import { validateRequest, HttpRequest } from "./request-validator";
import {
  BODY_SIZE_MAX_BYTES,
  PATH_LENGTH_MAX,
  ALLOWED_METHODS,
  REQUIRED_HEADERS,
} from "./request-validator-boundaries";

describe("validateRequest", () => {

  // ── Rejection tests come first ──────────────────────────────────────────────

  describe("rejects invalid Content-Type", () => {
    it("rejects missing Content-Type header", () => {
      const req = makeRequest({ headers: {} });
      expect(() => validateRequest(req)).toThrow(/content-type/i);
    });
    it("rejects unsupported Content-Type", () => {
      const req = makeRequest({ headers: { "content-type": "text/plain" } });
      expect(() => validateRequest(req)).toThrow(/content-type/i);
    });
    it("accepts application/json", () => {
      expect(() => makeRequest({ headers: { "content-type": "application/json" } }))
        .not.toThrow();
    });
  });

  describe("rejects invalid body size", () => {
    it("rejects empty body", () => {
      const req = makeRequest({ body: Buffer.alloc(0) });
      expect(() => validateRequest(req)).toThrow(/body/i);
    });
    it("rejects body at exactly BODY_SIZE_MAX_BYTES + 1", () => {
      const req = makeRequest({ body: Buffer.alloc(BODY_SIZE_MAX_BYTES + 1) });
      expect(() => validateRequest(req)).toThrow(/body size/i);
    });
    it("accepts body at exactly BODY_SIZE_MAX_BYTES (boundary)", () => {
      const req = makeRequest({ body: Buffer.alloc(BODY_SIZE_MAX_BYTES) });
      expect(() => validateRequest(req)).not.toThrow();
    });
    it("accepts body at 1 byte (minimum)", () => {
      const req = makeRequest({ body: Buffer.alloc(1) });
      expect(() => validateRequest(req)).not.toThrow();
    });
  });

  describe("rejects invalid HTTP method", () => {
    it("rejects GET (NOT HANDLED: read methods — this validator is for mutations)", () => {
      const req = makeRequest({ method: "GET" });
      expect(() => validateRequest(req)).toThrow(/method/i);
    });
    it("rejects DELETE", () => {
      const req = makeRequest({ method: "DELETE" });
      expect(() => validateRequest(req)).toThrow(/method/i);
    });
    it("rejects unknown method", () => {
      const req = makeRequest({ method: "FROBNICATE" });
      expect(() => validateRequest(req)).toThrow(/method/i);
    });
    for (const method of ALLOWED_METHODS) {
      it(`accepts ${method}`, () => {
        expect(() => makeRequest({ method })).not.toThrow();
      });
    }
  });

  describe("rejects invalid path", () => {
    it("rejects empty path", () => {
      const req = makeRequest({ path: "" });
      expect(() => validateRequest(req)).toThrow(/path/i);
    });
    it("rejects path without leading slash", () => {
      const req = makeRequest({ path: "api/transfers" });
      expect(() => validateRequest(req)).toThrow(/path/i);
    });
    it("rejects path longer than PATH_LENGTH_MAX", () => {
      const req = makeRequest({ path: "/" + "a".repeat(PATH_LENGTH_MAX) });
      expect(() => validateRequest(req)).toThrow(/path/i);
    });
    it("accepts path at PATH_LENGTH_MAX - 1 (boundary)", () => {
      const req = makeRequest({ path: "/" + "a".repeat(PATH_LENGTH_MAX - 2) });
      expect(() => validateRequest(req)).not.toThrow();
    });
  });

  describe("rejects missing required headers", () => {
    for (const header of REQUIRED_HEADERS) {
      it(`rejects missing ${header}`, () => {
        const headers = makeValidHeaders();
        delete headers[header];
        const req = makeRequest({ headers });
        expect(() => validateRequest(req)).toThrow(new RegExp(header, "i"));
      });
    }
  });

  // ── Acceptance tests follow ─────────────────────────────────────────────────

  describe("accepts valid requests", () => {
    it("returns a parsed request for a fully valid POST", () => {
      const req = makeValidRequest();
      const result = validateRequest(req);
      expect(result).toBeDefined();
    });
  });

});
```

The structure makes the acceptance test feel like an afterthought — which is the goal. The rejection surface is the primary specification; acceptance is just verification that valid input gets through.

---

## NS7 — Negative Interface Contracts: Three Class Examples

### File Reader

```python
class FileReader:
    """
    GUARANTEES:
      - read(path) returns the full contents of the file at path as bytes.
      - read() raises FileNotFoundError if the file does not exist.
      - read() raises PermissionError if the process lacks read access.
      - The file is closed after every call; no file handles are held.

    MUST NOT ASSUME:
      - NOT thread-safe: concurrent read() calls on the same instance require external locking.
      - Does NOT cache: every call to read() performs a fresh disk read.
      - Does NOT watch for changes: contents returned reflect the file at call time only.
      - Returned bytes are NOT guaranteed to be UTF-8: caller must decode if text is expected.
      - Does NOT follow symlinks by default: symlinks are read as-is unless follow_symlinks=True.
    """
```

### Connection Pool

```python
class ConnectionPool:
    """
    GUARANTEES:
      - acquire() returns a live, authenticated connection to the database.
      - release(conn) returns the connection to the pool for reuse.
      - acquire() blocks until a connection is available if the pool is exhausted.
      - acquire() raises PoolTimeoutError if no connection becomes available within timeout_s.
      - Connections are validated before being returned by acquire().

    MUST NOT ASSUME:
      - NOT thread-safe at the connection level: a single acquired connection must not be
        shared across threads; acquire one connection per thread.
      - Does NOT auto-release: caller must call release() or use the context manager;
        failing to release causes pool exhaustion.
      - Connections are NOT isolated: two connections see each other's committed writes.
      - Does NOT retry failed queries: connection validity is checked at acquire() time only;
        a connection may fail mid-use due to network interruption.
      - Pool size is NOT elastic: the pool is sized at init time and does not grow.
    """
```

### Cache

```python
class ResultCache:
    """
    GUARANTEES:
      - get(key) returns the cached value if present and unexpired, else None.
      - set(key, value, ttl_s) stores value for at least ttl_s seconds.
      - delete(key) removes the entry immediately if present; is a no-op if absent.
      - Expired entries are never returned.

    MUST NOT ASSUME:
      - NOT thread-safe: concurrent access requires the caller to hold an external lock.
      - Does NOT persist: cache is in-memory; all entries are lost on process restart.
      - set() does NOT deduplicate: a second set() for the same key silently overwrites
        the first value and resets the TTL.
      - Returned values are NOT copies: mutating a returned object mutates the cached entry;
        call copy.deepcopy() if mutation is needed.
      - Eviction is NOT instantaneous: an entry may be returned for up to one eviction cycle
        (default 1 second) after its TTL expires.
    """
```

The five items in each `MUST NOT ASSUME:` block map to the five common caller mistake categories: thread-safety, persistence, idempotency, mutability, and timing.

---

## Negative Space Audit: Applying All 7 Principles to a Function

Starting function — 50 lines, zero negative space:

```python
def process_payment(user_id: int, amount: float, currency: str) -> dict:
    user = db.get_user(user_id)
    rate = fx.get_rate(currency, "USD")
    amount_usd = amount * rate
    db.debit_user(user_id, amount_usd)
    result = gateway.charge(user.payment_method, amount_usd)
    db.record_transaction(user_id, amount_usd, result["transaction_id"])
    return {"status": "ok", "transaction_id": result["transaction_id"]}
```

**NS1 — Add exclusion specification:**
```python
def process_payment(user_id: int, amount: float, currency: str) -> dict:
    """
    NOT HANDLED: user_id <= 0 — caller must validate user exists before calling.
    NOT HANDLED: amount <= 0 — caller must ensure amount is positive.
    NOT HANDLED: amount > PAYMENT_AMOUNT_MAX — caller must enforce single-payment cap.
    NOT HANDLED: unsupported currency codes — caller must validate against SUPPORTED_CURRENCIES.
    NOT HANDLED: user with no payment method — caller must verify payment method before calling.
    NOT HANDLED: concurrent payments for the same user — caller must hold a per-user lock.
    """
```

**NS2 — Add non-goal documentation at the module level:**
```python
"""
GOALS:
  - Debit the user account and submit a charge to the payment gateway.
  - Record the transaction in the database.

NON-GOALS:
  - Does NOT authenticate the caller — API layer owns authentication.
  - Does NOT validate user identity or existence — caller must confirm user exists.
  - Does NOT handle currency not in SUPPORTED_CURRENCIES — caller must validate.
  - Does NOT retry on gateway failure — caller is responsible for retry logic.
  - Does NOT handle refunds — see process_refund().

Caller is responsible for:
  - Holding a per-user lock before calling to prevent double-charge.
  - Validating payment method presence before calling.
"""
```

**NS3 — Name forbidden states:**
```python
# A payment may not be submitted for a user in a suspended state.
# Suspended users have had their payment method revoked; charging them bypasses
# the suspension and is a compliance violation.
FORBIDDEN_PAYMENT_STATES: frozenset[str] = frozenset({"suspended", "fraud_hold", "closed"})
```

**NS4 — Add paired invariant assertions:**
```python
AMOUNT_USD_MIN = 0.01
AMOUNT_USD_MAX = 10_000.00

# Pre-conditions
assert user.account_balance >= 0, f"pre: user {user_id} has negative balance"
assert user.status not in FORBIDDEN_PAYMENT_STATES, (
    f"pre: payment forbidden for user in state {user.status}"
)

# ... operation ...

# Post-conditions (NEVER)
# NEVER: balance may not go negative after debit
assert user.account_balance >= 0, f"post: debit caused negative balance for user {user_id}"
# NEVER: amount_usd must remain within bounds after fx conversion
assert AMOUNT_USD_MIN <= amount_usd <= AMOUNT_USD_MAX, (
    f"post: converted amount {amount_usd} out of bounds"
)
```

**NS5 — Extract boundary predicates:**
```python
# Valid: [AMOUNT_MIN, PAYMENT_AMOUNT_MAX] inclusive.
# Zero rejected: zero-value payments are no-ops and indicate caller bugs.
# Above PAYMENT_AMOUNT_MAX rejected: single-payment cap per compliance policy.
PAYMENT_AMOUNT_MIN = 0.01
PAYMENT_AMOUNT_MAX = 10_000.00

def is_valid_payment_amount(amount: float) -> bool:
    return PAYMENT_AMOUNT_MIN <= amount <= PAYMENT_AMOUNT_MAX
```

**NS6 — Write rejection tests first:**
```python
class TestProcessPayment:
    # Rejection tests first
    def test_rejects_zero_amount(self):
        with pytest.raises(AssertionError, match="amount"):
            process_payment(valid_user_id, 0.0, "USD")

    def test_rejects_negative_amount(self):
        with pytest.raises(AssertionError, match="amount"):
            process_payment(valid_user_id, -1.0, "USD")

    def test_rejects_amount_above_maximum(self):
        with pytest.raises(AssertionError, match="amount"):
            process_payment(valid_user_id, PAYMENT_AMOUNT_MAX + 0.01, "USD")

    def test_accepts_amount_at_maximum_boundary(self):
        result = process_payment(valid_user_id, PAYMENT_AMOUNT_MAX, "USD")
        assert result["status"] == "ok"

    def test_rejects_suspended_user(self):
        with pytest.raises(AssertionError, match="forbidden.*suspended"):
            process_payment(suspended_user_id, 10.0, "USD")

    def test_rejects_unsupported_currency(self):
        with pytest.raises(AssertionError):
            process_payment(valid_user_id, 10.0, "XYZ")

    # Acceptance tests follow
    def test_processes_valid_payment(self):
        result = process_payment(valid_user_id, 10.0, "USD")
        assert result["status"] == "ok"
        assert "transaction_id" in result
```

**NS7 — Add negative interface contract:**
```python
def process_payment(user_id: int, amount: float, currency: str) -> dict:
    """
    GUARANTEES:
      - Returns {"status": "ok", "transaction_id": str} on success.
      - Debits user account and records transaction atomically.
      - Raises AssertionError with a descriptive message on invalid inputs.
      - Raises GatewayError on payment gateway failure (account is NOT debited).

    MUST NOT ASSUME:
      - NOT idempotent: calling twice with the same arguments submits two charges.
      - NOT concurrency-safe: caller must hold a per-user lock to prevent double-charge.
      - Does NOT retry: a GatewayError means no charge was made; caller handles retry.
      - Returned transaction_id is NOT guaranteed unique across retries.
      - Currency conversion rate is NOT locked: rate is fetched at call time and may differ
        from any rate shown to the user in a prior step.
    """
```

After applying all seven principles, the function's negative space — what it rejects, ignores, and forbids — is as precisely specified as what it accepts.
