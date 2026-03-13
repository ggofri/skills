# TigerStyle Skill

A Claude Code skill that encodes the [TigerStyle](https://tigerstyle.dev) development methodology from [TigerBeetle](https://tigerbeetle.com).

## What it does

When active, this skill guides Claude to apply TigerStyle's three core values — **Safety**, **Performance**, and **Experience** — when writing, reviewing, or discussing code.

## Triggers

This skill activates when you:

- Mention "tigerstyle", "tiger style", or "TigerBeetle"
- Ask to review code for TigerStyle compliance
- Ask about static allocation, assertions, zero-copy, or mechanical sympathy
- Ask about the primary colors framework
- Ask about naming conventions (big-endian qualification, unit suffixes)
- Are working in a TigerBeetle-style codebase

**Example prompts:**
```
Apply TigerStyle principles to this function
Review this code for TigerBeetle style compliance
How should I name this variable in TigerStyle?
We're using static allocation — is this TigerStyle compliant?
```

## Files

```
tigerstyle/
├── SKILL.md                  # Main skill definition
├── README.md                 # This file
└── references/
    └── rules.md              # Extended rules with rationale and code examples
```

## Core Values

| Value | Key Rules |
|-------|-----------|
| **Safety** | Assert everything · static alloc · explicit bounds · no recursion · zero deps |
| **Performance** | Back-of-envelope first · primary colors framework · control/data plane split · zero copy |
| **Experience** | Big-endian names · unit suffixes · no abbreviations · one function one thing |

## References

- [tigerstyle.dev](https://tigerstyle.dev) — official TigerStyle documentation
- [TigerBeetle](https://github.com/tigerbeetle/tigerbeetle) — source codebase
