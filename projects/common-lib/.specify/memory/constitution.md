<!--
SYNC IMPACT REPORT
==================
Version Change: 1.0.0 -> 2.0.0
Bump Rationale: MAJOR — persistence platform redefined (MySQL/H2 -> PostgreSQL/embedded PostgreSQL),
  testing discipline redefined (tests-after -> mandatory TDD with enforced coverage), inter-service
  communication restricted to REST, and the principle set restructured (6 -> 8).
Modified Principles:
  - I. Clean Architecture & Absolute Dependency Rule -> I. Clean Architecture & the Dependency Rule
  - II. Stateless Microservice & Explicit API Contracts -> split into
      III. Stateless, Single-Context REST Microservice and IV. Explicit API Contracts
  - III. Test-Driven Quality & Isolated Domain Testing -> V. Test-Driven Development
  - IV. Defense-in-Depth Security & Stateless Authentication -> VI. Secure by Default (adds RBAC)
  - V. First-Class Observability & Structured Telemetry -> VII. Built-In Observability (Grafana stack)
  - VI. Clean Code & Modern Java 25 Standards -> II. Clean Code Craftsmanship
Added Sections:
  - VIII. Resilience & Scalability (new principle)
  - Technology Stack & Engineering Standards: Deployment (Kubernetes on AWS), Automated Quality
    Tooling (Spotless, Checkstyle, ArchUnit, JaCoCo), DI/DTOs, Validation & Exceptions,
    Persistence, Configuration, Logging, Coding Standards
Removed Sections: None (all prior rules retained or strengthened)
Templates Requiring Updates:
  - ✅ .specify/templates/plan-template.md — Constitution Check gate list populated
  - ✅ .specify/templates/tasks-template.md — tests mandatory and written first (TDD)
  - ✅ .specify/templates/spec-template.md — no constitution-driven mandatory sections; no change
  - ✅ CLAUDE.md — persistence, testing, tooling, and deployment summary updated
  - ✅ AGENTS.md — contains only the Spec Kit pointer; no change
Follow-up TODOs:
  - TODO(BOOT_VERSION): pom.xml uses spring-boot-starter-parent 4.2.0-SNAPSHOT; this constitution
    mandates 4.1.1 stable. Align pom.xml in the first foundational feature.
  - TODO(TOOLING): JaCoCo, Spotless, Checkstyle, ArchUnit, Resilience4j, Springdoc, embedded
    PostgreSQL, Flyway, and the OpenTelemetry tracing bridge are mandated but not yet in pom.xml.
-->

# Common Lib Constitution

## Core Principles

### I. Clean Architecture & the Dependency Rule (NON-NEGOTIABLE)

Source code MUST be organised into four explicit layers: **Domain**, **Application**,
**Infrastructure**, and **Presentation (API)**.

- **Dependency Rule**: Outer layers MAY depend on inner layers; inner layers MUST NEVER depend on
  outer layers. There are no exceptions.
- **Domain**: Entities, value objects, aggregates, and domain services. ZERO dependencies on
  frameworks, persistence, or infrastructure libraries. Framework annotations (`@Entity`,
  `@Service`, `@Repository`, `@Component`, Jakarta Persistence, Jackson, etc.) MUST NOT appear here.
- **Application**: Use-case interactors and application services. Depends only on Domain.
  Defines ports (interfaces) for every external dependency — persistence, outbound HTTP, clocks,
  ID generation.
- **Use cases**: Every use case MUST be a dedicated class exposing exactly one public method.
  Grouping several operations into one service class is forbidden.
- **Infrastructure**: Implements Application ports — Spring Data JPA repositories, JPA entities,
  outbound HTTP clients, persistence configuration.
- **Presentation (API)**: Spring MVC REST controllers and request/response DTOs. Depends only on
  Application-layer use cases and ports. Controllers MUST only translate HTTP input into a use-case
  call and serialise the result; they MUST contain zero business logic.
- **Business logic** lives exclusively in the Domain and Application layers.
- **Enforcement**: The Dependency Rule MUST be verified automatically by ArchUnit tests in the build.

**Rationale**: Isolating business rules from frameworks and storage keeps the core testable without
infrastructure and lets technology choices change without rewriting behaviour.

### II. Clean Code Craftsmanship (NON-NEGOTIABLE)

Every class and method MUST meet these standards; reviewers MUST reject code that does not.

- **Meaningful names**: Class, method, and variable names MUST express intent without a comment.
- **Small methods**: A method does one thing. Any method over 20 lines MUST be justified in review
  or decomposed.
- **Single Responsibility**: Every class has exactly one reason to change.
- **No magic numbers or strings**: Use named constants or enums.
- **No dead code**: Commented-out code, unused imports, and unreachable branches MUST be removed.
- **Shallow nesting**: Nesting MUST NOT exceed two levels; use guard clauses, early returns, and
  extraction.
- **DRY**: Duplicate logic MUST be extracted into a shared abstraction before a second use is
  committed.
- **Comments explain why, not what**: Comments are reserved for non-obvious business rationale.
- **Boy Scout Rule**: Every change leaves the touched files slightly cleaner than before.

**Rationale**: Code is read far more often than it is written; consistent craftsmanship keeps
the cost of change low for the life of the service.

### III. Stateless, Single-Context REST Microservice

- **Pure API**: The service is a REST API only. It MUST NOT serve a UI, render server-side views,
  or host static web assets.
- **Statelessness**: No server-side session state. All state lives in the database or in the
  validated client token. Any instance MUST be able to serve any request.
- **Bounded context**: The service owns exactly one bounded context. Capabilities belonging to
  another domain MUST be placed in another service, not added here.
- **Inter-service communication**: Communication with other services MUST be synchronous REST/HTTP.
  In-process side effects MAY use Spring application events. Introducing a message broker or gRPC
  requires a constitution amendment.
- **Single datastore**: PostgreSQL is the only datastore. Adding another store (cache, document,
  key-value) requires a constitution amendment.
- **Single tenant**: The service is not multi-tenant; tenant isolation mechanisms MUST NOT be
  introduced without a constitution amendment.
- **12-Factor**: Configuration is externalised, backing services are attached resources, and
  dev/prod parity is maintained.
- **No hard-coded environment values**: All environment-specific values MUST be injected via
  environment variables or a secrets manager.

**Rationale**: Stateless, single-purpose services scale horizontally, deploy independently, and
fail in isolation; a deliberately narrow technology surface keeps them operable.

### IV. Explicit API Contracts

- **REST semantics**: APIs MUST follow REST principles and standard HTTP method semantics
  (GET, POST, PUT, PATCH, DELETE) and status codes.
- **Versioning**: Every endpoint MUST be versioned in the URL path (`/api/v1/...`). Breaking
  changes to a published version are forbidden; they require a new version.
- **Errors**: Every error response MUST be RFC 9457 Problem Details (`application/problem+json`).
  Stack traces, internal class names, exception messages, and database details MUST NEVER be
  returned.
- **Consistency**: Success responses MUST use a consistent response shape and HTTP status-code
  conventions across the whole service.
- **Documentation**: Every endpoint MUST be documented with OpenAPI 3.1 annotations (Springdoc),
  including its required roles. The specification is generated from code and MUST always match
  the implementation.
- **Pagination**: Every collection endpoint MUST be paginated (cursor- or offset-based) with an
  enforced maximum page size.
- **Idempotency**: Mutating endpoints MUST be idempotent wherever technically possible (e.g., via
  natural keys or an `Idempotency-Key` header for non-idempotent creates).

**Rationale**: Consumers depend on the contract, not the code; a stable, documented, predictable
contract allows independent evolution of clients and service.

### V. Test-Driven Development (NON-NEGOTIABLE)

- **TDD**: All production code MUST be written test-first: write a failing test (red), write the
  minimum code to pass (green), then refactor. Production code without a preceding failing test
  MUST be rejected in review.
- **Unit tests**: JUnit 5 + Mockito, with NO Spring context. Domain and Application tests MUST run
  instantly and deterministically.
- **Use-case coverage**: Every Application use case MUST have dedicated unit tests covering the
  happy path, boundary conditions, and failure scenarios.
- **Slice tests**: Controllers MUST be covered with `@WebMvcTest`; repositories with
  `@DataJpaTest`.
- **Integration tests**: `@SpringBootTest` with the `test` profile against embedded PostgreSQL, so
  tests exercise the production SQL dialect and the real Flyway migrations. H2 MUST NOT be used.
- **N+1 detection**: Repository tests MUST assert query counts using a query-counting interceptor.
- **Coverage gate**: JaCoCo MUST enforce at least 80% line and 80% branch coverage for the Domain
  and Application layers; the build MUST fail below the threshold. Coverage MUST NOT be gamed with
  assertion-free tests.
- **Isolation**: Tests MUST be readable, isolated, deterministic, and self-contained, with no
  shared mutable state between test cases.

**Rationale**: Writing the test first proves each behaviour is specified and verifiable; fast,
trustworthy tests are what make continuous refactoring — and therefore Clean Code — affordable.

### VI. Secure by Default

- **Authentication**: All endpoints MUST be protected by Spring Security configured as a stateless
  OAuth2 Resource Server. JWTs MUST be cryptographically verified on every request; claims MUST
  NOT be trusted before verification. JWT bearer tokens are the only accepted authentication
  mechanism.
- **Authorization (RBAC)**: Access control is role-based. Roles MUST be derived from a verified JWT
  claim through a single, centrally configured authorities converter. Every endpoint MUST declare
  its required role(s) explicitly; anything not explicitly granted is denied.
- **CSRF**: CSRF protection MAY be disabled only because the API is stateless and JWT-based, and
  the reason MUST be documented where it is disabled.
- **Security headers**: HSTS, `X-Content-Type-Options`, and `X-Frame-Options` MUST be configured
  through Spring Security.
- **Sensitive data**: Secrets, tokens, passwords, and PII MUST NEVER appear in responses, error
  messages, logs, or version control.
- **Passwords**: Any password the service stores MUST be hashed with BCrypt using an
  environment-appropriate cost factor.

**Rationale**: A zero-trust, least-privilege posture limits the blast radius of any single defect.

### VII. Built-In Observability

- **Actuator**: `/actuator/health`, `/actuator/info`, and `/actuator/prometheus` MUST be exposed.
  Liveness and readiness probes MAY be unauthenticated but MUST NOT reveal details; all other
  management endpoints MUST be restricted.
- **Metrics**: Meaningful custom metrics (use-case execution counts, latency histograms, error
  counters) MUST be published through Micrometer to the Prometheus registry and visualised in
  Grafana.
- **Tracing**: W3C Trace Context MUST be propagated on all inbound and outbound requests via
  Micrometer Tracing, exported over OTLP to the Grafana tracing backend (Tempo).
- **Structured logs**: All logs MUST be structured JSON written to stdout for collection into the
  Grafana logging backend (Loki).
- **Correlation**: Every log line MUST carry correlation identifiers (`traceId`, `spanId`, and a
  request ID) so logs, traces, and metrics can be linked in Grafana.

**Rationale**: In a distributed system, a failure that cannot be observed cannot be diagnosed.

### VIII. Resilience & Scalability

- **Transient failures**: Every operation MUST tolerate transient failures of downstream
  dependencies.
- **Timeouts**: Every outbound HTTP call and database query MUST have an explicit timeout; unbounded
  waits are forbidden.
- **Circuit breakers**: Calls to external services MUST be guarded by circuit breakers
  (Resilience4j).
- **Safe retries**: Write operations MUST be idempotent so retries cannot duplicate side effects.
- **Thread safety**: No shared mutable state; every bean MUST be thread-safe or explicitly scoped.
- **Bounded pools**: Thread pools and connection pools MUST be sized explicitly; unbounded defaults
  are forbidden in production.
- **Graceful shutdown**: The service MUST shut down gracefully, completing in-flight requests
  before terminating.

**Rationale**: Downstream failure is normal operation in a microservice landscape; the service
MUST degrade gracefully rather than cascade.

## Technology Stack & Engineering Standards

### Technology Stack

- **Language**: Java 25 (LTS).
- **Framework**: Spring Boot 4.1.1 (stable) with Spring Framework 7.0.x managed transitively.
- **Build**: Maven via the Maven wrapper.
- **Persistence**: Spring Data JPA; PostgreSQL in production; embedded PostgreSQL for development
  and integration testing; Flyway for all schema migrations.
- **Security**: Spring Security, stateless OAuth2 Resource Server (JWT).
- **Operations**: Spring Boot Actuator; Micrometer with the Prometheus registry; Micrometer Tracing
  with the OpenTelemetry bridge and OTLP exporter.
- **API documentation**: OpenAPI 3.1 via Springdoc.
- **Resilience**: Resilience4j.
- **Quality tooling**: JaCoCo, Spotless, Checkstyle, ArchUnit.
- **New libraries**: A dependency outside this list MUST have a clear, documented justification in
  the feature plan before it is added. Reflection-based mappers (e.g., MapStruct, ModelMapper) and
  Lombok are forbidden.

### Deployment

- **Target**: Kubernetes on AWS.
- **Packaging**: The service ships as an OCI container image running as a non-root user.
- **Probes**: Kubernetes liveness and readiness probes MUST use the Actuator health groups
  (`/actuator/health/liveness`, `/actuator/health/readiness`).
- **Resources**: Every deployment MUST declare CPU and memory requests and limits; JVM memory MUST
  be sized relative to the container limit.
- **Configuration & secrets**: Non-secret configuration is supplied through environment variables
  (e.g., ConfigMaps); secrets come from AWS Secrets Manager surfaced as Kubernetes Secrets.

### Automated Quality Tooling

All checks run in `./mvnw verify` and MUST pass before merge.

- **Spotless** (Palantir Java Format): Enforces formatting and removes unused imports.
- **Checkstyle**: Enforces no wildcard imports, no unused imports, magic-number detection, nesting
  depth ≤ 2, and flags methods over 20 lines.
- **ArchUnit**: Enforces the Dependency Rule, the absence of framework annotations in Domain, the
  absence of field injection, and the one-public-method rule for use cases.
- **JaCoCo**: Enforces the coverage gate defined in Principle V.
- **EditorConfig**: A committed `.editorconfig` keeps whitespace and encoding consistent across
  editors.

### Dependency Injection, DTOs & Mapping

- Constructor injection only. Field injection (`@Autowired` on fields) is forbidden.
- Application and Domain code MUST depend on interfaces (ports), never concrete implementations.
- Request DTOs, response DTOs, and value objects MUST be Java records.
- JPA entities MUST NEVER be exposed in API responses.
- Mapping between layers MUST use explicit mapper classes or static factory methods at the
  boundary.

### Validation & Exception Handling

- Inbound payloads MUST be validated with Jakarta Bean Validation at the Presentation boundary.
- Domain invariants MUST be enforced inside Domain entities and value objects, throwing
  domain-specific exceptions.
- A validation rule MUST live in exactly one layer; rules MUST NOT be duplicated across layers.
- A single global `@RestControllerAdvice` MUST translate all exceptions into RFC 9457 Problem
  Details.
- Domain exceptions are defined in the Domain layer. Infrastructure exceptions (e.g.,
  `DataAccessException`) MUST be translated in Infrastructure and MUST NOT leak into the
  Application or Presentation layers.

### Persistence

- Spring Data JPA is used only in the Infrastructure layer.
- All schema changes MUST be Flyway versioned migrations. An applied migration MUST NEVER be
  modified; corrections are made with a new migration.
- N+1 queries MUST be prevented with fetch joins or entity graphs and verified by query-count
  assertions.
- Associations are lazy by default; eager loading requires profiling evidence.
- Unbounded collection queries MUST be paginated.
- In-process caching MAY be introduced only with a measured, documented benefit and a documented
  invalidation strategy; an external cache requires a constitution amendment (Principle III).

### Configuration

- Profiles: `dev` (embedded PostgreSQL, relaxed security for local use only), `prod` (PostgreSQL,
  full security), and `test` (automated tests only). Relaxed security MUST NEVER be active in any
  deployed environment.
- `application.yaml` holds defaults only; overrides live in `application-{profile}.yaml`.
- Secrets and credentials MUST NEVER be committed in any form.

### Logging

- SLF4J is the only logging API. `System.out` / `System.err` are forbidden.
- JSON output MUST use Spring Boot's built-in structured logging; an additional encoder requires
  justification per the New libraries rule.
- Every log statement at INFO or above MUST include `traceId`, `spanId`, and the relevant business
  entity ID.
- Levels: DEBUG for diagnostics, INFO for significant business events, WARN for recoverable
  anomalies, ERROR for failures requiring attention.
- Passwords, tokens, secrets, and PII MUST NEVER be logged.

### Coding Standards

- No wildcard imports. No Lombok.
- Prefer composition over inheritance.
- Search the codebase and reuse existing components before creating new ones; never commit
  duplicate code.
- Use sealed interfaces and pattern matching where they improve clarity and exhaustiveness.
- Write self-explanatory code; comment only intent that the code cannot express.

## Development Workflow & Quality Gates

- **Feature lifecycle**: Every feature MUST follow the Spec Kit lifecycle:
  `/speckit.specify` → `/speckit.clarify` → `/speckit.plan` → `/speckit.tasks` →
  `/speckit.implement`. Specs and plans MUST NOT be hand-written outside this workflow.
- **Branches & review**: Each feature is developed on its own Spec Kit feature branch and merged
  through a reviewed pull request.
- **Constitution Check**: Every `plan.md` MUST pass the Constitution Check before research begins
  and again after design. Any violation MUST be recorded with justification in the plan's
  Complexity Tracking table.
- **Task ordering**: Every `tasks.md` MUST place each test task before the implementation task it
  drives (Principle V).
- **Build gate**: `./mvnw clean verify` MUST pass — compilation, all tests, coverage, formatting,
  static analysis, and architecture rules — before merge.
- **Contract gate**: Any API change MUST be reflected in the generated OpenAPI 3.1 specification
  and MUST NOT break an existing API version.
- **Migration gate**: Reviews MUST confirm no applied Flyway migration was modified.

## Governance

- **Supremacy**: This constitution supersedes all other practices, conventions, and preferences for
  `common-lib`. Runtime guidance files (`CLAUDE.md`, `AGENTS.md`) MUST NOT contradict it.
- **Amendments**: Changes are made only via `/speckit.constitution`, with documented justification,
  team approval, an updated Sync Impact Report, and propagation to dependent templates.
- **Versioning**: Semantic versioning.
  - **MAJOR**: Backward-incompatible removal or redefinition of a principle or governance rule.
  - **MINOR**: New principle or section, or materially expanded guidance.
  - **PATCH**: Clarifications, wording, and typo fixes.
- **Compliance**: Every pull request review and every feature plan MUST verify compliance.
  Unjustified complexity or deviation is grounds for rejection.

**Version**: 2.0.0 | **Ratified**: 2026-10-19 | **Last Amended**: 2026-10-19
