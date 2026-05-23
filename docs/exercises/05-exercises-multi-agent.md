# Exercises: Domain 5 — Multi-Agent Coordination

## Exercise 5.1: Choose the Orchestration Pattern

**Task**: For each scenario, choose the best orchestration pattern and justify your choice.

| # | Scenario | Pattern | Justification |
|---|----------|---------|---------------|
| 1 | Three agents: one plans features, one writes code, one reviews code | | |
| 2 | An agent reviews every PR, another handles deployments, another triages issues | | |
| 3 | A complex refactoring needs frontend, backend, and database changes simultaneously | | |
| 4 | A main agent breaks a large task into subtasks and assigns them to specialized workers | | |

<details>
<summary>✅ Answers</summary>

| # | Scenario | Pattern | Justification |
|---|----------|---------|---------------|
| 1 | Plan → Code → Review | **Sequential Pipeline** | Each output feeds the next; order matters; code needs plan, review needs code |
| 2 | PR review, deploy, triage | **Event-Driven** | Independent triggers (PR event, merge event, issue event); no coordination needed |
| 3 | Frontend + Backend + DB simultaneously | **Parallel Fan-Out/Fan-In** | Independent work streams that merge; reduces total time |
| 4 | Main breaks task into subtasks | **Hierarchical** | Central orchestrator delegates; workers report back; orchestrator integrates |

</details>

---

## Exercise 5.2: Design Agent Isolation

**Scenario**: You have a monorepo with this structure:
```
src/
├── frontend/     (React app)
├── backend/      (Node.js API)
├── shared/       (Shared types)
├── database/     (Migrations)
└── infrastructure/ (Terraform)
```

Three agents will work in parallel. Design their isolation boundaries:

| Agent | Scope | Allowed Paths | Forbidden Paths | Branch |
|-------|-------|---------------|-----------------|--------|
| `frontend-agent` | | | | |
| `backend-agent` | | | | |
| `infra-agent` | | | | |

Also define: What happens when two agents need to modify `src/shared/`?

<details>
<summary>✅ Sample Answer</summary>

| Agent | Scope | Allowed Paths | Forbidden Paths | Branch |
|-------|-------|---------------|-----------------|--------|
| `frontend-agent` | UI components, pages, styles | `src/frontend/**`, `src/shared/types/ui/**` | `src/backend/`, `src/database/`, `src/infrastructure/` | `agent/frontend-{task-id}` |
| `backend-agent` | API routes, services, models | `src/backend/**`, `src/shared/types/api/**` | `src/frontend/`, `src/database/`, `src/infrastructure/` | `agent/backend-{task-id}` |
| `infra-agent` | Infrastructure configs | `src/infrastructure/**`, `src/database/**` | `src/frontend/`, `src/backend/` | `agent/infra-{task-id}` |

**Shared directory conflict resolution:**

When two agents need `src/shared/`:
1. **Prevention**: Subdivide shared into owned sections (`shared/types/ui/`, `shared/types/api/`)
2. **Sequencing**: If both need the same file, serialize — first agent to claim it wins, second waits
3. **Merge strategy**: After parallel execution, integration job merges branches and runs full tests
4. **Conflict protocol**: If git merge fails on shared files:
   - Automated: Try `git merge -X ours` for additive changes (new types)
   - Manual: Escalate to human if changes conflict within same type

```yaml
# Integration job
jobs:
  integrate:
    needs: [frontend-agent, backend-agent, infra-agent]
    steps:
      - name: Merge all agent branches
        run: |
          git checkout -b agent/integrated
          git merge agent/frontend-${{ env.TASK_ID }} || exit 1
          git merge agent/backend-${{ env.TASK_ID }} || exit 1
          git merge agent/infra-${{ env.TASK_ID }} || exit 1
          
      - name: Run full test suite
        run: npm test
        
      - name: Create integrated PR
        if: success()
        run: gh pr create --title "Integrated agent changes" --base main
```

</details>

---

## Exercise 5.3: Conflict Detection and Resolution

**Scenario**: Two agents ran in parallel and produced these changes:

**Agent A** (backend) modified `src/shared/types.ts`:
```typescript
// Added:
export interface UserResponse {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}
```

**Agent B** (frontend) modified `src/shared/types.ts`:
```typescript
// Added:
export interface UserResponse {
  id: string;
  displayName: string;
  email: string;
  avatarUrl: string;
}
```

**Questions:**
1. What type of conflict is this?
2. How would you detect it automatically?
3. What are three possible resolution strategies?
4. Which strategy would you recommend and why?

<details>
<summary>✅ Answers</summary>

**1. Conflict type:** Contradictory outputs — both agents defined the same interface with different fields

**2. Automatic detection:**
- Git merge will fail (conflicting changes to same lines in same file)
- TypeScript compiler will error on duplicate interface declarations
- Integration test job can catch: `git merge agent/backend agent/frontend` → CONFLICT

**3. Resolution strategies:**

| Strategy | Implementation | Pros | Cons |
|----------|---------------|------|------|
| **A. Union merge** | Combine all fields from both interfaces | No data loss | May have incompatible fields (`name` vs `displayName`) |
| **B. Owner wins** | Backend owns API types, frontend adapts | Clear ownership | Frontend may need mapping layer |
| **C. Human arbitration** | Flag conflict, pause, wait for decision | Best outcome | Slow |

**4. Recommended: B (Owner wins) + adapter pattern**

Rationale:
- Backend agent owns the API contract (it defines what the server returns)
- Frontend agent should map/adapt to whatever the API provides
- This follows the "single source of truth" principle

Resolution:
```typescript
// src/shared/types.ts (Backend's version wins)
export interface UserResponse {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

// src/frontend/adapters/user.ts (Frontend adapts)
import { UserResponse } from '@shared/types';

export interface DisplayUser {
  id: string;
  displayName: string;  // mapped from name
  email: string;
  avatarUrl: string;    // derived or fetched separately
}

export function toDisplayUser(user: UserResponse): DisplayUser {
  return {
    id: user.id,
    displayName: user.name,
    email: user.email,
    avatarUrl: `https://avatars.example.com/${user.id}`,
  };
}
```

</details>

---

## Exercise 5.4: Multi-Agent Workflow YAML

**Task**: Write a complete GitHub Actions workflow that orchestrates three agents:
1. **Planner** (reads issue, produces plan)
2. **Implementers** (2 parallel agents: frontend + backend)
3. **Integrator** (merges results, runs tests, creates PR)

Include: error handling, timeouts, and conflict detection.

<details>
<summary>✅ Sample Answer</summary>

```yaml
name: Multi-Agent Development Pipeline
on:
  issues:
    types: [labeled]

jobs:
  # Phase 1: Planning (sequential)
  plan:
    if: contains(github.event.label.name, 'agent-implement')
    runs-on: ubuntu-latest
    timeout-minutes: 10
    outputs:
      plan-json: ${{ steps.generate.outputs.plan }}
      frontend-scope: ${{ steps.parse.outputs.frontend }}
      backend-scope: ${{ steps.parse.outputs.backend }}
    steps:
      - uses: actions/checkout@v4
      - name: Generate plan from issue
        id: generate
        run: |
          # Planner agent analyzes issue and produces structured plan
          echo "plan={...}" >> $GITHUB_OUTPUT
      - name: Parse plan into scopes
        id: parse
        run: |
          echo "frontend=$(echo '${{ steps.generate.outputs.plan }}' | jq -c '.frontend')" >> $GITHUB_OUTPUT
          echo "backend=$(echo '${{ steps.generate.outputs.plan }}' | jq -c '.backend')" >> $GITHUB_OUTPUT

  # Phase 2: Parallel Implementation
  implement-frontend:
    needs: plan
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - name: Create agent branch
        run: git checkout -b agent/frontend-${{ github.event.issue.number }}
      - name: Frontend agent executes
        id: frontend
        continue-on-error: true
        env:
          SCOPE: ${{ needs.plan.outputs.frontend-scope }}
        run: |
          # Frontend agent implements its scope
          git add -A
          git commit -m "feat(frontend): agent implementation"
          git push origin agent/frontend-${{ github.event.issue.number }}
      - name: Report failure
        if: steps.frontend.outcome == 'failure'
        run: echo "::error::Frontend agent failed"

  implement-backend:
    needs: plan
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - name: Create agent branch
        run: git checkout -b agent/backend-${{ github.event.issue.number }}
      - name: Backend agent executes
        id: backend
        continue-on-error: true
        env:
          SCOPE: ${{ needs.plan.outputs.backend-scope }}
        run: |
          # Backend agent implements its scope
          git add -A
          git commit -m "feat(backend): agent implementation"
          git push origin agent/backend-${{ github.event.issue.number }}
      - name: Report failure
        if: steps.backend.outcome == 'failure'
        run: echo "::error::Backend agent failed"

  # Phase 3: Integration
  integrate:
    needs: [plan, implement-frontend, implement-backend]
    if: always() && needs.plan.result == 'success'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Merge agent branches
        id: merge
        continue-on-error: true
        run: |
          git checkout -b agent/integrated-${{ github.event.issue.number }}
          
          # Try merging frontend
          if git ls-remote --heads origin agent/frontend-${{ github.event.issue.number }} | grep -q .; then
            git merge origin/agent/frontend-${{ github.event.issue.number }} --no-edit || {
              echo "merge_conflict=frontend" >> $GITHUB_OUTPUT
              exit 1
            }
          fi
          
          # Try merging backend
          if git ls-remote --heads origin agent/backend-${{ github.event.issue.number }} | grep -q .; then
            git merge origin/agent/backend-${{ github.event.issue.number }} --no-edit || {
              echo "merge_conflict=backend" >> $GITHUB_OUTPUT
              exit 1
            }
          fi
          
      - name: Run integration tests
        if: steps.merge.outcome == 'success'
        run: npm test
        
      - name: Create PR
        if: steps.merge.outcome == 'success'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          git push origin agent/integrated-${{ github.event.issue.number }}
          gh pr create \
            --title "Agent: Implement #${{ github.event.issue.number }}" \
            --body "Multi-agent implementation. Closes #${{ github.event.issue.number }}" \
            --base main
            
      - name: Escalate conflicts
        if: steps.merge.outcome == 'failure'
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: '⚠️ **Multi-agent conflict detected**\n\nAgents produced conflicting changes. Human intervention needed.\n\nConflict in: ${{ steps.merge.outputs.merge_conflict }}'
            });
```

</details>

---

## Exercise 5.5: Post-Hoc Analysis

**Scenario**: A multi-agent workflow completed. Here are the results:

| Agent | Started | Completed | Status | Files Changed | Conflicts |
|-------|---------|-----------|--------|---------------|-----------|
| Planner | 10:00 | 10:03 | ✅ Success | 0 (plan only) | 0 |
| Frontend | 10:03 | 10:25 | ✅ Success | 12 | 0 |
| Backend | 10:03 | 10:08 | ✅ Success | 5 | 0 |
| Test-Writer | 10:03 | 10:45 | ⚠️ Partial | 3 (of 8 expected) | 0 |
| Integrator | 10:45 | 10:48 | ❌ Failed | 0 | 2 conflicts |

**Questions:**
1. What's the total execution time vs. theoretical minimum?
2. Which agent is the bottleneck?
3. Why might the test-writer have produced partial output?
4. What caused the integrator to fail?
5. How would you improve this workflow?

<details>
<summary>✅ Answers</summary>

**1. Total time:** 48 minutes (10:00 → 10:48)
- Theoretical minimum: ~25 min (plan 3min + max(frontend 22min, backend 5min, tests 42min) = 3 + 42 = 45min... actually the test-writer is the bottleneck)
- The bottleneck makes parallel execution nearly as slow as sequential for this run

**2. Bottleneck:** Test-Writer agent (42 minutes, and only partial completion)

**3. Possible reasons for partial output:**
- Timeout limit hit (configured max execution time)
- Context window exhausted after writing 3 tests
- Dependencies on frontend/backend changes that weren't yet available
- Encountered errors writing certain tests and exhausted retries

**4. Integrator failure cause:**
- 2 merge conflicts likely between frontend (12 files) and test-writer (3 files)
- Or between frontend and backend on shared types
- The partial test-writer output may reference code that frontend changed

**5. Improvements:**
- **Test-writer**: Increase timeout, or split into multiple smaller test-writing agents
- **Sequencing**: Make test-writer depend on frontend+backend completion (needs their changes)
- **Conflict prevention**: Pre-assign file ownership more carefully
- **Fallback**: If integrator fails, auto-attempt `git mergetool` or escalate immediately
- **Monitoring**: Add 10-minute checkpoint alerts for long-running agents

</details>

---

## Exercise 5.6: Agent Lifecycle Management

**Task**: You need to replace `legacy-review-agent` with `smart-review-agent-v2` in a production multi-agent workflow. Design the migration plan.

Requirements:
- Zero downtime (workflow must continue operating during migration)
- Rollback capability if v2 performs worse
- Audit trail of the transition

<details>
<summary>✅ Sample Answer</summary>

```yaml
# Migration Plan: legacy-review-agent → smart-review-agent-v2

phase_1_prepare:
  duration: "1 week"
  actions:
    - Deploy smart-review-agent-v2 alongside legacy (both active)
    - Route 0% traffic to v2 (shadow mode — runs but output not used)
    - Compare outputs between legacy and v2 on same PRs
    - Collect quality metrics for both
    
phase_2_canary:
  duration: "1 week"
  actions:
    - Route 10% of new PRs to v2 (real reviews)
    - Route 90% to legacy
    - Monitor: quality score, false positive rate, developer satisfaction
    - Criteria to proceed: v2 quality ≥ legacy quality
    - Rollback trigger: v2 quality < 80% of legacy or any critical failure
    
phase_3_ramp:
  duration: "1 week"
  actions:
    - Increase v2 to 50%, then 75%, then 100%
    - Legacy remains deployed but inactive (standby)
    - Continue monitoring all metrics
    - Rollback: Revert routing to legacy within 5 minutes
    
phase_4_retire:
  duration: "After 2 weeks stable at 100%"
  actions:
    - Archive legacy-review-agent configuration
    - Document retirement: date, reason, replacement
    - Remove legacy from active deployment
    - Keep audit logs for compliance retention period (2 years)
    - Update workflow documentation
    
rollback_procedure:
  trigger: "v2 quality score drops below threshold OR critical failure"
  steps:
    1. Immediately route all traffic back to legacy
    2. Notify team via Slack/email
    3. Capture v2 failure logs for analysis
    4. Create issue documenting what went wrong
    5. Fix v2, return to phase_2_canary
    
audit_trail:
  - Configuration versions (git history)
  - Routing decisions (workflow logs)
  - Performance comparison data (artifacts)
  - Retirement decision record (issue/ADR)
  - All v2 outputs during canary (stored as workflow artifacts)
```

</details>

---

## Exercise 5.7: Handoff Protocol Design

**Task**: Design a handoff protocol between a "Research Agent" and an "Implementation Agent". Define exactly what data passes between them.

<details>
<summary>✅ Sample Answer</summary>

```typescript
interface AgentHandoff {
  metadata: {
    handoff_id: string;
    from_agent: string;
    to_agent: string;
    timestamp: string;
    task_id: string;
  };
  
  context: {
    // What the research agent discovered
    relevant_files: Array<{
      path: string;
      purpose: string;
      key_exports: string[];
    }>;
    
    existing_patterns: Array<{
      description: string;
      example_file: string;
      example_lines: [number, number];
    }>;
    
    dependencies: Array<{
      package: string;
      version: string;
      usage: string;
    }>;
    
    constraints_discovered: string[];
  };
  
  plan: {
    objective: string;
    steps: Array<{
      id: number;
      action: string;
      target_file: string;
      details: string;
      risk: 'low' | 'medium' | 'high';
    }>;
    
    decisions_made: Array<{
      decision: string;
      rationale: string;
      alternatives_considered: string[];
    }>;
  };
  
  instructions: {
    do: string[];      // Explicit instructions
    do_not: string[];  // Explicit prohibitions
    ask_if: string[];  // Conditions that need human input
  };
  
  validation: {
    expected_test_command: string;
    success_indicators: string[];
    failure_indicators: string[];
  };
}

// Example:
const handoff: AgentHandoff = {
  metadata: {
    handoff_id: "hf-20240115-001",
    from_agent: "researcher",
    to_agent: "implementer",
    timestamp: "2024-01-15T10:05:00Z",
    task_id: "issue-42-add-caching",
  },
  context: {
    relevant_files: [
      { path: "src/db/queries.ts", purpose: "Database query layer", key_exports: ["findUser", "findProducts"] },
      { path: "src/cache/redis.ts", purpose: "Existing Redis wrapper", key_exports: ["RedisClient", "CacheConfig"] },
    ],
    existing_patterns: [
      { description: "Cache-aside pattern used in product queries", example_file: "src/db/products.ts", example_lines: [45, 62] },
    ],
    dependencies: [
      { package: "ioredis", version: "5.3.0", usage: "Redis client already in project" },
    ],
    constraints_discovered: ["Redis TTL must be < 5 minutes per team policy", "Cache keys must be prefixed with service name"],
  },
  plan: {
    objective: "Add Redis caching to user database queries",
    steps: [
      { id: 1, action: "Create cache key generator", target_file: "src/cache/keys.ts", details: "Prefix: 'user-service:'", risk: "low" },
      { id: 2, action: "Add cache wrapper to findUser", target_file: "src/db/queries.ts", details: "TTL: 3 minutes", risk: "medium" },
    ],
    decisions_made: [
      { decision: "Use existing ioredis, not new package", rationale: "Already in project, team familiar", alternatives_considered: ["node-cache (in-memory)", "new redis package"] },
    ],
  },
  instructions: {
    do: ["Follow cache-aside pattern from products.ts", "Add cache invalidation on user update"],
    do_not: ["Don't cache sensitive user data (passwords, tokens)", "Don't modify the Redis connection config"],
    ask_if: ["If cache hit rate optimization seems needed (out of scope)"],
  },
  validation: {
    expected_test_command: "npm test -- --grep 'user cache'",
    success_indicators: ["All tests pass", "Cache hit reduces query time by >50%"],
    failure_indicators: ["Redis connection errors", "Stale data served after update"],
  },
};
```

</details>
