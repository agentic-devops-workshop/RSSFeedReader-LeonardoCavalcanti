<!--
SYNC IMPACT REPORT
==================
Version change: (initial template) → 1.0.0
Rationale: First ratified version. All placeholder tokens replaced with concrete,
project-specific principles derived from StakeholderDocuments/ (ProjectGoals.md,
AppFeatures.md, TechStack.md) and the user directive emphasizing security,
maintainability, and code quality.

Modified principles (template slot → concrete principle):
- [PRINCIPLE_1_NAME] → I. MVP-First, Incremental Delivery
- [PRINCIPLE_2_NAME] → II. Security by Default
- [PRINCIPLE_3_NAME] → III. Maintainable Architecture & Separation of Concerns
- [PRINCIPLE_4_NAME] → IV. Code Quality & Readability (NON-NEGOTIABLE)
- [PRINCIPLE_5_NAME] → V. Reproducible Cross-Platform Local Development

Added sections:
- Security & Compliance Requirements (was [SECTION_2_NAME])
- Development Workflow & Quality Gates (was [SECTION_3_NAME])

Removed sections: none

Templates requiring updates:
- ✅ .specify/templates/plan-template.md — Constitution Check gate is generic
  ("[Gates determined based on constitution file]") and now resolves to the
  principles below; no edits required.
- ✅ .specify/templates/spec-template.md — Scope/requirements structure remains
  compatible with the MVP-First and Security principles; no edits required.
- ✅ .specify/templates/tasks-template.md — Task categorization compatible with
  added quality, security, and maintainability principles; no edits required.
- ✅ .github/prompts/speckit.constitution.prompt.md — agent reference unchanged.
- ⚠ README.md — pending: optional update to link to this constitution once the
  MVP feature work begins.

Follow-up TODOs: none. RATIFICATION_DATE recorded as today since this is the
initial adoption.
-->

# RSS Feed Reader Constitution

## Core Principles

### I. MVP-First, Incremental Delivery

The project MUST ship the smallest demonstrable slice before adding scope.
The MVP is strictly limited to: (a) adding a feed subscription by URL and
(b) displaying the current subscription list from in-memory storage. Feed
fetching, parsing, persistence, removal, polling, and advanced UI are
Extended-MVP or post-MVP and MUST NOT be implemented as part of MVP work.

Rules:

- New functionality MUST map to the current phase (MVP, Extended-MVP, or
  post-MVP) declared in `StakeholderDocuments/AppFeatures.md`; out-of-phase
  work MUST be rejected in review.
- Storage in MVP MUST be in-memory only (e.g., `List<string>` or a minimal
  model); database, file persistence, and caching are forbidden until the
  post-MVP persistence feature is explicitly planned.
- Feature additions beyond the MVP MUST be justified in the plan's
  Complexity Tracking section before implementation begins.

Rationale: The project's stated purpose is a proof-of-concept focused on
subscription management. Scope creep destroys the value of a POC.

### II. Security by Default

Every change MUST default to the safer option, even in a proof-of-concept.
Security concerns escalate the moment Extended-MVP introduces network I/O
and XML parsing, so safe defaults are codified now.

Rules (MUST):

- No secrets, tokens, connection strings, or credentials may be committed
  to the repository. Configuration values that resemble secrets MUST be
  read from environment variables or local user-secrets.
- CORS in `backend/RSSFeedReader.Api/Program.cs` MUST whitelist only the
  explicit frontend origin(s) from `launchSettings.json`. Wildcards
  (`AllowAnyOrigin`) are prohibited.
- When feed fetching is introduced (Extended-MVP), the XML reader MUST
  disable DTD processing and external entity resolution to prevent XXE
  (e.g., `XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit,
  XmlResolver = null }`).
- User-supplied URLs MUST be parsed with `Uri.TryCreate(..., UriKind.Absolute, ...)`
  and restricted to `http`/`https` schemes before any network call.
- Any feed content rendered in the UI beyond plain `title` + `link` (a
  post-MVP feature) MUST pass through an HTML sanitizer; raw HTML
  injection into the DOM is prohibited.
- HTTP calls MUST use a configured `HttpClient` with an explicit timeout
  (≤ 10 seconds for MVP-scale usage) and MUST NOT follow non-HTTPS
  redirects from an HTTPS origin without explicit allow-listing.

Rationale: RSS readers historically suffer from XXE, SSRF, and stored-HTML
XSS. Codifying these defaults up front prevents costly retrofits.

### III. Maintainable Architecture & Separation of Concerns

The two-project layout (ASP.NET Core Web API backend + Blazor WebAssembly
frontend) MUST be preserved with strict separation of responsibilities.

Rules (MUST):

- Backend owns data and (later) feed operations; frontend owns UI and
  user interaction. Business logic in Razor components is prohibited
  beyond presentation concerns — move logic into services.
- API base URLs, ports, and other environment values MUST be read from
  configuration (`appsettings.json`, environment variables). Hardcoded
  URLs in `Program.cs` or component code are prohibited.
- Blazor template demonstration pages (`Home.razor`, `Counter.razor`,
  `Weather.razor`) and their `NavMenu` entries MUST be removed before
  any feature page using `@page "/"` is added; routing ambiguity is a
  blocker, not a warning.
- A class or component SHOULD remain under ~200 lines; exceeding that
  threshold MUST trigger an extraction discussion in review.
- Cross-cutting concerns (logging, configuration, HTTP client setup)
  MUST be registered in `Program.cs` via dependency injection, not
  instantiated ad hoc inside features.

Rationale: The stakeholder docs explicitly call out that incomplete
template cleanup and hardcoded ports waste development time at runtime.

### IV. Code Quality & Readability (NON-NEGOTIABLE)

Code quality gates apply to every change, regardless of scope.

Rules (MUST):

- `dotnet build` on both `backend/` and `frontend/` solutions MUST
  succeed with zero errors and zero new warnings before a change is
  merged. Suppressing warnings requires justification in the PR.
- Public types and public members MUST follow .NET naming conventions
  (`PascalCase` types/methods, `camelCase` locals/parameters,
  `_camelCase` private fields). Nullable reference types MUST remain
  enabled in project files.
- Public API surfaces (controllers, service interfaces, DTOs) MUST have
  XML doc comments describing intent; private helpers do not require
  comments unless behavior is non-obvious.
- Dead code, commented-out blocks, and `TODO` comments without a
  tracked owner or issue link are prohibited in merged code.
- Each change MUST update or extend any directly affected
  `StakeholderDocuments/` or `specs/` artifact when behavior changes.

Rationale: A POC that is hard to read becomes a POC that is hard to
extend into Extended-MVP. Quality discipline now is cheaper than
remediation later.

### V. Reproducible Cross-Platform Local Development

The application MUST run on Windows, macOS, and Linux from a clean
clone without manual file editing beyond documented configuration.

Rules (MUST):

- The three port/CORS coordination points described in
  `StakeholderDocuments/TechStack.md` (backend `launchSettings.json`,
  frontend `launchSettings.json`, `wwwroot/appsettings.json`, and
  backend CORS policy) MUST stay consistent; a change to one MUST be
  matched in the others within the same commit.
- Scripts and tooling MUST work in both POSIX shells and PowerShell, or
  provide an equivalent for the other platform. PowerShell-only
  commands in docs MUST be paired with a bash equivalent.
- The repository MUST NOT depend on globally installed local state
  (user-specific paths, IDE-specific files) for build or run. Any
  required SDK versions MUST be declared (e.g., `global.json` if a
  specific .NET SDK is pinned).
- A first-time contributor MUST be able to start backend and frontend
  with two documented `dotnet run` commands and reach a working UI
  with no further setup for MVP scope.

Rationale: The stakeholder docs explicitly target cross-platform
developer experience; reproducibility is part of the product.

## Security & Compliance Requirements

These constraints apply globally and supplement Principle II.

- **Dependencies**: New NuGet packages MUST be reviewed for license
  compatibility (permissive OSS only) and active maintenance (released
  within the last 24 months) before being added.
- **Input boundaries**: Every external input (HTTP request body/query,
  feed URL, feed XML payload) is treated as untrusted and validated at
  the boundary; downstream code MAY assume validated input.
- **Logging**: Logs MUST NOT contain feed URLs that include credentials
  (e.g., `https://user:pass@host/feed`); strip user-info before logging.
- **Error responses**: API error responses MUST NOT leak stack traces,
  framework versions, or internal paths to clients in any environment
  other than `Development`.
- **Transport**: Production-style runs (post-MVP) MUST default to HTTPS
  endpoints; HTTP is acceptable only for `localhost` development.

## Development Workflow & Quality Gates

These workflow rules supplement Principles III and IV.

- **Spec-Kit flow is authoritative**: feature work MUST follow
  `/speckit.specify` → `/speckit.plan` → `/speckit.tasks` →
  `/speckit.implement`. Skipping steps requires justification in the
  pull request description.
- **Constitution Check**: every `plan.md` MUST contain a populated
  Constitution Check section verifying each of the five principles;
  a violation MUST appear in the plan's Complexity Tracking table
  with a justification, or the plan MUST be revised.
- **Pull requests**: every PR MUST state (a) the phase it targets
  (MVP / Extended-MVP / post-MVP), (b) the principles it touches, and
  (c) confirmation that `dotnet build` is clean on both projects.
- **Review**: at least one reviewer MUST confirm compliance with this
  constitution before merge. Self-merge is prohibited except for
  documentation-only changes under `StakeholderDocuments/` or
  `specs/`.
- **Testing posture**: automated tests are not required for the MVP
  scope per `AppFeatures.md`, but any code introduced in or after the
  Extended-MVP phase that performs feed fetching, parsing, or URL
  validation MUST ship with at least one unit test covering the
  happy path and one covering an explicit failure mode.

## Governance

- **Supremacy**: This constitution supersedes ad hoc conventions,
  README notes, and prior verbal agreements. Where a stakeholder
  document conflicts with this constitution, the constitution wins
  until the document is updated and the constitution is amended.
- **Amendments**: Proposed changes MUST be made via a pull request
  that (a) edits this file, (b) updates the Sync Impact Report block
  at the top, (c) bumps the version per the rules below, and (d)
  updates any templates or docs listed as impacted.
- **Versioning policy** (semantic):
  - **MAJOR**: removing or redefining a principle in a backward-
    incompatible way, or removing a governance rule.
  - **MINOR**: adding a new principle or section, or materially
    expanding the scope of an existing principle.
  - **PATCH**: clarifications, wording fixes, typo corrections, or
    non-semantic refinements that do not change required behavior.
- **Compliance review**: every PR description MUST include a
  one-line statement of constitution compliance (e.g., "Complies
  with Principles I–V; no Complexity Tracking entries required.").
  Reviewers MUST reject PRs that omit this statement.
- **Runtime guidance**: agents and contributors MUST consult this
  file and the documents under `StakeholderDocuments/` before
  proposing implementation work; in case of conflict, raise an
  amendment PR rather than working around the constitution.

**Version**: 1.0.0 | **Ratified**: 2026-05-19 | **Last Amended**: 2026-05-19
