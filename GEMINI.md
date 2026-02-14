# gemini-review

Code review extension with sub-agent delegation. Splits reviews into specialized dimensions (security, correctness, quality), then synthesizes findings with confidence scoring.

**Commands**: `/review`, `/review:branch`, `/review:commit`, `/review:pr`
**Prerequisite**: `experimental.enableAgents: true` in Gemini CLI settings

---

## Review Orchestration Procedure

When a review command is invoked, follow these steps exactly:

### Step 1: Identify Scope

The command prompt provides the diff and scope. Note the scope type (uncommitted, branch, commit, PR) and the list of changed files.

### Step 2: Create Output Directory

```bash
mkdir -p /tmp/gemini-review/$(date +%Y%m%d-%H%M%S)/
```

Store the directory path — you will write all reports here.

### Step 3: Delegate to Sub-Agents

Call each sub-agent **in sequence**, passing:
- The review scope (e.g., "uncommitted changes", "PR #123")
- The list of changed files from the command output
- The shell command to retrieve the diff (e.g., `git diff HEAD`)
- Any additional context from the user's conversation

Call these three sub-agents:
1. **security-reviewer** — security vulnerabilities, auth issues, silent failures
2. **correctness-reviewer** — bugs, logic errors, edge cases, error handling
3. **quality-reviewer** — patterns, test gaps, comment accuracy, performance

Each sub-agent returns findings in this format:
```
[Confidence: XX] [P0] [category] Description — file:line
```

### Step 4: Synthesize Findings

Collect all findings from the three sub-agents. Then apply:

**4a. De-duplicate (Consensus Detection)**
If two or more sub-agents flag the same file:line with a similar description:
- Mark the finding as **CONSENSUS**
- Add **+15 confidence bonus** to each matching finding (cap at 100)
- This rescues borderline findings: e.g., 75 + 15 = 90

**4b. Apply Threshold**
Discard any finding with confidence < 80 (after consensus bonus).

**4c. Sort**
1. CONSENSUS findings first (highest trust)
2. Then by confidence descending
3. Then by severity: P0 > P1 > P2 > P3

**4d. Determine Verdict**
- **PASS**: 0 findings at P0 or P1, and 2 or fewer at P2
- **PASS WITH CONCERNS**: 0 findings at P0, any at P1 or P2
- **REVISE**: any P0, or 3+ findings at P1

### Step 5: Write Detailed Reports

Write these files to the output directory:
- `summary.md` — All findings grouped by severity, then by sub-agent source. Include evidence and reasoning for each finding.
- `security.md` — Full security-reviewer output
- `correctness.md` — Full correctness-reviewer output
- `quality.md` — Full quality-reviewer output

Each finding in the detailed reports must include:
- Confidence score, severity, category
- Description of the issue
- File path and line number
- Evidence from the code (quote the relevant lines)
- Why this is a problem and what to do about it

### Step 6: Return Terminal Summary

Print a brief summary (15-20 lines max) to the terminal:

```
**Review Complete** — {scope}
**Verdict**: {PASS | PASS WITH CONCERNS | REVISE}
**Findings**: {count} issues (confidence >= 80) | {consensus_count} consensus

**Critical Issues** ({count}):
- [P0] [Confidence: 95] {description} — {file:line}
- [P1] [Confidence: 88] {description} — {file:line}

**Top Recommendations**:
1. {most important action item}
2. {second most important}

**Detailed Report**: /tmp/gemini-review/{id}/summary.md
```

If there are no findings above the confidence threshold, report:
```
**Review Complete** — {scope}
**Verdict**: PASS
**Findings**: 0 issues above confidence threshold

No actionable findings. Code looks clean.
```

---

## Ignore List

Do NOT report any of the following:
- Pre-existing issues not introduced in this changeset
- Issues that linters, typecheckers, or CI would catch
- Pedantic nitpicks that a senior engineer would not flag
- Issues on lines that were not modified in the diff
- Intentional functionality changes that are part of the broader changeset
- Issues silenced by lint-ignore comments (e.g., `// eslint-disable`, `# noqa`)

---

## Severity Reference

- **P0 / CRITICAL**: Exploitable security issue, data loss or corruption, authorization bypass, crash in common path
- **P1 / HIGH**: High-likelihood bug, major performance regression, concurrency hazard, broken API contract
- **P2 / MEDIUM**: Edge-case bug, maintainability risk, incomplete error handling, moderate test gap
- **P3 / LOW**: Minor cleanup, nits only if they materially improve clarity
