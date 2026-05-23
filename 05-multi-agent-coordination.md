# Domain 5: Orchestrate Multi-Agent Coordination (15–20%)

## 📚 References & Documentation

| Resource | Link |
|----------|------|
| GitHub Docs | [Custom Agents (Multi-Agent)](https://docs.github.com/en/copilot/how-tos/copilot-sdk/use-copilot-sdk/custom-agents) |
| Copilot SDK | [github/copilot-sdk](https://github.com/github/copilot-sdk) |
| GitHub Docs | [GitHub Actions - Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) |
| GitHub Docs | [GitHub Actions - Matrix Strategy](https://docs.github.com/en/actions/using-jobs/using-a-matrix-strategy-for-your-jobs) |
| GitHub Docs | [Concurrency](https://docs.github.com/en/actions/using-jobs/using-concurrency) |
| MS Learn | [Designing Agent Architecture](https://learn.microsoft.com/en-us/training/modules/design-agent-architecture-integration/) |
| MS Learn | [Tooling, MCP, and Execution](https://learn.microsoft.com/en-us/training/modules/agent-tooling-mcp-execution-environments/) |

---

## Section 1: Operate and Manage Multi-Agent Workflows

### 1.1 Orchestration Patterns

**Pattern 1: Sequential Pipeline (Chain)**
```
Agent A → Agent B → Agent C → Final Output
(Plan)    (Code)    (Review)
```
- Each agent's output is the next agent's input
- Simple, predictable, easy to debug
- Bottleneck: slowest agent determines total time

**Pattern 2: Parallel Fan-Out / Fan-In**
```
              ┌─── Agent B (Frontend) ───┐
              │                          │
Agent A ──────┼─── Agent C (Backend) ────┼──── Agent F (Integration)
(Plan)        │                          │     (Merge & Test)
              └─── Agent D (Tests) ──────┘
```
- Multiple agents work simultaneously
- Faster total execution time
- Requires merge/integration step
- Risk of conflicts between agents

**Pattern 3: Hierarchical (Orchestrator + Workers)**
```
        ┌─────────────────┐
        │  Orchestrator   │
        │  (Parent Agent) │
        └────────┬────────┘
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
  ┌─────────┐ ┌─────────┐ ┌─────────┐
  │Worker A │ │Worker B │ │Worker C │
  │(Research)│ │(Code)   │ │(Test)   │
  └─────────┘ └─────────┘ └─────────┘
```
- Central coordinator delegates tasks
- Workers are specialized and isolated
- Orchestrator handles conflicts and sequencing

**Pattern 4: Event-Driven (Reactive)**
```
Event Bus / GitHub Events
    │
    ├──── PR Opened ──────► Review Agent
    ├──── PR Approved ────► Deploy Agent  
    ├──── Issue Created ──► Triage Agent
    └──── Push to main ──► Release Agent
```
- Agents respond to events independently
- Highly decoupled
- Scales naturally
- Harder to trace end-to-end flows

### 1.2 Apply an Orchestration Pattern

**Copilot SDK Multi-Agent Implementation:**
```typescript
import { CopilotClient } from "@github/copilot-sdk";

const client = new CopilotClient();
await client.start();

const session = await client.createSession({
  model: "gpt-4.1",
  customAgents: [
    {
      name: "planner",
      displayName: "Planning Agent",
      description: "Analyzes requirements and creates implementation plans",
      tools: ["grep", "glob", "view"],
      prompt: `You are a planning agent. Analyze the codebase and create 
               a structured implementation plan. Never modify files.`,
      infer: true,
    },
    {
      name: "implementer",
      displayName: "Implementation Agent", 
      description: "Writes code based on approved plans",
      tools: ["view", "edit", "create", "bash"],
      prompt: `You implement code changes based on plans. Follow the plan
               exactly. Report any deviations.`,
      infer: true,
    },
    {
      name: "reviewer",
      displayName: "Review Agent",
      description: "Reviews code for quality, security, and correctness",
      tools: ["grep", "glob", "view"],
      prompt: `You review code changes. Check for bugs, security issues,
               style violations, and test coverage. Never modify files.`,
      infer: true,
    },
  ],
  onPermissionRequest: async () => ({ kind: "approved" }),
});
```

### 1.3 Configure Agent Isolation for Parallel Execution

**Isolation strategies:**

| Strategy | Mechanism | Pros | Cons |
|----------|-----------|------|------|
| **Branch isolation** | Each agent works on separate branch | No conflicts during execution | Merge conflicts later |
| **File isolation** | Each agent owns specific files/dirs | Clear boundaries | Requires upfront planning |
| **Container isolation** | Separate containers per agent | Full environment isolation | Resource-heavy |
| **Context isolation** | Separate sessions/windows | No shared state pollution | Harder to share results |

**GitHub Actions parallel execution:**
```yaml
jobs:
  agent-frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Agent works on frontend
        run: # Agent modifies src/frontend/ only

  agent-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Agent works on backend
        run: # Agent modifies src/backend/ only

  integrate:
    needs: [agent-frontend, agent-backend]
    runs-on: ubuntu-latest
    steps:
      - name: Merge agent changes
        run: |
          git fetch origin agent/frontend agent/backend
          git merge agent/frontend agent/backend
          npm test  # Verify integration
```

### 1.4 Detect and Resolve Agent Conflicts

**Types of conflicts:**

| Conflict Type | Description | Detection |
|---------------|-------------|-----------|
| **Overlapping edits** | Two agents modify same file/line | Git merge conflicts |
| **Duplicated effort** | Two agents solve the same problem | Duplicate function/file detection |
| **Contradictory outputs** | Agents produce incompatible results | Integration test failures |
| **Resource contention** | Agents compete for same resource | Lock contention, race conditions |

**Conflict resolution strategies:**
```
1. PREVENT: Assign non-overlapping scopes before execution
2. DETECT: Run integration checks after parallel execution
3. RESOLVE: 
   - Automatic: Use git merge strategies (ours/theirs/union)
   - Manual: Escalate to human for ambiguous conflicts
   - Retry: Re-run conflicting agent with updated context
```

---

## Section 2: Configure Observability for Multi-Agent Behavior

### 2.1 Produce Artifacts Suitable for Review and Audit

**Required artifacts per agent:**
```yaml
agent_execution_record:
  agent_id: "implementer-001"
  task: "Add user validation"
  started_at: "2024-01-15T10:00:00Z"
  completed_at: "2024-01-15T10:05:23Z"
  status: "success"
  
  inputs:
    - plan_sha: "abc123"
    - source_branch: "main"
    
  outputs:
    - branch: "agent/add-validation"
    - pr_number: 42
    - files_modified: ["src/validators/user.ts", "tests/user.test.ts"]
    - diff_stats: "+45 -3"
    
  tools_used:
    - { tool: "view", calls: 5, duration: "2s" }
    - { tool: "edit", calls: 3, duration: "1s" }
    - { tool: "bash", calls: 2, duration: "30s" }
    
  decisions:
    - "Used zod for validation (matches existing pattern)"
    - "Added unit test instead of integration test (faster feedback)"
```

### 2.2 Document Key Decisions, Handoffs, and Outcomes

**Handoff documentation pattern:**
```
┌─────────────────────────────────────────────────┐
│ HANDOFF RECORD                                  │
├─────────────────────────────────────────────────┤
│ From: Planner Agent                             │
│ To: Implementer Agent                           │
│ Timestamp: 2024-01-15T10:02:00Z                 │
│                                                 │
│ Context Passed:                                 │
│   - Plan document (plan.md @ sha:abc123)        │
│   - Target files: [src/auth/*, tests/auth/*]    │
│   - Constraints: No breaking changes            │
│                                                 │
│ Decisions Made by Planner:                      │
│   - Use JWT with RS256 signing                  │
│   - Keep backward compat for 30 days            │
│                                                 │
│ Expected Outcome:                               │
│   - PR with implementation of steps 1-5         │
│   - All existing tests pass                     │
│   - New tests for added functionality           │
└─────────────────────────────────────────────────┘
```

### 2.3 Perform Post-Hoc Analysis of Multi-Agent Behavior

**Analysis questions:**
1. Did agents stay within their assigned scope?
2. Were handoffs clean (no missing context)?
3. Did any agent produce unnecessary work?
4. Were conflicts resolved correctly?
5. What was the total execution time vs. sequential?
6. Were there any cascade failures?

**Metrics to track:**
| Metric | Purpose |
|--------|---------|
| Total execution time | Efficiency |
| Per-agent execution time | Bottleneck identification |
| Conflict count | Isolation quality |
| Handoff success rate | Coordination quality |
| Rework percentage | Wasted effort |
| Human intervention count | Autonomy effectiveness |

---

## Section 3: Detect and Respond to Multi-Agent Failures

### 3.1 Identify Failed, Partial, or Stalled Agent Executions

**Failure states:**

| State | Description | Indicator |
|-------|-------------|-----------|
| **Failed** | Agent errored out | Non-zero exit code, error in logs |
| **Partial** | Agent completed some but not all steps | Incomplete artifacts, missing outputs |
| **Stalled** | Agent is stuck, not making progress | No new output after timeout period |
| **Degraded** | Agent completes but with poor quality | Tests fail, high lint errors |

**Detection mechanisms:**
```yaml
# Workflow-level failure detection
jobs:
  agent-task:
    runs-on: ubuntu-latest
    timeout-minutes: 30  # Detect stalls
    steps:
      - name: Agent execution
        id: agent
        continue-on-error: true
        run: |
          # Agent work here
          
      - name: Check completion
        if: always()
        run: |
          if [ "${{ steps.agent.outcome }}" == "failure" ]; then
            echo "::error::Agent failed - triggering recovery"
          fi
          
          # Check for partial completion
          if [ ! -f "expected-output.json" ]; then
            echo "::warning::Agent produced incomplete output"
          fi
```

### 3.2 Respond to Degraded Behavior

**Response hierarchy:**
```
1. RETRY: Re-run the failed agent with same inputs
2. FALLBACK: Run alternative agent with different strategy
3. ISOLATE: Remove failed agent from pipeline, continue without
4. ESCALATE: Notify human, pause pipeline
5. ROLLBACK: Undo all changes from the failed execution
```

### 3.3 Multi-Agent Recovery Patterns

**Pattern: Rollback and Retry**
```yaml
jobs:
  execute:
    runs-on: ubuntu-latest
    steps:
      - name: Save checkpoint
        run: echo "CHECKPOINT=$(git rev-parse HEAD)" >> $GITHUB_ENV
        
      - name: Agent A execution
        id: agent-a
        continue-on-error: true
        run: # Agent A work
        
      - name: Rollback A on failure
        if: steps.agent-a.outcome == 'failure'
        run: git reset --hard ${{ env.CHECKPOINT }}
        
      - name: Human-in-the-loop on repeated failure
        if: steps.agent-a.outcome == 'failure'
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: 'Agent A failed - needs human intervention',
              body: 'Agent A failed after retry. Please investigate.',
              labels: ['agent-failure', 'needs-human']
            });
```

---

## Section 4: Manage the Lifecycle of Agents

### 4.1 Add Agents to Existing Workflows

**Integration checklist:**
- [ ] Define the new agent's scope and responsibilities
- [ ] Ensure no overlap with existing agents
- [ ] Configure isolation boundaries
- [ ] Add to orchestration logic
- [ ] Update handoff documentation
- [ ] Test with existing agents (integration test)
- [ ] Deploy gradually (feature flag or canary)

### 4.2 Update, Reconfigure, or Replace Agents

**Safe update pattern:**
```
1. Deploy new agent version alongside old (blue/green)
2. Route a percentage of tasks to new version
3. Compare outputs between versions
4. If new version meets/exceeds criteria → full rollout
5. If new version degrades → rollback to old version
```

**Configuration change management:**
```yaml
# Version your agent configurations
# .github/agents/v2/implementer.yml
version: 2
name: "implementer"
changes_from_v1:
  - "Added TypeScript strict mode support"
  - "Reduced max diff size from 500 to 300 lines"
  - "Added security scanning before commit"
effective_date: "2024-02-01"
rollback_to: "v1"
```

### 4.3 Retire Agents While Preserving Auditability

**Retirement checklist:**
- [ ] Archive agent configuration (don't delete)
- [ ] Document retirement reason and date
- [ ] Ensure all past agent outputs remain accessible
- [ ] Update workflow to remove agent invocations
- [ ] Redirect any dependencies to replacement agent
- [ ] Keep audit logs for compliance period
- [ ] Remove from active orchestration

```yaml
# Retired agent record
retired_agents:
  - name: "legacy-formatter"
    retired_date: "2024-03-15"
    reason: "Replaced by 'style-agent' with broader capabilities"
    replacement: "style-agent"
    audit_retention: "2 years"
    archived_config: ".github/agents/retired/legacy-formatter.yml"
```

---

## 🧠 Key Memorization Points

1. **Four orchestration patterns**: Sequential, Parallel Fan-Out/In, Hierarchical, Event-Driven
2. **Isolation strategies**: Branch, File, Container, Context
3. **Conflict types**: Overlapping edits, duplicated effort, contradictory outputs, resource contention
4. **Failure states**: Failed, Partial, Stalled, Degraded
5. **Recovery patterns**: Retry → Fallback → Isolate → Escalate → Rollback
6. **Handoff documentation**: From/To, Context passed, Decisions made, Expected outcome
7. **Post-hoc metrics**: Execution time, conflict count, handoff success rate, rework %
8. **Agent lifecycle**: Add → Operate → Update → Replace → Retire
9. **Safe updates**: Blue/green deployment, canary routing, output comparison
10. **Retirement**: Archive config, document reason, maintain audit trail

---

## 📝 Self-Assessment Questions

1. Describe the four multi-agent orchestration patterns and when to use each.
2. How do you configure agent isolation for parallel execution?
3. What are the types of multi-agent conflicts and how do you resolve each?
4. Describe the failure detection mechanisms for stalled agents.
5. What's the recovery hierarchy when a multi-agent workflow fails?
6. What information should a handoff record contain?
7. How do you safely update an agent without disrupting active workflows?
8. What metrics indicate poor multi-agent coordination?
9. How do you retire an agent while maintaining auditability?
10. What's the difference between "parallel fan-out" and "hierarchical" orchestration?

➡️ **Practice these concepts with exercises**: [05-exercises-multi-agent.md](./exercises/05-exercises-multi-agent.md)
