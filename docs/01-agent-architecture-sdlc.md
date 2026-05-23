# Domain 1: Prepare Agent Architecture and SDLC Processes (15–20%)

## 📚 References & Documentation

| Resource | Link |
|----------|------|
| MS Learn Module | [Foundations of Agentic AI in GitHub](https://learn.microsoft.com/en-us/training/modules/foundations-agentic-ai/) |
| MS Learn Module | [Designing Agent Architecture and SDLC Integration](https://learn.microsoft.com/en-us/training/modules/design-agent-architecture-integration/) |
| GitHub Docs | [Prepare for Custom Agents](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-organization/prepare-for-custom-agents) |
| GitHub Docs | [Cloud Agent](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent) |
| GitHub Docs | [Agent Firewall](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/customize-the-agent-firewall) |
| GitHub Docs | [Rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets) |
| GitHub Docs | [Required Status Checks](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) |

---

## Section 1: Integrate Agents into the Software Development Lifecycle (SDLC)

### 1.1 Key Concepts

**Agentic AI vs. Assistive AI:**
- **Assistant**: Responds to prompts, provides suggestions, requires human to act (e.g., Copilot autocomplete)
- **Agent**: Plans, reasons, acts autonomously within defined boundaries (e.g., Copilot cloud agent creating PRs)

**The Plan → Act → Evaluate Lifecycle:**
```
┌─────────┐     ┌─────────┐     ┌──────────┐
│  PLAN   │────▶│   ACT   │────▶│ EVALUATE │
│         │     │         │     │          │
│ Analyze │     │ Execute │     │ Review   │
│ Decide  │     │ Create  │     │ Validate │
│ Propose │     │ Modify  │     │ Approve  │
└─────────┘     └─────────┘     └──────────┘
      ▲                                │
      └────────────────────────────────┘
              (Feedback Loop)
```

**The Contributor Model:**
Agents are treated like junior developers:
- Their work goes through PR review
- They follow branch protection rules
- Their access is scoped via permissions
- Their outputs are traceable and auditable

### 1.2 Identify Steps for Agents to Perform

Agents are best suited for:
- **Repetitive, well-defined tasks** (code generation, test writing, documentation)
- **Tasks with clear success criteria** (linting, formatting, migration)
- **Tasks that benefit from parallelism** (multi-file refactoring)

Agents are NOT suited for:
- Ambiguous requirements without clear boundaries
- Tasks requiring deep domain expertise with no training data
- Decisions with irreversible business consequences

### 1.3 Identify and Mitigate Common Anti-Patterns

| Anti-Pattern | Description | Mitigation |
|--------------|-------------|------------|
| **Unbounded autonomy** | Agent acts without approval gates | Implement PR-based governance, required reviews |
| **Opaque reasoning** | Agent decisions can't be inspected | Require structured plans before execution |
| **Scope creep** | Agent modifies files outside its mandate | Use tool restrictions, repository scoping |
| **Infinite loops** | Agent retries indefinitely on failure | Implement max retries, timeout, escalation |
| **Context pollution** | Agent accumulates irrelevant context | Prune memory, scope to task-relevant info |
| **Trust escalation** | Agent gains privileges beyond what's needed | Apply least-privilege, scoped tokens |
| **Silent failure** | Agent fails without notification | Require status reporting, artifact production |

### 1.4 Define Inputs, Outputs, and Success Criteria

**Structured Task Definition:**
```yaml
task:
  name: "Generate unit tests for auth module"
  inputs:
    - source_files: "src/auth/*.ts"
    - test_framework: "vitest"
    - coverage_target: 80%
  outputs:
    - test_files: "tests/auth/*.test.ts"
    - coverage_report: "coverage/auth-coverage.json"
  success_criteria:
    - all_tests_pass: true
    - coverage_meets_target: true
    - no_new_lint_errors: true
    - pr_created: true
  constraints:
    - max_execution_time: "10m"
    - no_production_file_changes: true
```

---

## Section 2: Define Boundaries Between Planning, Reasoning, and Action

### 2.1 Configure Agent Planning Distinct from Execution

**Why separate planning from execution?**
- Planning can be reviewed before any changes are made
- Humans can approve/modify the plan
- Failed plans don't leave partial changes
- Plans create an audit trail

**Implementation Pattern:**
```
Phase 1: PLANNING (read-only tools only)
  → Agent analyzes codebase
  → Agent produces structured plan
  → Plan is committed as artifact (e.g., plan.md)

Phase 2: APPROVAL (human-in-the-loop)
  → Human reviews plan
  → Human approves, modifies, or rejects

Phase 3: EXECUTION (write tools enabled)
  → Agent executes approved plan
  → Changes are submitted as PR
  → Standard review process applies
```

### 2.2 Configure an Agent to Output a Structured Plan

Example structured plan output:
```json
{
  "plan": {
    "objective": "Refactor authentication to use JWT",
    "steps": [
      {
        "id": 1,
        "action": "Create JWT utility module",
        "files_affected": ["src/utils/jwt.ts"],
        "risk": "low",
        "reversible": true
      },
      {
        "id": 2,
        "action": "Update auth middleware",
        "files_affected": ["src/middleware/auth.ts"],
        "risk": "medium",
        "reversible": true
      }
    ],
    "estimated_changes": 5,
    "requires_approval": true
  }
}
```

### 2.3 Validate Agent Plans

Validation checklist:
- [ ] Plan scope matches the original request
- [ ] No unexpected files are being modified
- [ ] Risk assessment is reasonable
- [ ] Changes are reversible where claimed
- [ ] Success criteria are testable
- [ ] No security-sensitive operations without explicit approval

### 2.4 Prevent Action Until Plan Is Approved

**GitHub-native enforcement:**
- Use **required status checks** that validate plan approval
- Use **branch protection rules** preventing direct pushes
- Use **CODEOWNERS** requiring specific reviewers for agent-generated PRs
- Use **environment protection rules** for deployment-related agent actions

---

## Section 3: Configure Observability and Control for Autonomous Agents

### 3.1 Plan and Implement the Degree of Agent Autonomy

**Autonomy Spectrum:**
```
FULL MANUAL ◀─────────────────────────────────▶ FULL AUTONOMY
     │              │              │              │
  Human does    Agent suggests  Agent acts,    Agent acts
  everything    human decides   human reviews  independently
                                after the fact
```

**Deciding autonomy level factors:**
- Risk of the action (reversible vs. irreversible)
- Confidence in agent capability for this task type
- Regulatory/compliance requirements
- Organizational trust maturity

### 3.2 Configure Agent to Produce Inspectable Artifacts

Agents should produce:
- **Structured plans** (committed to the repository)
- **Execution logs** (stored as workflow artifacts)
- **Diff summaries** (in PR descriptions)
- **Decision rationale** (in commit messages or PR comments)
- **Metrics** (execution time, tokens used, tools invoked)

### 3.3 Configure Human Intervention Without Slowing Delivery

**Patterns for efficient human oversight:**

| Pattern | When to Use | Implementation |
|---------|-------------|----------------|
| **Async review** | Low-risk changes | PR review with auto-merge after approval |
| **Pre-approved templates** | Repetitive tasks | Agent follows approved patterns without per-task review |
| **Exception-based review** | High-volume tasks | Agent proceeds unless flagged by automated checks |
| **Tiered approval** | Mixed-risk workflows | Low-risk auto-merges; high-risk requires human |

---

## 🧠 Key Memorization Points

1. **Agents follow the Plan → Act → Evaluate lifecycle**
2. **Agents are treated as contributors** (their work goes through standard PR review)
3. **Planning must be separate from execution** to enable review
4. **Anti-patterns**: unbounded autonomy, opaque reasoning, scope creep, infinite loops
5. **Inputs + Outputs + Success Criteria** = well-defined agent tasks
6. **Observability requires**: structured plans, logs, diffs, decision rationale
7. **Human intervention patterns**: async review, pre-approved templates, exception-based, tiered
8. **Autonomy level** depends on: risk, confidence, compliance, trust maturity

---

## 📝 Self-Assessment Questions

1. What distinguishes an AI agent from an AI assistant in the SDLC context?
2. Name three anti-patterns in agent systems and their mitigations.
3. How do you enforce that an agent cannot execute before its plan is approved?
4. What GitHub features enable the "contributor model" for agents?
5. Describe the Plan → Act → Evaluate lifecycle and why each phase matters.
6. How do you configure human intervention without slowing delivery velocity?
7. What artifacts should an autonomous agent produce for observability?
8. When should you NOT use an agent for a task?

➡️ **Practice these concepts with exercises**: [01-exercises-architecture.md](./exercises/01-exercises-architecture.md)
