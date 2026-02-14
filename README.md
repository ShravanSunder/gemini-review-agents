# gemini-review

Gemini CLI extension for code review with sub-agent delegation. Splits reviews into three specialized dimensions (security, correctness, quality), then synthesizes findings with confidence scoring and progressive disclosure.

Methodology ported from [quorum-counsel](../plugins/quorum-counsel/) and Claude Code's pr-review-toolkit.

## Installation

```bash
# From the ai-tools repo
gemini extensions link ./extensions/gemini-review

# Or from a git URL
gemini extensions install https://github.com/ShravanSunder/gemini-review-agents
```

### Prerequisites

Enable experimental sub-agents in Gemini CLI settings:

```json
{
  "experimental": {
    "enableAgents": true
  }
}
```

Without this setting, the `/review` commands will still work but Gemini will handle the review as a single pass instead of delegating to specialized sub-agents.

## Commands

| Command | Scope | Usage |
|---------|-------|-------|
| `/review` | Uncommitted changes (staged + unstaged) | `/review` |
| `/review:branch` | Current branch vs base branch | `/review:branch main` |
| `/review:commit` | Specific commit | `/review:commit a1b2c3d` |
| `/review:pr` | GitHub pull request | `/review:pr 123` |

### Examples

```
# Review your working directory changes
/review

# Review everything on this branch vs main
/review:branch main

# Review a specific commit
/review:commit HEAD~1

# Review a GitHub PR
/review:pr 456
```

## Sub-Agents

Each review delegates to three specialized sub-agents:

| Agent | Focus | Categories |
|-------|-------|------------|
| **security-reviewer** | Injection, auth bypass, data exposure, SSRF, silent failures | `security`, `silent-failure` |
| **correctness-reviewer** | Logic errors, null safety, type issues, edge cases, error handling | `bug`, `error-handling` |
| **quality-reviewer** | Pattern compliance, test gaps, comment accuracy, performance | `pattern`, `test-gap`, `comment-rot`, `performance` |

## Methodology

### Severity Scale

| Level | Meaning |
|-------|---------|
| **P0 / CRITICAL** | Exploitable security, data loss, authz bypass, crash in common path |
| **P1 / HIGH** | High-likelihood bug, major perf regression, concurrency hazard, broken API contract |
| **P2 / MEDIUM** | Edge-case bug, maintainability risk, incomplete error handling, moderate test gap |
| **P3 / LOW** | Minor cleanup, nits only if they materially improve clarity |

### Confidence Scoring

Every finding is scored 0-100. Only findings with **confidence >= 80** are reported.

When two or more sub-agents flag the same issue (same file:line, similar description), a **+15 consensus bonus** is applied (cap at 100). This rescues borderline findings that multiple reviewers independently identified.

### Verdict

| Verdict | Criteria |
|---------|----------|
| **PASS** | 0 findings at P0-P1, 2 or fewer at P2 |
| **PASS WITH CONCERNS** | 0 P0, any P1 or P2 |
| **REVISE** | Any P0, or 3+ findings at P1 |

### Ignore List

The following are NOT reported:
- Pre-existing issues not introduced in this changeset
- Issues linters/typecheckers/CI would catch
- Pedantic nitpicks a senior engineer wouldn't flag
- Issues on unmodified lines
- Intentional functionality changes
- Issues silenced by lint-ignore comments

## Output

### Terminal (brief summary, 15-20 lines)

```
**Review Complete** — uncommitted
**Verdict**: PASS WITH CONCERNS
**Findings**: 4 issues (confidence >= 80) | 1 consensus

**Critical Issues** (1):
- [P1] [Confidence: 92] Missing null check on user input — src/api/users.ts:42

**Top Recommendations**:
1. Add input validation before database query
2. Add test for empty user ID case

**Detailed Report**: /tmp/gemini-review/20260214-153022/summary.md
```

### Detailed reports (written to /tmp/)

```
/tmp/gemini-review/{timestamp}/
├── summary.md        # All findings with evidence
├── security.md       # Security reviewer output
├── correctness.md    # Correctness reviewer output
└── quality.md        # Quality reviewer output
```

## Architecture

```
/review command (TOML)
    │
    ├── Injects diff via !{git diff HEAD}
    │
    └── Gemini main agent (guided by GEMINI.md)
         │
         ├── Delegates to security-reviewer (sub-agent)
         ├── Delegates to correctness-reviewer (sub-agent)
         ├── Delegates to quality-reviewer (sub-agent)
         │
         ├── Synthesizes: de-duplicate → consensus bonus → threshold → sort
         ├── Writes detailed reports to /tmp/gemini-review/
         └── Returns brief terminal summary
```
