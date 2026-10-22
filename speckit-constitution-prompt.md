I am building a Java Spring Boot microservice.

Help me create a project constitution that defines the permanent engineering standards for this project. These rules must apply to every feature that will ever be developed.

The constitution should focus only on long-term project principles. Do not include feature-specific requirements, business rules, or implementation details for any individual feature.

Use the following project standards when creating the constitution.

## Technology Stack

- Java 25 (latest LTS, released September 2025)
- Java virtual threads for concurrency (standard since Java 21 and available on Java 25)
- Spring Boot 4.1.1 (latest stable, requires Java 17+, compatible with Java 25)
- Spring Framework 7.0.9 (latest stable, managed transitively via Spring Boot)
- Maven
- JaCoCo Maven plugin for test coverage measurement and build enforcement
- Spring Data JPA
- Spring Data Redis (with Spring Cache abstraction) as the distributed caching layer
- PostgreSQL for production
- Embedded PostgreSQL for development and integration testing
- Flyway for database migrations
- Spring Security with stateless OAuth2 Resource Server (JWT)
- Spring Boot Actuator for health, metrics, and observability endpoints
- Micrometer with Prometheus registry for metrics
- OpenAPI 3.1 (Springdoc) for API documentation
- Spring Cloud OpenFeign for declarative clients used to call external APIs
- Spring HATEOAS for hypermedia-driven RESTful APIs and HAL link model representation

## Architecture

Apply Clean Architecture principles throughout the project:

- Organise source code into four explicit layers: Domain, Application, Infrastructure, and Presentation (API).
- The Dependency Rule is absolute: outer layers may depend on inner layers, but inner layers must never depend on outer layers.
- The Domain layer contains entities, value objects, aggregates, and domain services. It has zero dependencies on frameworks or infrastructure.
- The Application layer contains use-case interactors and application services. It depends only on the Domain layer and defines ports (interfaces) for all external dependencies.
- The Infrastructure layer implements ports defined by the Application layer: JPA repositories, messaging adapters, external HTTP clients, and persistence configuration.
- Use Spring Cloud OpenFeign clients for all synchronous outbound calls to external APIs. Keep Feign client interfaces and their configuration in the Infrastructure layer, and expose them to the Application layer through its ports.
- The Presentation (API) layer contains Spring MVC REST controllers, hypermedia representation models/assemblers, and request/response DTOs. It depends only on Application layer ports.
- Business logic belongs exclusively in the Domain and Application layers.
- Controllers must only translate HTTP input into use-case calls and serialise the result to a response — no business logic in controllers.
- Hypermedia assembly belongs strictly in the Presentation (API) layer: Domain and Application layers must remain completely decoupled from hypermedia and framework concerns; Domain entities and Application use-case DTOs must never import or reference Spring HATEOAS types (such as `RepresentationModel`, `EntityModel`, `CollectionModel`, `PagedModel`, `Link`, or `WebMvcLinkBuilder`).
- No framework annotations (e.g., `@Entity`, `@Service`, `@Repository`) may appear in the Domain layer.
- Every use case is represented by a dedicated class with a single public method, keeping application behaviour explicit and traceable.

## Microservice Design Principles

- The service must be stateless: no server-side session state. All state lives in the database or the client token.
- Design all mutating endpoints to be idempotent where technically possible.
- Expose a single bounded context per microservice; do not grow a service beyond one domain.
- Communicate synchronously over REST/HTTP for query operations; prefer asynchronous messaging (e.g., Spring Events, or a broker) for cross-service side effects.
- Follow the 12-Factor App methodology: externalise configuration, treat backing services as attached resources, and keep dev/prod parity.
- Never hard-code environment-specific values; inject all configuration via environment variables or a secrets manager.

## API Design

- Design all APIs following REST principles and standard HTTP semantics (GET, POST, PUT, PATCH, DELETE).
- Version all APIs in the URL path (e.g., `/api/v1/`). Never introduce breaking changes to an existing version.
- Implement HATEOAS (Hypermedia as the Engine of Application State) across all REST endpoints to achieve Richardson Maturity Model Level 3.
- Use standard hypermedia representations adhering to HAL (`application/hal+json`) or standard link relation conventions, exposing an explicit `_links` object containing standard IANA relations and registered custom relations.
- Every single resource representation must include a canonical `self` link pointing directly to its own resource URI.
- Provide relational links to associated resources and child collections (e.g., parent, sub-resources, related aggregates) so clients navigate domain relationships through discovered links rather than hard-coded client-side URI templates.
- Support dynamic state-transition affordances: conditionally expose action links based on the current lifecycle state of the resource and the caller's authorized permissions (e.g., an order in `PENDING` state includes `cancel` and `pay` action links, which are dynamically omitted when the order is in `COMPLETED` or `CANCELLED` state, or when the caller lacks permission). Clients discover permitted operations through hypermedia affordances rather than duplicating business workflow rules on the client.
- Paginate collections using hypermedia-driven pagination models (`PagedModel` / `CollectionModel`): include standard navigation link relations (`self`, `first`, `prev`, `next`, `last`) alongside collection page metadata (`size`, `totalElements`, `totalPages`, `number`).
- Generate all hypermedia links dynamically using Spring HATEOAS link builders (e.g., `WebMvcLinkBuilder.linkTo(...)` and `methodOn(...)`) tied directly to controller methods. Never construct URLs using manual string concatenation or hard-coded URI paths.
- Ensure link generation is fully aware of reverse proxies, ingress controllers, and API gateways: enable and configure Spring's `ForwardedHeaderFilter` (or server-level forwarded header handling) so links preserve external protocol, host, port, and path prefix (`X-Forwarded-*` headers) when deployed behind an edge gateway.
- Return RFC 9457 Problem Details (`application/problem+json`) for all error responses — never expose stack traces or internal exception messages.
- Use consistent response envelopes and HTTP status codes throughout the service.
- Document every endpoint with OpenAPI 3.1 annotations, including hypermedia link relations, media types (`application/hal+json`, `application/problem+json`), and representation schemas; the spec must always be up to date and generated from code.
- Paginate all collection endpoints using cursor-based or offset pagination with explicit page-size limits.
- Every API endpoint must support localization through the standard `Accept-Language` request header. Resolve the locale once per request (e.g., Spring `LocaleResolver`/`AcceptHeaderLocaleResolver`) against an explicit list of supported locales. The default and fallback language is English (`en`): use it when the caller omits `Accept-Language`, sends an unsupported or malformed value, or when a message key is missing from the requested language's resource bundle (resolve per key, so the English text is returned for that key only). Configure this explicitly (e.g., `AcceptHeaderLocaleResolver.setDefaultLocale(Locale.ENGLISH)`, and `spring.messages.fallback-to-system-locale=false` so the server's OS locale is never used). The English bundle is the source of truth and must contain every message key.
- Localize all user-facing response messages, validation errors, and exception/RFC 9457 Problem Details fields (`title`, `detail`, and field-level errors) from resource bundles using the resolved locale; never hard-code message text.
- Return the applied locale in the `Content-Language` response header, and add `Vary: Accept-Language` where responses differ by locale.
- Document the `Accept-Language` header and supported locales on every endpoint in the OpenAPI specification.
- Handle missing, malformed, or unsupported `Accept-Language` values gracefully by falling back to the default locale rather than failing the request.
- Machine-readable identifiers (error codes, field names, enum values) must remain locale-independent and stable across languages.
- Test localization for each supported locale and the English fallback (no header, unsupported locale, and missing key in a non-English bundle), including validation and exception responses.

## Clean Code Standards

Apply the following Clean Code principles to every class and method:

- **Meaningful names**: class names, method names, and variable names must clearly express intent without requiring a comment to explain them.
- **Small methods**: each method should do one thing and do it well. Methods longer than twenty lines are a signal to decompose.
- **Single Responsibility**: every class must have exactly one reason to change.
- **No magic numbers or strings**: extract constants or enums with descriptive names.
- **No dead code**: remove commented-out code, unused imports, and unreachable branches immediately.
- **Avoid deep nesting**: use early returns, guard clauses, and extraction to limit nesting to at most two levels.
- **DRY (Don't Repeat Yourself)**: duplicate logic must be extracted into a shared abstraction before a second use is committed.
- **Comments explain why, not what**: the code itself must express what it does; comments are reserved for non-obvious business rationale.

## Dependency Injection

- Use constructor injection exclusively.
- Never use field injection (`@Autowired` on fields).
- Declare dependencies through interfaces (ports), never concrete implementations, in the Application and Domain layers.

## Data Transfer Objects

- Use Java records for all request DTOs, response DTOs, and value objects.
- Do not expose JPA entities directly in API responses under any circumstances.
- Map between layers at the boundary using explicit mapper classes or static factory methods. Do not use reflection-based mapping frameworks.
- In the Presentation layer, wrap response record DTOs into Spring HATEOAS representation models (`EntityModel<T>`, `CollectionModel<T>`, `PagedModel<T>`, or custom types extending `RepresentationModel<T>`) using dedicated `RepresentationModelAssembler` components to attach canonical, relational, and conditional transition links cleanly and consistently.

## Validation

- Validate all inbound request payloads using Jakarta Bean Validation at the Presentation layer boundary.
- Define user-facing Jakarta Bean Validation messages as keys in `ValidationMessages.properties` and locale-specific resource bundles (or an explicitly configured equivalent); constraints must reference message keys rather than hard-coded message text.
- Resolve validation messages using the request locale, falling back to English when the header is absent or a key is missing in the requested language, and verify that all referenced message keys exist.
- Apply domain invariant validation inside Domain entities and value objects; throw domain-specific exceptions on invariant violations.
- Validation rules must be enforced consistently and never duplicated across layers.

## Exception Handling

- Use a single global `@RestControllerAdvice` to translate all exceptions into RFC 9457 Problem Details responses.
- Define domain-specific exceptions in the Domain layer; represent expected errors with stable error codes/message keys and interpolation arguments, not hard-coded or localized user-facing text.
- Resolve user-facing exception messages from localized resource bundles through the presentation/error-handling boundary, using the request locale and falling back to English when the header is absent or a key is missing in the requested language. Do not put translated messages or framework-specific message-resolution dependencies in Domain exceptions.
- Use localized Bean Validation messages for validation failures; do not replace them with hard-coded text in exception handlers or error responses.
- Never expose raw unexpected exception messages; use a safe localized generic message for unexpected failures.
- Never let infrastructure exceptions (e.g., `DataAccessException`) leak into the Application or Presentation layers.
- Never return a stack trace, internal class name, or database error detail in an API response.

## Persistence

- Use Spring Data JPA for data access within the Infrastructure layer only.
- Manage all schema changes with Flyway versioned migrations. Never modify an already-applied migration script.
- Prevent N+1 query problems using fetch joins or entity graphs; verify query counts in integration tests.
- Use lazy loading by default; switch to eager loading only when profiling justifies it.
- Use pagination for all unbounded collection queries.

## Caching

- Introduce caching only when there is a measurable, documented performance benefit; document the invalidation strategy.
- Use Spring Data Redis, through the Spring Cache abstraction (`@Cacheable`, `@CacheEvict`, `CacheManager`), as the only caching mechanism so all service instances share one consistent cache in the distributed architecture.
- Never cache inside the container or JVM process: do not use in-memory caches (e.g., Caffeine, Ehcache, `ConcurrentMapCacheManager`, static maps, or local Hibernate second-level/query caches) for application data. Containers are ephemeral and horizontally scaled, so local caches become stale and inconsistent across instances.
- Keep caching annotations and Redis configuration in the Infrastructure layer (or Application-layer ports implemented there); the Domain layer must remain free of caching concerns.
- Set an explicit TTL on every cache entry or cache region; never rely on unbounded or non-expiring entries. Namespace keys by service and cache name (and version where the cached shape may change).
- Serialize cached values with a safe, explicit format (e.g., JSON); never use unrestricted Java deserialization.
- Treat the cache as an optimization, not a source of truth: the service must remain correct if Redis is unavailable, degrading gracefully to the database with timeouts and a circuit breaker rather than failing requests.
- Apply explicit connection and command timeouts and size the Redis connection pool; secure Redis with authentication and TLS, and supply credentials only through environment variables or a secrets manager.
- Never cache sensitive data (credentials, tokens, PII) unless justified, encrypted, and approved; never cache per-user data under shared keys.
- Expose cache hit/miss and latency metrics via Micrometer, and test cache behavior (hits, eviction, TTL expiry, and Redis failure) in integration tests against a real Redis instance (e.g., Testcontainers).

## Security

- Align security principles and implementation requirements with the current published OWASP Top 10 for Web Application Security and OWASP API Security Top 10. Treat both as minimum baselines, review them when OWASP publishes updates, and translate relevant risks into enforceable project-wide controls rather than merely naming the lists.
- Protect all endpoints using Spring Security configured as a stateless OAuth2 Resource Server.
- Validate JWT tokens on every request; never trust claims without cryptographic verification.
- Follow deny-by-default access control: explicitly grant permissions per endpoint or role.
- Enforce authorization at both endpoint and resource/object level; ensure users can access only the records and properties they are permitted to access, and prevent unauthorized property changes or data exposure.
- Prevent injection vulnerabilities by validating input at trust boundaries and using parameterized queries and safe framework APIs; never construct executable queries or commands from untrusted input.
- Minimize sensitive data collection and exposure; use approved cryptography and secure secret management, and do not expose sensitive fields through API responses or mass assignment.
- Protect APIs against unrestricted resource consumption with bounded payloads, pagination limits, timeouts, and appropriate rate or workload limits.
- Validate and constrain outbound requests and data received from external services to mitigate SSRF and unsafe consumption of third-party APIs.
- Keep API documentation and deployed endpoints in sync; remove or secure obsolete, undocumented, and non-production endpoints.
- Never disable CSRF protection without a documented reason (it is not needed for stateless JWT but must be a conscious decision).
- Configure HTTP security headers (HSTS, X-Content-Type-Options, X-Frame-Options) via Spring Security.
- Never expose sensitive information in responses, error messages, or logs.
- Store passwords using BCrypt with a cost factor appropriate for the target environment.
- Include security-focused tests for authentication, authorization, input handling, data exposure, and resource limits in the relevant unit and integration test suites.

## Resilience and Scalability

- Design every operation to tolerate transient failures from downstream dependencies.
- Apply timeouts to all outbound HTTP calls and database queries; never allow unbounded waits.
- Configure explicit connection and read timeouts for every Feign client; do not make direct outbound HTTP calls outside the Feign client adapters.
- Use circuit breakers (e.g., Resilience4j) for calls to external services.
- Make write operations idempotent so that safe retries do not produce duplicate side effects.
- Prefer Java virtual threads for high-concurrency blocking I/O workloads; enable Spring Boot virtual-thread support with `spring.threads.virtual.enabled=true` and verify that application dependencies are compatible.
- Do not pool virtual threads. Bound actual resource use independently with explicit database and HTTP connection-pool limits, semaphores or bulkheads, and rate limits; virtual threads do not increase downstream capacity.
- Avoid shared mutable state; all beans must be thread-safe or explicitly scoped.
- Size platform-thread executors and connection pools explicitly; never rely on unbounded defaults in production. Use virtual threads for task concurrency rather than increasing platform-thread pool sizes.

## Observability

- Expose `/actuator/health`, `/actuator/info`, and `/actuator/prometheus` endpoints. Secure management endpoints appropriately.
- Emit structured JSON logs (not plain text) to stdout so that log aggregators can index them without parsing.
- Use Micrometer Tracing with W3C Trace Context (`traceparent` and `tracestate`) to propagate trace context across inbound and outbound service calls and asynchronous messaging; preserve context across executor and virtual-thread task boundaries.
- Define and publish meaningful custom metrics (e.g., use-case execution counts, latency histograms) via Micrometer.
- Carry a consistent correlation ID and trace/span IDs across service-to-service calls and internal communication, including events, message brokers, scheduled/background work, and executor tasks. Accept or create a safe request correlation ID at the API boundary and propagate it using the established headers or message metadata.
- Instrument PostgreSQL/JPA and Redis operations with Micrometer-compatible tracing so each database/cache operation is associated with the active trace and can be correlated with the originating request across services. Do not expose correlation data by concatenating it into SQL or Redis commands.
- Include correlation ID, `traceId`, and `spanId` in structured logs for every request and associated internal operation, including database and Redis interactions; ensure the context is restored or cleared when work completes so it cannot leak between requests.

## Configuration

- Externalise all environment-specific configuration via environment variables or a secrets manager; nothing environment-specific lives in committed code.
- Maintain distinct Spring profiles: `local` (PostgreSQL, relaxed developer security), `dev` (PostgreSQL, relaxed non-production security), `stage` (PostgreSQL, production-like security and configuration for staging/pre-deployment validation), and `prod` (PostgreSQL, full production security).
- Reserve the `test` profile for isolated automated tests and release-gating/pre-deployment validation. Use embedded or ephemeral PostgreSQL for integration tests so database behavior matches production; never use production data or credentials in tests.
- Relaxed security settings are permitted only in `local` and `dev`; never carry them into `stage`, `test` release validation, or `prod`.
- Use `application.yml`) for defaults only; profile-specific overrides use `application-{profile}.yml`.
- Secrets and credentials must never be committed to version control in any form.

## Logging

- Use SLF4J with a structured JSON appender (e.g., Logback with `logstash-logback-encoder`) for all logging.
- Never use `System.out.println()` or `System.err.println()`.
- Never log passwords, tokens, secrets, PII, or any other sensitive information.
- Every log statement at INFO level or above must include context fields: at minimum `traceId`, `spanId`, and the relevant business entity ID.
- Log at DEBUG for detailed diagnostic information; INFO for significant business events; WARN for recoverable anomalies; ERROR for failures requiring attention.

## Containerization

- Build and run the service as a standards-compliant OCI container image. Use current, actively supported, security-patched versions of the JDK, build tools, base images, and container tooling; review and update them regularly.
- Pin base images and build-tool versions to explicit versions, and use immutable digests for release builds where supported. Do not use floating `latest` tags in production; update pins deliberately as patched releases become available.
- Use a multi-stage build: compile and package with a Maven/JDK builder image, then copy only the application artifact and required runtime files into a minimal, trusted Java runtime image. Do not include Maven, a JDK, source code, test reports, or build caches in the runtime image.
- Use Spring Boot's layered executable JAR support or equivalent OCI layer ordering. Place stable dependencies and runtime layers before frequently changing application classes so code changes reuse dependency layers; avoid both a single unnecessarily large layer and excessive layers that add no caching or maintainability benefit.
- Maintain a `.dockerignore` that excludes version-control data, local configuration, credentials, build output not needed by the image, and other unnecessary files from the build context.
- Optimize image size and attack surface by using minimal trusted base images, installing only required runtime packages, removing package-manager caches in the same build layer, and avoiding unnecessary shells, utilities, and services.
- Run the application as a dedicated non-root user. Drop unnecessary Linux capabilities, do not run privileged, and use a read-only root filesystem when compatible with the application; explicitly configure writable locations when required.
- Never bake secrets, credentials, environment-specific configuration, or private keys into an image or build arguments. Supply runtime configuration and secrets through approved deployment mechanisms.
- Scan the built image and its operating-system and application dependencies for known vulnerabilities and produce an SBOM as part of CI. Define severity thresholds and remediation timelines; block releases for vulnerabilities that exceed policy, and rebuild/redeploy promptly when base-image or dependency security fixes are available.
- Validate each image build in CI: verify the image starts with production-equivalent configuration, passes health and readiness checks and smoke tests, and contains no unintended secrets or development/build artifacts. Test the published image, not only the application running outside a container.
- Keep `Containerfile`s, build configuration, and container validation steps reproducible and version-controlled; document any security-scanner exceptions with scope, rationale, owner, and expiry.

## Testing

- Write unit tests using JUnit 5 and Mockito. All unit tests must run without a Spring context.
- Write integration tests using `@SpringBootTest` with the `test` profile and embedded or ephemeral PostgreSQL; do not use H2 as a substitute for production database behavior.
- Use `@WebMvcTest` for controller slice tests; use `@DataJpaTest` for repository slice tests.
- Verify HATEOAS hypermedia links in Presentation slice tests (`@WebMvcTest`) and integration tests: assert the presence, correct URI paths, and valid link relations (`_links.self`, related links, pagination links, and state/authorization-dependent transition affordances).
- Every use-case class in the Application layer must have dedicated unit tests covering happy-path, boundary, and failure scenarios.
- Domain entities and value objects must be fully tested without any Spring context.
- Verify N+1 query counts in repository integration tests using a query-counting interceptor.
- Tests must be easy to read, isolated, deterministic, and self-contained — no shared mutable state between test cases.
- Configure the JaCoCo Maven plugin to instrument tests, generate coverage reports, and enforce coverage checks in the Maven build and CI.
- Require at least 80% aggregate line coverage and 70% aggregate branch coverage across production code, and at least 90% line coverage separately for the Domain and Application layers.
- Apply coverage gates to unit and integration tests; report and combine their coverage where the build runs them separately.
- Exclude only generated code and trivial non-behavioral boilerplate from coverage measurements; document exclusions and do not exclude handwritten business logic to satisfy thresholds.
- Treat coverage thresholds as minimum safeguards, not goals; never sacrifice test quality for coverage numbers or lower a threshold without a documented justification.

## Coding Standards

- Do not use wildcard imports.
- Do not use Lombok; rely on Java records, plain constructors, and IDE generation for boilerplate.
- Prefer composition over inheritance.
- Reuse existing components before creating new ones; search the codebase before adding a class.
- Never generate duplicate code.
- Do not introduce additional libraries unless there is a clear, justified, and documented need.
- Keep classes and methods focused on a single responsibility (Clean Code SRP).
- Write self-explanatory code. Add comments only when the intent cannot be understood from the code itself.
- Apply the Boy Scout Rule: leave every file slightly cleaner than you found it.
- Use Java sealed interfaces and pattern matching (available since Java 21) where they improve clarity and exhaustiveness checking.

## General Principles

Create a constitution that enforces consistency, maintainability, readability, testability, security, and long-term scalability for a production-grade microservice.

The constitution must embed Clean Code and Clean Architecture as first-class, non-negotiable constraints. It must also enforce microservice-specific principles: statelessness, observability, resilience, and explicit API contracts. Every future feature generated by GitHub Spec Kit should automatically follow these standards without exception.

## Optional Project Discovery

The project type and most engineering defaults are already defined above. Do not ask questions whose answers are already specified. Ask only about unresolved choices that would materially change a permanent project standard, and ask any needed questions together in one concise, structured list.

If clarification is useful, ask:

1. Which additional API locales should be supported besides the English default?
2. Does the service require multi-tenancy, and are there project-specific roles or authorization rules beyond the stated deny-by-default policy?
3. What deployment platform or organization-mandated security, CI, or code-style standards should be reflected?

If the user does not answer or declines, retain the stated defaults and proceed without blocking constitution generation. Incorporate explicit answers where applicable, but identify conflicts with the established security and architecture principles instead of silently creating contradictory requirements.
