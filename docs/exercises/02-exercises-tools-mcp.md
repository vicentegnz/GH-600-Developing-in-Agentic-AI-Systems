# Exercises: Domain 2 — Tool Use and Environment Interaction

## Exercise 2.1: Tool Selection and Scoping

**Scenario**: You're configuring three specialized agents for a TypeScript project. Assign the appropriate tools to each agent using the principle of least privilege.

**Available tools**: `grep`, `glob`, `view`, `edit`, `create`, `bash`, `git`, `npm`, `deploy`, `delete`

| Agent | Role | Tools (select from available) |
|-------|------|------|
| `code-reader` | Analyzes code, finds patterns, answers questions | |
| `test-writer` | Creates new test files for existing code | |
| `refactorer` | Modifies existing code to improve quality | |

<details>
<summary>✅ Answers</summary>

| Agent | Tools | Rationale |
|-------|-------|-----------|
| `code-reader` | `grep`, `glob`, `view` | Read-only; no modification capability needed |
| `test-writer` | `grep`, `glob`, `view`, `create`, `bash` | Needs read + create new files + run tests; no `edit` (doesn't modify existing) |
| `refactorer` | `grep`, `glob`, `view`, `edit`, `bash` | Needs read + modify existing files + run tests; no `create` or `delete` (refactoring only) |

Note: None get `deploy`, `delete`, or `git` — these are not needed for their tasks.

</details>

---

## Exercise 2.2: MCP Server Configuration

**Task**: Write a complete `.github/copilot/mcp.json` configuration that includes:
1. A PostgreSQL database server (for query access)
2. A filesystem server (scoped to `/workspace/docs`)
3. A GitHub MCP server (for issue/PR access)

<details>
<summary>✅ Sample Answer</summary>

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/workspace/docs"
      ]
    },
    "github": {
      "url": "https://api.github.com/mcp",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    }
  }
}
```

**Key points:**
- PostgreSQL uses stdio transport (local command)
- Filesystem is scoped to `/workspace/docs` only (least privilege)
- GitHub uses HTTP transport (remote server)
- Secrets use variable substitution, never hardcoded

</details>

---

## Exercise 2.3: MCP Allow List Design

**Scenario**: Your organization wants to control which MCP servers agents can use. Design an allow list policy.

**Requirements:**
- Developers can use: postgres, filesystem, GitHub
- Only senior devs can use: AWS, Kubernetes
- Nobody should use: arbitrary HTTP servers, eval/exec servers

Write the allow list configuration:

<details>
<summary>✅ Sample Answer</summary>

```json
{
  "mcpPolicy": {
    "defaultAction": "deny",
    "allowList": {
      "all-developers": [
        "@modelcontextprotocol/server-postgres",
        "@modelcontextprotocol/server-filesystem",
        "github-mcp-server"
      ],
      "senior-developers": [
        "@modelcontextprotocol/server-postgres",
        "@modelcontextprotocol/server-filesystem",
        "github-mcp-server",
        "@modelcontextprotocol/server-aws",
        "@modelcontextprotocol/server-kubernetes"
      ]
    },
    "denyList": [
      "@modelcontextprotocol/server-exec",
      "@modelcontextprotocol/server-eval",
      "*-experimental",
      "*-unsafe"
    ],
    "auditLog": true
  }
}
```

</details>

---

## Exercise 2.4: CI Workflow Agent Integration

**Task**: Write a GitHub Actions workflow that:
1. Triggers on `pull_request` (opened or synchronized)
2. Runs an agent that reviews the changed files
3. Posts review comments on the PR
4. Fails the check if critical issues are found

```yaml
# Write your workflow here:
name: 
on:

jobs:
```

<details>
<summary>✅ Sample Answer</summary>

```yaml
name: Agent Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for diff
          
      - name: Get changed files
        id: changes
        run: |
          FILES=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)
          echo "files=$FILES" >> $GITHUB_OUTPUT
          
      - name: Run agent review
        id: review
        env:
          CHANGED_FILES: ${{ steps.changes.outputs.files }}
        run: |
          # Agent reviews changed files
          # Outputs: review-results.json with findings
          echo "critical_count=$(jq '.critical | length' review-results.json)" >> $GITHUB_OUTPUT
          
      - name: Post review comments
        uses: actions/github-script@v7
        with:
          script: |
            const results = require('./review-results.json');
            for (const finding of results.findings) {
              await github.rest.pulls.createReviewComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                pull_number: context.issue.number,
                body: `🤖 **${finding.severity}**: ${finding.message}`,
                path: finding.file,
                line: finding.line,
                commit_id: context.sha
              });
            }
            
      - name: Fail on critical issues
        if: steps.review.outputs.critical_count > 0
        run: |
          echo "::error::Found ${{ steps.review.outputs.critical_count }} critical issues"
          exit 1
```

</details>

---

## Exercise 2.5: Branch-Based Agent Scope

**Scenario**: Configure an agent that:
- Can only create branches with prefix `agent/`
- Can only push to its own branches
- Must create PRs targeting `main`
- Cannot directly push to `main` or `develop`

Write the branch protection rules and agent configuration:

<details>
<summary>✅ Sample Answer</summary>

**Branch Protection (main):**
```yaml
# Repository settings → Branches → Branch protection rules
branch: main
rules:
  require_pull_request:
    required_approving_review_count: 1
    dismiss_stale_reviews: true
  require_status_checks:
    strict: true
    contexts: ["ci/tests", "security/scan"]
  restrict_pushes:
    allow_force_pushes: false
    restrict_who_can_push:
      teams: ["maintainers"]  # Agent's token NOT included
```

**Branch Protection (develop):**
```yaml
branch: develop
rules:
  require_pull_request:
    required_approving_review_count: 1
  restrict_pushes:
    restrict_who_can_push:
      teams: ["developers"]  # Agent's token NOT included
```

**Ruleset for agent branches:**
```yaml
# Ruleset: Allow agent to create/push to agent/* branches
ruleset:
  name: "Agent branch permissions"
  target: "branch"
  conditions:
    ref_name:
      include: ["refs/heads/agent/*"]
  rules:
    - type: creation
      parameters:
        allowed_actors: ["agent-bot"]  # Only agent can create these
```

**Agent configuration:**
```yaml
agent:
  branch_prefix: "agent/"
  target_branch: "main"
  permissions:
    create_branch: true
    push_to_own_branch: true
    create_pr: true
    merge_pr: false
    push_to_main: false
    push_to_develop: false
```

</details>

---

## Exercise 2.6: Error Handling Design

**Task**: Design an error handling strategy for an agent that deploys to staging. For each failure type, specify: the retry policy, rollback action, and escalation path.

| Failure Type | Retry Policy | Rollback Action | Escalation |
|--------------|-------------|-----------------|------------|
| Network timeout during deploy | | | |
| Build failure | | | |
| Test failures in staging | | | |
| Permission denied | | | |
| Dependency not found | | | |
| Out of memory | | | |

<details>
<summary>✅ Sample Answer</summary>

| Failure Type | Retry Policy | Rollback Action | Escalation |
|--------------|-------------|-----------------|------------|
| Network timeout | Retry 3x with exponential backoff (1s, 2s, 4s) | None (deploy didn't start) | After 3 retries: notify DevOps channel |
| Build failure | No retry (deterministic failure) | None (nothing deployed) | Immediately notify developer who triggered |
| Test failures in staging | Retry once (could be flaky) | Redeploy previous version | After 1 retry: block + assign to developer |
| Permission denied | No retry (won't resolve itself) | None | Immediately escalate to platform team |
| Dependency not found | Retry once after `npm install` | None | Notify developer, suggest lock file update |
| Out of memory | No retry (needs config change) | Kill stuck processes | Escalate to DevOps for resource increase |

</details>

---

## Exercise 2.7: Traceability Implementation

**Task**: An agent has just completed a task. Write the commit message and PR description that provides full traceability.

**Context:**
- Agent: `test-generator-v2`
- Task: Generate missing unit tests for `src/utils/date.ts`
- Duration: 45 seconds
- Tools used: grep, view, create, bash
- Tests generated: 12 new test cases
- Coverage improvement: 45% → 92%

<details>
<summary>✅ Sample Answer</summary>

**Commit message:**
```
test: add unit tests for date utility functions

Generated 12 test cases covering:
- formatDate() with various locales
- parseDate() with valid/invalid inputs
- dateDiff() with timezone edge cases
- isWeekend() boundary conditions

Coverage: 45% → 92% (+47%)

Agent: test-generator-v2
Task-ID: task-20240115-abc123
Tools: grep, view, create, bash
Duration: 45s
Tokens: 8,200

Co-authored-by: test-generator-v2[bot] <test-generator-v2[bot]@users.noreply.github.com>
```

**PR Description:**
```markdown
## 🤖 Agent-Generated: Unit Tests for Date Utilities

### Summary
Added comprehensive unit tests for `src/utils/date.ts` to improve code coverage.

### Changes
- Created `tests/utils/date.test.ts` with 12 test cases
- No modifications to source code

### Metrics
| Metric | Value |
|--------|-------|
| Test cases added | 12 |
| Coverage before | 45% |
| Coverage after | 92% |
| Execution time | 45 seconds |
| Agent version | test-generator-v2 |

### Test Cases Added
1. ✅ formatDate - formats ISO string to locale format
2. ✅ formatDate - handles null input gracefully
3. ✅ parseDate - parses valid date strings
4. ✅ parseDate - throws on invalid format
... (8 more)

### Validation
- [x] All new tests pass
- [x] All existing tests still pass
- [x] No lint errors introduced
- [x] Coverage meets threshold (>80%)

### Decision Log
- Used `vitest` (matches existing test setup)
- Added edge cases for timezone handling (identified gap via coverage analysis)
- Skipped integration tests (out of scope for this task)
```

</details>

---

## Exercise 2.8: Agent Firewall Configuration

**Scenario**: Configure a firewall for a cloud agent that:
- Can access the GitHub API
- Can access npm registry
- Can access your company's internal documentation site
- CANNOT access production databases
- CANNOT access external AI services
- CANNOT access social media sites

Write the firewall configuration:

<details>
<summary>✅ Sample Answer</summary>

```json
{
  "firewall": {
    "defaultAction": "deny",
    "allowedUrls": [
      "https://api.github.com/*",
      "https://github.com/*",
      "https://registry.npmjs.org/*",
      "https://docs.internal.company.com/*",
      "https://raw.githubusercontent.com/*"
    ],
    "blockedUrls": [
      "https://*.database.internal.company.com/*",
      "https://db-*.company.com/*",
      "https://api.openai.com/*",
      "https://api.anthropic.com/*",
      "https://*.facebook.com/*",
      "https://*.twitter.com/*",
      "https://*.linkedin.com/*"
    ],
    "blockedPorts": [5432, 3306, 27017, 6379],
    "notes": "Production databases blocked by URL and port. External AI services blocked to prevent data exfiltration."
  }
}
```

**Rationale:**
- `defaultAction: deny` — Allowlist approach (safer)
- GitHub/npm allowed for code operations
- Internal docs allowed for context
- Databases blocked by both URL pattern AND common ports
- AI services blocked to prevent sending code to external models
- Social media blocked as unnecessary and potential data leak vector

</details>
