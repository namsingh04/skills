---
name: "adr-microservices-blueprint"
description: "Architecture Decision Record blueprint for microservices modernization with Java 21, Spring Boot 4.x, multi-tenant architecture, BFF pattern, and Azure cloud deployment"
version: 2
created: "2026-06-08"
updated: "2026-06-08"
---
## When to Use
Use when creating or documenting architecture decisions for microservices modernization projects, especially when migrating from legacy applications to cloud-native Java microservices with multi-tenancy requirements. Apply when setting up new services, defining API standards, implementing observability, or establishing distributed system patterns.

## Procedure
1. **Technology Stack Selection**: Java 21 LTS runtime, Spring Boot 4.0.6, Maven 3.9+, PostgreSQL 15+, Azure Managed Redis, React 19.x frontend, OpenTelemetry + Grafana Cloud for observability
2. **Project Structure**: Implement strict 3-tier architecture - Controller (routing, DTO validation), Service (business logic, transactions), Repository (data access with Spring Data JPA). Use standard directory layout with separate packages for config, dto/request, dto/response, exception, interceptor, mapper, model, service/impl, client, util
3. **API Design Standards**: Use POST for sensitive/complex data (create/update/delete/read), GET only for non-sensitive retrieval. Implement hybrid versioning: URI versioning (/api/v1/) for breaking changes, Content-Negotiation (Accept header) for minor versions. Standardize request headers (X-Correlation-ID, userId, Content-Type) and response format with status, statusCode, message, data, errors array
4. **Multi-Tenant Architecture**: Implement application-level routing using X-Tenant-Ids header with AbstractRoutingDataSource pattern. Store tenant mapping in dedicated database table with Azure Redis cache (TTL: 24h). Use cache-aside pattern. Ensure all domain tables include tenant_id column for data isolation
5. **BFF Pattern Implementation**: Create Backend-for-Frontend layer with Spring Boot + FeignClient for domain service aggregation. Implement Resilience4j patterns (circuit breaker, retry, timeout, bulkhead). Handle UI orchestration, data transformation, and service composition. Support parallel calls for multi-tenant requests
6. **Observability & Tracing**: Implement dual identifier strategy - correlationId (business-level, UI-generated) for end-to-end user journey tracking, and traceId (technical-level, OpenTelemetry-generated at BFF) for distributed tracing. Use SLF4J + Log4j2 with MDC context propagation, structured JSON logging via JsonTemplateLayout, async logging with LMAX Disruptor. Integrate with Grafana Cloud
7. **Caching Strategy**: Implement multi-layer caching - APIM level (selective static/reference data), BFF level (user session, aggregated responses with Azure Redis), Domain service level (business entities with Caffeine + Azure Redis), HTTP level (browser/CDN). Use cache-aside pattern with write-through at domain layer
8. **Testing Requirements**: Achieve 90% unit test coverage enforced by JaCoCo. Use JUnit 5 (Jupiter) with @ExtendWith(MockitoExtension.class) for unit tests (no Spring context), @SpringBootTest with WireMock for integration tests, @WebMvcTest/@DataJpaTest for layer testing. Enable parallel test execution. Strictly prohibit test skipping in CI
9. **Inter-Service Communication**: Use non-reactive stack (Spring MVC + Virtual Threads). Implement FeignClient for declarative REST calls. Apply Resilience4j patterns for fault tolerance. Use MapStruct for compile-time DTO mapping. Enable virtual threads in Spring Boot configuration
10. **Database Management**: Use Liquibase for schema migrations with version control. Each domain service manages its own schema with tenant_id in all tables. Store migration files in src/main/resources/db/changelog/. Implement automated rollback capabilities
11. **Security & Configuration**: Store all secrets in Azure Key Vault. Use constructor injection with final fields. Implement proper error handling with safe messages (never expose stack traces). Always propagate correlation IDs. Redact PII from logs
12. **CI/CD Quality Gates**: Enforce build failures for: Checkstyle violations, JUnit 5 test failures, JaCoCo coverage below 90%, OWASP Dependency Check findings with CVSS >= 7. No test skipping allowed
13. **Naming Conventions**: packages (lowercase reverse domain: com.sales.service), Classes/Records (PascalCase), methods/variables (camelCase), constants (UPPER_SNAKE_CASE), REST APIs (kebab-case with versioning: /api/v1/customer-orders)

## Pitfalls
- Do NOT put business logic in controllers - controllers should only handle routing and DTO validation
- Avoid field injection - use constructor injection with final fields for immutability and testability
- Never hardcode secrets or credentials - always use Azure Key Vault or environment variables
- Do NOT expose stack traces in API responses - use safe error messages with trace_id for debugging
- Avoid mixing reactive and non-reactive patterns - stick to Spring MVC + Virtual Threads consistently
- Do NOT implement single-layer caching - use multi-layer strategy (APIM, BFF L2, Domain L1+L2, HTTP) for optimal performance
- Never skip correlation ID propagation - both correlationId (business) and traceId (technical) must flow through all services
- Avoid synchronous-only communication for >3 step transactions - use distributed Saga patterns with Spring State Machine
- Do NOT store tenant mapping in configuration files - use database as source of truth with Redis caching
- Never allow test skipping in CI/CD pipeline - tests must always run and pass
- Avoid using GET for sensitive data or complex queries - use POST to prevent URL length limits and security issues
- Do NOT create tables without tenant_id column in multi-tenant domain services
- Never load Spring context in unit tests - use Mockito extensions (@ExtendWith(MockitoExtension.class)) for faster execution
- Do NOT skip parent POM - all services MUST inherit from parent-project/pom.xml with Spring Boot 4.0.6
- Never use Logback - ALWAYS use SLF4J + Log4j2 with JsonTemplateLayout and exclude logback-classic from ALL dependencies
- Avoid creating common-framework without proper separation - NO datasource/Azure config in common-framework
- Do NOT manually extract X-Tenant-Id, X-User-ID, X-Correlation-ID from HttpServletRequest - use TenantContextHolder (auto-populated by filters)
- Never add Authorization, X-User-ID, X-Tenant-Ids headers to Feign method signatures - DomainFeignInterceptor auto-injects these
- Do NOT use FallbackFactory for Feign fallbacks - use class-based fallback implementing the Feign interface
- Never manually set MDC fields (correlationId, traceId, tenantId, userId) - MdcFilter auto-populates these
- Avoid caching JPA entities - ALWAYS cache DTOs only (entities have Hibernate proxies)
- Do NOT use @Cacheable without composite SpEL key and unless condition
- Never skip @CacheEvict on write/update/delete operations
- Do NOT classify caches incorrectly - reference/static → L1+L2, transient/per-request → L2 only
- Never put per-request/transient data in L1 (Caffeine) - L1 is for reference data ONLY
- Avoid using @SpringBootTest for unit tests - use @ExtendWith(MockitoExtension.class) for unit, @WebMvcTest for controller slice, @DataJpaTest for repository slice
- Do NOT use deprecated @MockBean - use @MockitoBean (Spring Boot 4.x)
- Never use JUnit 4 - ALWAYS use JUnit 5 (Jupiter) and exclude junit-vintage-engine
- Avoid using Mockito.when()/verify() - use BDD-style given()/then()
- Do NOT use JUnit assertions - use AssertJ (assertThat, assertThatThrownBy)
- Never use @Data on @Entity classes - use @Getter/@Setter and manual equals()/hashCode() on @Id field only
- Avoid FetchType.EAGER on JPA relationships - ALWAYS use FetchType.LAZY with explicit fetch joins
- Do NOT expose JPA entities in REST responses - ALWAYS use DTOs
- Never skip @Transactional(readOnly = true) on read service methods
- Do NOT use RestTemplate - ALWAYS use FeignClient for inter-service calls
- Never modify applied Liquibase changelogs - ALWAYS add new changeset
- Avoid hardcoding database credentials - ALWAYS use Azure Managed Identity
- Do NOT implement custom JWT parsing or tenant authorization - TenantSecurityFilter handles this
- Never re-implement user profile caching or tenant-DB mapping - CoreFramework provides UserProfileCacheService and TenantDbMappingCacheService
- Avoid using System.out.println or e.printStackTrace() - use SLF4J logger with parameterized logging
- Do NOT log raw PII, passwords, or JWT tokens - use LogMaskingUtil.maskEmail() and mask sensitive data
- Never use string concatenation in log statements - use parameterized logging (log.info("msg id={}", id))
- Avoid catching Exception broadly - catch specific exceptions and handle appropriately
- Do NOT return null from public API methods - use Optional or empty collections
- Never use RELEASE/+ dynamic versions in Maven - use dependencyManagement with explicit versions
- Avoid *.md in .gitignore - this excludes README.md and documentation
- Do NOT commit .slingshot/, .idea/, .vscode/, target/ to version control
- Never leave commented-out functional code or stale TODOs
- Avoid using fully-qualified class names inline - ALWAYS use imports
- Do NOT initialize collection fields as null - initialize as empty lists
- Never use synchronized with virtual-thread workloads unless justified - it pins carrier threads
- Avoid shared mutable static state - use per-call instances for SimpleDateFormat, DecimalFormat (or use DateTimeFormatter)
- Do NOT create multiple StringUtils classes - use one consistently
- Never put pagination validation in both controller and service - define in one layer only
- Avoid non-descriptive test names - follow methodName_scenario_expectedBehavior pattern
- Do NOT skip @DisplayName on tests
- Never commit unused imports
- Avoid files without trailing newline
## Verification
1. Verify project structure matches 3-tier architecture with standard package layout (controller, service, repository, dto, model, etc.)
2. Confirm API endpoints follow naming conventions (kebab-case, versioned /api/v1/) and return standardized response format with status, statusCode, message, data, errors
3. Validate multi-tenant routing works correctly with X-Tenant-Ids header and tenant mapping cached in Redis with 24h TTL
4. Check BFF layer uses FeignClient for domain service calls with Resilience4j patterns (circuit breaker, retry, timeout) configured
5. Verify logs contain both correlationId and traceId in structured JSON format, with PII redacted and async logging enabled
6. Confirm JaCoCo reports show >= 90% code coverage, JUnit 5 tests run in parallel, and CI pipeline fails when tests are skipped
7. Validate multi-layer caching is active at APIM, BFF (Azure Redis), Domain (Caffeine + Azure Redis), and HTTP levels
8. Check all secrets are retrieved from Azure Key Vault, not hardcoded or in configuration files
9. Verify Liquibase migrations execute successfully with proper schema isolation and tenant_id columns in all domain tables
10. Confirm OpenTelemetry integration sends traces to Grafana Cloud with proper context propagation across services
11. Validate virtual threads are enabled in Spring Boot configuration for improved I/O-bound performance
12. Check all REST endpoints use POST for sensitive/complex data and GET only for non-sensitive retrieval