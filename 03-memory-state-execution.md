# Domain 3: Manage Memory, State, and Execution (10–15%)

## 📚 References & Documentation

| Resource | Link |
|----------|------|
| GitHub Docs | [Copilot Memory](https://docs.github.com/en/copilot/concepts/agents/copilot-memory) |
| GitHub Docs | [Managing Copilot Memory](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/copilot-memory) |
| GitHub Docs | [Custom Instructions](https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions) |
| GitHub Docs | [Copilot Setup Steps](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/copilot-setup-steps) |
| MS Learn | [Foundations of Agentic AI](https://learn.microsoft.com/en-us/training/modules/foundations-agentic-ai/) |

---

## Section 1: Implement Agent Memory Strategies

### 1.1 Types of Agent Memory

| Memory Type | Duration | Scope | Examples |
|-------------|----------|-------|----------|
| **Short-term (Working)** | Single session/task | Current conversation | Chat history, current file context |
| **Long-term (Persistent)** | Across sessions | Repository or user | Copilot Memory, custom instructions |
| **External (Artifact)** | Permanent until deleted | Repository | Files, docs, databases, issues |

### 1.2 Short-Term Memory

**Characteristics:**
- Limited by context window size (token limits)
- Exists only during a single interaction/session
- Includes: current conversation, loaded files, tool outputs
- Lost when session ends

**When to use:**
- Task-specific context that won't be needed again
- Intermediate reasoning steps
- Temporary exploration results

**Managing short-term memory:**
```
Context Window Budget:
┌──────────────────────────────────────────────┐
│ System Prompt (fixed)          ~10%          │
│ Custom Instructions (fixed)    ~5%           │
│ Loaded Memory/Facts (dynamic)  ~15%          │
│ Conversation History           ~30%          │
│ Current Task Context           ~25%          │
│ Reserved for Response          ~15%          │
└──────────────────────────────────────────────┘
```

### 1.3 Long-Term Memory

**Copilot Memory (GitHub-native):**
- **Repository-level facts**: Coding conventions, architectural decisions, build commands
- **User-level preferences**: Personal coding style, tool preferences
- Auto-expires after 28 days of non-use
- Validated against current codebase before use
- Created only by users with write access + Memory enabled

**Custom Instructions (`.github/copilot-instructions.md`):**
```markdown
# Repository Custom Instructions

## Architecture
- This project uses Clean Architecture with CQRS
- Domain layer has no external dependencies
- All API responses use the Result<T, Error> pattern

## Conventions
- Use PascalCase for public members
- All async methods must have CancellationToken parameter
- Never use string interpolation for SQL queries
```

### 1.4 External Memory

**Artifact-based memory:**
- README files, ADRs (Architecture Decision Records)
- Issue templates, PR templates
- Configuration files (`.editorconfig`, `copilot-instructions.md`)
- Wiki pages and documentation

### 1.5 Choose the Right Memory Type

| Scenario | Best Memory Type | Rationale |
|----------|-----------------|-----------|
| Current refactoring context | Short-term | Only needed for this task |
| "We use tabs, not spaces" | Long-term (Custom Instructions) | Applies to all future work |
| Database schema decisions | External (ADR file) | Must survive beyond any agent interaction |
| User prefers concise responses | Long-term (User preference) | Personal across all repos |
| Intermediate API response | Short-term | Temporary data for current step |

### 1.6 Scope Agent Memory to Task-Relevant Information

**Problem**: Agents that load too much context become confused or hit token limits.

**Solutions:**
- Only load files directly relevant to the current task
- Use targeted search (grep/glob) rather than loading entire directories
- Summarize large contexts before including them
- Separate "reference" information from "active" information

### 1.7 Define Memory Expiration, Pruning, and Reset Rules

| Rule | Copilot Memory Behavior | Custom Configuration |
|------|------------------------|---------------------|
| **Expiration** | 28 days of non-use | Set per-fact TTL in custom systems |
| **Pruning** | Automatic (unused facts removed) | Implement relevance scoring |
| **Reset** | Manual deletion by owner/user | Clear on new major version |
| **Validation** | Facts checked against current code | Automated staleness detection |

---

## Section 2: Persist Agent State and Manage Context Drift

### 2.1 Capture Task Progress as Durable Artifacts

**Patterns for persisting agent state:**

```yaml
# State captured in a plan artifact (plan.md or similar)
task_state:
  id: "refactor-auth-module"
  started_at: "2024-01-15T10:00:00Z"
  current_step: 3
  total_steps: 7
  completed_steps:
    - { id: 1, description: "Analyzed existing auth code", status: "done" }
    - { id: 2, description: "Created JWT utility", status: "done" }
    - { id: 3, description: "Updated middleware", status: "in_progress" }
  decisions:
    - "Using RS256 for JWT signing (team preference)"
    - "Keeping backward compatibility with session tokens for 30 days"
  blocked_on: null
```

**Where to persist state:**
- **Git commits** — Each meaningful step is a commit with descriptive message
- **PR description** — Running summary of progress and decisions
- **Issue comments** — Status updates for long-running tasks
- **Workflow artifacts** — Structured data files (JSON/YAML)
- **Branch state** — The branch itself represents current progress

### 2.2 Resume Agent Work Without Repeating Steps

**Resumption Strategy:**
```
1. Load the last known state artifact
2. Verify which steps are actually complete (check file existence, test results)
3. Identify the next pending step
4. Resume from that point with full context of prior decisions
5. Do NOT re-execute completed steps unless validation fails
```

**Implementation in Copilot SDK:**
```typescript
// Resume pattern: load prior state before starting
const priorState = await loadArtifact("task-state.json");

const session = await client.createSession({
  prompt: `You are resuming a task. Prior state:
    - Steps 1-3 are complete
    - Decision: Using RS256 for JWT
    - Current step: 4 (Update user model)
    Resume from step 4. Do not redo steps 1-3.`,
});
```

### 2.3 Detect and Correct Drift During Extended Execution

**What is context drift?**
When an agent's understanding diverges from the actual state of the codebase during a long-running task.

**Causes of drift:**
- Other developers push changes while agent is working
- Agent's assumptions become invalid over time
- Agent loses track of its own changes
- Context window fills with irrelevant information

**Detection strategies:**
| Strategy | Implementation |
|----------|---------------|
| **Periodic validation** | Re-run tests/lints at checkpoints |
| **Git status checks** | Compare expected vs actual file state |
| **Conflict detection** | `git fetch` + `git diff` against upstream |
| **Assertion checks** | Verify assumptions before each step |

**Correction strategies:**
- Rebase on latest main before continuing
- Re-read affected files to refresh context
- Re-validate prior decisions against current state
- If drift is severe, restart from last known-good checkpoint

---

## Section 3: Ensure Continuity Across Tools and Environments

### 3.1 Share Agent State

**Cross-tool state sharing patterns:**

| Pattern | Mechanism | Use Case |
|---------|-----------|----------|
| **File-based** | Shared files in repository | Agent in IDE ↔ Agent in CI |
| **PR-based** | PR description/comments | Cloud agent ↔ Human reviewer |
| **Issue-based** | Issue body/comments | Task tracking across sessions |
| **Environment variables** | `GITHUB_ENV`, secrets | Between workflow steps |
| **Artifacts** | GitHub Actions artifacts | Between workflow jobs |

```yaml
# Sharing state between workflow jobs
jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      plan-json: ${{ steps.plan.outputs.result }}
    steps:
      - id: plan
        run: echo "result=$(cat plan.json | jq -c)" >> $GITHUB_OUTPUT

  execute:
    needs: plan
    runs-on: ubuntu-latest
    steps:
      - run: echo '${{ needs.plan.outputs.plan-json }}' | jq .
```

### 3.2 Prevent Conflicting Context

**Problem**: Multiple agents or tools provide contradictory information.

**Solutions:**
- **Single source of truth**: Designate one authoritative source per concern
- **Conflict resolution rules**: "Most recent wins" or "Most specific wins"
- **Locking mechanisms**: File-level or branch-level locks during agent execution
- **Merge strategies**: Use Git's conflict resolution for overlapping changes

```yaml
# Prevent conflicts with concurrency controls
concurrency:
  group: agent-${{ github.ref }}
  cancel-in-progress: false  # Don't cancel, queue instead
```

### 3.3 Prevent Stale Context

**Staleness indicators:**
- Files loaded into context have been modified since loading
- Referenced branches have new commits
- External APIs return different schemas
- Configuration values have changed

**Prevention mechanisms:**
- **Freshness checks**: Re-read files before acting on them
- **TTL on loaded context**: Refresh every N minutes for long tasks
- **Watch for events**: Subscribe to push events, PR updates
- **Validate before commit**: Final check that assumptions still hold

---

## 🧠 Key Memorization Points

1. **Three memory types**: Short-term (session), Long-term (persistent), External (artifacts)
2. **Copilot Memory**: Repository facts + User preferences, auto-expires in 28 days
3. **Custom instructions** = `.github/copilot-instructions.md`
4. **Context drift** = agent's understanding diverges from actual state
5. **Drift detection**: periodic validation, git status checks, conflict detection
6. **State persistence**: commits, PR descriptions, artifacts, issue comments
7. **Resume pattern**: Load state → Verify completion → Identify next step → Continue
8. **Prevent conflicts**: Single source of truth, locking, concurrency controls
9. **Prevent staleness**: Freshness checks, TTL, event subscriptions
10. **Memory scoping**: Only load task-relevant information to avoid context pollution

---

## 📝 Self-Assessment Questions

1. What are the three types of agent memory and when do you use each?
2. How does Copilot Memory validate stored facts before using them?
3. What is context drift and how do you detect it?
4. Describe a pattern for resuming agent work without repeating completed steps.
5. How do you prevent conflicting context when multiple agents work simultaneously?
6. What is the expiration policy for Copilot Memory?
7. Where should you persist agent state for cross-session continuity?
8. How do you scope agent memory to avoid context pollution?
9. What mechanisms prevent stale context in long-running agent tasks?
10. How do custom instructions differ from Copilot Memory?

➡️ **Practice these concepts with exercises**: [03-exercises-memory-state.md](./exercises/03-exercises-memory-state.md)
