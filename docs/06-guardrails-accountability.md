# Domain 6: Implement Guardrails and Accountability (10–15%)

## 📚 References & Documentation

| Resource | Link |
|----------|------|
| GitHub Docs | [Build Guardrails](https://docs.github.com/en/copilot/tutorials/cloud-agent/build-guardrails) |
| GitHub Docs | [Risks and Mitigations](https://docs.github.com/en/copilot/concepts/agents/cloud-agent/risks-and-mitigations) |
| GitHub Docs | [Agent Firewall](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/customize-the-agent-firewall) |
| GitHub Docs | [Branch Protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches) |
| GitHub Docs | [Rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets) |
| GitHub Docs | [CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) |
| GitHub Docs | [Environment Protection Rules](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) |
| GitHub Docs | [Required Status Checks](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches#require-status-checks-before-merging) |
| MS Learn | [Designing Agent Architecture](https://learn.microsoft.com/en-us/training/modules/design-agent-architecture-integration/) |
| Responsible AI | [Microsoft Responsible AI](https://www.microsoft.com/en-us/ai/responsible-ai) |

---

## Section 1: Define Autonomy Levels

### 1.1 Classify Agent Actions by Risk

**Risk Classification Matrix:**

| Risk Level | Operational Impact | Security Impact | Compliance Impact | Examples |
|------------|-------------------|-----------------|-------------------|----------|
| **Low** | Easily reversible | No sensitive data | No regulatory concern | Read files, run tests, format code |
| **Medium** | Reversible with effort | Touches config | Minor policy concern | Create branches, modify non-critical code |
| **High** | Difficult to reverse | Accesses secrets | Regulatory implications | Deploy to staging, modify auth code |
| **Critical** | Irreversible | Exposes sensitive data | Audit-triggering | Deploy to production, delete resources, modify access controls |

### 1.2 Autonomy Levels Definition

| Level | Name | Agent Behavior | Human Role | Use Cases |
|-------|------|---------------|------------|-----------|
| **0** | Disabled | Agent cannot act | Human does everything | Security-critical operations |
| **1** | Suggest | Agent proposes, never acts | Human reviews + executes | Architecture decisions |
| **2** | Act + Review | Agent acts, human reviews after | Human approves or reverts | Code changes (PR-based) |
| **3** | Act + Notify | Agent acts and notifies | Human monitors | Low-risk automated tasks |
| **4** | Fully Autonomous | Agent acts independently | Human audits periodically | Formatting, dependency updates |

### 1.3 Assign Autonomy Levels

**Decision Framework:**
```
                    ┌─────────────────────────┐
                    │ Is the action reversible?│
                    └────────────┬────────────┘
                          Yes    │    No
                    ┌────────────┴────────────┐
                    ▼                          ▼
         ┌──────────────────┐      ┌──────────────────┐
         │ Does it touch     │      │ LEVEL 0-1        │
         │ sensitive data?   │      │ (Disabled/Suggest)│
         └────────┬──────────┘      └──────────────────┘
             Yes  │  No
         ┌────────┴────────┐
         ▼                  ▼
  ┌──────────────┐  ┌──────────────────┐
  │ LEVEL 1-2    │  │ Does it affect   │
  │ (Suggest/    │  │ production?      │
  │  Act+Review) │  └────────┬─────────┘
  └──────────────┘       Yes │  No
                    ┌────────┴────────┐
                    ▼                  ▼
             ┌──────────┐     ┌──────────────┐
             │ LEVEL 2  │     │ LEVEL 3-4    │
             │(Act+Review)│   │(Notify/Auto) │
             └──────────┘     └──────────────┘
```

**Example Autonomy Assignment:**

| Action | Risk | Autonomy Level | Rationale |
|--------|------|---------------|-----------|
| Format code | Low | 4 (Full auto) | Easily reversible, no logic changes |
| Write unit tests | Low-Med | 3 (Act + Notify) | Additive, doesn't break existing |
| Refactor function | Medium | 2 (Act + Review) | Could introduce bugs |
| Modify authentication | High | 1 (Suggest only) | Security-critical |
| Delete database table | Critical | 0 (Disabled) | Irreversible data loss |
| Update dependencies | Medium | 2 (Act + Review) | Could break builds |
| Deploy to production | Critical | 1 (Suggest only) | Business impact |

### 1.4 Balance Delivery Speed with Compliance

**Principles:**
1. **Right-size interventions**: Don't require approval for trivial changes
2. **Risk-proportional oversight**: More risk = more human involvement
3. **Fast-path for safe actions**: Pre-approved patterns skip review
4. **Audit everything, approve selectively**: Log all actions, only gate risky ones

---

## Section 2: Implement Guardrails and Human-in-the-Loop Workflows

### 2.1 Identify Actions Requiring Human Judgment

**Human judgment required when:**
- Trade-offs between competing priorities (speed vs. security)
- Ambiguous requirements with multiple valid interpretations
- Changes that affect user experience or business logic
- Actions with legal or compliance implications
- Novel situations not covered by existing rules

**Human judgment NOT required when:**
- Action follows a well-defined, repeatable pattern
- Success/failure is objectively measurable
- Action is fully reversible
- Existing automated checks provide sufficient validation

### 2.2 Block Actions Violating Policies

**Policy enforcement layers:**

```
┌─────────────────────────────────────────────────────┐
│ Layer 1: INSTRUCTION-LEVEL (Agent prompt)           │
│   "Never modify files in /security/ directory"      │
├─────────────────────────────────────────────────────┤
│ Layer 2: TOOL-LEVEL (Permission restrictions)       │
│   tools: ["grep", "view"] // No write access        │
├─────────────────────────────────────────────────────┤
│ Layer 3: ENVIRONMENT-LEVEL (Firewall/Network)       │
│   Block access to production APIs                   │
├─────────────────────────────────────────────────────┤
│ Layer 4: PLATFORM-LEVEL (GitHub controls)           │
│   Branch protection, CODEOWNERS, required checks    │
├─────────────────────────────────────────────────────┤
│ Layer 5: ORGANIZATIONAL (Policies & Rulesets)       │
│   Organization-wide rulesets, audit requirements    │
└─────────────────────────────────────────────────────┘
```

**GitHub-native policy enforcement:**

```yaml
# Branch protection rules
branches:
  main:
    protection:
      required_reviews: 2
      dismiss_stale_reviews: true
      require_code_owner_review: true
      required_status_checks:
        strict: true
        contexts:
          - "ci/tests"
          - "security/codeql"
          - "lint/eslint"
      restrictions:
        users: []
        teams: ["maintainers"]
      enforce_admins: true
```

**CODEOWNERS for agent-modified paths:**
```
# .github/CODEOWNERS

# Security-critical code requires security team review
/src/auth/          @security-team
/src/crypto/        @security-team

# Infrastructure requires platform team
/terraform/         @platform-team
/.github/workflows/ @platform-team

# Agent-generated code requires designated reviewer
/agent-output/      @agent-supervisors
```

### 2.3 Scope Permissions — Least Privilege

**Token scoping:**
```yaml
# GitHub Actions - scoped permissions
jobs:
  agent-job:
    permissions:
      contents: read     # Can read code
      pull-requests: write  # Can create/comment on PRs
      issues: read       # Can read issues
      # NOT granted:
      # contents: write  # Cannot push directly
      # admin: write     # Cannot change settings
      # deployments: write # Cannot deploy
```

**Agent execution boundaries:**
```json
{
  "agent_permissions": {
    "file_access": {
      "read": ["src/**", "tests/**", "docs/**"],
      "write": ["src/features/**", "tests/**"],
      "deny": ["src/core/**", ".env*", "secrets/**"]
    },
    "network_access": {
      "allow": ["api.github.com", "registry.npmjs.org"],
      "deny": ["*.production.internal", "*.database.internal"]
    },
    "actions": {
      "allow": ["create_branch", "create_pr", "run_tests"],
      "deny": ["merge_pr", "delete_branch", "deploy"]
    }
  }
}
```

### 2.4 Require Authorization for Irreversible Changes

**Gate patterns for sensitive actions:**

```yaml
# Environment protection rules for deployments
environments:
  production:
    protection_rules:
      - type: required_reviewers
        reviewers:
          - team: "release-managers"
      - type: wait_timer
        minutes: 15
      - type: branch_policy
        branches: ["main"]
```

**Copilot cloud agent firewall:**
```json
{
  "firewall": {
    "allowedUrls": [
      "https://api.github.com/*",
      "https://registry.npmjs.org/*"
    ],
    "blockedUrls": [
      "https://*.production.*",
      "https://*.internal.*"
    ]
  }
}
```

### 2.5 Preserve Execution Velocity

**Strategies to minimize unnecessary friction:**

| Strategy | Implementation | Benefit |
|----------|---------------|---------|
| **Pre-approved patterns** | Template-based tasks skip review | Fast repetitive work |
| **Auto-merge for safe PRs** | PRs passing all checks merge automatically | No waiting for humans |
| **Tiered review** | Only high-risk changes need senior review | Right-sized oversight |
| **Async notification** | Notify but don't block for low-risk | No interruption |
| **Batch approvals** | Approve multiple similar changes at once | Efficient review |

**Auto-merge configuration:**
```yaml
# Enable auto-merge for PRs that pass all checks
name: Auto-merge safe agent PRs
on:
  pull_request:
    types: [labeled]
    
jobs:
  auto-merge:
    if: contains(github.event.pull_request.labels.*.name, 'agent-safe')
    runs-on: ubuntu-latest
    steps:
      - name: Enable auto-merge
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.pulls.merge({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
              merge_method: 'squash'
            });
```

---

## 🧠 Key Memorization Points

1. **Risk levels**: Low (reversible) → Medium (config) → High (secrets) → Critical (irreversible)
2. **Autonomy levels**: 0 (Disabled) → 1 (Suggest) → 2 (Act+Review) → 3 (Act+Notify) → 4 (Full Auto)
3. **Policy enforcement layers**: Instruction → Tool → Environment → Platform → Organization
4. **Least privilege**: Scope tokens, restrict file access, limit network, deny dangerous actions
5. **CODEOWNERS** enforces reviewers for specific paths
6. **Branch protection** prevents direct pushes, requires reviews and status checks
7. **Environment protection** gates deployments with required reviewers and wait timers
8. **Agent firewall** restricts network access in cloud environments
9. **Velocity preservation**: Pre-approved patterns, auto-merge, tiered review, async notification
10. **Human judgment needed for**: Trade-offs, ambiguity, UX impact, legal/compliance, novel situations

---

## Responsible AI Considerations

### Key Principles for Agent Systems

| Principle | Application to Agents |
|-----------|----------------------|
| **Fairness** | Agents should not introduce bias in code or decisions |
| **Reliability** | Agents must produce consistent, predictable results |
| **Safety** | Agents must not cause harm to systems or data |
| **Privacy** | Agents must not expose or collect sensitive information |
| **Inclusiveness** | Agent outputs should be accessible and inclusive |
| **Transparency** | Agent reasoning and actions must be inspectable |
| **Accountability** | Clear ownership of agent actions and outcomes |

### Responsible AI Guardrails

```yaml
responsible_ai_checks:
  - name: "Bias detection"
    action: "Scan generated code for discriminatory patterns"
    enforcement: "Block PR if detected"
    
  - name: "Data privacy"
    action: "Ensure no PII in logs, commits, or artifacts"
    enforcement: "Secret scanning + custom regex"
    
  - name: "Transparency"
    action: "Agent must explain its reasoning in PR description"
    enforcement: "Required PR template fields"
    
  - name: "Human oversight"
    action: "Critical decisions flagged for human review"
    enforcement: "Required reviewers for high-risk paths"
```

---

## 📝 Self-Assessment Questions

1. Define the five autonomy levels and give an example action for each.
2. How do you classify agent actions by risk level?
3. What are the five layers of policy enforcement?
4. How does CODEOWNERS help implement guardrails for agents?
5. Describe least-privilege implementation for an agent's file access.
6. When is human judgment required vs. when can agents act autonomously?
7. How do environment protection rules gate deployments?
8. What strategies preserve execution velocity while maintaining guardrails?
9. How does the agent firewall enforce network access controls?
10. What Responsible AI principles apply specifically to agent systems?

➡️ **Practice these concepts with exercises**: [06-exercises-guardrails.md](./exercises/06-exercises-guardrails.md)
