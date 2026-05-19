# Quickstart — Subscription Management MVP

**Feature**: 001-subscription-mvp · **Date**: 2026-05-19 · **Plan**: [plan.md](plan.md)

This document walks a first-time contributor through running the MVP and
verifying every Success Criterion in [spec.md](spec.md). It is the only
testing artifact required for the MVP (research D-008).

## Prerequisites

- .NET 8 SDK installed and on `PATH` (`dotnet --version` ≥ `8.0.0`).
- A modern browser (Chromium, Firefox, or Safari).
- The repository cloned and the `001-subscription-mvp` branch checked out.

No HTTPS development certificate setup is required (research D-009).

## Run the application

Open two terminals from the repository root.

### Terminal 1 — backend (Web API)

```bash
dotnet run --project backend/RSSFeedReader.Api
```

Expected: the console reports `Now listening on: http://localhost:5151`.

### Terminal 2 — frontend (Blazor WebAssembly)

```bash
dotnet run --project frontend/RSSFeedReader.UI
```

Expected: the console reports `Now listening on: http://localhost:5213`.

Open <http://localhost:5213> in your browser. The Subscriptions page should
load with an empty list and a single input + button for adding a URL.

## Verify Success Criteria

The numbering matches the spec.

### SC-001 — first success in under 30 seconds

1. Start the stopwatch when the page renders.
2. Paste `https://devblogs.microsoft.com/dotnet/feed/` into the input.
3. Click **Add**.
4. The URL appears in the list below.

**Pass** if steps 1–4 complete in under 30 seconds without consulting any
documentation other than the on-screen UI.

### SC-002 — every successful add appears immediately

1. Type `https://example.com/feed` and submit.
2. The list updates with two entries: the previous URL and the new one,
   in insertion order, **before** you click or type anything else.

**Pass** if the list reflects the addition with no manual refresh.

### SC-003 — empty/whitespace submissions are blocked

1. Clear the input.
2. Click **Add**.
3. The UI shows a non-technical message such as
   `Please enter a feed URL before adding a subscription.` The list is
   unchanged.
4. Repeat with `"   "` (three spaces). Same outcome.

**Pass** if neither attempt adds an entry and the message is shown both
times.

### SC-004 — restart clears the list

1. With at least one subscription in the list, stop the backend
   (`Ctrl+C` in Terminal 1).
2. Restart it with the same `dotnet run` command.
3. Reload the browser tab.

**Pass** if the list is empty after the reload.

### SC-005 — failure messages are free of internals

1. With the frontend page open, stop the backend (`Ctrl+C` in Terminal 1).
2. Try to add `https://example.com/feed`.
3. The UI shows a user-friendly message such as
   `We couldn't reach the subscriptions service. Please try again.`

**Pass** if the message contains no stack trace, no `Microsoft.*` /
`System.*` namespace, no `Program.cs:line` reference, and no internal
file paths.

## Acceptance scenario coverage

| Spec scenario | Quickstart step |
|---------------|------------------|
| Story 1, scenario 1 (add to empty list) | SC-001 |
| Story 1, scenario 2 (add to non-empty list) | SC-002 |
| Story 1, scenario 3 (empty submission rejected) | SC-003 |
| Story 2, scenario 1 (list shows all added) | SC-001 + SC-002 |
| Story 2, scenario 2 (list updates immediately) | SC-002 |
| Story 2, scenario 3 (empty-state indicator) | Initial page load + SC-004 |
| Edge case — backend unreachable | SC-005 |
| Edge case — restart loses state | SC-004 |

## If something goes wrong

- **Browser console shows a CORS error**: verify
  `backend/RSSFeedReader.Api/appsettings.json` includes the frontend
  origin from `frontend/RSSFeedReader.UI/Properties/launchSettings.json`
  under `Cors:AllowedOrigins`. Constitution Principle II forbids
  `AllowAnyOrigin`.
- **Browser console shows the wrong API URL**: verify
  `frontend/RSSFeedReader.UI/wwwroot/appsettings.json` contains the
  backend port from
  `backend/RSSFeedReader.Api/Properties/launchSettings.json` in
  `ApiBaseUrl`. The three port references (backend launchSettings,
  frontend launchSettings, frontend appsettings) plus the backend CORS
  list MUST stay in sync (constitution Principle V).
- **`InvalidOperationException: ambiguous routes`**: confirm
  `Home.razor`, `Counter.razor`, and `Weather.razor` were deleted and
  the NavMenu cleaned up (constitution Principle III; TechStack.md).
- **Both terminals print warnings on build**: stop, fix, and re-run.
  Constitution Principle IV requires warning-free builds.