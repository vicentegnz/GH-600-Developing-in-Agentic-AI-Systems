# Exercises: Domain 1 — Agent Architecture & SDLC Processes

## Exercise 1.1: Identify Agent-Suitable Tasks

**Scenario**: You're a tech lead at a company with a large monorepo. Your team receives the following requests. For each, determine if it's suitable for an agent and explain why.

| # | Task | Suitable? | Why? |
|---|------|-----------|------|
| 1 | Auto-format all Python files to match PEP 8 | | |
| 2 | Decide whether to use microservices or monolith for a new product | | |
| 3 | Generate unit tests for all utility functions | | |
| 4 | Migrate 200 API endpoints from REST to GraphQL | | |
| 5 | Write the company's privacy policy | | |
| 6 | Update all import statements after a package rename | | |
| 7 | Review PRs for common security vulnerabilities | | |
| 8 | Design the database schema for a new feature | | |

<details>
<summary>✅ Answers</summary>

1. **YES** — Well-defined, repeatable, clear success criteria (passes lint)
2. **NO** — Requires deep business context, no clear success criteria, irreversible architectural decision
3. **YES** — Repetitive, clear inputs/outputs, testable success criteria
4. **PARTIAL** — Agent can help but needs human oversight due to complexity and potential breaking changes
5. **NO** — Legal document requiring domain expertise and human judgment
6. **YES** — Mechanical transformation, easily verifiable, reversible
7. **YES** — Pattern matching against known vulnerabilities, clear criteria
8. **NO** — Requires business domain understanding, has long-term architectural implications

</details>

---

## Exercise 1.2: Identify Anti-Patterns

**Scenario**: Review the following agent configurations and identify the anti-pattern(s) present.

### Configuration A:
```yaml
agent:
  name: "code-bot"
  permissions: admin
  scope: entire-organization
  approval_required: false
  max_retries: unlimited
  timeout: none
```

### Configuration B:
```yaml
agent:
  name: "refactor-agent"
  tools: [bash, edit, create, delete, deploy]
  output: 
    plan: false
    logs: minimal
    artifacts: none
```

### Configuration C:
```yaml
agent:
  name: "test-generator"
  scope: "src/**"
  on_failure: retry_same_approach
  on_success: auto_merge_without_review
  context: load_entire_repository
```

<details>
<summary>✅ Answers</summary>

**Configuration A Anti-Patterns:**
- **Unbounded autonomy** — admin permissions with no approval required
- **Scope creep** — entire organization scope
- **Infinite loops** — unlimited retries with no timeout
- **Trust escalation** — admin-level permissions

**Configuration B Anti-Patterns:**
- **Opaque reasoning** — no plan output, minimal logs, no artifacts
- **Overly broad toolset** — includes `deploy` and `delete` unnecessarily
- **Silent failure** — no artifacts means failures can't be diagnosed

**Configuration C Anti-Patterns:**
- **Infinite loops** — retries same approach on failure
- **Bypassed review** — auto-merge without review
- **Context pollution** — loads entire repository instead of relevant files

</details>

---

## Exercise 1.3: Define Task Specification

**Task**: Write a complete task specification for an agent that will "Add input validation to all API endpoints in the `/api/users` directory."

Fill in the template:
```yaml
task:
  name: ""
  description: ""
  
  inputs:
    - 
    - 
    
  outputs:
    - 
    - 
    
  success_criteria:
    - 
    - 
    - 
    
  constraints:
    - 
    - 
    
  autonomy_level: ""
  required_reviews: 
```

<details>
<summary>✅ Sample Answer</summary>

```yaml
task:
  name: "Add input validation to user API endpoints"
  description: "Add request body validation using zod schemas for all POST/PUT endpoints in /api/users"
  
  inputs:
    - source_files: "src/api/users/*.ts"
    - validation_library: "zod"
    - existing_patterns: "src/api/products/validation.ts"  # Reference implementation
    
  outputs:
    - validation_schemas: "src/api/users/schemas/*.ts"
    - updated_handlers: "src/api/users/*.ts (modified)"
    - tests: "tests/api/users/validation.test.ts"
    
  success_criteria:
    - all_endpoints_have_validation: true
    - existing_tests_pass: true
    - new_validation_tests_pass: true
    - no_lint_errors: true
    - validation_errors_return_400: true
    - matches_existing_patterns: true
    
  constraints:
    - max_execution_time: "10m"
    - no_changes_outside_users_dir: true
    - no_breaking_changes_to_response_format: true
    - must_handle_all_edge_cases: ["empty body", "wrong types", "missing required fields"]
    
  autonomy_level: "2 (Act + Review)"
  required_reviews: 1
```

</details>

---

## Exercise 1.4: Design Plan-Before-Execute Workflow

**Task**: Design a GitHub Actions workflow that:
1. Triggers when an issue is labeled `agent-task`
2. Has the agent create a plan (Phase 1)
3. Requires human approval of the plan (Phase 2)
4. Executes the plan (Phase 3)

Write the workflow YAML:

<details>
<summary>✅ Sample Answer</summary>

```yaml
name: Agent Plan-Execute Workflow

on:
  issues:
    types: [labeled]

jobs:
  plan:
    if: contains(github.event.label.name, 'agent-task')
    runs-on: ubuntu-latest
    permissions:
      contents: read
      issues: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate plan
        id: plan
        run: |
          # Agent generates plan based on issue body
          echo "plan=$(cat generated-plan.json)" >> $GITHUB_OUTPUT
          
      - name: Post plan for review
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: '## 🤖 Agent Plan\n\n' + process.env.PLAN + '\n\n**Approve by commenting `/approve-plan`**'
            });
            
  execute:
    needs: plan
    runs-on: ubuntu-latest
    # This job only runs after human approval (triggered by /approve-plan comment)
    if: github.event.comment.body == '/approve-plan'
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Execute approved plan
        run: |
          # Agent executes the approved plan
          git checkout -b agent/issue-${{ github.event.issue.number }}
          # ... make changes ...
          git push origin agent/issue-${{ github.event.issue.number }}
          
      - name: Create PR
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh pr create \
            --title "Agent: ${{ github.event.issue.title }}" \
            --body "Closes #${{ github.event.issue.number }}" \
            --base main
```

</details>

---

## Exercise 1.5: Observability Configuration

**Task**: For an agent that autonomously fixes lint errors across a repository, design what artifacts it should produce. List at least 6 artifacts with their format and purpose.

<details>
<summary>✅ Sample Answer</summary>

| # | Artifact | Format | Purpose |
|---|----------|--------|---------|
| 1 | Execution plan | `plan.md` (Markdown) | Shows what the agent intends to fix before acting |
| 2 | Changes summary | `changes.json` (JSON) | Machine-readable list of all modifications |
| 3 | Before/after diffs | PR diff | Human-reviewable changes |
| 4 | Lint report comparison | `lint-before.json` + `lint-after.json` | Quantitative improvement evidence |
| 5 | Decision log | `decisions.md` (Markdown) | Why certain fixes were chosen over alternatives |
| 6 | Execution metrics | `metrics.json` (JSON) | Time, files scanned, fixes applied, tokens used |
| 7 | Skipped items log | `skipped.md` (Markdown) | Lint errors the agent chose NOT to fix and why |
| 8 | Test results | `test-results.xml` (JUnit) | Proof that fixes didn't break anything |

</details>

---

## Exercise 1.6: Scenario Analysis

**Scenario**: Your organization has deployed a coding agent that:
- Runs nightly to fix TODO comments by implementing the described functionality
- Has write access to all repositories
- Auto-merges its PRs if CI passes
- Sends a weekly summary email to the team

**Question**: Identify at least 5 risks with this setup and propose mitigations for each.

<details>
<summary>✅ Sample Answer</summary>

| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| 1 | Agent misinterprets TODO and implements wrong functionality | High | Require human review before merge (remove auto-merge) |
| 2 | Agent has org-wide write access (too broad) | Critical | Scope to specific repositories, use least-privilege tokens |
| 3 | No plan review before execution | High | Add planning phase with approval gate |
| 4 | TODO implementation could introduce bugs | High | Require test coverage for new code, don't auto-merge |
| 5 | Agent could create breaking changes | High | Run full test suite + integration tests before merge |
| 6 | No scope limit on which TODOs to address | Medium | Add label/priority filter, only address tagged TODOs |
| 7 | Weekly summary is too infrequent for oversight | Medium | Add per-PR notifications + daily summary |
| 8 | No rollback mechanism if merged PR breaks things | High | Implement auto-revert on test failures post-merge |

</details>

---

## Exercise 1.7: Design Autonomy Levels

**Task**: For a CI/CD pipeline agent, assign autonomy levels to each of the following actions. Justify your choice.

| Action | Your Level (0-4) | Justification |
|--------|------------------|---------------|
| Run unit tests | | |
| Add a new test file | | |
| Modify existing source code | | |
| Create a feature branch | | |
| Merge PR to main | | |
| Deploy to staging | | |
| Deploy to production | | |
| Rollback production deployment | | |
| Update dependencies | | |
| Modify CI/CD workflow files | | |

<details>
<summary>✅ Sample Answer</summary>

| Action | Level | Justification |
|--------|-------|---------------|
| Run unit tests | 4 (Full Auto) | Read-only, no risk, always safe |
| Add a new test file | 3 (Act+Notify) | Additive, doesn't break existing code |
| Modify existing source code | 2 (Act+Review) | Could introduce bugs, needs human check |
| Create a feature branch | 4 (Full Auto) | Low risk, easily deleted |
| Merge PR to main | 1 (Suggest) | Affects main branch, needs human decision |
| Deploy to staging | 2 (Act+Review) | Reversible but affects shared environment |
| Deploy to production | 0-1 (Disabled/Suggest) | High business impact, irreversible |
| Rollback production | 1 (Suggest) | Emergency action but needs human judgment |
| Update dependencies | 2 (Act+Review) | Could break builds, security implications |
| Modify CI/CD workflows | 1 (Suggest) | Meta-level change affecting all automation |

</details>
