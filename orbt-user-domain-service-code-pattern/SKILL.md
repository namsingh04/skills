---
name: orbt-user-domain-service-code-pattern
description: >-
  Generates ORBT User Domain Java services matching client-reviewed orbt-user-api-src:
  Central DB (shared.*), single DataSource with Azure MI, ApiResponseDTO envelope,
  domain.entity/repository layout, Lombok DTOs, static mappers, L1+L2 cache.
  Use when generating or fixing User Domain / Central-DB domain services from story packs
  (UserDomain, Sales-Rep Directory, RBAC read APIs), or when orbt-domain-service-code-pattern
  would wrongly apply tenant-DB routing or CQRS sales-api patterns.
---

# ORBT User Domain Service Code Pattern

Reference implementation: `orbt-user-api-src-develop/orbt-user-api-src` (`com.orbt.user`).

## When to use

| Use this skill | Use `orbt-domain-service-code-pattern` instead |
|---|---|
| User Domain, Central DB, `shared.*` tables | Tenant-scoped feature domains (`sales.*`, etc.) |
| Sales-Rep Directory, RBAC read, user-tenant mapping | CQRS Command/Query + multi-tenant routing DataSource |
| Story pack under `Story/Shared/UserDomain/` | Story pack under `Story/Schedule/` or other tenant DB features |

**Precedence:** ADR > Story pack > this skill. This skill is HOW only; story pack / build contract is WHAT.

## Procedure

### 1. Package layout (mandatory)

Root package: `com.orbt.user` (or `com.orbt.{user-capability}` from inventory — keep `user` for User Domain).

```
com.orbt.{user}/
├── OrbtUserApiApplication.java          # or {Capability}Application
├── config/
│   ├── CentralDatabaseConfig.java       # Central DB DataSource (required)
│   ├── JpaConfig.java                   # EMF + TransactionManager
│   ├── AuthSecurityConfig.java          # Stateless SecurityFilterChain
│   ├── OpenApiConfig.java               # Swagger — dev/qat profiles only
│   ├── RbacAsyncConfig.java             # @Async executor (if async audit/events)
│   └── ResilienceConfig.java            # ONLY if pack explicitly requires; prefer none
├── constant/                            # AppConstants — headers, API prefixes
├── controller/                          # Thin REST controllers per capability area
├── domain/
│   ├── entity/                          # JPA entities (NOT model/)
│   └── repository/
│       └── projection/                  # Interface projections for native queries
├── dto/
│   ├── constant/                        # CacheConstants (cache name strings)
│   ├── request/
│   └── response/
├── exception/                           # Domain exceptions + RbacGlobalExceptionHandler
├── mapper/                              # Static mapper utilities (entity → DTO)
├── service/
│   ├── impl/
│   └── model/                           # Internal service models (e.g. ResolvedPermissionSet)
└── util/                                # LogSanitizer, helpers
```

Every package gets `package-info.java`. Tests mirror under `src/test/java` with `TestDataBuilder`, `RbacTestSecurityConfig`.

### 2. Boot / Maven / framework

- **Parent:** `com.renuity:orbt-common-parent-lib` (GitHub Packages).
- **Deps:** `core-framework-common`, `core-framework-data` only. **NEVER** `core-framework-web`, `jjwt`, or BFF Resilience4j starters unless ADR overrides.
- **Starters:** `spring-boot-starter-web` (exclude logback), `spring-boot-starter-data-jpa`, `spring-boot-starter-security`, `postgresql` (runtime), `liquibase-core`, `springdoc-openapi-starter-webmvc-ui`.
- **Application class:**

```java
@SpringBootApplication
@EnableScheduling
@EnableCaching
@EnableAsync
@ComponentScan(basePackages = {
    "com.orbt.user",
    "com.renuity.core.common",
    "com.renuity.core.domain",
    "com.renuity.core.data"
})
public final class OrbtUserApiApplication { ... }
```

- **Context path:** `server.servlet.context-path: /api/user` (adjust slug per service).
- **Virtual threads:** `spring.threads.virtual.enabled: true`.
- **Logging:** Log4j2 via `logging.config: classpath:log4j2.xml` — never Logback.
- **Import:** `spring.config.import: optional:classpath:common-framework-default.yaml`.

### 3. Central DB — NOT tenant routing

User Domain connects to the **Central DB** (`shared.*`). Do **not** copy sales-api multi-tenant `RoutingDataSource`.

| Property | Value |
|---|---|
| `central.database.enabled` | `true` |
| `multitenant.database.enabled` | `false` |
| `multitenant.cache.enabled` | `false` |
| `spring.datasource.*` | Present for framework defaults; **primary** bean comes from `CentralDatabaseConfig` |

**CentralDatabaseConfig pattern:**
- `@ConditionalOnProperty(name = "central.database.enabled", havingValue = "true")`
- `@Primary @Bean(name = "dataSource")` — single HikariCP pool to `central.datasource.url`
- Azure Managed Identity: token as Hikari password + `TokenRefreshingDataSource` wrapper
- Local/H2: password auth when `azure.managed-identity.enabled=false`
- Pool name: `central-HikariPool`

**JpaConfig pattern:**
- `@EnableJpaRepositories(basePackages = "com.orbt.user", entityManagerFactoryRef = "entityManagerFactory", transactionManagerRef = "transactionManager")`
- `@Primary` `entityManagerFactory` + `transactionManager` wired to `@Qualifier("dataSource")`
- `packages("com.orbt.user")`, `persistenceUnit("auth")` (rename per service if needed)

### 4. Tenancy rules (critical)

| Table type | Rule |
|---|---|
| **Global** (`shared.emp_employees`) | **No `tenant_id` predicate.** Table is global. |
| **Tenant-scoped** (`shared.employee_tenant_map`) | Filter via `tenant_id` from `X-Tenant-Id` header / `TenantContextHolder` |
| **Request payload** | **Never** accept `tenantId` in body for directory/read APIs |

`default_flag` on `employee_tenant_map` = **last-login tracking**, not branch/market default — do not use for roster logic.

For **Sales-Rep Directory** (subset): read-only; no `core-framework-rbac` injection; implement pre-ratified wire contract exactly.

### 5. Controllers

- One controller per capability area: `UserProfileController`, `RbacReadController`, `{Feature}Controller`.
- `@RestController` + `@RequestMapping("/v1/...")` + `@Tag` + `@Slf4j` + `@RequiredArgsConstructor` (or explicit constructor).
- **Thin only:** validate input → delegate to service → wrap in `ApiResponseDTO`.
- **Envelope:** always `com.renuity.core.common.dto.response.ApiResponseDTO`:

```java
return ResponseEntity.ok(ApiResponseDTO.success(data, "message"));
```

- OpenAPI: `@Operation`, `@ApiResponses` on every endpoint.
- Tenant from header when needed — **never** in URL path for RBAC/directory APIs.
- **No CQRS Command/Query split required** unless the build contract explicitly demands it.

### 6. DTOs

- Lombok classes in `dto.request` / `dto.response`:
  - `@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder`
  - `@JsonInclude(NON_NULL)`, `@JsonProperty` on every serialized field
  - `@ToString(exclude = {...})` for PII fields
- Request DTOs: Jakarta `@Valid`, `@NotNull`, `@NotBlank`, `@NotEmpty` as per contract.
- **Do not** use Java records for wire DTOs in this service family (differs from sales-api skill).
- Cacheable DTOs must declare `value-type` in `application.yml` under `cache.l2.per-cache`.

### 7. Services

- Interface in `service/`, implementation in `service/impl/`.
- `@Service @Slf4j`, constructor injection, `Objects.requireNonNull` on deps.
- Reads: `@Transactional(readOnly = true)`; writes: `@Transactional`.
- **Caching:** `@Cacheable` / `@CacheEvict` with names from `CacheConstants`; always `unless = "#result == null"`.
- **Logging:** `MDC.put` for correlation keys; `LogSanitizer.sanitize()` on PII (email, names).
- **No business logic in controllers.** No Feign clients (BFF-only).
- **No Resilience4j** `@CircuitBreaker` / `@Retry` on service methods unless ADR explicitly requires.

### 8. Entities (`domain.entity`)

```java
@Entity
@Table(name = "emp_employees", schema = "shared", indexes = { ... })
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
@ToString(exclude = { "sensitiveField" })
public class Employee { ... }
```

- `@Column(name = "snake_case")` matching DB exactly (preserve legacy names like `brn_id`).
- `@SQLRestriction("is_deleted = false")` on soft-delete tables.
- **NEVER `@Data`** on entities. LAZY fetch on `@ManyToOne`.
- Global tables: no `tenant_id` column. Tenant map tables: `tenant_id` required.

### 9. Repositories

- `@Repository` extends `JpaRepository<Entity, Id>`.
- Prefer Spring Data method names for simple lookups.
- Complex SQL → `@Query(nativeQuery = true)` with named `@Param`.
- Interface projections under `domain.repository.projection` for multi-column native results.
- `@Modifying` + `@Transactional(propagation = MANDATORY)` for updates.
- Unknown ids in batch lookups: omit from result, do not error.

### 10. Mappers

- Static utility classes in `mapper/` (e.g. `EmployeeMapper`, `RbacMapper`).
- `private` constructor; `public static` mapping methods.
- MapStruct is **optional** — this reference uses hand-written static mappers. Follow reference unless pack mandates MapStruct.

### 11. Exceptions

- Typed exceptions extending `RbacException` (or feature base) with `HttpStatus` + `errorCode`.
- `RbacGlobalExceptionHandler`:
  - `@RestControllerAdvice(basePackages = "com.orbt.user")`
  - `@Order(Ordered.HIGHEST_PRECEDENCE)`
  - Returns `ApiResponseDTO` with incident UUID; **never** expose `e.getMessage()` or stack traces in HTTP body.
  - WARN for 4xx, ERROR for 5xx.
- Use framework `ResourceNotFoundException` where appropriate.
- Configure `common.error.*` keys in `application.yml` (service-specific prefix).

### 12. Security

`AuthSecurityConfig`:
- CSRF disabled, stateless sessions, `permitAll()` on API paths (gateway/BFF enforces auth).
- Permit: `/actuator/**`, `/swagger-ui/**`, `/v3/api-docs/**`, `/h2-console/**` (test).
- **Do not** add custom JWT parsing in Domain — authorization is caller's BFF concern for directory APIs.

### 13. Caching (`application.yml`)

```yaml
cache:
  key:
    namespace: domain
    tenant-scoped-by-default: false    # User Domain is NOT tenant-keyed by default
  allow-null-values: false
  l1:
    cache-names: [ role-name ]
  l2:
    cache-names: [ user-profile, user-permissions, branch-access, role-name ]
    per-cache:
      user-profile:
        ttl: 15m
        value-type: 'com.orbt.user.dto.response.UserProfileDTO'
```

- Register cache names in `CacheConstants` as `public static final String` mirrors.
- L1 for static reference data only; mutable per-user data → L2 Redis only.
- Every cache needs eviction path or documented TTL-only rationale.

### 14. Liquibase

- `db/changelog/db.changelog-master.xml` includes numbered `changes/NNN-*.xml`.
- `schemaName` / `schema="shared"` on table creates.
- `spring.liquibase.change-log: classpath:db/changelog/db.changelog-master.xml`
- `spring.jpa.hibernate.ddl-auto: validate` — never `create` or `update` in prod.

### 15. Tests

| Layer | Pattern |
|---|---|
| Controller | `@WebMvcTest` + `RbacTestSecurityConfig` + `@MockitoBean` services |
| Service | `@ExtendWith(MockitoExtension.class)` + `@Mock` + `@InjectMocks` |
| Repository (integration) | Testcontainers PostgreSQL — **NEVER H2** for integration tests |
| Repository (unit) | Mockito mock of repository interface acceptable for query-contract tests |
| DTO | Direct field/assertion tests for serialization contracts |
| Test data | `TestDataBuilder` utility class |

Coverage targets per build contract (typical: service 100%, controller 90%, repository 80%).

### 16. Sales-Rep Directory adaptation (UserDomain V1)

When story pack is `UserDomain` / `sales_rep_directory_api.md`:

1. **Scope:** domain service only — two read endpoints, no BFF, no Feign, no `sales.*`.
2. **Implement wire contract exactly** — emit `contract_conformance_report.md`, never edit the contract.
3. **Entities:** `Employee` (`shared.emp_employees`), `EmployeeTenantMap` (`shared.employee_tenant_map`).
4. **Endpoints:** `GET /api/v1/users/sales-reps`, `POST /api/v1/users/batch` (paths relative to service context).
5. **Response record fields:** `id`, `firstName`, `lastName`, `homeBranch`, `isCurrent` on **both** endpoints.
6. **No RBAC bean injection** (DR-26).
7. Reuse Central DB config + caching + exception patterns from this skill; skip RBAC controllers/entities not in allowlist.

## Pitfalls

- **NEVER** apply tenant `RoutingDataSource` / `multitenant.database.enabled: true` from sales-api skill.
- **NEVER** put `tenant_id` predicate on global `emp_employees`.
- **NEVER** use `default_flag` for branch/market logic.
- **NEVER** add `core-framework-web`, Feign, Resilience4j circuit breakers, or BFF layers.
- **NEVER** hand-roll `GlobalExceptionHandler` when framework + `RbacGlobalExceptionHandler` suffice.
- **NEVER** cache empty rosters (`unless = "#result == null || #result.isEmpty()"`).
- **NEVER** use `@Data` on entities; **NEVER** field `@Autowired`.
- **NEVER** skip `homeBranch` on batch endpoint (breaks Schedule board classification).
- **NEVER** commit secrets — env placeholders only (`AZURE_DB_*`, `AZURE_REDIS_*`).

## Workflow integration

For UserDomain codegen runs, attach this skill alongside:
- `domain-service-coreframework-guidelines` (framework binding)
- ADR zip / `_ADR_INDEX.md` (authoritative HOW)

In `AdditionalNotes`:
```
User Domain only. Follow orbt-user-domain-service-code-pattern.
Central DB shared.*. Implement 20_lld/domain/_BUILD.md only.
Do not generate BFF or Schedule artifacts.
```

## Additional reference

- Package tree, YAML snippets, and controller/service templates: [reference.md](reference.md)
