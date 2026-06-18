# PostgreSQL Create Policy Enforcement

## Overview
This work enables PostgreSQL create-action database policy enforcement for Data API builder create operations and aligns configuration validation with that runtime support. Before this change, create policy behavior for PostgreSQL diverged from other SQL engines, allowing policy expressions to be configured but not reliably enforced in create flows.

The implementation closes that gap by enforcing create policy during PostgreSQL insert query generation, preserving expected create semantics, and expanding regression coverage across REST and GraphQL. It also standardizes create-policy compatibility validation across host modes so unsupported combinations are rejected consistently.

## Architecture and Design

### High-Level Architecture
The feature spans four layers:
- Query generation in PostgreSQL SQL builder.
- PostgreSQL query execution parameter typing.
- Runtime configuration validation semantics.
- End-to-end and unit test coverage.

At runtime, create policy expressions are obtained from the existing authorization-policy pipeline and applied in PostgreSQL insert SQL generation. Validation checks are performed independently of development-mode-only validation gates for create-policy support decisions.

### Design Decisions
- PostgreSQL create SQL now follows the existing SQL-engine pattern for policy-aware inserts when the create policy is non-baseline.
- The base predicate path preserves simple insert behavior for policy-baseline scenarios.
- PostgreSQL parameter DbType propagation remains provider-specific in the PostgreSQL executor to avoid broad behavior changes in shared query execution paths.
- Create-policy compatibility checks are enforced in all host modes to remove mode-dependent ambiguity.

### Integration Points
- Query builder: [src/Core/Resolvers/PostgresQueryBuilder.cs](src/Core/Resolvers/PostgresQueryBuilder.cs#L68)
- PostgreSQL executor parameter typing: [src/Core/Resolvers/PostgreSqlExecutor.cs](src/Core/Resolvers/PostgreSqlExecutor.cs#L142)
- Validation support matrix and mode-independent create-policy validation:
  - [src/Core/Configurations/RuntimeConfigValidator.cs](src/Core/Configurations/RuntimeConfigValidator.cs#L47)
  - [src/Core/Configurations/RuntimeConfigValidator.cs](src/Core/Configurations/RuntimeConfigValidator.cs#L1918)

## User Guide

### Prerequisites
- PostgreSQL-backed DAB configuration.
- Entity permissions with create action and a database policy expression where required.
- Test environment with reachable PostgreSQL instance for integration tests.

### Basic Usage
1. Define create-action database policy for a PostgreSQL entity role.
2. Submit create request via REST or GraphQL with policy-compliant payload.
3. Expect successful insert and standard create response behavior.
4. Submit create request with policy-violating payload.
5. Expect policy failure behavior and no unauthorized row persistence.

### Advanced Usage
- Host-mode behavior:
  - PostgreSQL create policy is accepted in both development and production modes.
  - Unsupported engines for create-policy remain rejected consistently in both modes.
- Nullable/type-sensitive payloads:
  - PostgreSQL parameter DbType metadata propagation is applied where available to improve stability in policy-aware create flows.

## API Reference

### Key Components
- PostgreSQL policy-aware insert SQL generation:
  - [Build(SqlInsertStructure)](src/Core/Resolvers/PostgresQueryBuilder.cs#L68)
- PostgreSQL DbType parameter assignment:
  - [PopulateDbTypeForParameter](src/Core/Resolvers/PostgreSqlExecutor.cs#L142)
- Create-policy support validation:
  - [_databaseTypesSupportingCreatePolicy](src/Core/Configurations/RuntimeConfigValidator.cs#L47)
  - [ValidateCreatePolicySupport](src/Core/Configurations/RuntimeConfigValidator.cs#L1940)

### Configuration Options
- No new configuration schema fields were introduced.
- Existing create-action database policy configuration for PostgreSQL is now treated as supported.
- Unsupported engine combinations continue to return config validation errors.

## Testing

### How to Test
- Build:
  - `dotnet build src/Azure.DataApiBuilder.sln`
- Focused unit tests for create-policy validation and PostgreSQL DbType handling:
  - `dotnet test src/Service.Tests/Azure.DataApiBuilder.Service.Tests.csproj --filter "FullyQualifiedName~Test_PostgreSql_DbCommandParameter_PopulatedWithCorrectDbTypes|FullyQualifiedName~AddDatabasePolicyToCreateOperation"`
- PostgreSQL REST create-policy coverage:
  - [src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs](src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs#L365)
- PostgreSQL GraphQL create-policy coverage:
  - [src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs](src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs#L142)

### Edge Cases
- Create policy violation should fail with policy-failure behavior and no unauthorized persistence.
- Policy-referenced fields absent in request body should fail authorization checks.
- Baseline policy path should preserve normal insert behavior.
- Host-mode parity for create-policy compatibility validation is covered in:
  - [src/Service.Tests/UnitTests/ConfigValidationUnitTests.cs](src/Service.Tests/UnitTests/ConfigValidationUnitTests.cs#L237)

## Limitations and Future Work
- Full PostgreSQL integration tests require a running local PostgreSQL instance; in environments without DB connectivity these tests cannot be fully executed.
- Additional stress/perf coverage for multi-create/linking-specific scenarios can be added in a follow-up if required.
- Broader policy matrix coverage can be extended as additional database engines gain create-policy support.
