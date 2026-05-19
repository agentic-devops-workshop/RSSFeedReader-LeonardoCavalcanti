# Contract: Subscriptions API (MVP)

**Feature**: 001-subscription-mvp · **Date**: 2026-05-19 · **Plan**: [../plan.md](../plan.md)

This is the only external interface the backend exposes in MVP scope. It is
consumed by the Blazor WebAssembly frontend (`SubscriptionsClient`). All
endpoints are unauthenticated (single local user) and return JSON.

Base URL (development): `http://localhost:5151/api/`

---

## `POST /api/subscriptions`

Add a new subscription to the in-memory list.

### Request

- **Headers**:
  - `Content-Type: application/json`
- **Body**:

  ```json
  {
    "url": "https://devblogs.microsoft.com/dotnet/feed/"
  }
  ```

- **Field**: `url` — string, required. Server trims leading/trailing
  whitespace before validation. Must be non-empty after trimming. No
  format check is performed (research D-005).

### Responses

- **`201 Created`** — Subscription accepted.

  - **Headers**:
    - `Location: /api/subscriptions` *(the list endpoint; MVP does not
      assign per-resource URLs)*
  - **Body**:

    ```json
    {
      "url": "https://devblogs.microsoft.com/dotnet/feed/"
    }
    ```

- **`400 Bad Request`** — `url` is null, missing, or only whitespace.

  - **Body** (`application/problem+json` — RFC 7807):

    ```json
    {
      "type": "about:blank",
      "title": "URL is required",
      "status": 400,
      "detail": "Please enter a feed URL before adding a subscription."
    }
    ```

  - The `detail` text MUST be user-friendly and MUST NOT include stack
    traces, framework names, or internal paths (spec FR-009, constitution
    Security & Compliance).

### Behavior

- Duplicates are accepted (spec Edge Cases). A second POST with the same
  `url` produces another `201` and another list entry.
- Ordering is preserved: the new entry is appended to the end of the list
  returned by `GET /api/subscriptions`.

---

## `GET /api/subscriptions`

Return all subscriptions added during the current backend session.

### Request

- **Headers**: none required.
- **Body**: none.

### Responses

- **`200 OK`** — Always, even when the list is empty.

  - **Body** — array of subscriptions in insertion order:

    ```json
    [
      { "url": "https://devblogs.microsoft.com/dotnet/feed/" },
      { "url": "https://example.com/feed" }
    ]
    ```

  - **Empty body for empty list**:

    ```json
    []
    ```

### Behavior

- Returns a snapshot at the time of the call; concurrent additions during
  serialization are not reflected.
- No pagination, filtering, or sorting parameters are accepted in MVP
  scope (spec FR-010).

---

## Error model (global)

- All error responses use `application/problem+json` per RFC 7807.
- In `Production`, `Staging`, or any non-`Development` environment, error
  responses MUST NOT include `detail`, `instance`, or `extensions` fields
  that contain stack traces, framework identifiers, or internal paths
  (constitution Security & Compliance, spec FR-009).
- In `Development`, the standard ASP.NET Core developer exception page is
  acceptable for unhandled exceptions; the `Production` pipeline maps
  exceptions to opaque `500 Internal Server Error` problem responses.

## CORS

- Preflight and actual requests from the explicit frontend origin
  (`http://localhost:5213` by default, plus any other origin listed in
  `Cors:AllowedOrigins`) MUST be allowed for `GET` and `POST` on
  `/api/subscriptions`.
- No other origin is permitted. Wildcards (`*`) are forbidden by
  constitution Principle II.

## Out of scope for this contract (MVP)

- `DELETE /api/subscriptions/{id}` — deferred to post-MVP (spec FR-010).
- `GET /api/subscriptions/{id}/items` — Extended-MVP feature.
- `POST /api/subscriptions/refresh` — Extended-MVP feature.
- Authentication/authorization headers — single local user.
- ETag / `If-None-Match` caching — post-MVP optimization.