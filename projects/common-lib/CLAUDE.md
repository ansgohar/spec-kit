# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This is a freshly-scaffolded Spring Boot service (`common-lib`) — currently just the Spring Initializr skeleton (`CommonLibApplication`, one context-load test, `application.yaml`). No domain code, controllers, or persistence exist yet. The repo is set up with **GitHub Spec Kit** for spec-driven development; new features are expected to be built through that workflow rather than ad hoc, and must conform to `.specify/memory/constitution.md`, which is the authoritative architecture/quality contract for this project.

## Commands

Use the Maven wrapper (no local Maven install required):

```bash
./mvnw clean install          # full build + tests
./mvnw test                   # run all tests
./mvnw test -Dtest=ClassName                       # single test class
./mvnw test -Dtest=ClassName#methodName            # single test method
./mvnw spring-boot:run         # run the application locally
```

Build artifacts land under `target/` (gitignored). Java version is pinned to **25** via `pom.xml`'s `java.version` property.

Note: the parent POM is `spring-boot-starter-parent:4.2.0-SNAPSHOT`, which requires the Spring snapshot repository declared in `pom.xml` (`https://repo.spring.io/snapshot`) — expect pre-release Spring Boot/Framework 7 APIs.

## Spec Kit workflow

This project is driven through Spec Kit slash-command skills (installed under `.agents/skills/speckit-*`). The required feature lifecycle, per the constitution's Governance section, is:

```
/speckit.specify → /speckit.clarify → /speckit.plan → /speckit.tasks → /speckit.implement
```

Supporting commands: `/speckit.constitution` (amend the constitution), `/speckit.analyze`, `/speckit.checklist`, `/speckit.taskstoissues`, and the git-extension commands under `.specify/extensions/git/commands/` (`speckit.git.commit`, `.feature`, `.initialize`, `.remote`, `.validate`). Feature specs/plans, once generated, live under a `specs/` directory (not yet created). Don't bypass this lifecycle by hand-writing a plan or spec file outside the skills.

## Architecture (mandatory, from `.specify/memory/constitution.md`)

All production code — even though none exists yet beyond the bootstrap class — MUST be organized into four strictly decoupled layers with inner layers never depending on outer ones:

- **Domain**: entities, value objects, domain services. Zero framework dependencies — no `@Entity`/`@Service`/`@Repository` annotations here. Invariant validation throws domain-specific exceptions.
- **Application**: use-case interactor classes (one public method each), depending only on Domain. Defines outbound ports (interfaces) for persistence/external APIs — infrastructure implements these, never the reverse.
- **Infrastructure**: implements Application-layer ports (Spring Data JPA repos, outbound REST clients).
- **Presentation (API)**: Spring MVC controllers + request/response DTOs only. Controllers translate HTTP ↔ use-case calls; zero business logic.

Key cross-cutting rules worth knowing before writing any code here:

- **DTOs/value objects are immutable Java records**; constructor injection only (no field `@Autowired`, no Lombok).
- **Mapping between layers** uses explicit mapper classes/static factories — never reflection-based mappers, and JPA entities must never leak into API responses.
- **API contracts**: REST under `/api/v1/...`, documented with OpenAPI 3.1 (Springdoc); errors returned as RFC 9457 Problem Details (`application/problem+json`) — never raw stack traces/SQL errors; all collection endpoints paginated; mutating endpoints designed idempotent.
- **Service shape**: pure REST API (no UI), single tenant, stateless; inter-service calls are REST only (no broker/gRPC without a constitution amendment).
- **Security**: stateless Spring Security OAuth2 Resource Server (JWT only), RBAC with roles from a verified JWT claim, required roles declared per endpoint, deny-by-default, no server-side sessions.
- **Persistence**: Spring Data JPA, PostgreSQL in prod / embedded PostgreSQL for dev & test (no H2, no other datastores), schema changes only via immutable Flyway migrations.
- **Observability**: SLF4J structured JSON logs only (no `System.out`/`System.err`), `traceId`/`spanId` correlation via Micrometer Tracing, metrics via Micrometer → Prometheus; Grafana stack (Prometheus metrics, Tempo traces over OTLP, Loki logs); actuator health/info/prometheus endpoints exposed.
- **Deployment**: Kubernetes on AWS — non-root container image, liveness/readiness via actuator health groups, explicit resource limits, graceful shutdown, secrets from AWS Secrets Manager.
- **Automated checks in `./mvnw verify`**: Spotless (Palantir Java Format), Checkstyle, ArchUnit (layer rules), JaCoCo (≥ 80% line/branch on Domain + Application).
- **Testing** (see below) and **Clean Code** limits: methods >20 lines need decomposition review, no wildcard imports, nesting ≤ 2 levels, no magic numbers.

Treat the constitution as the source of truth if anything here goes stale — re-read `.specify/memory/constitution.md` for exact wording before a Constitution Check / architecture-gate review.

## Testing conventions

- **TDD is mandatory**: write a failing test first, then the minimum code to pass, then refactor.
- **Unit tests** (Domain/Application layers): JUnit 5 + Mockito, no Spring context — must be instant and deterministic. Every use case needs happy-path, boundary, and domain-failure-case coverage.
- **Slice tests**: `@WebMvcTest` for controllers, `@DataJpaTest` for repositories (the latter must assert query counts to catch N+1 regressions).
- **Integration tests**: `@SpringBootTest` with the `test` profile against embedded PostgreSQL.
- Tests must be isolated/deterministic with no shared mutable state.

## Known project quirks

- The Maven `groupId`/`artifactId` combo `com.jdax.common.common-lib` is an invalid Java package name; the actual package is `com.jdax.common.common_lib` (underscore) — see `HELP.md`.

<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan
<!-- SPECKIT END -->
