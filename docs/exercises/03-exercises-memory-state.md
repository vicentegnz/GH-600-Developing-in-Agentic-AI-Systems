# Exercises: Domain 3 — Memory, State, and Execution

## Exercise 3.1: Memory Type Classification

**Task**: For each scenario, identify the best memory type and justify your choice.

| # | Scenario | Memory Type | Justification |
|---|----------|-------------|---------------|
| 1 | "Our team uses 2-space indentation" | | |
| 2 | Agent is mid-task analyzing a complex function | | |
| 3 | The database uses UUID v7 for primary keys | | |
| 4 | Agent found 15 files that need refactoring | | |
| 5 | User prefers verbose commit messages | | |
| 6 | The CI pipeline requires Node.js 20 | | |
| 7 | Agent's current conversation context | | |
| 8 | Architecture Decision: "We chose PostgreSQL over MongoDB" | | |

<details>
<summary>✅ Answers</summary>

| # | Scenario | Memory Type | Justification |
|---|----------|-------------|---------------|
| 1 | 2-space indentation | Long-term (Custom Instructions or Copilot Memory) | Applies to all future work in this repo |
| 2 | Analyzing a complex function | Short-term (Working memory) | Only needed for current task |
| 3 | UUID v7 for PKs | Long-term (Copilot Memory - repo fact) | Architectural pattern that persists |
| 4 | 15 files needing refactoring | External (Issue/artifact) | Needs to survive across sessions |
| 5 | Verbose commit messages | Long-term (Copilot Memory - user pref) | Personal preference across repos |
| 6 | Node.js 20 required | Long-term (Custom Instructions) | Build requirement that applies always |
| 7 | Current conversation | Short-term (Session context) | Ephemeral, lost when session ends |
| 8 | PostgreSQL choice | External (ADR document) | Major decision that must be documented permanently |

</details>

---

## Exercise 3.2: Write Custom Instructions

**Scenario**: You're setting up a new repository for a Go microservice. Write a `.github/copilot-instructions.md` that covers:
- Architecture pattern
- Error handling conventions
- Testing requirements
- Naming conventions
- Prohibited patterns

<details>
<summary>✅ Sample Answer</summary>

```markdown
# Repository Custom Instructions

## Architecture
- This is a Go microservice using Clean Architecture (ports & adapters)
- Layer order: Handler → Service → Repository
- Dependency injection via constructor functions, no globals
- All interfaces defined in the consuming package

## Error Handling
- Always wrap errors with context: `fmt.Errorf("operation failed: %w", err)`
- Use custom error types for domain errors (see `internal/domain/errors.go`)
- Never swallow errors silently
- HTTP handlers map domain errors to appropriate status codes
- Log at the handler level only, not in services or repositories

## Testing
- Table-driven tests for all public functions
- Test file naming: `*_test.go` in the same package
- Use testify/assert for assertions
- Mock interfaces using counterfeiter (see `go generate` comments)
- Minimum 80% coverage for service layer
- Integration tests use Docker Compose (see `docker-compose.test.yml`)

## Naming Conventions
- Package names: lowercase, single word, no underscores
- Interfaces: verb-based (e.g., `UserFinder`, not `IUserService`)
- Exported functions: PascalCase with descriptive verbs
- Unexported helpers: camelCase
- Constants: PascalCase (not SCREAMING_SNAKE)
- Context key types: unexported struct type (not string)

## Prohibited Patterns
- NO `init()` functions (use explicit initialization)
- NO `panic()` in library/service code (only in main if unrecoverable)
- NO direct SQL string concatenation (use parameterized queries)
- NO importing `internal/` from other services
- NO global state or package-level variables (except for `var _ Interface = (*Struct)(nil)`)
- NO `any`/`interface{}` in public APIs without strong justification
```

</details>

---

## Exercise 3.3: Context Drift Detection

**Scenario**: An agent has been working on a refactoring task for 30 minutes. During this time:
- Another developer pushed 3 commits to `main`
- One of those commits modified a file the agent is about to edit
- The agent's plan references a function signature that was changed

**Questions:**
1. What type of drift has occurred?
2. How should the agent detect this?
3. What's the correct recovery action?
4. How could this have been prevented?

<details>
<summary>✅ Answers</summary>

**1. Type of drift:** Context drift caused by upstream changes (stale file content + invalidated assumptions)

**2. Detection methods:**
- `git fetch origin main && git diff HEAD...origin/main` before each edit
- Check file modification timestamps against when they were loaded
- Re-read target files before applying edits
- Validate function signatures match expectations before modifying callers

**3. Recovery action:**
- Stop current execution
- `git rebase origin/main` (or merge) to incorporate new changes
- Re-read affected files to update context
- Re-validate the plan against updated code
- If the plan is still valid → continue from current step
- If plan is invalidated → regenerate affected plan steps
- If conflicts are complex → escalate to human

**4. Prevention measures:**
- Short-lived tasks (reduce exposure window)
- Lock affected files during agent execution (branch isolation)
- Periodic freshness checks (every N steps, fetch and compare)
- Subscribe to push events and pause if affected files change
- Use concurrency controls to prevent parallel work on same files

</details>

---

## Exercise 3.4: State Persistence Design

**Task**: Design a state persistence strategy for an agent performing a large-scale migration (200 files, estimated 2 hours). The agent may be interrupted (timeout, error, infrastructure restart).

Define:
1. What state to persist
2. Where to persist it
3. How to resume after interruption
4. How to detect which steps completed

<details>
<summary>✅ Sample Answer</summary>

**1. State to persist:**
```yaml
migration_state:
  task_id: "migrate-api-v2-to-v3"
  total_files: 200
  completed_files: ["src/api/users.ts", "src/api/products.ts", ...]
  current_file: "src/api/orders.ts"
  current_step: "update_imports"  
  decisions_made:
    - "Using named exports (confirmed by team)"
    - "Keeping deprecated functions with @deprecated tag"
  errors_encountered:
    - { file: "src/api/legacy.ts", error: "circular dependency", resolution: "skipped" }
  started_at: "2024-01-15T10:00:00Z"
  last_checkpoint: "2024-01-15T10:45:00Z"
```

**2. Where to persist:**
- **Primary**: Git commits (one per batch of files) — serves as both state and progress
- **Secondary**: Workflow artifact (`migration-state.json`) — updated every 5 files
- **Tertiary**: PR description (human-readable progress) — updated every 20 files

**3. How to resume:**
```
1. Load migration-state.json from latest artifact
2. List all .ts files in migration scope
3. Diff against completed_files list
4. For completed files: verify changes are committed (git show)
5. For current_file: check if partially modified
   - If partially modified → rollback that file, restart it
   - If not modified → start fresh
6. Continue with remaining files
7. Apply same decisions_made (don't re-ask)
```

**4. Detecting completion:**
- **Committed files** = completed (git log shows migration commit for that file)
- **Modified but uncommitted** = in-progress (needs verification or rollback)
- **Unmodified** = not started
- **Validation**: Run full test suite on completed batch to confirm correctness

</details>

---

## Exercise 3.5: Cross-Tool State Sharing

**Scenario**: Three tools/environments need to share state:
1. **Copilot Cloud Agent** (creates the PR)
2. **GitHub Actions CI** (runs tests on the PR)
3. **Copilot Code Review** (reviews the PR)

Design how state flows between them:

<details>
<summary>✅ Sample Answer</summary>

```
┌────────────────────┐
│ Copilot Cloud Agent│
│                    │
│ Creates:           │
│ - PR with changes  │
│ - PR description   │──────────┐
│   (plan + context) │          │
│ - Commit messages   │          │
│   (decision log)   │          │
└────────┬───────────┘          │
         │                       │
         │ Triggers:             │ Shares via:
         │ pull_request event    │ PR description
         ▼                       │ Commit messages
┌────────────────────┐          │ Labels
│ GitHub Actions CI  │          │
│                    │          │
│ Reads:             │          │
│ - PR metadata      │          │
│ - Branch changes   │          │
│                    │          │
│ Produces:          │          │
│ - Test results     │──────────│──── Shared via:
│ - Coverage report  │          │     Status checks
│ - Lint report      │          │     PR comments
│ - Artifacts        │          │     Workflow artifacts
└────────┬───────────┘          │
         │                       │
         │ Triggers:             │
         │ check_suite complete  │
         ▼                       ▼
┌────────────────────────────────────┐
│ Copilot Code Review                │
│                                    │
│ Reads:                             │
│ - PR diff (from PR)                │
│ - PR description (agent's context) │
│ - Test results (from CI artifacts) │
│ - Copilot Memory (repo facts)      │
│                                    │
│ Produces:                          │
│ - Review comments on PR            │
│ - Approval or request changes      │
└────────────────────────────────────┘
```

**State sharing mechanisms:**
| From → To | Mechanism | Data |
|-----------|-----------|------|
| Agent → CI | PR event trigger | Changed files, branch name |
| Agent → Review | PR description | Plan, decisions, context |
| CI → Review | Status checks + artifacts | Test/lint results |
| Review → Agent | Review comments | Feedback for iteration |
| All → Future | Copilot Memory | Facts learned during process |

</details>

---

## Exercise 3.6: Memory Pruning Strategy

**Scenario**: An agent's context window is filling up during a complex task. It has loaded:
- 15 source files (some relevant, some not)
- Full git log (500 lines)
- Previous conversation (40 turns)
- 3 large JSON API responses
- Project README (2000 words)

**Task**: Design a pruning strategy. What do you keep, summarize, or drop?

<details>
<summary>✅ Sample Answer</summary>

| Content | Action | Rationale |
|---------|--------|-----------|
| 15 source files | **Keep 5 most relevant, drop 10** | Only files being directly modified/referenced |
| Full git log | **Summarize to last 10 relevant commits** | Full log is noise; keep recent + relevant |
| Conversation (40 turns) | **Keep last 10 turns, summarize earlier** | Recent context matters most; compress history |
| 3 JSON responses | **Extract key fields, drop raw responses** | Only specific values needed, not full payloads |
| README | **Drop** (move to long-term memory) | Already internalized; can be re-read if needed |

**Priority order for context window:**
```
1. [KEEP] Current task definition and constraints
2. [KEEP] Files being actively modified
3. [KEEP] Recent conversation turns (last 10)
4. [SUMMARIZE] Older conversation (key decisions only)
5. [SUMMARIZE] Reference files (function signatures only)
6. [EXTRACT] API responses (relevant fields only)
7. [DROP] Already-completed exploration results
8. [DROP] General documentation (available on re-read)
```

**Pruning triggers:**
- Context usage > 70% → Start summarizing old turns
- Context usage > 85% → Drop non-essential files
- Context usage > 95% → Emergency: keep only current step context

</details>

---

## Exercise 3.7: Preventing Stale Context

**Task**: Write a pre-edit validation function that checks for stale context before the agent modifies a file.

```typescript
// Complete this function:
async function validateFreshness(
  filePath: string, 
  expectedContent: string
): Promise<ValidationResult> {
  // Your implementation here
}

interface ValidationResult {
  isFresh: boolean;
  action: 'proceed' | 'refresh' | 'abort';
  reason?: string;
}
```

<details>
<summary>✅ Sample Answer</summary>

```typescript
import { readFile } from 'fs/promises';
import { execSync } from 'child_process';

interface ValidationResult {
  isFresh: boolean;
  action: 'proceed' | 'refresh' | 'abort';
  reason?: string;
}

async function validateFreshness(
  filePath: string,
  expectedContent: string
): Promise<ValidationResult> {
  // Step 1: Check if file still exists
  try {
    const currentContent = await readFile(filePath, 'utf-8');
    
    // Step 2: Check if content matches what we loaded
    if (currentContent === expectedContent) {
      return { isFresh: true, action: 'proceed' };
    }
    
    // Step 3: Content differs - check if upstream changed
    const upstreamChanged = execSync(
      `git diff origin/main -- "${filePath}"`,
      { encoding: 'utf-8' }
    ).trim().length > 0;
    
    if (upstreamChanged) {
      // Someone else changed this file - need to refresh context
      return {
        isFresh: false,
        action: 'refresh',
        reason: `File modified upstream since loaded. Diff length: ${currentContent.length - expectedContent.length} chars`
      };
    }
    
    // Step 4: We changed it ourselves (from earlier step) - that's ok
    const weChangedIt = execSync(
      `git diff --name-only HEAD -- "${filePath}"`,
      { encoding: 'utf-8' }
    ).trim().length > 0;
    
    if (weChangedIt) {
      return { isFresh: true, action: 'proceed' };
    }
    
    // Unknown change source - abort to be safe
    return {
      isFresh: false,
      action: 'abort',
      reason: 'File content differs from expected but change source is unknown'
    };
    
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === 'ENOENT') {
      return {
        isFresh: false,
        action: 'abort',
        reason: 'File no longer exists'
      };
    }
    throw error;
  }
}
```

</details>
