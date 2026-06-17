---
date: 2026-06-17T14:32:54.9698261-07:00
git_commit: 0488fc2f129585b718b1937f7abb99ebee3418ce
branch: feature/pg_create_policy
repository: data-api-builder
topic: "PostgreSQL create policy enforcement research"
tags: [research, postgresql, create-policy, authorization, sql]
status: complete
last_updated: 2026-06-17
---

# Research: PostgreSQL Create Policy Enforcement

## Research Question

Where in the current DAB codebase are create-action database policies enforced (or not enforced) for PostgreSQL create paths, how policy plumbing and parameter typing flow at query-build/execute time, what configuration validation gates apply, and which existing tests are the closest fit for PostgreSQL create-policy coverage.

## Summary

- SQL create execution is routed through `SqlMutationEngine` into `SqlInsertStructure`, then DB-specific `IQueryBuilder.Build(SqlInsertStructure)` and `IQueryExecutor.ExecuteQuery*` paths (`src/Core/Resolvers/SqlMutationEngine.cs:919`, `src/Core/Resolvers/SqlMutationEngine.cs:939`, `src/Core/Resolvers/SqlMutationEngine.cs:1483`).
- PostgreSQL insert builder currently emits direct `INSERT ... VALUES|DEFAULT VALUES ... RETURNING` without integrating create DB policy predicates (`src/Core/Resolvers/PostgresQueryBuilder.cs:68`, `src/Core/Resolvers/PostgresQueryBuilder.cs:81`).
- MSSQL and DWSQL insert builders both integrate create DB policy by rewriting to `SELECT ... FROM (VALUES(...)) ... WHERE <policy>` when policy is not base predicate (`src/Core/Resolvers/MsSqlQueryBuilder.cs:81`, `src/Core/Resolvers/MsSqlQueryBuilder.cs:89`, `src/Core/Resolvers/DWSqlQueryBuilder.cs:388`, `src/Core/Resolvers/DWSqlQueryBuilder.cs:396`).
- Authorization policy plumbing for create at query-build time exists: policy text is resolved from role+operation, parsed to filter clause, converted to SQL predicate, and stored in query structure for retrieval by builders (`src/Core/Resolvers/Sql Query Structures/SqlInsertQueryStructure.cs:71`, `src/Core/Resolvers/AuthorizationPolicyHelpers.cs:35`, `src/Core/Authorization/AuthorizationResolver.cs:208`, `src/Core/Resolvers/Sql Query Structures/BaseSqlQueryStructure.cs:596`).
- Parameter metadata (`DbType`/`SqlDbType`) is captured at parameter creation time, including null values, but generic executor does not apply `DbType`; MSSQL overrides and applies it explicitly (`src/Core/Resolvers/BaseQueryStructure.cs:119`, `src/Core/Models/DbConnectionParam.cs:10`, `src/Core/Resolvers/QueryExecutor.cs:424`, `src/Core/Resolvers/MsSqlQueryExecutor.cs:706`). PostgreSQL executor has no `PopulateDbTypeForParameter` override (`src/Core/Resolvers/PostgreSqlExecutor.cs:22`).
- Config validation currently defines create-policy support list as MSSQL/DWSQL only (`src/Core/Configurations/RuntimeConfigValidator.cs:47`), and permission validation is invoked only in development-mode entity validation (`src/Core/Configurations/RuntimeConfigValidator.cs:1918`, `src/Core/Configurations/RuntimeConfigValidator.cs:1928`, `src/Core/Services/MetadataProviders/SqlMetadataProvider.cs:346`). Host mode defaults to Production (`src/Config/ObjectModel/HostOptions.cs:38`).
- Existing PostgreSQL create-policy test hooks are present but ignored/unimplemented in REST and GraphQL subclasses (`src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs:344`, `src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs:710`), while base tests assert expected create-policy failure behavior (`src/Service.Tests/SqlTests/RestApiTests/Insert/InsertApiTestBase.cs:847`, `src/Service.Tests/SqlTests/GraphQLMutationTests/GraphQLMutationTestBase.cs:111`).

## Documentation System

- Framework: repository markdown plus external Learn docs reference (`docs/readme.md:3`).
- Docs directory: `docs/` (`docs/readme.md:1`).
- Navigation config: none found in repo root for static doc site engines (no `mkdocs.yml`/docusaurus config observed in current workspace listing); docs entry indicates migration to Learn (`docs/readme.md:3`).
- Style conventions: markdown docs and design notes in `docs/` plus root contribution docs (`README.md:1`, `CONTRIBUTING.md:1`).
- Build command: no local docs build command documented in `docs/readme.md`; it points to Learn contribution flow (`docs/readme.md:5`).
- Standard files: `README.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` at repo root.

## Verification Commands

- Test command: `dotnet test --filter "TestCategory=PostgreSql"` (`.github/copilot-instructions.md:63`).
- Lint/format command: `dotnet format src/Azure.DataApiBuilder.sln --verify-no-changes` (`.github/copilot-instructions.md:94`).
- Build command: `dotnet build src/Azure.DataApiBuilder.sln` (`.github/copilot-instructions.md:49`).
- Type check: none documented as a distinct command in repo guidance; C# compile is validated via `dotnet build` (`.github/copilot-instructions.md:49`).

## Detailed Findings

### 1) PostgreSQL Insert Query Generation Path and Create-Policy Integration Points

- Mutation flow constructs `SqlInsertStructure` for both REST insert and GraphQL create, then calls DB-specific builder: `queryBuilder.Build(insertQueryStruct)` (`src/Core/Resolvers/SqlMutationEngine.cs:919`, `src/Core/Resolvers/SqlMutationEngine.cs:939`).
- In multi-create/linking flow, insert execution also uses `SqlInsertStructure` + `queryBuilder.Build(sqlInsertStructure)` (`src/Core/Resolvers/SqlMutationEngine.cs:1466`, `src/Core/Resolvers/SqlMutationEngine.cs:1483`).
- PostgreSQL insert builder emits:
  - `INSERT INTO schema.table (cols) VALUES (...)` when columns exist,
  - `INSERT INTO schema.table DEFAULT VALUES` when no insert columns,
  - then `RETURNING ...`.
  No create-policy predicate is referenced in this method (`src/Core/Resolvers/PostgresQueryBuilder.cs:68`, `src/Core/Resolvers/PostgresQueryBuilder.cs:72`, `src/Core/Resolvers/PostgresQueryBuilder.cs:78`, `src/Core/Resolvers/PostgresQueryBuilder.cs:81`).
- Postgres update/delete builders do incorporate operation-specific DB policy via `GetDbPolicyForOperation(...)`, showing create is the missing operation in this builder (`src/Core/Resolvers/PostgresQueryBuilder.cs:88`, `src/Core/Resolvers/PostgresQueryBuilder.cs:99`).

### 2) Existing MSSQL/DWSQL Create-Policy Enforcement Pattern

- MSSQL insert build computes create policy predicates and uses two paths:
  - base predicate: normal `VALUES` insert,
  - non-base predicate: `SELECT <insertColumns> FROM (VALUES(...)) T(<insertColumns>) WHERE <dbPolicy>`.
  (`src/Core/Resolvers/MsSqlQueryBuilder.cs:81`, `src/Core/Resolvers/MsSqlQueryBuilder.cs:89`, `src/Core/Resolvers/MsSqlQueryBuilder.cs:90`).
- DWSQL insert build mirrors the same predicate-conditional `VALUES` vs `SELECT FROM (VALUES(...)) WHERE ...` pattern (`src/Core/Resolvers/DWSqlQueryBuilder.cs:388`, `src/Core/Resolvers/DWSqlQueryBuilder.cs:396`).
- MSSQL query executor maps empty-result create/update outcomes to `DatabasePolicyFailure` in relevant update/upsert flows (`src/Core/Resolvers/MsSqlQueryExecutor.cs:646`, `src/Core/Resolvers/MsSqlQueryExecutor.cs:659`).

### 3) Authorization Policy Plumbing Used at Query-Build Time

- `SqlInsertStructure` is created with `operationType: EntityActionOperation.Create` via base constructor path (`src/Core/Resolvers/Sql Query Structures/SqlInsertQueryStructure.cs:71`).
- `BaseSqlQueryStructure` constructor calls `AuthorizationPolicyHelpers.ProcessAuthorizationPolicies(...)` when `httpContext` is present and entity is not linking (`src/Core/Resolvers/Sql Query Structures/BaseSqlQueryStructure.cs:77`).
- `AuthorizationPolicyHelpers.ProcessAuthorizationPolicies` resolves role from request header, maps operation (including compound operation handling), reads DB policy from `AuthorizationResolver.ProcessDBPolicy`, parses into OData filter clause, and stores operation-specific predicate through callback (`src/Core/Resolvers/AuthorizationPolicyHelpers.cs:35`, `src/Core/Resolvers/AuthorizationPolicyHelpers.cs:55`, `src/Core/Resolvers/AuthorizationPolicyHelpers.cs:154`, `src/Core/Resolvers/AuthorizationPolicyHelpers.cs:184`).
- `AuthorizationResolver.ProcessDBPolicy` retrieves operation DB policy and substitutes claims values (`src/Core/Authorization/AuthorizationResolver.cs:208`, `src/Core/Authorization/AuthorizationResolver.cs:217`).
- Query builders consume policy through `GetDbPolicyForOperation(operation)` (`src/Core/Resolvers/Sql Query Structures/BaseSqlQueryStructure.cs:596`).
- For create requests, `SqlInsertStructure` also enforces that all fields referenced in create DB policy are present in request body by tracking `FieldsReferencedInDbPolicyForCreateAction` and throwing authorization failure when unresolved (`src/Core/Resolvers/Sql Query Structures/SqlInsertQueryStructure.cs:76`, `src/Core/Resolvers/Sql Query Structures/SqlInsertQueryStructure.cs:83`, `src/Core/Resolvers/Sql Query Structures/SqlInsertQueryStructure.cs:106`).

### 4) Parameter Typing Path and PostgreSQL/Null Typing Constraints

- Parameter objects are created by `MakeDbConnectionParam`, which stores value plus `DbType`, `SqlDbType`, and optional length derived from source definition (`src/Core/Resolvers/BaseQueryStructure.cs:119`, `src/Core/Resolvers/BaseQueryStructure.cs:126`).
- This same method is used for null values in insert/update paths (`src/Core/Resolvers/Sql Query Structures/SqlInsertQueryStructure.cs:119`).
- `DbConnectionParam` explicitly carries nullable `DbType` and `SqlDbType` (`src/Core/Models/DbConnectionParam.cs:10`, `src/Core/Models/DbConnectionParam.cs:27`, `src/Core/Models/DbConnectionParam.cs:32`).
- `DatabaseObject.GetDbTypeForParam` resolves column `DbType` from metadata (`src/Config/DatabasePrimitives/DatabaseObject.cs:211`).
- Generic executor creates `DbParameter`, assigns value, then calls `PopulateDbTypeForParameter`; base implementation is a no-op and interface comment states DbType population is currently only for MSSQL (`src/Core/Resolvers/QueryExecutor.cs:355`, `src/Core/Resolvers/QueryExecutor.cs:361`, `src/Core/Resolvers/QueryExecutor.cs:424`, `src/Core/Resolvers/IQueryExecutor.cs:167`).
- MSSQL executor overrides and applies `DbType`/`SqlDbType` (`src/Core/Resolvers/MsSqlQueryExecutor.cs:706`).
- PostgreSQL executor class extends generic executor but does not provide an override for parameter type population in this file (`src/Core/Resolvers/PostgreSqlExecutor.cs:22`).

### 5) Validation Logic for Create-Policy Support Lists and Mode Gating

- Allowed database types for create-action DB policy are currently `{MSSQL, DWSQL}` (`src/Core/Configurations/RuntimeConfigValidator.cs:47`).
- Permission validation rejects create-action database policy when datasource DB type is not in that set and policy for create is non-empty (`src/Core/Configurations/RuntimeConfigValidator.cs:1338`, `src/Core/Configurations/RuntimeConfigValidator.cs:1341`).
- `IsValidDatabasePolicyForAction` currently allows create only when database policy is null/whitespace (`src/Core/Configurations/RuntimeConfigValidator.cs:1381`, `src/Core/Configurations/RuntimeConfigValidator.cs:1383`).
- `ValidatePermissionsInConfig` is called from `ValidateEntityAndAutoentityConfigurations`, which runs only when runtime is development mode (`src/Core/Configurations/RuntimeConfigValidator.cs:1916`, `src/Core/Configurations/RuntimeConfigValidator.cs:1918`, `src/Core/Configurations/RuntimeConfigValidator.cs:1928`).
- Metadata provider invokes this entity validation during initialization (`src/Core/Services/MetadataProviders/SqlMetadataProvider.cs:346`).
- Host mode defaults to Production, and `RuntimeConfig.IsDevelopmentMode` gates this path (`src/Config/ObjectModel/HostOptions.cs:38`, `src/Config/ObjectModel/RuntimeConfig.cs:580`).

### 6) Existing Tests Relevant to PostgreSQL Create-Policy Behavior

- REST base create-policy failure expectations exist in `InsertApiTestBase`:
  - `InsertOneFailingDatabasePolicy` expects `DatabasePolicyFailure` for create-policy violation (`src/Service.Tests/SqlTests/RestApiTests/Insert/InsertApiTestBase.cs:847`, `src/Service.Tests/SqlTests/RestApiTests/Insert/InsertApiTestBase.cs:865`).
  - `InsertOneInTableWithFieldsInDbPolicyNotPresentInBody` expects `AuthorizationCheckFailed` for missing policy-referenced fields (`src/Service.Tests/SqlTests/RestApiTests/Insert/InsertApiTestBase.cs:873`, `src/Service.Tests/SqlTests/RestApiTests/Insert/InsertApiTestBase.cs:894`).
- PostgreSQL REST subclass currently marks both relevant overrides as `[Ignore]`/`NotImplementedException` (`src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs:344`, `src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs:351`).
- GraphQL mutation base has helper methods validating create-policy fail/pass outcomes (`src/Service.Tests/SqlTests/GraphQLMutationTests/GraphQLMutationTestBase.cs:111`, `src/Service.Tests/SqlTests/GraphQLMutationTests/GraphQLMutationTestBase.cs:134`).
- PostgreSQL GraphQL subclass currently has create-policy-referenced-field test override ignored/not implemented (`src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs:710`).
- Config validation unit tests currently assert PostgreSQL create-policy-defined config fails and MSSQL/DWSQL passes (`src/Service.Tests/UnitTests/ConfigValidationUnitTests.cs:188`, `src/Service.Tests/UnitTests/ConfigValidationUnitTests.cs:196`, `src/Service.Tests/UnitTests/ConfigValidationUnitTests.cs:197`).

## Risks / Unknowns and Validation Checks (Evidence-backed)

### Observed Risks/Unknowns

- Risk: PostgreSQL create builder currently has no create-policy predicate application while MSSQL/DWSQL do (`src/Core/Resolvers/PostgresQueryBuilder.cs:68`, `src/Core/Resolvers/MsSqlQueryBuilder.cs:81`, `src/Core/Resolvers/DWSqlQueryBuilder.cs:388`).
- Risk: Generic executor does not apply `DbType` and PostgreSQL executor does not add a PostgreSQL-specific override in current file; null-valued parameters therefore rely on provider defaults (`src/Core/Resolvers/QueryExecutor.cs:424`, `src/Core/Resolvers/PostgreSqlExecutor.cs:22`, `src/Core/Resolvers/Sql Query Structures/SqlInsertQueryStructure.cs:119`).
- Risk: Validation behavior for create-policy is gated behind development mode in metadata initialization flow, while host default is production (`src/Core/Configurations/RuntimeConfigValidator.cs:1918`, `src/Core/Services/MetadataProviders/SqlMetadataProvider.cs:346`, `src/Config/ObjectModel/HostOptions.cs:38`).
- Unknown: Final runtime behavior differences between host modes for rejected/accepted create-policy config require environment-specific verification because permission validation call is mode-gated (`src/Core/Configurations/RuntimeConfigValidator.cs:1928`, `src/Config/ObjectModel/RuntimeConfig.cs:580`).
- Unknown: PostgreSQL REST/GraphQL create-policy scenarios are not represented by implemented subclass tests today due to ignored methods (`src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs:344`, `src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs:710`).

### Proposed Validation Checks

- Check 1: Verify generated PostgreSQL insert SQL for create operations contains or omits create-policy predicate as intended, by asserting `PostgresQueryBuilder.Build(SqlInsertStructure)` output patterns (`src/Core/Resolvers/PostgresQueryBuilder.cs:68`).
- Check 2: Validate REST create-policy fail/pass behavior for PostgreSQL by implementing/enabling the currently ignored overrides in PostgreSQL insert tests (`src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs:344`, `src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs:351`) and using base expected status/substatus assertions (`src/Service.Tests/SqlTests/RestApiTests/Insert/InsertApiTestBase.cs:865`, `src/Service.Tests/SqlTests/RestApiTests/Insert/InsertApiTestBase.cs:894`).
- Check 3: Validate GraphQL create-policy fail/pass behavior for PostgreSQL using existing base helper expectations and subclass coverage (`src/Service.Tests/SqlTests/GraphQLMutationTests/GraphQLMutationTestBase.cs:111`, `src/Service.Tests/SqlTests/GraphQLMutationTests/GraphQLMutationTestBase.cs:134`, `src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs:710`).
- Check 4: Validate parameter typing behavior for null create payload fields in PostgreSQL execution path where generic `PopulateDbTypeForParameter` is no-op (`src/Core/Resolvers/QueryExecutor.cs:424`, `src/Core/Resolvers/Sql Query Structures/SqlInsertQueryStructure.cs:119`).
- Check 5: Validate config acceptance/rejection semantics for PostgreSQL create-policy in unit tests around current support-list expectations (`src/Service.Tests/UnitTests/ConfigValidationUnitTests.cs:188`, `src/Core/Configurations/RuntimeConfigValidator.cs:47`, `src/Core/Configurations/RuntimeConfigValidator.cs:1338`).
- Check 6: Validate mode-gated behavior by executing config-load/init checks under both development and production host modes (`src/Core/Configurations/RuntimeConfigValidator.cs:1918`, `src/Config/ObjectModel/HostOptions.cs:38`, `src/Config/ObjectModel/RuntimeConfig.cs:580`).

## Code References

- `src/Core/Resolvers/SqlMutationEngine.cs:919` - Create/insert operations instantiate `SqlInsertStructure`.
- `src/Core/Resolvers/SqlMutationEngine.cs:939` - Mutation path invokes query builder for insert.
- `src/Core/Resolvers/SqlMutationEngine.cs:1483` - Multi-create/linking insert path also invokes builder.
- `src/Core/Resolvers/PostgresQueryBuilder.cs:68` - PostgreSQL insert builder entrypoint.
- `src/Core/Resolvers/PostgresQueryBuilder.cs:81` - PostgreSQL insert returns `RETURNING` without create-policy predicate.
- `src/Core/Resolvers/MsSqlQueryBuilder.cs:81` - MSSQL create-policy predicate retrieval.
- `src/Core/Resolvers/MsSqlQueryBuilder.cs:89` - MSSQL conditional `VALUES` vs `SELECT FROM (VALUES...) WHERE policy`.
- `src/Core/Resolvers/DWSqlQueryBuilder.cs:388` - DWSQL create-policy predicate retrieval.
- `src/Core/Resolvers/AuthorizationPolicyHelpers.cs:35` - Policy processing helper entrypoint.
- `src/Core/Authorization/AuthorizationResolver.cs:208` - DB policy resolution with claims.
- `src/Core/Resolvers/Sql Query Structures/SqlInsertQueryStructure.cs:71` - Create operation binding in insert structure.
- `src/Core/Resolvers/BaseQueryStructure.cs:119` - Parameter creation with type metadata.
- `src/Core/Resolvers/QueryExecutor.cs:424` - Generic parameter type population is no-op.
- `src/Core/Resolvers/MsSqlQueryExecutor.cs:706` - MSSQL parameter type population override.
- `src/Core/Configurations/RuntimeConfigValidator.cs:47` - Create-policy support list.
- `src/Core/Configurations/RuntimeConfigValidator.cs:1338` - Create-policy validation enforcement by DB type.
- `src/Core/Configurations/RuntimeConfigValidator.cs:1918` - Development-mode gate for entity+permission validation.
- `src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs:344` - PostgreSQL REST create-policy test override currently ignored.
- `src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs:710` - PostgreSQL GraphQL create-policy test override currently ignored.
- `src/Service.Tests/UnitTests/ConfigValidationUnitTests.cs:188` - Config test expecting PostgreSQL create-policy validation failure.

## Open Questions

- Host-mode validation semantics were later decided in planning artifacts to be consistent across development and production modes; implementation details remain to be validated against current mode-gated behavior (`src/Core/Configurations/RuntimeConfigValidator.cs:1918`).
- For PostgreSQL create-policy query rewriting, should null parameter typing rely on provider inference or explicit type assignment (current generic no-op typing path: `src/Core/Resolvers/QueryExecutor.cs:424`)?
- Should PostgreSQL test coverage parity include both REST and GraphQL create-policy scenarios by un-ignoring existing subclass overrides (`src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs:344`, `src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs:710`)?
