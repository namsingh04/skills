# ORBT User Domain — Reference Detail

Companion to [SKILL.md](SKILL.md). Read when implementing a specific layer.

## Full package tree (orbt-user-api-src)

```
src/main/java/com/orbt/user/
├── OrbtUserApiApplication.java
├── config/
│   ├── AuthSecurityConfig.java
│   ├── CentralDatabaseConfig.java
│   ├── JpaConfig.java
│   ├── OpenApiConfig.java
│   ├── RbacAsyncConfig.java
│   └── ResilienceConfig.java
├── constant/
│   └── AppConstants.java
├── controller/
│   ├── RbacReadController.java
│   └── UserProfileController.java
├── domain/
│   ├── entity/
│   │   ├── Authority.java
│   │   ├── Employee.java
│   │   ├── RbacUserAccessAuditLog.java
│   │   ├── Role.java
│   │   ├── RoleAuthority.java
│   │   ├── Tenant.java
│   │   ├── UserProfile.java
│   │   ├── UserProfileAuthority.java
│   │   └── UserTenantMap.java
│   └── repository/
│       ├── AuthorityRepository.java
│       ├── EmployeeRepository.java
│       ├── RbacAuditLogRepository.java
│       ├── RoleAuthorityRepository.java
│       ├── RoleRepository.java
│       ├── UserProfileAuthorityRepository.java
│       ├── UserProfileRepository.java
│       ├── UserTenantMapRepository.java
│       └── projection/
│           ├── AccessibleMarketProjection.java
│           └── AuthorityProjection.java
├── dto/
│   ├── constant/
│   │   └── CacheConstants.java
│   ├── request/
│   │   ├── TenantSelectionDTO.java
│   │   └── UpdateActiveTenantRequestDTO.java
│   └── response/
│       ├── FeaturePermissionResponse.java
│       ├── MarketAccessItem.java
│       ├── MarketAccessResponse.java
│       ├── PagePermissionItem.java
│       ├── PagePermissionsResponse.java
│       ├── TenantDetailDTO.java
│       ├── UserProfileDTO.java
│       ├── UserProfilePagePermissionDTO.java
│       ├── UserProfilePermissionsDTO.java
│       └── UserRoleResponse.java
├── exception/
│   ├── DataModelPendingException.java
│   ├── DuplicateAuthorityException.java
│   ├── DuplicateProfileException.java
│   ├── InvalidAuthoritySourceException.java
│   ├── InvalidAuthorityTypeException.java
│   ├── MarketAccessDeniedException.java
│   ├── ProfileNotFoundException.java
│   ├── RbacAccessDeniedException.java
│   ├── RbacException.java
│   ├── RbacGlobalExceptionHandler.java
│   ├── RbacReadOnlyException.java
│   ├── RoleNotFoundException.java
│   └── TenantMembershipNotFoundException.java
├── mapper/
│   ├── EmployeeMapper.java
│   └── RbacMapper.java
├── service/
│   ├── AuditService.java
│   ├── MarketAccessService.java
│   ├── PermissionResolutionService.java
│   ├── UserRbacProfileService.java
│   ├── UserTenantMappingService.java
│   ├── impl/
│   │   ├── AuditServiceStubImpl.java
│   │   ├── MarketAccessServiceImpl.java
│   │   ├── PermissionResolutionServiceImpl.java
│   │   ├── UserRbacProfileServiceImpl.java
│   │   └── UserTenantMappingServiceImpl.java
│   └── model/
│       └── ResolvedPermissionSet.java
└── util/
    └── LogSanitizer.java
```

## Resources layout

```
src/main/resources/
├── application.yml
├── application-dev.yml          # profile overrides
├── log4j2.xml
└── db/changelog/
    ├── db.changelog-master.xml
    └── changes/
        ├── 001-create-emp-employees.xml
        ├── 002-create-user-tenant-map.xml
        └── ... (RBAC migrations 004–012)
```

## Key application.yml blocks

```yaml
server:
  servlet:
    context-path: /api/user

spring:
  application:
    name: orbt-user-api-src
  threads:
    virtual:
      enabled: true
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.xml

central:
  database:
    enabled: true
  datasource:
    url: jdbc:postgresql://${AZURE_DB_HOST_1}:${AZURE_DB_PORT}/${AZURE_DB_NAME}
    username: ${AZURE_DB_USER}

multitenant:
  database:
    enabled: false
  cache:
    enabled: false

app:
  tenant:
    header: X-Tenant-Id
    validation: true

common:
  error:
    enabled: true
    incident-prefix: User API Service
```

## Controller template

```java
@RestController
@RequestMapping("/v1/users")
@Tag(name = "Sales Rep Directory", description = "...")
@Slf4j
@RequiredArgsConstructor
public class SalesRepDirectoryController {

    private final SalesRepDirectoryService salesRepDirectoryService;

    @GetMapping("/sales-reps")
    @Operation(summary = "List sales reps by home branch")
    public ResponseEntity<ApiResponseDTO<List<SalesRepDto>>> listByHomeBranch(
            @RequestParam String homeBranch,
            @RequestParam(defaultValue = "true") boolean currentOnly) {
        var data = salesRepDirectoryService.listByHomeBranch(homeBranch, currentOnly);
        return ResponseEntity.ok(ApiResponseDTO.success(data, "Sales reps retrieved successfully"));
    }
}
```

## Service + cache template

```java
@Service
@Slf4j
public class SalesRepDirectoryServiceImpl implements SalesRepDirectoryService {

    private final EmployeeRepository employeeRepository;

    public SalesRepDirectoryServiceImpl(final EmployeeRepository employeeRepository) {
        this.employeeRepository = Objects.requireNonNull(employeeRepository);
    }

    @Override
    @Cacheable(
        cacheNames = CacheConstants.SALES_REPS_BY_BRANCH,
        key = "T(com.renuity.core.common.tenant.TenantContextHolder).current() + ':' + #homeBranch + ':' + #currentOnly",
        unless = "#result == null || #result.isEmpty()")
    @Transactional(readOnly = true)
    public List<SalesRepDto> listByHomeBranch(final String homeBranch, final boolean currentOnly) {
        final var tenantId = TenantContextHolder.current();
        // R1 query: join emp_employees + employee_tenant_map; NO tenant predicate on emp_employees
        return employeeRepository.findRosterByHomeBranch(tenantId, homeBranch, currentOnly)
            .stream()
            .map(SalesRepMapper::toDto)
            .toList();
    }
}
```

## Entity template (global + tenant map)

```java
// Global — NO tenant_id
@Entity
@Table(name = "emp_employees", schema = "shared")
public class Employee {
    @Id
    @Column(name = "id")
    private Long id;

    @Column(name = "brn_id")          // preserve legacy column name
    private String homeBranch;

    @Column(name = "sales_rep")
    private Boolean salesRep;
}

// Tenant-scoped map
@Entity
@Table(name = "employee_tenant_map", schema = "shared")
@SQLRestriction("is_deleted = false")
public class EmployeeTenantMap {
    @Column(name = "tenant_id", nullable = false)
    private Long tenantId;

    @Column(name = "emp_id", nullable = false)
    private Long empId;
}
```

## Exception template

```java
public class SalesRepNotFoundException extends RbacException {
    public SalesRepNotFoundException() {
        super(HttpStatus.NOT_FOUND, "RN_USER_DOMAIN_NOT_FOUND_404", "Requested resource not found");
    }
}
```

## Test template

```java
@WebMvcTest(SalesRepDirectoryController.class)
@Import(RbacTestSecurityConfig.class)
class SalesRepDirectoryControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private SalesRepDirectoryService salesRepDirectoryService;

    @Test
    void listByHomeBranch_returns200() throws Exception {
        given(salesRepDirectoryService.listByHomeBranch("NW01", true))
            .willReturn(List.of(new SalesRepDto(1L, "Dana", "Reyes", "NW01", true)));

        mockMvc.perform(get("/v1/users/sales-reps")
                .param("homeBranch", "NW01")
                .header("X-Tenant-Id", "1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.success").value(true));
    }
}
```

## Difference from orbt-domain-service-code-pattern

| Aspect | User Domain (this skill) | Tenant Domain (sales-api skill) |
|---|---|---|
| Database | Central DB, `shared.*` | Tenant DB routing |
| Entity package | `domain.entity` | `model` |
| DTO style | Lombok classes + `@JsonProperty` | Java records |
| Mapper | Static utility classes | MapStruct |
| Controller | Feature controllers | CQRS Command + Query |
| Multi-tenancy | Header + tenant map join | Routing DataSource + tenant SQL bind |
| Resilience4j | Not in service layer | Not in domain |
| Security config | `AuthSecurityConfig` permitAll | No SecurityConfig in domain |
