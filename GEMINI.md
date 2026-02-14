# gemini-review

Code review extension. Output all findings directly to the terminal.

**Do NOT create directories. Do NOT write files. All output goes to the terminal.**

## Review Procedure

When a review command is invoked:

1. Read the diff provided by the command.
2. Read surrounding code for context using `read_file` and `grep_search`.
3. If sub-agents are available (security-reviewer, correctness-reviewer, quality-reviewer), delegate to them. Otherwise, review all dimensions yourself.
4. Score each finding for confidence and severity.
5. Print the full review to the terminal using the output format below.

## Review Dimensions

### Security & Silent Failures
- Injection (SQL, NoSQL, XSS, command injection)
- Auth bypass, privilege escalation, missing auth checks
- Secrets in code, PII leaks, path traversal, SSRF
- Race conditions, TOCTOU
- For EVERY catch/fallback/error-handler: is error logged? Does user get feedback? Could catch hide unrelated errors? Empty catch = P0

### Bugs & Correctness
- Logic errors, off-by-one, wrong variable, inverted conditions
- Null/undefined dereference, missing boundary checks
- Type coercion bugs, implicit conversions
- Edge cases: empty collections, boundary values, overflow
- Missing error handling, unhandled rejections, unclosed resources

### Quality & Patterns
- Does code follow existing codebase patterns?
- Test gaps (rate criticality 1-10)
- Stale comments describing old behavior (comment rot)
- Performance: O(n^2) where O(n) possible, N+1 queries, memory leaks

## Severity

- **P0 / CRITICAL**: Exploitable security, data loss/corruption, authz bypass, crash in common path
- **P1 / HIGH**: High-likelihood bug, major perf regression, concurrency hazard, broken API contract
- **P2 / MEDIUM**: Edge-case bug, maintainability risk, incomplete error handling, moderate test gap
- **P3 / LOW**: Minor cleanup, nits only if they materially improve clarity

## Confidence Scoring

Score every finding 0-100. **Only report findings with confidence >= 80.**

## Ignore (Do NOT Report)

- Pre-existing issues not in this changeset
- Issues linters/typecheckers/CI would catch
- Nitpicks a senior engineer wouldn't flag
- Issues on unmodified lines
- Intentional changes
- Lint-ignore silenced issues

## Output Format

Print directly to terminal:

```
## Review — {scope}

**Verdict**: {PASS | PASS WITH CONCERNS | REVISE}

### Findings

- **[P0] [95] [security]** SQL injection in user query — `src/api/users.ts:42`
  Unsanitized input passed to query builder. Use parameterized queries.

- **[P1] [88] [bug]** Missing null check — `src/utils/parse.ts:17`
  `data.items` can be undefined when API returns empty response.

### Recommendations

1. Most important action
2. Second most important
```

Verdict rules:
- **PASS**: 0 findings P0-P1, ≤2 P2
- **PASS WITH CONCERNS**: 0 P0, any P1-P2
- **REVISE**: any P0, or 3+ P1
