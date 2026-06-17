# Feature Specification: PostgreSQL Create Policy Enforcement

**Branch**: feature/pg_create_policy  |  **Created**: 2026-06-17  |  **Status**: Draft
**Input Brief**: Enforce create-action database policies for PostgreSQL inserts to prevent cross-user writes while preserving expected create behavior.

## Overview
PostgreSQL create operations currently allow writes even when a create-action database policy should block them. This creates a security and data-integrity gap where policy-protected resources can be written by unauthorized users under specific request patterns. The behavior diverges from other SQL engines where create policy is enforced as part of insert query execution.

The goal is to close this gap by making PostgreSQL create behavior consistent with policy expectations: requests that satisfy create policy are accepted, and requests that violate create policy are rejected with the standard database-policy failure behavior. This must work for the existing API surfaces where create operations are exposed, and it must preserve normal successful create flows.

Because create-policy enforcement can involve query rewriting patterns, correctness includes both authorization outcome and runtime reliability. The system must avoid introducing new failures in valid requests, especially for payloads that include nullable values or type-sensitive fields. The end result should be a secure, predictable create path for PostgreSQL that aligns with operator intent and existing policy semantics.

## Objectives
- Ensure PostgreSQL create operations enforce configured create-action database policies at runtime.
- Prevent unauthorized cross-user row creation in policy-protected entities.
- Maintain expected successful behavior for authorized creates.
- Keep behavior consistent across REST and GraphQL create entry points.
- Avoid regressions in create handling for nullable and type-sensitive inputs.

## User Scenarios & Testing
### User Story P1 - Enforce policy-protected creates
Narrative: As an API operator, I want create-action database policies to be enforced for PostgreSQL so unauthorized users cannot insert protected rows.
Independent Test: Submit a create request that violates policy and verify it fails with policy-failure behavior.
Acceptance Scenarios:
1. Given a PostgreSQL entity with a create-action database policy and a caller that does not satisfy the policy, when the caller submits a create request, then the request is rejected with policy-failure behavior and no row is inserted.
2. Given the same policy and a caller that satisfies the policy, when the caller submits a create request, then the request succeeds and the row is inserted.

### User Story P1 - Consistent API-surface behavior
Narrative: As a client developer, I want create-policy enforcement to behave consistently across API surfaces so security behavior does not depend on protocol choice.
Independent Test: Execute equivalent policy pass/fail create operations through REST and GraphQL and confirm matching outcomes.
Acceptance Scenarios:
1. Given equivalent create operations, when executed through REST and GraphQL, then policy pass/fail outcomes are consistent.

### User Story P2 - Reliable handling of nullable payloads
Narrative: As a service owner, I want policy-enforced creates to handle nullable/type-sensitive inputs reliably so valid requests do not fail unexpectedly.
Independent Test: Submit authorized create requests with nullable fields and verify expected success without type-inference failures.
Acceptance Scenarios:
1. Given authorized create requests containing nullable fields, when requests are processed, then operations complete without runtime type-inference errors.

### Edge Cases
- Create request uses default-value behavior (no explicit non-key fields provided).
- Policy expression evaluates to always-allow baseline behavior.
- Payload includes null values for optional numeric/text/date fields.
- Multi-row create variants (if supported) preserve policy outcome semantics for each inserted row.

## Requirements
### Functional Requirements
- FR-001: The system SHALL enforce create-action database policy checks for PostgreSQL create operations before insert completion. (Stories: P1)
- FR-002: The system SHALL reject PostgreSQL create requests that violate create-action database policy with standard database-policy failure behavior and SHALL NOT persist unauthorized rows. (Stories: P1)
- FR-003: The system SHALL allow PostgreSQL create requests that satisfy create-action database policy to complete successfully. (Stories: P1)
- FR-004: The system SHALL provide equivalent create-policy pass/fail behavior for PostgreSQL across REST and GraphQL create flows. (Stories: P1)
- FR-005: The system SHALL preserve expected create behavior for default-value inserts and policy baseline-allow cases. (Stories: P1)
- FR-006: The system SHALL process authorized PostgreSQL creates with nullable/type-sensitive inputs without introducing new runtime type-inference failures. (Stories: P2)
- FR-007: Configuration validation for create-action database policy SHALL align with supported PostgreSQL runtime behavior. (Stories: P1)

### Key Entities
- Create-action database policy: Authorization predicate that governs whether a create operation is allowed.
- Policy-protected entity: Entity configured with a create-action database policy.
- Create request context: Caller identity/role and request payload used to evaluate authorization outcome.

### Cross-Cutting / Non-Functional
- Security: Unauthorized create attempts must be blocked without partial persistence.
- Reliability: Policy-enforced create flow must not reduce stability for valid requests.
- Consistency: Authorization outcomes must be protocol-agnostic across supported create entry points.

## Success Criteria
- SC-001: In policy-fail test scenarios, 100% of PostgreSQL unauthorized create attempts are rejected and persist zero rows. (FR-001, FR-002)
- SC-002: In policy-pass test scenarios, 100% of authorized PostgreSQL create attempts succeed and persist expected rows. (FR-003)
- SC-003: REST and GraphQL produce matching policy pass/fail outcomes for equivalent PostgreSQL create scenarios. (FR-004)
- SC-004: Default-value and baseline-allow create scenarios continue to pass with no behavioral regression. (FR-005)
- SC-005: Nullable/type-sensitive authorized create scenarios complete without newly introduced runtime type-inference failures. (FR-006)
- SC-006: Validation behavior accepts supported PostgreSQL create-policy configurations and rejects unsupported combinations consistently with runtime support guarantees. (FR-007)

## Assumptions
- Existing tests and runtime patterns already define standard database-policy failure behavior.
- PostgreSQL create-policy support is intended for the same deployment modes where PostgreSQL create operations are currently supported.
- Existing entity permission modeling remains unchanged; this work updates enforcement consistency rather than policy syntax design.

## Scope
In Scope:
- PostgreSQL runtime enforcement of create-action database policy during create operations.
- Behavior consistency verification across REST and GraphQL create paths.
- Handling of nullable/type-sensitive create inputs required by policy-enforced flow.
- Validation-rule alignment with supported PostgreSQL create-policy runtime behavior.
- Regression coverage for policy pass/fail and unauthorized row-insertion prevention.

Out of Scope:
- New policy language/features beyond existing create-action database policy semantics.
- Redesign of non-PostgreSQL engine policy behavior.
- Broad authorization model changes outside create-operation enforcement.
- Non-create operations (read/update/delete) behavior changes.

## Dependencies
- Existing authorization policy evaluation semantics and failure mapping.
- PostgreSQL create query-generation and execution paths.
- Existing integration/unit test infrastructure for PostgreSQL, REST, GraphQL, and config validation.

## Risks & Mitigations
- Risk: Enforcement implementation introduces regressions in valid create flows. Mitigation: Add explicit authorized create and default-value regression tests.
- Risk: Nullable/type-sensitive payload handling causes runtime failures under enforcement path. Mitigation: Add targeted nullable/type-sensitive tests and verify stable parameter typing behavior.
- Risk: Validation changes diverge from runtime capability. Mitigation: Couple validation updates with runtime support and include focused validation tests.
- Risk: Security gap remains in adjacent create variants. Mitigation: Include policy pass/fail coverage across supported create surfaces and representative edge cases.

## References
- Investigation Summary: PostgreSQL Create DB Policy Investigation and Handoff Summary
- GitHub Issue: https://github.com/Azure/data-api-builder/issues/1334
- Azure DevOps Work Item: 2141216
