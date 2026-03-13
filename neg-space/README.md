# Negative Space Programming Skill

A Claude Code skill for Negative Space Programming — a methodology where code is specified as much by what it rejects, ignores, and forbids as by what it accepts.

## What it does

When active, this skill guides Claude to treat the invalid domain as a first-class design artifact. Every function must enumerate its rejected inputs, every module must document its non-goals, every invariant must have a paired negative form, and every public interface must specify what callers may not assume. The methodology is organized around seven principles across three disciplines: Specification (NS1–NS3), Invariants (NS4–NS5), and Testing & Contracts (NS6–NS7).

## Triggers

This skill activates when you:

- Say "negative space programming" or "negative space"
- Ask to enumerate invalid inputs or forbidden states
- Ask to document what a function or module does NOT handle
- Ask about `NOT HANDLED:` annotations or `NON-GOALS:` blocks
- Ask about rejection-first testing or boundary testing
- Ask about invariant negation or `NEVER` assertions
- Ask about `MUST NOT ASSUME:` or negative interface contracts
- Ask to define exclusion specifications or boundary predicates

**Example prompts:**
```
Apply negative space programming to this module
Add NOT HANDLED annotations to this function
Write the NON-GOALS block for this service
Review this code for negative space compliance
Add rejection-first tests for this validator
Document the MUST NOT ASSUME contract for this class
Enumerate the forbidden states in this state machine
Define a named boundary predicate for this validation logic
```

## Files

```
negative-space/
├── SKILL.md                  # Main skill definition (NS1–NS7 principles, checklist, quick ref)
├── README.md                 # This file
└── references/
    └── patterns.md           # Extended worked examples per principle
```

## Core Disciplines

| Discipline | Principles | Key Artifacts |
|------------|------------|---------------|
| **Specification** | NS1, NS2, NS3 | `NOT HANDLED:` annotations · `NON-GOALS:` blocks · named forbidden states |
| **Invariants** | NS4, NS5 | Paired `NEVER` assertions · named boundary predicates · closed-interval docs |
| **Testing** | NS6 | Rejection-first test suites · boundary at min−1/min/max/max+1 |
| **Contracts** | NS7 | `GUARANTEES:` + `MUST NOT ASSUME:` on every public interface |

## Principles

| ID | Name | Description |
|----|------|-------------|
| NS1 | Exclusion Specification | Enumerate rejected inputs before implementation; annotate with `NOT HANDLED:` |
| NS2 | Non-Goal Documentation | Pair every `GOALS:` block with a `NON-GOALS:` block; delegate explicitly |
| NS3 | Forbidden State Enumeration | Name impossible states as constants; document WHY each is forbidden |
| NS4 | Invariant Negation | Pair every positive invariant with a `NEVER` assertion; assert at every transition |
| NS5 | Boundary Articulation | Extract valid/invalid boundary into a named predicate; document as closed interval |
| NS6 | Rejection-First Testing | Write rejection tests first; test boundary explicitly; test every `NOT HANDLED:` annotation |
| NS7 | Negative Interface Contracts | Require both `GUARANTEES:` and `MUST NOT ASSUME:` on every public interface |
