---
name: "legacy-sql-to-modern-jpa-transformation"
description: "Transform legacy SQL (stored procedures, views, dynamic SQL) to modern JPA/JPQL patterns with proper schema mapping, parameterization, and type safety. Use with Database Implementation agents to migrate legacy database code to modern microservices architecture."
version: 1
created: "2026-06-09"
updated: "2026-06-09"
---
## When to Use
Use this skill when:
- Migrating from legacy databases (VB.NET, SQL Server stored procedures, Oracle PL/SQL) to modern microservices with PostgreSQL/JPA
- Database agent needs to replace stored procedures with JPA repositories
- Converting dynamic SQL strings to type-safe JPQL queries
- Mapping legacy views/functions to JPA entity graphs
- Transforming legacy schema (denormalized, no constraints) to normalized schema with proper relationships
- Creating Liquibase/Flyway migrations from legacy DDL
- Legacy code uses EXEC sp_*, string concatenation for SQL, or cursors
- Target architecture: Spring Data JPA + PostgreSQL with Liquibase migrations

## Procedure
1. STEP 1: Analyze Legacy SQL Pattern - Identify the legacy pattern (stored procedure, dynamic SQL, view, cursor-based logic, temp tables). Extract: inputs (parameters), outputs (result sets/columns), business logic, joins, filters, sorting. Document legacy schema: table names, column names, data types, relationships (even if not enforced).
2. STEP 2: Design Target Schema - Map legacy tables to normalized target schema (sales_schema, etc.). Add missing constraints (PRIMARY KEY, FOREIGN KEY, NOT NULL, UNIQUE). Add tenant isolation columns (tenant_id) if multi-tenant. Decide entity relationships (@ManyToOne, @OneToMany) with FetchType.LAZY. Create schema migration (Liquibase changeset): V{version}__{description}.sql with CREATE TABLE, ALTER TABLE, CREATE INDEX, comments.
3. STEP 3: Create JPA Entities - Map each target table to @Entity class with @Table(schema="target_schema"). Use @Getter/@Setter (NEVER @Data on entities). Add @Column(name="...") for all fields matching exact DB column names. Use FetchType.LAZY on ALL relationships. Add tenant_id column with @Column(name="tenant_id"). Use @EqualsAndHashCode(onlyExplicitlyIncluded=true) with @EqualsAndHashCode.Include on @Id only.
4. STEP 4: Transform SQL Logic to JPQL - Convert legacy SELECT with table joins to JPQL with entity joins (LEFT JOIN entity.relationship, not LEFT JOIN table_name). Replace dynamic SQL parameters (:param) with named JPQL parameters. Convert CASE/COALESCE/aggregate functions to JPQL equivalents or native query if needed. Replace cursors/loops with batch queries or streaming (Stream<Entity>). Convert temp tables to @Transient fields or separate queries.
5. STEP 5: Create Repository Interfaces - Extend JpaRepository<Entity, ID> for basic CRUD. Create CustomRepository interface for complex queries (replaces stored procedures). Implement CustomRepositoryImpl with @PersistenceContext EntityManager for JPQL or native SQL. Use @Query with JPQL for read queries, @Modifying for updates/deletes. Add @Param annotations for named parameters. Use DTO projections (constructor expression in JPQL) for read-only result sets.
6. STEP 6: Handle Legacy SQL Idioms - VARCHAR concatenation → CONCAT() or Java String.format in service layer. ISNULL/COALESCE → Java Optional or JPQL COALESCE. CONVERT/CAST date formats → Java DateTimeFormatter in service layer. Dynamic ORDER BY → Pageable/Sort in Spring Data or CASE in JPQL. String-based IN clauses → List<T> parameters in JPQL. EXEC sp_name → Custom repository method with @Query.
7. STEP 7: Create Migration Mapping Documentation - Document legacy-to-target mapping: legacy_table → target_entity, legacy_sp → repository_method, legacy_column → entity_field. Include performance notes (indexes needed, N+1 prevention with @EntityGraph). Add rollback strategy if migration fails. Document any business logic moved from SQL to Java service layer.

## Pitfalls
- DO NOT use table names in JPQL - use entity names and relationship navigation (e.g., 'FROM Lead l LEFT JOIN l.branch b' not 'FROM leads l LEFT JOIN branches b')
- DO NOT use FetchType.EAGER on relationships - always LAZY to prevent N+1 queries and performance issues
- DO NOT put @Data on @Entity classes - use @Getter/@Setter + @EqualsAndHashCode(onlyExplicitlyIncluded=true) to avoid lazy-loading issues
- DO NOT copy legacy table structure exactly - normalize schema, add constraints, add tenant_id, fix data types (VARCHAR(MAX) → TEXT, DATETIME → TIMESTAMP WITH TIME ZONE)
- DO NOT use string concatenation for SQL parameters - always use named parameters (@Param) to prevent SQL injection
- DO NOT convert all logic to SQL/JPQL - complex business rules belong in Java service layer, not database queries
- DO NOT forget indexes on foreign keys and tenant_id - legacy SQL may have relied on implicit indexes that don't exist in target schema
- DO NOT use native SQL unless absolutely necessary - prefer JPQL for database portability and type safety; use native SQL only for database-specific functions
- DO NOT assume legacy data types match target - MONEY → NUMERIC(19,4), BIT → BOOLEAN, IMAGE/BLOB → BYTEA, XML → JSONB (PostgreSQL)
- DO NOT ignore legacy comments/documentation - stored procedure comments often contain critical business rules not obvious from SQL alone

## Verification
1. Verify schema migration runs successfully: mvn liquibase:update or Liquibase CLI, check target database has all tables/columns/indexes/constraints
2. Verify entities compile and map correctly: mvn clean compile, check no JPA validation errors, run schema validation (spring.jpa.hibernate.ddl-auto=validate)
3. Verify JPQL queries return same results as legacy SQL: compare row counts, column values, sort order using integration tests with Testcontainers
4. Verify no N+1 queries: enable SQL logging (spring.jpa.show-sql=true), run query, check console shows single query (or explicit batch), not N+1 SELECT statements
5. Verify parameterization prevents SQL injection: test with malicious input ('; DROP TABLE--), ensure parameterized query rejects it safely
6. Verify performance: compare query execution time (legacy vs JPQL), check EXPLAIN ANALYZE plans in PostgreSQL, ensure indexes are used
7. Verify tenant isolation: test with multiple tenant_id values, ensure queries filter by tenant_id, test with TenantContextHolder integration
8. Run integration tests: @DataJpaTest with Testcontainers for PostgreSQL, test all repository methods, verify result sets match expected DTOs