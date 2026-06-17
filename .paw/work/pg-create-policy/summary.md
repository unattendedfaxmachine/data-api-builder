# PostgreSQL Create DB Policy Investigation and Handoff Summary

## Scope
This document summarizes investigation work for correlating:
- Azure DevOps work item 2141216 (AppDev/Rayfin)
- GitHub issue #1334 in Azure/data-api-builder

The core problem is PostgreSQL create-action database policy handling and security impact.

## Executive Summary
The issue is real and explainable from current OSS code:
1. PostgreSQL create-action DB policy is not enforced in query generation.
2. Validation rules that block PostgreSQL create-policy are only enforced in development mode during metadata init.
3. In production mode (default), those validation checks can be skipped, allowing configs with PostgreSQL create-policy to still run.
4. A complete fix likely needs both SQL rewrite logic and PostgreSQL parameter typing improvements (for NULL/type inference stability).

## Correlated Ticket Perspective

### GitHub issue #1334 perspective
- Technical/implementation perspective.
- Focus: adding create-policy support for PostgreSQL similar to MSSQL path introduced by #1325.
- Known blocker discussed historically: PostgreSQL type inference errors when NULL appears in VALUES/SELECT without explicit typing.

### Azure DevOps 2141216 perspective
- Security/impact perspective.
- Focus: cross-user write injection in PostgreSQL due to create-policy not being enforced.
- Symptoms include successful commit with policy expectation violated.

These are the same root issue viewed from two lenses: engineering gap and security impact.

## Key Confirmed Findings (Code)

### 1) PostgreSQL create path does not apply create DB policy
- File: src/Core/Resolvers/PostgresQueryBuilder.cs
- Method: Build(SqlInsertStructure structure)
- Behavior: emits INSERT ... VALUES ... RETURNING with no create-policy predicate integration.

### 2) MSSQL and DWSQL do apply create DB policy
- File: src/Core/Resolvers/MsSqlQueryBuilder.cs
- Method: Build(SqlInsertStructure structure)
- Behavior: uses GetDbPolicyForOperation(Create) and rewrites to INSERT ... SELECT ... FROM (VALUES(...)) WHERE <policy>.

- File: src/Core/Resolvers/DWSqlQueryBuilder.cs
- Method: Build(SqlInsertStructure structure)
- Behavior mirrors MSSQL policy application.

### 3) Runtime validation gap by mode
- File: src/Core/Configurations/RuntimeConfigValidator.cs
- Gate exists:
  - _databaseTypesSupportingCreatePolicy = MSSQL, DWSQL
  - Create DB policy for PostgreSQL/MySQL deemed invalid.
- But this permission validation is invoked from development-mode-only validation path during metadata initialization:
  - src/Core/Services/MetadataProviders/SqlMetadataProvider.cs
  - ValidateEntityAndAutoentityConfigurations(runtimeConfig)
  - which calls ValidatePermissionsInConfig(...) only if runtimeConfig.IsDevelopmentMode() is true.

### 4) Production default makes bypass realistic
- File: src/Config/ObjectModel/HostOptions.cs
- Default host mode is Production.
- Therefore, in default/prod scenarios, the create-policy validation block can be skipped while policy strings may still be present in runtime config.

### 5) Policy still flows at runtime
- File: src/Core/Authorization/AuthorizationResolver.cs
- Database policy text is stored and retrievable per role/operation.
- File: src/Core/Resolvers/AuthorizationPolicyHelpers.cs
- ProcessAuthorizationPolicies can process create policy into SQL-side predicate structures.
- Net: policy can exist and be parsed, but PostgreSQL insert builder does not enforce it.

### 6) Additional technical blocker for robust PostgreSQL implementation
- DbType metadata is created in query structures via MakeDbConnectionParam(...).
- Generic QueryExecutor path does not apply DbType for non-MSSQL providers.
- File: src/Core/Resolvers/QueryExecutor.cs
  - PrepareDbCommand sets parameter values but generic PopulateDbTypeForParameter is a no-op.
- MSSQL has special handling in MsSqlQueryExecutor; PostgreSQL currently does not.
- This likely contributes to the NULL/type inference issue referenced in #1334 when using VALUES/SELECT policy rewrites.

## What This Means
To fully resolve PostgreSQL create-policy support safely:
1. Add create-policy enforcement in PostgreSQL insert query generation.
2. Ensure PostgreSQL parameter typing is stable (especially NULL handling).
3. Update validation semantics so PostgreSQL create policy is no longer rejected once enforcement is implemented.
4. Add regression tests for security behavior and type edge cases.

## Proposed Implementation Plan

### Phase 1: PostgreSQL SQL enforcement (builder)
Owner: Query builder implementer

- Update src/Core/Resolvers/PostgresQueryBuilder.cs Build(SqlInsertStructure).
- Pattern target (analogous to MSSQL logic):
  - When create policy resolves to BASE_PREDICATE: keep simple insert path.
  - Otherwise, rewrite insert to use SELECT FROM (VALUES(...)) WHERE <createPolicy>.
- Preserve current RETURNING behavior and default-values behavior.

Deliverable:
- Query text contains create-policy WHERE for PostgreSQL inserts.

### Phase 2: PostgreSQL parameter typing for policy rewrite safety
Owner: Query executor implementer

- Evaluate where to apply db type for Npgsql parameters.
- Candidate approaches:
  1. Override parameter population in PostgreSQL executor.
  2. Improve generic executor path to set DbType when provided and safe.
- Validate NULL typed parameters in insert policy scenarios.

Deliverable:
- No type mismatch errors for known nullable integer/text/date cases under policy rewrite.

### Phase 3: Validation rule alignment
Owner: Config/validation implementer

- Update _databaseTypesSupportingCreatePolicy to include PostgreSQL (and MySQL if simultaneously supported).
- Adjust/replace unit tests that currently expect create-policy rejection for PostgreSQL.

Deliverable:
- Validation accepts create DB policy for supported engines after runtime enforcement exists.

### Phase 4: Test coverage
Owner: Test implementer

Add/adjust tests across:
- REST insert create-policy pass/fail (PostgreSQL)
- GraphQL create mutation create-policy pass/fail (PostgreSQL)
- Negative test for cross-user injection prevention
- NULL typing scenarios for rewritten insert queries

Likely files to touch:
- src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs
- src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs
- src/Service.Tests/UnitTests/ConfigValidationUnitTests.cs
- possibly policy-focused test bases under Authorization/GraphQL/Policies.

## Acceptance Criteria
- PostgreSQL create requests that violate create DB policy fail with DatabasePolicyFailure behavior.
- PostgreSQL create requests that satisfy policy succeed.
- No silent cross-user row insertion in policy-protected scenarios.
- No NULL type inference regressions in policy-rewritten PostgreSQL inserts.
- Config validation allows PostgreSQL create-policy only when runtime support is in place.

## Risks and Pitfalls
- Policy rewrite SQL may break default-values insert path if not handled separately.
- NULL parameter typing can still fail if only SQL text is changed without executor typing work.
- Validation changes without runtime enforcement would increase exposure.
- Upsert and multiple-create paths may have adjacent behavior requiring explicit checks.

## Suggested Task Breakdown for Swarm

### Agent A: Builder changes
- Implement PostgreSQL create-policy SQL rewrite in PostgresQueryBuilder.
- Add unit-style query generation assertions if available.

### Agent B: Parameter typing
- Add PostgreSQL parameter type propagation.
- Verify typed NULL behavior with targeted tests.

### Agent C: Validation and config tests
- Update RuntimeConfigValidator support list and related tests.

### Agent D: Integration/security tests
- Add REST/GraphQL end-to-end tests covering policy pass/fail and cross-user injection prevention.

### Agent E: Final integration review
- Run relevant test subsets and capture before/after behavior.
- Verify no regressions in MSSQL/DWSQL existing create-policy logic.

## Quick Start Commands (for assignee)
- Build:
  - dotnet build src/Azure.DataApiBuilder.sln
- Run focused tests by category:
  - dotnet test --filter "TestCategory=PostgreSql"
- Run format check before PR:
  - dotnet format src/Azure.DataApiBuilder.sln --verify-no-changes

## Current Status
Investigation and correlation complete.
No code changes applied yet for runtime behavior.
This handoff is ready for implementation delegation.
