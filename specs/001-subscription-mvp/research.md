# Phase 0 Research — Subscription Management MVP

**Feature**: 001-subscription-mvp · **Date**: 2026-05-19 · **Plan**: [plan.md](plan.md)

The Technical Context in [plan.md](plan.md) contains no `NEEDS CLARIFICATION`
markers — the stakeholder documents and the project constitution constrain
every meaningful decision. This document records the decisions that were
implicit in those constraints so that downstream commands and reviewers can
trace each choice.

## Decisions

### D-001: Two-project layout (`backend/` + `frontend/`)

- **Decision**: Place the ASP.NET Core Web API under
  `backend/RSSFeedReader.Api/` and the Blazor WebAssembly app under
  `frontend/RSSFeedReader.UI/`.
- **Rationale**: The exact paths are already referenced verbatim in
  [StakeholderDocuments/TechStack.md](../../StakeholderDocuments/TechStack.md)
  (port configuration block). Reusing them keeps the port/CORS guidance in
  that doc directly actionable without editing it.
- **Alternatives considered**:
  - Single-project Blazor Server: rejected — TechStack.md mandates Blazor
    WebAssembly + Web API to demonstrate the API contract.
  - Solution file with a `shared/` project for DTOs: rejected for MVP —
    the DTO has one field; duplicating two tiny records is cheaper than
    introducing a third project and changing build scripts. Will be
    reconsidered if a third consumer emerges in Extended-MVP.

### D-002: Minimal APIs over MVC controllers

- **Decision**: Use ASP.NET Core minimal APIs (`MapPost`, `MapGet`) inside
  a `SubscriptionEndpoints` static class.
- **Rationale**: Two endpoints with trivial payloads; minimal APIs cut
  ceremony and align with the MVP-First principle. Class size stays well
  under the constitution's ~200-line soft limit.
- **Alternatives considered**:
  - `[ApiController]` MVC: rejected — adds attribute routing, model
    binders, and filter pipeline that bring no value at this scale.

### D-003: In-memory store as a singleton service

- **Decision**: Introduce `ISubscriptionStore` with an
  `InMemorySubscriptionStore` implementation backed by a `List<Subscription>`
  guarded by a `lock` object; register it as a singleton in DI.
- **Rationale**: Singleton lifetime matches "for the duration of the running
  session" (FR-004) and the constitution's "in-memory only" rule. Putting
  the store behind an interface localizes the future replacement when
  persistence is introduced post-MVP, satisfying Principle III.
- **Alternatives considered**:
  - `ConcurrentBag<Subscription>`: rejected — ordering is part of the
    contract (FR-006 "in the order they were added"), and `ConcurrentBag`
    does not preserve insertion order on enumeration.
  - Static field on the endpoints class: rejected — defeats DI, blocks
    future swap-in of a persistent store, and violates Principle III.

### D-004: Configuration-driven API base URL and CORS origin

- **Decision**: Read `ApiBaseUrl` from
  `frontend/RSSFeedReader.UI/wwwroot/appsettings.json`; read allowed CORS
  origins from a `Cors:AllowedOrigins` array in
  `backend/RSSFeedReader.Api/appsettings.json`. No hardcoded URLs in
  `Program.cs` on either side.
- **Rationale**: Constitution Principle III explicitly forbids hardcoded
  URLs; TechStack.md identifies the three coordination points (backend
  launchSettings, frontend launchSettings, frontend appsettings) plus the
  CORS policy. Keeping the values in configuration keeps the four locations
  in sync via a single source of truth on each side.
- **Alternatives considered**:
  - `WithOrigins("http://localhost:5213", ...)` hardcoded: rejected —
    Principle III violation.
  - Wildcard CORS (`AllowAnyOrigin`): explicitly forbidden by
    Principle II.

### D-005: URL accepted as opaque string (no validation beyond non-empty)

- **Decision**: The backend trims the incoming `Url` string; if the result
  is empty, it returns HTTP 400 with a non-technical message. Otherwise it
  stores the value as-is.
- **Rationale**: Directly mirrors FR-002 and FR-003 in the spec, which in
  turn mirror the explicit "no validation of feed URLs (assume user
  provides valid URLs)" direction in
  [StakeholderDocuments/ProjectGoals.md](../../StakeholderDocuments/ProjectGoals.md).
- **Alternatives considered**:
  - `Uri.TryCreate(..., UriKind.Absolute, ...)` with `http`/`https`
    restriction: deferred to Extended-MVP. The constitution's Principle II
    requires this *before any network call* — and MVP makes no network
    call. Adding the check now would contradict MVP-First and the spec.

### D-006: Single landing page handles both add and list

- **Decision**: A single `Subscriptions.razor` page at `@page "/"` renders
  the add form and the list. No tabs, no separate detail page.
- **Rationale**: User Stories 1 and 2 are both P1 and described as a
  single demonstrable slice. One page minimises navigation and clicks for
  SC-001 (30-second first success).
- **Alternatives considered**:
  - Split add / list routes: rejected — adds routing, navigation, and
    state-sharing concerns out of proportion to the MVP value.

### D-007: Template demo pages removed before adding `Subscriptions.razor`

- **Decision**: Delete `Home.razor`, `Counter.razor`, `Weather.razor` and
  their NavMenu entries as part of the foundational setup, *before* the
  first Razor page is added.
- **Rationale**: Constitution Principle III and TechStack.md both call
  this out by name as a runtime-error trap (ambiguous route on `/`).
- **Alternatives considered**:
  - Leave demo pages and use `@page "/subscriptions"`: rejected — does not
    satisfy Principle III's literal wording ("MUST be removed before any
    feature page using `@page \"/\"` is added"), and stakeholder docs
    require the landing page to be the subscriptions page.

### D-008: No automated test project for MVP

- **Decision**: Do not create a test project as part of this feature.
- **Rationale**: AppFeatures.md and constitution Development Workflow &
  Quality Gates explicitly state "automated tests are not required for the
  MVP scope". The mandatory testing posture kicks in at Extended-MVP for
  fetching/parsing/validation. Manual smoke testing via `quickstart.md` is
  sufficient and covers all five SC-* outcomes.
- **Alternatives considered**:
  - xUnit project skeleton with placeholder tests: rejected — adds CI
    surface and dead code (forbidden by Principle IV) without exercising
    any non-trivial behavior at MVP.

### D-009: HTTP-only loopback for MVP local development

- **Decision**: Both projects listen on `http://localhost:5151` (API) and
  `http://localhost:5213` (UI). HTTPS dev certificate is not part of MVP
  setup steps.
- **Rationale**: Constitution Security & Compliance explicitly permits
  HTTP for `localhost` development. It cuts one setup step
  (`dotnet dev-certs https --trust`) from the new-contributor flow,
  contributing to SC-001 (30 s) and Principle V (reproducibility).
- **Alternatives considered**:
  - HTTPS-only with dev certs: deferred until production-style runs or
    Extended-MVP, when actual feed fetching introduces real transport
    concerns.

## Open items

None. All Technical Context fields are resolved.