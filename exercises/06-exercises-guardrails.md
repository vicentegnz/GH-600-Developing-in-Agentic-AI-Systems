# Exercises: Domain 6 — Guardrails and Accountability

## Exercise 6.1: Risk Classification

**Task**: Classify each agent action by risk level (Low, Medium, High, Critical) and assign an autonomy level (0-4).

| # | Agent Action | Risk Level | Autonomy Level | Reasoning |
|---|-------------|-----------|----------------|-----------|
| 1 | Run `npm test` | | | |
| 2 | Create a new file in `src/utils/` | | | |
| 3 | Modify `.github/workflows/*.yml` | | | |
| 4 | Push to a feature branch | | | |
| 5 | Merge a PR to `main` | | | |
| 6 | Delete a database migration file | | | |
| 7 | Add a new dependency to `package.json` | | | |
| 8 | Modify environment variables in production | | | |
| 9 | Create a GitHub issue | | | |
| 10 | Rotate API keys | | | |
| 11 | Format code (prettier) | | | |
| 12 | Modify authentication middleware | | | |

<details>
<summary>✅ Answers</summary>

| # | Agent Action | Risk Level | Autonomy Level | Reasoning |
|---|-------------|-----------|----------------|-----------|
| 1 | Run `npm test` | Low | 4 (Full Auto) | Read-only, no side effects |
| 2 | Create new file in `src/utils/` | Low | 3 (Act+Notify) | Additive, easily reversed |
| 3 | Modify workflow files | High | 1 (Suggest) | Meta-level: affects all CI/CD |
| 4 | Push to feature branch | Low | 4 (Full Auto) | Isolated, doesn't affect others |
| 5 | Merge PR to `main` | High | 1 (Suggest) | Affects shared branch, hard to undo |
| 6 | Delete migration file | Critical | 0 (Disabled) | Data loss risk, irreversible |
| 7 | Add dependency | Medium | 2 (Act+Review) | Supply chain risk, version conflicts |
| 8 | Modify prod env vars | Critical | 0 (Disabled) | Immediate production impact |
| 9 | Create GitHub issue | Low | 4 (Full Auto) | No code impact, informational |
| 10 | Rotate API keys | Critical | 1 (Suggest) | Could break integrations |
| 11 | Format code | Low | 4 (Full Auto) | No logic change, fully reversible |
| 12 | Modify auth middleware | High | 1 (Suggest) | Security-critical code path |

</details>

---

## Exercise 6.2: Policy Enforcement Layers

**Scenario**: You need to prevent an agent from:
1. Accessing production databases
2. Modifying security-critical code without review
3. Installing unapproved packages
4. Pushing directly to main

For each, implement the guardrail at the MOST APPROPRIATE layer.

<details>
<summary>✅ Sample Answer</summary>

**1. Accessing production databases:**

**Layer: Environment (Firewall/Network)**
```json
{
  "firewall": {
    "blockedUrls": [
      "https://*.prod.database.internal/*",
      "postgresql://*.prod.*"
    ],
    "blockedPorts": [5432, 3306]
  }
}
```
Why this layer: Network-level blocking is most reliable — can't be bypassed by prompt injection.

---

**2. Modifying security-critical code without review:**

**Layer: Platform (CODEOWNERS + Branch Protection)**
```
# .github/CODEOWNERS
/src/auth/        @security-team
/src/crypto/      @security-team
/src/middleware/auth* @security-team
```

```yaml
# Branch protection on main:
require_code_owner_review: true
required_approving_review_count: 2
```
Why this layer: Platform-level enforcement can't be overridden by the agent regardless of instructions.

---

**3. Installing unapproved packages:**

**Layer: Tool-level (CI Status Check)**
```yaml
# .github/workflows/check-dependencies.yml
name: Dependency Allowlist Check
on: pull_request

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check new dependencies against allowlist
        run: |
          NEW_DEPS=$(git diff origin/main -- package.json | grep '"+' | grep -v '"version"')
          ALLOWLIST=$(cat .github/approved-packages.json)
          for dep in $NEW_DEPS; do
            if ! echo "$ALLOWLIST" | grep -q "$dep"; then
              echo "::error::Unapproved package: $dep"
              exit 1
            fi
          done
```
Why this layer: Automated check catches any new dependency regardless of who/what added it.

---

**4. Pushing directly to main:**

**Layer: Platform (Branch Protection Rules)**
```yaml
# Repository Settings → Branches → main
protection_rules:
  require_pull_request:
    required_approving_review_count: 1
    dismiss_stale_reviews: true
  restrict_pushes:
    allow_force_pushes: false
    restrict_who_can_push:
      teams: ["release-managers"]
    # Agent's PAT/app NOT in allowed list
```
Why this layer: Branch protection is absolute — no code (agent or human) can bypass it.

</details>

---

## Exercise 6.3: Human-in-the-Loop Design

**Task**: Design a human-in-the-loop workflow for an agent that handles customer data migration. The workflow must balance speed with safety.

Requirements:
- Agent can read source and target databases
- Agent proposes a migration plan
- Migration of non-sensitive data can proceed with notification
- Migration of PII (emails, phone numbers) requires explicit approval
- Rollback must be possible within 24 hours

<details>
<summary>✅ Sample Answer</summary>

```yaml
workflow: customer-data-migration

steps:
  1_analysis:
    agent_action: "Analyze source and target schemas"
    autonomy: 4 (Full Auto)
    output: "schema-analysis.json"
    human_role: None
    
  2_plan:
    agent_action: "Generate migration plan with data classification"
    autonomy: 3 (Act + Notify)
    output: "migration-plan.md"
    human_role: "Notified, can intervene"
    data_classification:
      non_sensitive: ["product_ids", "timestamps", "categories"]
      sensitive_pii: ["emails", "phone_numbers", "addresses", "full_names"]
      
  3_approval_gate:
    type: "Human decision point"
    presented_to: "Data Protection Officer + Tech Lead"
    timeout: "4 hours (then escalate to manager)"
    decision_options:
      - approve_all: "Migrate everything as planned"
      - approve_non_sensitive: "Only migrate non-PII data"
      - modify: "Change the plan (human edits)"
      - reject: "Cancel migration"
      
  4a_non_sensitive_migration:
    trigger: "approval for non-sensitive data"
    agent_action: "Migrate non-PII data"
    autonomy: 3 (Act + Notify)
    human_role: "Monitoring dashboard"
    rollback_window: "24 hours"
    checkpoint: "Every 1000 records"
    
  4b_sensitive_migration:
    trigger: "EXPLICIT approval for PII"
    agent_action: "Migrate PII data with encryption"
    autonomy: 2 (Act + Review)
    human_role: "Watching each batch complete"
    batch_size: 100
    approval_per_batch: true  # Human confirms each batch
    rollback_window: "24 hours"
    audit_log: "every record logged"
    
  5_verification:
    agent_action: "Run data integrity checks"
    autonomy: 4 (Full Auto)
    output: "verification-report.json"
    checks:
      - record_count_matches: true
      - checksum_validation: true
      - sample_spot_check: "10 random records"
      
  6_rollback_capability:
    mechanism: "Soft delete + backup table"
    window: "24 hours from completion"
    trigger: "Human command OR automated integrity failure"
    action: "Restore from backup, delete migrated data"
```

**Key design decisions:**
- Non-sensitive data flows faster (Level 3) — notify but don't block
- PII requires per-batch approval — slower but compliant
- Rollback uses soft delete (reversible within window)
- Audit log captures every PII record touched
- Timeout on approval gate prevents indefinite blocking

</details>

---

## Exercise 6.4: Least Privilege Configuration

**Task**: Write the complete permission configuration for an agent that:
- Reviews PRs and leaves comments
- Can read all code
- Can read issues for context
- Cannot modify code
- Cannot merge PRs
- Cannot access secrets

Write configurations for: GitHub token permissions, tool restrictions, and file access.

<details>
<summary>✅ Sample Answer</summary>

**GitHub Actions Permissions:**
```yaml
jobs:
  agent-review:
    permissions:
      contents: read          # Read code
      pull-requests: write    # Comment on PRs (write needed for comments)
      issues: read            # Read issues for context
      # Explicitly NOT granted:
      # contents: write       # Cannot modify code
      # actions: write        # Cannot trigger workflows
      # admin: write          # Cannot change settings
      # deployments: write    # Cannot deploy
```

**Tool Restrictions:**
```typescript
const session = await client.createSession({
  customAgents: [{
    name: "reviewer",
    description: "Code review agent - read-only analysis",
    tools: ["grep", "glob", "view"],  // READ ONLY - no edit, create, bash
    prompt: `You are a code reviewer. 
             Analyze code for bugs, security issues, and style problems.
             Post findings as PR comments.
             NEVER suggest or make direct code modifications.
             NEVER access or reference secrets, API keys, or credentials.`,
  }],
});
```

**File Access Restrictions:**
```json
{
  "agent_scope": {
    "file_access": {
      "read": [
        "src/**",
        "tests/**",
        "docs/**",
        "package.json",
        "tsconfig.json",
        ".eslintrc.*"
      ],
      "write": [],
      "deny": [
        ".env*",
        "secrets/**",
        "*.key",
        "*.pem",
        "*credentials*",
        ".github/secrets/**"
      ]
    }
  }
}
```

**Network Access:**
```json
{
  "firewall": {
    "allowedUrls": [
      "https://api.github.com/*"
    ],
    "blockedUrls": ["*"],
    "notes": "Only GitHub API access needed for posting comments"
  }
}
```

**Summary of least-privilege layers:**
| Layer | Restriction | Enforced By |
|-------|------------|-------------|
| Token | Read code, write PR comments only | GitHub permissions |
| Tools | grep, glob, view only | Copilot SDK tool config |
| Files | Deny all secrets-related paths | Agent scope config |
| Network | Only GitHub API | Agent firewall |
| Instructions | "Never modify, never access secrets" | System prompt |

</details>

---

## Exercise 6.5: Responsible AI Guardrails

**Task**: An agent generates code suggestions. Design guardrails that address each Responsible AI principle.

| Principle | Risk in Agent Context | Guardrail Implementation |
|-----------|----------------------|-------------------------|
| Fairness | | |
| Reliability | | |
| Safety | | |
| Privacy | | |
| Inclusiveness | | |
| Transparency | | |
| Accountability | | |

<details>
<summary>✅ Sample Answer</summary>

| Principle | Risk in Agent Context | Guardrail Implementation |
|-----------|----------------------|-------------------------|
| **Fairness** | Agent generates biased variable names, assumptions in business logic | Lint rules for inclusive language; review generated business logic for assumptions about users |
| **Reliability** | Agent produces inconsistent outputs for same input | Deterministic settings (temperature=0), regression test suite, output comparison across runs |
| **Safety** | Agent introduces vulnerabilities, deletes data | Security scanning (CodeQL), restricted tool access, no delete operations, sandbox execution |
| **Privacy** | Agent exposes PII in logs, hardcodes sensitive data | Secret scanning, log sanitization, deny access to PII files, audit all data access |
| **Inclusiveness** | Agent generates inaccessible UI, non-i18n code | Accessibility linting (axe), i18n checks, screen reader compatibility tests |
| **Transparency** | Agent reasoning is opaque, decisions unexplained | Require structured plans, decision logs in PRs, explain "why" in commit messages |
| **Accountability** | No clear owner for agent-generated bugs | Co-authored-by tags, task-ID tracking, clear escalation to requesting developer |

</details>

---

## Exercise 6.6: Comprehensive Guardrail Scenario

**Scenario**: Your organization is deploying a fully autonomous code-fixing agent that:
- Monitors for failing CI builds
- Automatically creates fix PRs
- Runs 24/7 without human supervision
- Has access to all repositories in the org

Design a complete guardrail system. Address:
1. What autonomy level should it have?
2. What actions should be blocked?
3. What requires human approval?
4. How do you maintain velocity?
5. What audit trail is needed?

<details>
<summary>✅ Sample Answer</summary>

### 1. Autonomy Level: **Level 2 (Act + Review)** for fixes, **Level 3 (Act + Notify)** for trivial fixes

```yaml
autonomy_rules:
  trivial_fixes:  # Level 3
    criteria:
      - change_type: ["formatting", "unused_imports", "type_errors"]
      - diff_size: "< 5 lines"
      - files_changed: "≤ 2"
    action: "Auto-create PR, auto-merge if CI passes, notify channel"
    
  standard_fixes:  # Level 2
    criteria:
      - change_type: ["logic_fix", "dependency_update", "config_change"]
      - diff_size: "5-50 lines"
    action: "Create PR, require 1 human review"
    
  complex_fixes:  # Level 1
    criteria:
      - diff_size: "> 50 lines"
      - touches_security_paths: true
    action: "Create draft PR, require 2 reviews, tag team lead"
```

### 2. Blocked Actions:
```yaml
blocked_actions:
  - merge_to_main_without_review
  - modify_ci_workflows
  - modify_security_code_without_security_team
  - access_production_systems
  - delete_files
  - modify_permissions_or_access_controls
  - install_new_dependencies (must be approved)
  - modify_more_than_100_lines_in_one_PR
```

### 3. Human Approval Required:
```yaml
requires_approval:
  - Any change to authentication/authorization code
  - Any change to data models or database schemas
  - Any fix that modifies the public API surface
  - Any fix requiring a new dependency
  - Any fix the agent has attempted and failed 2+ times
  - Any change to files owned by CODEOWNERS
```

### 4. Velocity Preservation:
```yaml
velocity_optimizations:
  - auto_merge_trivial: true  # < 5 line formatting/import fixes
  - batch_similar_fixes: true  # Group related fixes into one PR
  - parallel_repos: true  # Work across repos simultaneously
  - skip_already_fixed: true  # Don't re-attempt if human already fixing
  - smart_scheduling: "Run during off-hours to avoid conflicts"
  - pre_approved_patterns:
    - "Remove unused imports"
    - "Fix type errors"
    - "Update deprecated API calls (from migration guide)"
```

### 5. Audit Trail:
```yaml
audit_requirements:
  per_action:
    - timestamp
    - agent_version
    - trigger_event (which CI failure)
    - files_analyzed
    - files_modified
    - tools_used
    - reasoning_summary
    - tokens_consumed
    
  per_pr:
    - link_to_failing_build
    - diff_with_annotations
    - test_results_before_after
    - security_scan_results
    - auto_merge_decision_rationale
    
  aggregate_weekly:
    - total_fixes_attempted
    - success_rate
    - auto_merged_count
    - human_rejected_count
    - average_fix_time
    - repos_affected
    
  retention: "2 years for compliance"
  access: "Org admins + security team"
```

</details>

---

## Exercise 6.7: Velocity vs. Safety Trade-off

**Task**: For each pair, choose the option that best balances velocity with safety. Explain your reasoning.

### Scenario 1:
A) Require manual approval for ALL agent PRs
B) Auto-merge agent PRs that pass all CI checks and are < 10 lines

### Scenario 2:
A) Give agent admin access so it can fix its own permission errors
B) Agent fails with permission error, creates issue for human to resolve

### Scenario 3:
A) Agent reviews its own code before creating PR (self-review)
B) Agent creates PR, separate review agent reviews it

<details>
<summary>✅ Answers</summary>

**Scenario 1: Choose B**
- Option A is too conservative — blocks trivial fixes that are objectively safe
- Option B preserves velocity for low-risk changes while CI provides safety net
- The 10-line threshold ensures complex changes still get human review
- Auto-merge only after ALL checks pass (tests, lint, security)

**Scenario 2: Choose B**
- Option A violates least-privilege catastrophically — admin access means agent could do anything
- Option B is slower but maintains security boundaries
- Permission errors are a signal that the agent is trying something it shouldn't
- Human can evaluate whether to grant the permission or adjust the task
- Speed loss is acceptable because permission errors should be rare

**Scenario 3: Choose B (separate review agent)**
- Self-review provides almost no value (same context, same blind spots)
- A separate review agent has different instructions, different perspective
- This follows the "separation of concerns" principle
- Like code review: the author shouldn't review their own code
- The review agent can catch issues the implementation agent's context missed

</details>
