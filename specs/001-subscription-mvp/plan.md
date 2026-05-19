# Implementation Plan: Subscription Management MVP

**Branch**: `001-subscription-mvp` | **Date**: 2026-05-19 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-subscription-mvp/spec.md`

## Summary

Deliver the smallest possible end-to-end slice of the RSS Feed Reader: a Blazor
WebAssembly UI that lets a single local user submit a feed URL as free text and
immediately see it appear in a subscription list, backed by an ASP.NET Core
Web API that stores subscriptions in an in-memory list for the lifetime of the
backend process. No feed fetching, parsing, validation, removal, or persistence
is included in this scope (per `StakeholderDocuments/AppFeatures.md` and the
project constitution's Principle I).

Technical approach: two .NET projects (`backend/RSSFeedReader.Api`,
`frontend/RSSFeedReader.UI`) wired via a thin `POST /api/subscriptions` and
`GET /api/subscriptions` contract that exchanges a `Subscription { url }` DTO.
A thread-safe in-memory store registered as a singleton holds the list. The
frontend uses a typed `HttpClient` configured from `appsettings.json` and a
single Razor page that handles both the add form and the list display.

## Technical Context

**Language/Version**: C# 12 / .NET 8 (LTS)

**Primary Dependencies**: ASP.NET Core 8 Web API (minimal APIs), Blazor
WebAssembly 8, `System.Net.Http.Json` for typed `HttpClient` use. No other
runtime packages required for MVP.

**Storage**: In-memory only — a singleton `List<Subscription>` guarded by a
lock (or `ConcurrentBag` / `ImmutableList` swap) inside the backend process.
No database, no file, no browser storage. Explicitly forbidden by
Constitution Principle I for MVP scope.

**Testing**: Not required for the MVP per `AppFeatures.md` and constitution
Principle IV/Workflow ("automated tests are not required for the MVP scope").
A single smoke verification via `quickstart.md` is sufficient. Unit tests
will be introduced from Extended-MVP onward when feed fetching is added.

**Target Platform**: Local developer machine — Windows, macOS, Linux —
running .NET 8 SDK. Browser: any modern evergreen (Chromium, Firefox,
Safari) capable of running Blazor WebAssembly.

**Project Type**: Web application (backend Web API + frontend SPA).

**Performance Goals**: First-time user reaches a working "add + see list"
flow within 30 seconds (SC-001). Single-user, sub-second add/list round
trips on `localhost`. Performance is not a constraint for MVP given
in-memory storage and trivial payloads.

**Constraints**:

- No persistence (FR-005).
- No URL/feed validation (FR-003); only empty/whitespace is rejected
  (FR-002).
- CORS in the backend must allow only the explicit frontend origin from
  `frontend/RSSFeedReader.UI/Properties/launchSettings.json`
  (Constitution Principle II).
- HTTP-only loopback is acceptable for MVP; HTTPS becomes mandatory
  starting at production-style runs (Constitution Security & Compliance).
- Error responses MUST NOT leak stack traces outside `Development`
  (Constitution Security & Compliance).

**Scale/Scope**: 1 local user, tens of subscriptions per session (FR + the
"Subscription list size during a single session is small" assumption).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Reference: [.specify/memory/constitution.md](../../.specify/memory/constitution.md) (v1.0.0).

| Principle | Verdict | Evidence |
|-----------|---------|----------|
| I. MVP-First, Incremental Delivery | PASS | Scope = `POST/GET /api/subscriptions` only; in-memory store; FR-010 explicitly excludes fetching/parsing/removal/persistence/deduplication. |
| II. Security by Default | PASS | CORS whitelists exactly the frontend origin from `launchSettings.json`; no secrets in repo; URL strings are stored as opaque text and never dereferenced in this scope, so XXE/SSRF are not in the threat model yet; empty/whitespace rejection at the boundary (FR-002). Logs do not include user-info components because no URL parsing occurs in MVP. |
| III. Maintainable Architecture & SoC | PASS | Backend owns data, frontend owns UI; API base URL read from `wwwroot/appsettings.json`; Blazor template demo pages (`Home.razor`, `Counter.razor`, `Weather.razor`) and their NavMenu entries removed in Phase 2 Foundational before adding `Subscriptions.razor`. No business logic embedded in Razor markup. |
| IV. Code Quality & Readability (NON-NEGOTIABLE) | PASS | Two `dotnet build` runs (backend + frontend) MUST be warning-free at every commit; nullable enabled; public types/members get XML docs; no dead code. |
| V. Reproducible Cross-Platform Local Dev | PASS | Two `dotnet run` commands documented in `quickstart.md`; ports coordinated across backend `launchSettings.json`, frontend `launchSettings.json`, `wwwroot/appsettings.json`, and CORS policy as required by Constitution. |

**Gate result**: PASS — no Complexity Tracking entries required.

## Project Structure

### Documentation (this feature)

```text
specs/001-subscription-mvp/
├── plan.md              # This file
├── research.md          # Phase 0 — decisions & alternatives
├── data-model.md        # Phase 1 — Subscription entity
├── quickstart.md        # Phase 1 — run + verify the MVP
├── contracts/
│   └── subscriptions-api.md  # Phase 1 — REST contract
├── checklists/
│   └── requirements.md  # Spec quality checklist (from /speckit.specify)
└── tasks.md             # Phase 2 — produced later by /speckit.tasks
```

### Source Code (repository root)

```text
backend/
└── RSSFeedReader.Api/
    ├── RSSFeedReader.Api.csproj
    ├── Program.cs                       # Minimal API entry point, DI, CORS
    ├── Properties/
    │   └── launchSettings.json          # Backend port (http://localhost:5151)
    ├── appsettings.json
    ├── appsettings.Development.json
    ├── Models/
    │   └── Subscription.cs              # { string Url }
    ├── Services/
    │   ├── ISubscriptionStore.cs        # Add(url), GetAll()
    │   └── InMemorySubscriptionStore.cs # Singleton, lock-guarded list
    └── Endpoints/
        └── SubscriptionEndpoints.cs     # POST /api/subscriptions, GET /api/subscriptions

frontend/
└── RSSFeedReader.UI/
    ├── RSSFeedReader.UI.csproj
    ├── Program.cs                       # HttpClient + config wiring
    ├── Properties/
    │   └── launchSettings.json          # Frontend port (http://localhost:5213)
    ├── wwwroot/
    │   ├── index.html
    │   └── appsettings.json             # { "ApiBaseUrl": "http://localhost:5151/api/" }
    ├── Layout/
    │   ├── MainLayout.razor
    │   └── NavMenu.razor                # Only "Subscriptions" link
    ├── Pages/
    │   ├── Subscriptions.razor          # @page "/" — add form + list
    │   └── NotFound.razor               # @page "/not-found"
    ├── Models/
    │   └── Subscription.cs              # Shared DTO shape (mirrors backend)
    └── Services/
        └── SubscriptionsClient.cs       # Typed HttpClient wrapper

specs/001-subscription-mvp/
└── (see Documentation tree above)
```

**Structure Decision**: Two .NET projects under `backend/` and `frontend/`,
matching the directory layout already referenced verbatim in
`StakeholderDocuments/TechStack.md` (`backend/RSSFeedReader.Api/`,
`frontend/RSSFeedReader.UI/`). This keeps port/CORS coordination
instructions in the stakeholder doc directly actionable and satisfies
Constitution Principle III.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No violations — this section is intentionally empty.
