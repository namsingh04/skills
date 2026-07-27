---
name: "compilation-error-triage"
description: "Systematic error classification and resolution order for Java/Spring Boot compilation, test, and runtime errors in microservices migration projects"
version: 1
created: "2026-06-09"
updated: "2026-06-09"
---
## When to Use
Use when encountering build failures, compilation errors, test failures, or runtime errors during microservices development or migration. Apply when facing multiple error types simultaneously and need prioritized resolution order. Essential for troubleshooting Spring Boot 4.x, Java 21, Maven, JPA, and framework integration issues.

## Procedure
1. STEP 1: Classify All Errors - Run 'mvn clean compile' and collect all error output. Categorize each error: COMPILE (syntax, missing class, unresolved dependency), TEST (test compilation, test execution failure, assertion failure), RUNTIME (startup failure, bean creation, configuration error). Document error count per category. Prioritize: COMPILE → TEST → RUNTIME (fix in this order)
2. STEP 2: Resolve Dependency Errors First - Check parent POM inheritance (must extend spring-boot-starter-parent 4.0.6+). Verify all core-framework-* dependencies are declared with correct scope. Run 'mvn dependency:tree' to check for conflicts. Add missing dependencies (spring-boot-starter-web, spring-boot-starter-data-jpa, etc.). Exclude logback-classic and junit-vintage-engine from ALL dependencies. Fix version conflicts using dependencyManagement section. Re-run 'mvn clean compile'
3. STEP 3: Fix Schema Mapping Errors - Review entity @Table annotations match actual database schema names. Verify @Column(name='...') matches exact DB column names (case-sensitive for PostgreSQL). Add missing @Id, @GeneratedValue on primary keys. Fix relationship mappings (@ManyToOne, @OneToMany) with proper FetchType.LAZY. Add tenant_id column to all tenant-specific entities. Ensure Liquibase changelogs exist for all tables. Validate JPA entities compile with 'mvn clean compile'
4. STEP 4: Resolve Import and Package Errors - Fix missing imports (auto-import in IDE or add manually). Remove unused imports. Verify package names follow reverse-DNS lowercase convention (com.sales.service, not com.Sales.Service). Check classes are in correct packages (controllers in .controller, services in .service, etc.). Ensure no duplicate class names across packages. Re-compile after each fix batch
5. STEP 5: Fix Type Errors and Annotations - Verify Records have @JsonProperty on all fields. Check service layer uses constructor injection with final fields (no @Autowired on fields). Ensure controllers use @RestController, services use @Service, repositories extend JpaRepository. Fix generic type mismatches (List<String> vs List<Object>). Add @Transactional on service methods. Add @Valid on controller DTOs. Verify MapStruct mappers have @Mapper(componentModel='spring')
6. STEP 6: Resolve Test Compilation Errors - Check test classes use JUnit 5 (@Test from org.junit.jupiter.api, NOT org.junit). Use @ExtendWith(MockitoExtension.class) for unit tests (not @RunWith). Replace @MockBean with @MockitoBean (Spring Boot 4.x). Fix test assertions to use AssertJ (assertThat, not assertEquals). Add missing test dependencies (junit-jupiter, mockito-core, assertj-core). Ensure test resources (application-test.yml) exist
7. STEP 7: Fix Test Execution Failures - Run 'mvn clean test' and collect failures. For 'No qualifying bean' errors in slice tests: add @MockitoBean for external dependencies. For 'Unable to find @SpringBootConfiguration': ensure test class is in same package or sub-package as @SpringBootApplication. For Testcontainers failures: verify Docker is running or switch to H2 for unit tests. For mock setup errors: use given().willReturn() pattern (BDD style). Fix assertions to match actual vs expected order (assertThat(actual).isEqualTo(expected))
8. STEP 8: Document and Track - Create error log with: error type, error message, root cause, resolution. Track resolution order and time per category. Document common patterns (missing dependency X causes error Y). Update project README with known issues and fixes. Create checklist for future iterations

## Pitfalls
- DO NOT fix errors in random order - always resolve dependencies first, then compile errors, then test errors, then runtime errors
- DO NOT assume all errors are code-related - many are configuration or dependency issues in pom.xml
- DO NOT add dependencies without checking for conflicts - use 'mvn dependency:tree' to validate
- DO NOT skip parent POM inheritance - services MUST extend spring-boot-starter-parent for Spring Boot 4.x
- DO NOT use case-insensitive column names - PostgreSQL is case-sensitive, use exact @Column(name='column_name') matching
- DO NOT fix test failures before fixing compilation errors - tests cannot run if code does not compile
- DO NOT use @SpringBootTest for unit tests - use @ExtendWith(MockitoExtension.class) to avoid slow Spring context loading
- DO NOT use JUnit 4 annotations - Spring Boot 4.x requires JUnit 5 (Jupiter), exclude junit-vintage-engine
- DO NOT use @MockBean - Spring Boot 4.x uses @MockitoBean instead
- DO NOT ignore schema mapping errors - JPA entities must exactly match database schema (table names, column names, data types)
- DO NOT add core-framework-web to Domain services - only BFF services should include core-framework-web
- DO NOT forget to exclude logback-classic - Spring Boot 4.x with Log4j2 requires explicit logback exclusion
- DO NOT assume error messages are accurate - 'Cannot find symbol' often means missing dependency, not missing code
- DO NOT fix symptoms without understanding root cause - one missing dependency can cause dozens of cascading errors

## Verification
1. Run 'mvn clean compile' successfully with zero compilation errors
2. Run 'mvn dependency:tree' and verify no conflicts, all core-framework-* dependencies present
3. Run 'mvn clean test' and verify all tests pass with no mocking errors or bean creation failures
4. Check JaCoCo report shows >= 90% coverage (mvn jacoco:report)
5. Verify application starts successfully with 'mvn spring-boot:run' (no bean creation or configuration errors)
6. Check logs show no WARN or ERROR during startup
7. Verify database schema matches JPA entities (run Liquibase validation or compare DDL)
8. Confirm all entities have @Id, @Table with correct schema, @Column names matching DB exactly
9. Validate no deprecated APIs or warnings in build output
10. Run 'mvn checkstyle:check' to ensure code style compliance