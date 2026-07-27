---
name: "domain-service-coreframework-guidelines"
description: "Domain service implementation guidelines based on orbt-coreframework-lib-src modules. Defines what Domain services MUST and MUST NEVER implement given cross-cutting concerns handled by core-framework-common and core-framework-data"
version: 1
created: "2026-06-08"
updated: "2026-06-08"
---
## When to Use
Use when developing Domain services that handle core business logic, persistence, and tenant-specific data access. Apply when implementing multi-tenant database routing, JPA repositories, caching strategies, or transaction management in Domain layer. Essential for ensuring Domain services properly leverage CoreFramework modules without duplicating framework-provided functionality.

## Procedure
1. **Module Dependencies**: MUST include core-framework-data (for caching, Redis, multi-tenant routing). MUST include core-framework-common (for logging, MDC, filters, DTOs). NEVER add core-framework-web (BFF-only module). ALWAYS exclude logback-classic and junit-vintage-engine from ALL dependencies
2. **Inbound Header & Context Extraction**: Use TenantContextHolder.getEffectiveTenantIds() to access tenant context. Use TenantContextHolder.getUserId() for authenticated user. NEVER manually extract X-Tenant-Id, X-User-ID, X-Correlation-ID from HttpServletRequest (MdcFilter and TenantSecurityFilter populate). Call TenantContextHolder.clear() in finally block for manual context manipulation
3. **Multi-Tenant Database Routing**: Rely on AbstractRoutingDataSource (provided by core-framework-data) for automatic tenant-based DB routing. Include tenantId as @Column(nullable = false) on ALL tenant-specific JPA entities. NEVER implement custom DataSource routing. NEVER hardcode database connections or tenant-to-DB mappings. NEVER perform cross-tenant queries bypassing routing datasource
4. **Caching Strategy**: Classify caches: L1+L2 (reference/static data) or L2-only (transient/per-request data). Use @Cacheable(value=..., key=#composite.spel, unless=#result.success == false). Add @CacheEvict on ALL write/update/delete methods. Register cache names in application.yml under cache.l1.cache-names and/or cache.l2.cache-names. Use @CacheTtl for method-level TTL overrides on L2. Use CacheService programmatic API for pattern eviction, existence checks
5. **JPA & Entity Best Practices**: NEVER use @Data on @Entity classes (use @Getter/@Setter with manual equals()/hashCode() on @Id). ALWAYS use FetchType.LAZY on relationships with explicit fetch joins. Use @Transactional(readOnly = true) on read methods. Use @Transactional on write methods. NEVER expose entities in REST responses (DTOs only). NEVER cache Hibernate proxies (cache DTOs). NEVER return lazy-loaded collections as cached values
6. **Logging & MDC**: Use SLF4J logger (private static final Logger log or @Slf4j). Use parameterized logging. NEVER manually set MDC fields (correlationId, traceId, tenantId, userId, requestUri, httpMethod, clientIp) - MdcFilter auto-populates. Use LogMaskingUtil.maskEmail() for PII. Pass exception object as last parameter to log.error(). Use appropriate log levels (DEBUG for cache/DB details, INFO for business events, WARN for fallbacks, ERROR for failures)
7. **Security & Authentication**: Use TenantContextHolder.getEffectiveTenantIds() for authorized tenant set (TenantSecurityFilter computes). Use TenantContextHolder.getUserId() for audit logging. Scope all queries to current tenant via AbstractRoutingDataSource. NEVER implement custom JWT parsing or tenant authorization (TenantSecurityFilter handles). NEVER disable Spring Security in production
8. **Resilience - Outbound Calls**: If calling other domain services via Feign, annotate wrapper methods with @CircuitBreaker, @Retry, @TimeLimiter. Define fallback with same return type plus Throwable parameter. Configure resilience4j instances in application.yml. NEVER implement custom retry/circuit breaker logic. Fallback MUST return safe default (never throw)
9. **User Identity & Config Resolution**: Use TenantContextHolder.getUserId() for current user (populated by TenantSecurityFilter). Call orbt-user-api-src ONLY for explicit business use cases (not authentication/authorization). Call orbt-configuration-api-src for system config, feature flags, reference data (NEVER hardcode). NEVER implement custom UserProfileCacheService or TenantDbMappingCacheService (CoreFramework provides)
10. **Architecture & Code Structure**: Follow Controller → Service → Repository layered architecture. Keep controllers thin (routing, DTO validation only). Use @RestControllerAdvice for global exception handling. Use @Valid on request DTOs. Version APIs as /api/v1/. Use noun-based URIs. Use constructor injection with final fields. Enable virtual threads (spring.threads.virtual.enabled: true). Include tenantId on all tenant-specific entities
11. **Testing Standards**: Use @ExtendWith(MockitoExtension.class) for unit tests. Use @WebMvcTest for controller slice tests. Use @DataJpaTest for repository slice tests. Use @MockitoBean (Spring Boot 4.x). Use AssertJ assertions (assertThat). Use BDD-style mocking (given/then). Name tests: methodName_scenario_expectedBehavior with @DisplayName. Configure JaCoCo exclusions for DTOs, models, constants. Targets: 100% service, 90% controller, 80% repository
12. **Configuration Requirements**: Set spring.threads.virtual.enabled: true. Configure cache.l1/l2 sections with all cache names. Configure spring.data.redis.* with Azure Redis cluster, SSL, Lettuce pool. Configure resilience4j instances if calling downstream services. Set management.tracing.sampling.probability: 1.0. Configure orbt.user.profile.cache.ttl-seconds: 900

## Pitfalls
- NEVER extract X-Tenant-Id, X-User-ID, X-Correlation-ID from HttpServletRequest manually - use TenantContextHolder
- Do NOT implement custom JWT parsing or tenant authorization logic - TenantSecurityFilter handles
- Never implement custom DataSource routing or tenant-DB mapping lookup - AbstractRoutingDataSource handles
- Do NOT create JPA entities without tenantId column for tenant-specific data
- Never implement custom Redis client or connection factory - core-framework-data manages
- Do NOT cache JPA entities or Hibernate proxies - ALWAYS cache DTOs only
- Never use @Cacheable without composite SpEL key and unless condition
- Do NOT skip @CacheEvict on write/update/delete operations
- Never put transient/per-request data in L1 Caffeine (L1 is reference/static only)
- Do NOT manually set MDC fields already managed by MdcFilter
- Never use System.out.println, e.printStackTrace(), or Logback - use SLF4J + Log4j2
- Do NOT log raw PII, passwords, or JWT tokens - use LogMaskingUtil
- Never use @Data on @Entity classes - use @Getter/@Setter with manual equals()/hashCode()
- Do NOT use FetchType.EAGER on JPA relationships - ALWAYS use FetchType.LAZY
- Never skip @Transactional(readOnly = true) on read service methods
- Do NOT expose JPA entities in REST responses - use DTOs
- Never call BFF layer from Domain service - no upward dependencies
- Do NOT implement presentation/aggregation logic in Domain (BFF responsibility)
- Never implement custom AbstractRoutingDataSource - framework provides
- Do NOT use @SpringBootTest for unit tests - use @ExtendWith(MockitoExtension.class)
- Never use deprecated @MockBean - use @MockitoBean (Spring Boot 4.x)
- Do NOT test repositories with @SpringBootTest - use @DataJpaTest
- Never use field injection - use constructor injection with final fields
- Do NOT hardcode environment-specific values - use ${ENV_VAR:default} placeholders
- Never configure FetchType.EAGER in JPA entity mappings
- Do NOT call orbt-user-api-src for current user identity (use TenantContextHolder)
- Never implement custom user profile cache or tenant-DB mapping cache - CoreFramework provides
- Do NOT hardcode configuration values - call orbt-configuration-api-src

## Verification
1. Verify core-framework-data and core-framework-common dependencies are included in pom.xml
2. Check NO core-framework-web dependency in Domain service pom.xml
3. Confirm TenantContextHolder.getEffectiveTenantIds() used instead of manual header extraction
4. Validate AbstractRoutingDataSource handles multi-tenant routing (no custom implementation)
5. Check ALL tenant-specific JPA entities have tenantId column with @Column(nullable = false)
6. Verify @Cacheable annotations include composite key, unless condition, and @CacheEvict on writes
7. Confirm cache names registered in application.yml under cache.l1.cache-names and/or cache.l2.cache-names
8. Check NO @Data on @Entity classes (use @Getter/@Setter)
9. Validate ALL JPA relationships use FetchType.LAZY
10. Verify @Transactional(readOnly = true) on ALL read service methods
11. Check @Transactional on ALL write service methods
12. Confirm NO JPA entities exposed in REST responses (DTOs only)
13. Validate logs use SLF4J parameterized logging with NO manual MDC.put() for auto-populated fields
14. Check application.yml contains spring.threads.virtual.enabled: true
15. Verify NO @SpringBootTest in unit tests - use @ExtendWith(MockitoExtension.class)
16. Confirm @DataJpaTest used for repository slice tests
17. Check all tests use @MockitoBean (not @MockBean), AssertJ, BDD-style mocking
18. Validate controllers are thin with business logic in service layer
19. Verify constructor injection with final fields (no field injection)
20. Confirm NO hardcoded credentials, URLs, or config values (use environment variables or orbt-configuration-api-src)
21. Check JaCoCo configuration excludes DTOs, models, constants, exceptions, *Application.java