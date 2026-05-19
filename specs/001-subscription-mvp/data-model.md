# Phase 1 Data Model — Subscription Management MVP

**Feature**: 001-subscription-mvp · **Date**: 2026-05-19 · **Plan**: [plan.md](plan.md)

The MVP introduces a single entity. There is no relational model, no
persistence, and no state machine.

## Entity: `Subscription`

Represents one feed URL the user has chosen to track during the current
backend session.

### Fields

| Field | Type   | Required | Notes |
|-------|--------|----------|-------|
| `Url` | string | yes      | Free-text URL submitted by the user. Stored verbatim after `Trim()`. Treated as opaque — never dereferenced in MVP scope. |

### Validation rules

- The deserialized `Url` MUST be non-null and, after `Trim()`, MUST have
  length ≥ 1. Otherwise the boundary (HTTP `POST /api/subscriptions`)
  rejects the request with HTTP 400 and a user-friendly message
  (spec FR-002, FR-009).
- No format, scheme, host, length-upper-bound, or reachability check is
  performed in MVP scope (spec FR-003; research D-005). A future
  Extended-MVP iteration will introduce `Uri.TryCreate` + `http`/`https`
  scheme allow-listing at the same boundary (constitution Principle II).
- Whitespace handling: leading and trailing whitespace is stripped via
  `Trim()` before storage. Internal whitespace is preserved as-is (the
  spec treats the value as an opaque string).

### Ordering and identity

- Subscriptions have no identifier in MVP scope. Equality is by `Url`
  value plus position; duplicates are explicitly allowed
  (spec Edge Cases — "the user submits the same URL twice in a row").
- The collection MUST preserve insertion order (spec FR-006). The
  in-memory store therefore uses a `List<Subscription>` (not
  `ConcurrentBag`); see research D-003.

### Lifetime and persistence

- The `Subscription` instance lives only inside the backend process
  memory. The store is reset to empty whenever the backend process
  restarts (spec FR-005, SC-004; research D-003).
- No serialization to disk, database, distributed cache, or browser
  storage is performed.

### Equality / comparison

- No custom equality. Two `Subscription` records with the same `Url`
  are considered distinct list entries (duplicates allowed).

## Collection: `SubscriptionList`

Not a distinct .NET type — it is the public surface of
`ISubscriptionStore`. Modeled here only because the spec treats it as a
named concept.

### Operations

| Operation | Inputs | Outputs / Effects |
|-----------|--------|-------------------|
| `Add(Subscription)` | A validated `Subscription` | Appends to the in-memory list under a `lock`. Idempotency is **not** guaranteed — repeated calls with equal `Url` append additional entries. |
| `GetAll()` | none | Returns a snapshot (`IReadOnlyList<Subscription>`) of the current entries in insertion order. The snapshot is decoupled from the underlying list so that callers cannot mutate internal state. |

### State transitions

There are no nontrivial state transitions. The list moves between:

```
Empty  ──Add──▶  N entries  ──Add──▶  N+1 entries
                      │
                      └── process restart ──▶  Empty
```

## Mapping to wire contract

The wire DTO is identical in shape to the entity (one `Url` string). See
[contracts/subscriptions-api.md](contracts/subscriptions-api.md) for the
HTTP payloads.