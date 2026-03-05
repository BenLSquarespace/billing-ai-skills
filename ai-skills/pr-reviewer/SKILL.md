# Billing PR Reviewer Skill

## Purpose

Guidelines for reviewing pull requests in billing team repositories, based on patterns and standards distilled from **12 months** of billing team code reviews (March 2025 – March 2026) across `billing-invoice-service`, `v6-billing-facade`, and `squarespace-v6`.

**Data Sources:**
- ~13,000 lines of review comments from 15 billing team members
- 3 repositories over 12 months
- Patterns validated across two independent 6-month windows for consistency

## Team Context

**Team:** `@sqsp/billing`

| Repository | Description |
|------------|-------------|
| `sqsp/billing-invoice-service` | Billing/invoice microservice (Ripley) |
| `sqsp/v6-billing-facade` | Billing facade layer between V6 and Ripley |
| `sqsp/squarespace-v6` | Main monolith — billing owns `business/.../billing/`, `site-server/.../billing/`, promotions, discounts, and related paths |

## Prerequisites

- GitHub CLI (`gh`) authenticated
- Access to `sqsp` GitHub organization
- Familiarity with billing domain concepts: subscriptions, contracts, invoices, discounts, promotions, fees

## Review Process

### 1. Understand the Change

1. Read the PR title and description — look for linked JIRA tickets (`BILL-XXXXX`)
2. Understand the scope: is this a new feature, refactor, bug fix, or config change?
3. Check if the PR spans multiple repos (facade + Ripley, or v6 + facade)
4. Review the diff with domain context

```bash
# View PR details
gh pr view PR_NUMBER -R sqsp/REPO_NAME

# View the diff
gh pr diff PR_NUMBER -R sqsp/REPO_NAME

# Check changed files
gh pr view PR_NUMBER -R sqsp/REPO_NAME --json files | jq '.files[].path'
```

### 2. Apply the Review Checklist

Run through each category below when reviewing billing PRs. Not all categories apply to every PR — focus on what's relevant to the change.

---

## Review Checklist

### ✅ Version & Dependency Management

This is the most commonly flagged issue across all billing repos. Check every PR for:

- [ ] **No SNAPSHOT versions remain**: `.drone.yml`, `gradle.properties`, `build.gradle`, and `site-server/build.gradle` should not contain snapshot or branch-specific dependency versions at merge time
- [ ] **Release versions are used**: Dependencies on `billing-service-client`, `billing-events`, `v6-billing-facade`, and `service-facade-clients` must reference release versions
- [ ] **Cross-repo version chain is tracked**: If changes span repos, verify the dependency release order (e.g., Ripley client → facade version bump → v6 version bump)
- [ ] **Use `- [x]` checkbox comments** to track version updates that must happen before merge
- [ ] **Post-release versioning**: After release, `gradle.properties` must be updated to `version+1-SNAPSHOT`

### ✅ Database & Migration Safety (billing-invoice-service)

- [ ] **Assess migration risk**: Does the migration acquire exclusive locks? How large is the table? Consider `ALTER TABLE` with `NOT NULL`, adding columns, or index creation
- [ ] **Use Change Management (CM) for risky migrations**: Large or locking schema changes should go through CM rather than automated Flyway migrations. For large tables, CM first then Flyway
- [ ] **Split risky operations**: Separate non-locking DDL from locking DDL into different migration files
- [ ] **Batch large data operations**: Large `DELETE` or `UPDATE` operations should use date ranges or batch sizes to avoid long-running locks
- [ ] **Validate query performance**: New queries should have evidence of performance in production-like environments (links to dashboards, `EXPLAIN ANALYZE`, Datagrip testing)
- [ ] **Primary key requirements**: Every table must have a primary key (CockroachDB migration: BILL-9187). Unique indexes being promoted to PKs
- [ ] **Non-blocking index creation**: Use `CREATE UNIQUE INDEX CONCURRENTLY` to avoid locking
- [ ] **Deferred constraints**: Prefer `ALTER CONSTRAINT ... NOT DEFERRABLE` over drop-and-recreate when removing deferred foreign key constraints
- [ ] **UUID generation performance**: Benchmark `gen_random_uuid()` for new tables in QA to verify no performance impact

### ✅ Naming, Clarity & Code Style

The billing team values explicit, readable code:

- [ ] **Explicit types over `var`**: Prefer explicit Java types; `var` is acceptable only when the type is obvious from context, but generally discouraged in the codebase
- [ ] **Descriptive names**: Variable, method, and class names should clearly convey purpose. Response objects use `Response` suffix. Avoid abbreviations that create ambiguity
- [ ] **Constants over magic values**: Extract hardcoded strings and numbers into `private static final` constants
- [ ] **`Optional` return types, `@Nullable` parameters**: Prefer returning `Optional` from getters but passing `@Nullable` parameters (not `Optional` params — IntelliJ warns about this pattern)
- [ ] **`Collections.emptyList()` over `new ArrayList<>()`**: Use explicit empty collection constructors when the intent is to return empty
- [ ] **`entrySet()` over double map lookups**: When iterating maps, use `entrySet()` instead of getting keys and looking up values separately
- [ ] **Avoid single-use helper methods**: Inline methods used only once unless they significantly improve readability
- [ ] **Use `Function.identity()`**: Prefer over `x -> x` lambdas
- [ ] **Flatten nested conditionals**: Prefer early returns and guard clauses over deep `if` nesting
- [ ] **Simplify stream operations**: Use idiomatic stream constructs (e.g., `.map(BillingRequest::getId).collect(Collectors.toList())`) over manual iteration
- [ ] **Comments match implementation**: When cron schedules, return types, or behavior changes, update associated comments
- [ ] **Remove debug logs before merge**: Any temporary debug logging should be cleaned up. Track with `- [x] logs will be removed before merge`

### ✅ API & Model Design

- [ ] **Nullable field annotations**: Review whether `@Nullable` is appropriate — prefer non-null fields when business logic guarantees values will be present
- [ ] **Javadoc on public APIs**: New public methods, class headers, and constructor parameters should have Javadoc — especially for nullable fields
- [ ] **Stale Javadoc**: When method signatures change (return type, parameters), verify Javadoc is updated to match
- [ ] **Method signatures prevent invalid states**: Design APIs that make it impossible to pass conflicting arguments
- [ ] **Breaking changes**: Verify all downstream consumers before changing facade method signatures — "We generally shouldn't be making breaking changes without warning clients"
- [ ] **Prefer interfaces over abstract classes**: When an abstract class only defines method signatures (no shared behavior), use an interface instead (except when shared state is needed, like `Promotion` hierarchy)
- [ ] **Enum value naming conventions**: Product-specific enum values should be prefixed (e.g., `DOMAIN_` for domain-specific `ValidationFailureReason` values)
- [ ] **Enum propagation to frontend**: When adding new enum values in v6, propagate to frontend TypeScript files using the documented enum release process
- [ ] **Boolean flags insufficient for >2 options**: When a boolean (e.g., `isPercentBased`) no longer covers all cases, expose the underlying enum value directly
- [ ] **Reduce data mutability**: Remove setters if not strictly needed. Prefer immutable data. "Remove them and if they're needed add them back in"
- [ ] **Avoid duplicate data structures**: When a map keyset serves the same purpose as a separate list, consolidate
- [ ] **Static utility methods over instance construction**: Prefer static methods when no instance state is needed

### ✅ Business Logic Correctness

- [ ] **Discount/pricing calculation accuracy**: Verify the math is correct, especially for proportional calculations. A common error: `(planPrice - promoPrice) * termRatio` is wrong; it should be `(planPrice * termRatio) - promoPrice`
- [ ] **Discount stacking rules**: Understand how candidate, code, and bundle discounts interact — verify precedence and stacking behavior
- [ ] **Feature flag check ordering**: Check cheaper conditions (plan type, null checks) before making external service calls (statsig, namespace client)
- [ ] **Edge case handling**: Consider null/absent fields, multi-currency scenarios, existing vs. new customers, and concurrent operations
- [ ] **Lifecycle awareness**: Consider how changes affect the full subscription lifecycle: creation → renewal → plan change → cancellation → refund
- [ ] **Domain descriptions accuracy**: Verify that comments and Javadoc accurately describe the business behavior
- [ ] **Reversed or swapped logic**: Watch for implementations where "consume" and "rollback" (or similar paired operations) are accidentally swapped
- [ ] **Contract state machine awareness**: Understand PENDING, ACTIVE, CANCELLED, PAST_DUE, EXPIRED transitions and ensure handlers process events correctly for each state
- [ ] **Multi-product cart (MPC) scenarios**: Consider how changes behave when multiple products are purchased together — race conditions, per-contract event processing, fulfillment ordering
- [ ] **Payment flow edge cases**: SCA/3DS, SEPA, e-mandates — billing request states (`REQUIRES_ACTION`, `AWAITING_ASYNC_PAYMENT`) and cancellation flows differ by payment method

### ✅ Error Handling & Resilience

- [ ] **Fail-fast with exceptions**: Prefer throwing exceptions for unexpected states rather than silently returning — especially in non-critical-path consumers (e.g., email, analytics)
- [ ] **Early returns over deep nesting**: Prefer early return patterns for readability
- [ ] **Validate inputs explicitly**: Throw on invalid inputs rather than silently proceeding with nulls
- [ ] **NPE safety**: Flag chained method calls on nullable objects (`.get(0)`, `.getDomainName()` on a possibly null reference)
- [ ] **Log levels matter**: 4xx responses → not errors (client issues); 5xx → warnings or errors. Bad client data → `warn`, not `error`. Don't log info where warn is appropriate for rare-but-important paths
- [ ] **Combine error log and exception**: Include identifiers in exception messages rather than logging separately before throwing
- [ ] **Try/catch placement**: Be careful about what goes inside `try` blocks — if email send succeeds but notification fails, should the flag be reset?
- [ ] **Avoid `AtomicBoolean` in lambdas**: Refactor streams to for-loops or collect intermediate results instead of using `AtomicBoolean` workarounds
- [ ] **Billing request failure reasons**: Must be generic (e.g., `VALIDATION_FAILURE`), not product-specific. Product-specific failures go in separate enums with appropriate prefixes

### ✅ Testing & QA Requirements

- [ ] **QA testing evidence in PR description**: Reviewers expect screenshots, testing notes, and links to QA validation — especially for billing flows
- [ ] **Don't delete tests during refactors**: When refactoring, update method calls in existing tests to the new API rather than removing tests
- [ ] **Test both positive and negative cases**: Include failure scenarios, not just happy paths
- [ ] **Remove commented-out test code**: Don't leave commented tests in the codebase
- [ ] **Integration test for complex flows**: Complex billing flows need both unit tests and integration/QA validation
- [ ] **Test coverage praise**: The team values thorough test coverage — call it out when it's done well ("your testing is awesome, as always!", "Nice tests!")
- [ ] **Performance validation under prod conditions**: For query changes, test manually in Datagrip against prod data and link monitoring dashboards
- [ ] **Detailed test plans in PR descriptions**: Steps to reproduce, QA validation, and prod monitoring plans are all valued
- [ ] **Algorithm complexity consideration**: For algorithms operating on variable-size inputs, consider runtime complexity. "What's the runtime complexity? We need to make sure it holds up for n<100ish"

### ✅ Observability & Monitoring

- [ ] **Log appropriate identifiers**: Include `billingRequestId`, `websiteId`, `contractId` in log messages — not objects without meaningful identifiers
- [ ] **Don't log user-generated values**: Security concern — avoid logging coupon codes, user inputs, or cookies directly
- [ ] **Remove debug logs before merge**: Any temporary debug logging should be cleaned up
- [ ] **Monitoring plan for new features**: Ask how changes will be monitored once deployed — dashboards, alerts, runbooks
- [ ] **Alert configuration**: Alerts should have correct thresholds, proper group names, linked runbooks, and appropriate stage vs. production separation
- [ ] **Log a single consolidated message**: Rather than logging one message per field, consolidate into a single structured log entry
- [ ] **Log warnings for unexpected scenarios**: When encountering edge cases that shouldn't happen, prefer logging a warning over silently proceeding
- [ ] **Include SQL for Hibernate queries**: When writing JPA queries, include the generated SQL for reviewer understanding

### ✅ Documentation & Comments

- [ ] **Avoid time-specific comments**: Don't write "as of today..." or "currently..." — these go stale. Use git history for that context
- [ ] **TODO comments with JIRA tickets**: Always pair `TODO` comments with a JIRA ticket number for tracking (e.g., `// TODO [BILL-XXXXX] clean up feature flag`)
- [ ] **Create cleanup tickets**: When deferring work, create a JIRA ticket and reference it in code
- [ ] **Explain non-obvious design decisions**: When doing something unusual, explain why in comments
- [ ] **PR descriptions with context**: Include context so future reviewers don't need to trace through chains of previous PRs
- [ ] **Javadoc on methods that throw**: Especially for methods that throw on certain conditions, document the behavior so callers handle appropriately
- [ ] **Scope reduction**: When PRs grow too large, explicitly call out scope reduction and create follow-up tickets. "I don't want to add scope to this PR, can we keep it in mind for upcoming changes?"

### ✅ Deployment & Cross-Service Coordination

- [ ] **Deployment order**: When changes span multiple services, verify deployment order and backward compatibility (e.g., deploy Ripley before facade, facade before v6)
- [ ] **Feature flag protection for risky deploys**: Type changes or breaking changes should be behind flags to prevent failures during rolling deploys
- [ ] **Revert CI/test changes**: Remove debug/test CI pipeline changes before merging (`.drone.yml` test branch targets, etc.)
- [ ] **Config consistency**: Keep `application.yml` (local dev) in sync with `configmap.yaml` (deployed)
- [ ] **Explicit rollback plans**: For risky changes, document the exact steps to revert before approving
- [ ] **On-call notification**: For high-volume or high-risk changes, inform on-call and be prepared to rollback pods
- [ ] **CM for risky prod changes**: Cutover plans should reference CM tickets for smoke tests

### ✅ Internationalization (i18n) — squarespace-v6

- [ ] **Dark translations workflow**: When updating email copy, follow the dark translations workflow — update `_en.yaml` first, wait for translations, then update templates in a follow-up PR
- [ ] **Verify translation impact**: Changes to `.yaml` message bundles may cause regressions in other languages if not handled properly
- [ ] **Split email copy PRs**: Separate text additions from template changes to allow the translation pipeline to work
- [ ] **Variable name changes in templates**: Renaming template variables (e.g., `{productType}` to `{planType}`) will break non-English emails until translation jobs run — consider keeping old variable names or coordinating with i18n team
- [ ] **HTML template formatting**: Spotless auto-formatting can break HTML template indentation — preserve original formatting

### ✅ Infrastructure — billing-invoice-service

- [ ] **Kafka consumer configuration**: Be mindful of consumer count relative to topic partitions. Use sticky assignment strategies
- [ ] **Connection pool sizing**: HikariCP pool sizes should reference production metrics
- [ ] **Feature flag cleanup**: Create follow-up tickets to remove feature flags once fully rolled out
- [ ] **Audit vs. external events**: Both must be published for contract changes — audit events for internal DB, external events for downstream consumers
- [ ] **Don't leak external models into internal classes**: Pass specific fields (e.g., `contractId`, `orderActionId`) rather than entire event objects into internal orchestrators

### ✅ Performance & Scalability

- [ ] **Avoid unnecessary external service calls**: Don't call namespace client or statsig on every request when the data can be cached or checked cheaply first
- [ ] **Short-circuit expensive operations**: Order condition checks so cheaper checks run first
- [ ] **Batch processing design**: Consider window-based querying, batch sizes, and edge cases when processing batches smaller than result sets
- [ ] **Avoid N+1 query patterns**: When iterating over collections, don't make a DB call per item
- [ ] **Reduce unnecessary method parameters**: If fields can be fetched from an already-available object, don't pass them as separate parameters

## Common Patterns to Flag

### Red Flags 🚩
- SNAPSHOT versions in merge-ready code
- Missing null checks before chained method calls
- Large schema migrations without CM consideration
- Breaking facade API changes without downstream coordination
- Debug logs or temporary test code still present
- Missing QA validation for billing flow changes
- `var` usage where the type isn't immediately obvious
- Hardcoded strings that should be constants
- Reversed/swapped implementation of paired operations
- Tables without primary keys (CockroachDB migration requirement)
- i18n template + copy changes merged together (should be separate PRs)
- `AtomicBoolean` used inside lambda to work around effectively-final requirement

### Green Flags ✅
- Thorough test coverage with edge cases
- QA screenshots in PR descriptions
- Checkbox tasks tracking pre-merge items
- Clear Javadoc on public APIs
- Feature flag protection for risky changes
- Cleanup tickets created for deferred work
- Proportional discount math with correct term length accounting
- Explicit rollback plan documented
- Non-blocking index creation (`CONCURRENTLY`)
- Detailed PR description with testing section and monitoring plan

## PR Review Comment Style

Based on the team's review culture:

1. **Be specific and constructive** — point to the exact line and suggest an alternative
2. **Use `nit:` prefix** for style/preference suggestions that shouldn't block merge
3. **Use `- [x]` checkboxes** to track tasks that must be done before merge
4. **Link to relevant code/docs** when explaining context (Backstage docs, Slack threads, RFCs, Confluence)
5. **Ask clarifying questions** rather than assuming — "Could you clarify what's changing here?"
6. **Acknowledge good work** — the team values positive feedback ("Nice tests!", "Awesome work!", "Great test coverage!", "This change is so thoroughly tested that I feel extremely confident about it merging")
7. **Provide code suggestions** using GitHub's suggestion syntax when proposing alternatives
8. **Call out scope concerns early** — if a PR is growing too large, suggest splitting
9. **Deep domain explanations** — senior reviewers often write multi-paragraph explanations of billing flow context, linking to specific code lines in other files
10. **Track follow-up work** — create JIRA tickets for deferred improvements and link them in comments

## Output Format

When presenting a PR review:

```markdown
## PR Review: #NUMBER — Title

### Summary
Brief description of what this PR does and its billing domain context.

### Review Findings

#### 🚩 Issues (must fix)
1. [Issue description with specific file/line reference]

#### ⚠️ Suggestions (should consider)
1. [Suggestion with rationale]

#### 💡 Nits (optional improvements)
1. [Minor style/clarity improvement]

#### ✅ Looks Good
- [Positive callouts — test coverage, clean design, etc.]

### Pre-Merge Checklist
- [ ] Version dependencies updated to release versions
- [ ] QA validation completed
- [ ] [Any PR-specific items]
```
