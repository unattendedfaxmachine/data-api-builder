# PostgreSQL Create Policy Enforcement Implementation Plan

## Overview
This plan implements runtime create-policy enforcement for PostgreSQL create operations so policy-violating inserts are blocked and policy-compliant inserts succeed consistently across REST and GraphQL. The approach aligns PostgreSQL create behavior with existing SQL-engine policy enforcement patterns while preserving default-value and authorized create behavior.

## Current State Analysis
- PostgreSQL create query generation currently emits direct `INSERT ... VALUES|DEFAULT VALUES ... RETURNING` and does not apply create DB policy.
- MSSQL/DWSQL already implement create-policy enforcement by rewriting insert query shape when policy is non-baseline.
- Authorization-policy plumbing for create is already available to query structures/builders.
- Generic query execution path does not populate parameter DbType; MSSQL has provider-specific type population override while PostgreSQL currently does not.
- Config validation currently rejects PostgreSQL create-action DB policy and applies permission validation in development-mode-gated path.
- Existing PostgreSQL create-policy tests are partially ignored/not implemented, creating regression coverage gaps.

## Desired End State
- PostgreSQL create operations enforce configured create-action DB policy at runtime.
- Policy-fail creates return standard database-policy failure behavior and persist no unauthorized row, including explicit empty-result-to-policy-failure mapping parity where required.
- Policy-pass creates succeed for REST and GraphQL with behavior parity.
- Null/type-sensitive payloads in authorized creates are handled without newly introduced runtime type-inference failures.
- Validation semantics for PostgreSQL create-policy align with implemented runtime support and are enforced consistently across development and production host modes.
- Regression coverage is in place for policy pass/fail, missing policy-fields in body, and representative null/type-sensitive inputs.

## What We're NOT Doing
- Introducing new policy language or policy-expression features.
- Changing read/update/delete authorization semantics.
- Refactoring unrelated SQL engines beyond targeted parity alignment.
- Broad re-architecture of mutation execution or authorization subsystems.

## Phase Status
- [x] **Phase 1: PostgreSQL Create Policy SQL Enforcement** - Apply create-policy predicate in PostgreSQL insert query generation.
- [x] **Phase 2: PostgreSQL Parameter Typing Stability** - Improve PostgreSQL parameter typing behavior for null/type-sensitive create-policy flows.
- [x] **Phase 3: Validation Semantics Alignment** - Align create-policy validation rules with PostgreSQL runtime support.
- [x] **Phase 4: PostgreSQL Policy Regression Coverage** - Add/enable REST and GraphQL tests for create-policy pass/fail and edge cases.
- [x] **Phase 5: Documentation** - Produce Docs.md and update project docs if warranted.

## Phase Candidates
- [x] Additional stress/perf coverage for multi-create or linking-specific create-policy scenarios beyond baseline functional verification. (Deferred to follow-up; not required for baseline acceptance.)
- [x] Broader host-mode validation behavior clarification tests if ambiguity remains after Phase 3. (Resolved by host-mode parity coverage in config validation tests.)

---

## Phase 1: PostgreSQL Create Policy SQL Enforcement

### Changes Required:
- **src/Core/Resolvers/PostgresQueryBuilder.cs**: Update insert build path to retrieve create-operation DB policy and apply enforcement query shape equivalent to existing SQL-engine policy pattern for non-baseline policies while preserving baseline path.
- **src/Core/Resolvers/PostgresQueryBuilder.cs**: Ensure default-values insert path remains valid and policy-aware where applicable.
- **src/Core/Resolvers/PostgresQueryBuilder.cs**: Preserve expected `RETURNING` behavior for create responses.
- **src/Core/Resolvers/SqlMutationEngine.cs**: Verify create-path empty-result handling for PostgreSQL policy-fail outcomes and align behavior to standard `DatabasePolicyFailure` semantics in FR-002.
- **src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs**: Enable or add PostgreSQL REST create-policy pass/fail coverage tied to insert enforcement behavior.
- **src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs**: Enable or add PostgreSQL GraphQL create-policy pass/fail coverage tied to create enforcement behavior.

### Evidence Mapping:
- PostgreSQL create path currently lacks create-policy predicate integration: `CodeResearch.md` findings citing `src/Core/Resolvers/PostgresQueryBuilder.cs:68-81`.
- Existing enforcement pattern for parity: `CodeResearch.md` findings citing `src/Core/Resolvers/MsSqlQueryBuilder.cs:81-90` and `src/Core/Resolvers/DWSqlQueryBuilder.cs:388-396`.
- Create-path policy-failure handling is implemented in mutation engine create flows: `src/Core/Resolvers/SqlMutationEngine.cs:1032` and `src/Core/Resolvers/SqlMutationEngine.cs:1526`.

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `dotnet test --filter "TestCategory=PostgreSql"`
- [ ] Build passes: `dotnet build src/Azure.DataApiBuilder.sln`
- [ ] Targeted tests/assertions validate that policy-fail creates surface standard `DatabasePolicyFailure` behavior for PostgreSQL.

#### Manual Verification:
- [ ] Policy-fail PostgreSQL create attempts are rejected with expected policy-failure behavior.
- [ ] Policy-pass PostgreSQL create attempts succeed and return expected payload shape.
- [ ] Default-values create path remains functional under policy-enforced flow.

---

## Phase 2: PostgreSQL Parameter Typing Stability

### Changes Required:
- **Validation-first scope**: Add a targeted validation step to confirm whether nullable/type-sensitive failures reproduce for PostgreSQL under create-policy rewrite query shape.
- **src/Core/Resolvers/PostgreSqlExecutor.cs** (preferred) and only if required, **src/Core/Resolvers/QueryExecutor.cs**: Introduce PostgreSQL-safe parameter type population behavior where metadata is available, with emphasis on null/type-sensitive values used in create-policy-enforced inserts.
- **src/Core/Models/DbConnectionParam.cs** and related usage sites (if needed): Preserve existing metadata contract while ensuring provider-specific consumption is reliable.
- **src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs**: Add targeted REST create tests with nullable/type-sensitive fields under policy-enforced create path.
- **src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs**: Add targeted GraphQL create tests with nullable/type-sensitive fields under policy-enforced create path.

### Evidence Mapping:
- Parameter metadata is carried but generic executor type population is a no-op: `CodeResearch.md` findings citing `src/Core/Resolvers/QueryExecutor.cs:424` and `src/Core/Resolvers/BaseQueryStructure.cs:119-126`.
- PostgreSQL executor currently lacks provider-specific type-population override in researched path: `CodeResearch.md` findings citing `src/Core/Resolvers/PostgreSqlExecutor.cs:22`.

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `dotnet test --filter "TestCategory=PostgreSql"`
- [ ] Format check passes: `dotnet format src/Azure.DataApiBuilder.sln --verify-no-changes`
- [ ] Validation artifacts/tests document whether null/type-sensitive failure is reproducible pre-fix and resolved post-fix (or not reproducible, with no implementation change).

#### Manual Verification:
- [ ] Authorized PostgreSQL creates with nullable numeric/text/date fields complete without newly introduced type-inference failures.
- [ ] No regressions observed in existing PostgreSQL mutation scenarios unrelated to create-policy enforcement.

---

## Phase 3: Validation Semantics Alignment

### Changes Required:
- **src/Core/Configurations/RuntimeConfigValidator.cs**: Update supported-database gating for create-action DB policy to include PostgreSQL once runtime support is in place.
- **Decision**: Enforce create-policy validation semantics consistently across development and production host modes for supported PostgreSQL create-policy scenarios.
- **src/Core/Configurations/RuntimeConfigValidator.cs** and initialization call paths: Ensure action-specific database policy validation logic aligns with intended PostgreSQL support behavior without mode-dependent acceptance ambiguity.
- **src/Service.Tests/UnitTests/ConfigValidationUnitTests.cs**: Update/add tests for accepted/rejected create-policy configs across supported/unsupported engines.
- **Host-mode verification matrix (development and production)**:
- **src/Service.Tests/UnitTests/ConfigValidationUnitTests.cs** or equivalent configuration-validation test surface: Add explicit tests that validate behavior under development mode and production mode for PostgreSQL create-policy configs.
- **src/Service.Tests/** integration test surface used for metadata initialization checks: Add or update targeted tests to verify that mode-gated validation behavior does not violate FR-007/SC-006 intent.

### Evidence Mapping:
- Create-policy support list and create-action validation constraints: `CodeResearch.md` findings citing `src/Core/Configurations/RuntimeConfigValidator.cs:47` and `src/Core/Configurations/RuntimeConfigValidator.cs:1338-1383`.
- Development-mode gating and production default risk: `CodeResearch.md` findings citing `src/Core/Configurations/RuntimeConfigValidator.cs:1918-1928`, `src/Core/Services/MetadataProviders/SqlMetadataProvider.cs:346`, and `src/Config/ObjectModel/HostOptions.cs:38`.

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `dotnet test src/Service.Tests/Service.Tests.csproj --filter "ConfigValidation|TestCategory=PostgreSql"`
- [ ] Build passes: `dotnet build src/Azure.DataApiBuilder.sln`

#### Manual Verification:
- [ ] PostgreSQL create-policy configs are accepted when runtime support is enabled by this feature.
- [ ] Unsupported create-policy combinations continue to fail validation with clear error behavior.
- [ ] Development-mode and production-mode behavior is explicitly verified against FR-007/SC-006/FR-008 expectations with no unresolved mode-dependent ambiguity.

---

## Phase 4: PostgreSQL Policy Regression Coverage

### Changes Required:
- **src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs**: Implement/enable create-policy-related tests currently ignored or unimplemented.
- **src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs**: Implement/enable create-policy-related GraphQL mutation tests currently ignored or unimplemented.
- **src/Service.Tests/SqlTests/RestApiTests/Insert/InsertApiTestBase.cs** and **src/Service.Tests/SqlTests/GraphQLMutationTests/GraphQLMutationTestBase.cs**: Reuse existing policy pass/fail helpers and extend only if needed for PostgreSQL-specific edge cases.
- **src/Service.Tests/SqlTests/RestApiTests/Insert/InsertApiTestBase.cs**: Reuse/extend base assertions for PostgreSQL cross-user unauthorized create prevention and missing policy-field handling.
- **src/Service.Tests/SqlTests/GraphQLMutationTests/GraphQLMutationTestBase.cs**: Reuse/extend base assertions for PostgreSQL GraphQL create policy pass/fail parity.
- **Multi-create/linking support determination**: Add explicit verification task for whether PostgreSQL multi-create/linking create paths are supported and affected by create-policy enforcement, and add baseline functional checks for supported paths.

### Evidence Mapping:
- PostgreSQL create-policy tests currently ignored/unimplemented: `CodeResearch.md` findings citing `src/Service.Tests/SqlTests/RestApiTests/Insert/PostgreSqlInsertApiTests.cs:344-351` and `src/Service.Tests/SqlTests/GraphQLMutationTests/PostgreSqlGraphQLMutationTests.cs:710`.
- Existing reusable pass/fail expectations in base suites: `CodeResearch.md` findings citing `src/Service.Tests/SqlTests/RestApiTests/Insert/InsertApiTestBase.cs:847-894` and `src/Service.Tests/SqlTests/GraphQLMutationTests/GraphQLMutationTestBase.cs:111-134`.

### Success Criteria:

#### Automated Verification:
- [ ] Tests pass: `dotnet test --filter "TestCategory=PostgreSql"`
- [ ] Targeted REST/GraphQL create-policy suites pass without ignored critical scenarios.
- [ ] Automated assertions verify policy-fail scenarios persist zero unauthorized rows (SC-001) for covered REST/GraphQL PostgreSQL create-policy tests.
- [ ] Automated checks document multi-create/linking support outcome (supported with tests, or explicitly documented as not supported).

#### Manual Verification:
- [ ] Cross-user unauthorized create scenario is reproducibly blocked.
- [ ] REST and GraphQL produce consistent pass/fail policy outcomes for equivalent create scenarios.

### Phase Entry/Exit Clarification

- **Phase 1 exit**: Runtime enforcement path and standard policy-failure mapping behavior are functionally validated.
- **Phase 4 exit**: Regression matrix is complete (REST/GraphQL parity, zero-persistence assertions, and multi-create/linking support determination).

---

## Phase 5: Documentation

### Changes Required:
- **.paw/work/pg-create-policy/Docs.md**: Document technical implementation decisions, behavior changes, verification strategy, and known constraints (load `paw-docs-guidance` during implementation).
- **docs/**: Update user-facing/internal docs if behavior-visible changes are confirmed for operators/users.
- **README.md or release-note surfaces**: Update when either of these triggers is true: (a) PostgreSQL create-policy support status changes from unsupported to supported, or (b) externally observable create-policy behavior/validation semantics change.

### Evidence Mapping:
- Documentation-system and repository-doc conventions captured in `CodeResearch.md` documentation section (docs/readme and contribution references).

### Success Criteria:

#### Automated Verification:
- [ ] Docs/style consistency check performed per repository conventions.
- [ ] Format check passes: `dotnet format src/Azure.DataApiBuilder.sln --verify-no-changes`

#### Manual Verification:
- [ ] Docs.md accurately reflects implemented behavior and test evidence.
- [ ] Any updated project docs are consistent with existing style and avoid undocumented behavior claims.

---

## References
- Issue: https://github.com/Azure/data-api-builder/issues/1334
- Spec: .paw/work/pg-create-policy/Spec.md
- Research: .paw/work/pg-create-policy/CodeResearch.md
