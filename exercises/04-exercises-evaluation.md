# Exercises: Domain 4 — Evaluation, Error Analysis, and Tuning

## Exercise 4.1: Define Success Criteria

**Task**: Write success criteria for the following agent tasks. Include both quantitative and qualitative signals.

### Task A: "Generate API documentation for all endpoints"
```yaml
success_criteria:
  quantitative:
    - 
    - 
    - 
  qualitative:
    - 
    - 
```

### Task B: "Refactor the authentication module to use JWT"
```yaml
success_criteria:
  quantitative:
    - 
    - 
    - 
  qualitative:
    - 
    - 
```

<details>
<summary>✅ Sample Answers</summary>

**Task A:**
```yaml
success_criteria:
  quantitative:
    - coverage: "100% of endpoints have documentation"
    - format: "All docs follow OpenAPI 3.0 specification"
    - validation: "openapi-lint passes with 0 errors"
    - examples: "Every endpoint has at least 1 request/response example"
  qualitative:
    - clarity: "Descriptions are understandable without reading source code"
    - accuracy: "Parameters, types, and responses match actual implementation"
    - consistency: "Naming and structure consistent across all endpoints"
```

**Task B:**
```yaml
success_criteria:
  quantitative:
    - tests_pass: "All existing auth tests still pass"
    - new_tests: "≥ 5 new tests covering JWT-specific scenarios"
    - coverage: "Auth module coverage ≥ 85%"
    - security_scan: "0 critical/high findings from CodeQL"
    - performance: "Token validation < 10ms (benchmark test)"
  qualitative:
    - backward_compat: "Existing sessions continue to work during migration period"
    - separation: "JWT logic is isolated in its own module, not scattered"
    - extensibility: "Easy to add new token types or claims in the future"
```

</details>

---

## Exercise 4.2: Root Cause Classification

**Scenario**: Review the following agent failure logs and classify the root cause.

### Failure A:
```
Step 1: grep "UserService" src/ → Found 3 files
Step 2: view src/services/UserService.ts → Success
Step 3: edit src/services/UserService.ts 
        old_str: "export class UserService {"
        → ERROR: String not found in file
Step 4: edit src/services/UserService.ts
        old_str: "export class UserService {"
        → ERROR: String not found in file (retry)
Step 5: edit src/services/UserService.ts
        old_str: "export class UserService {"
        → ERROR: String not found in file (retry)
```

### Failure B:
```
Step 1: Analyzed requirements → "Add caching to database queries"
Step 2: Created plan → "Add Redis caching layer"
Step 3: Installed redis package → Success
Step 4: Created src/cache/RedisCache.ts → Success
Step 5: Modified src/db/queries.ts → Added Redis calls
Step 6: Tests → FAILURE: "Cannot connect to Redis on localhost:6379"
```

### Failure C:
```
Step 1: Read issue: "Fix the login button on mobile"
Step 2: Searched for login component → Found LoginForm.tsx
Step 3: Modified LoginForm.tsx → Changed button styles
Step 4: Modified LoginForm.tsx → Changed form layout  
Step 5: Modified LoginForm.tsx → Changed validation logic
Step 6: Modified LoginForm.tsx → Changed API call handling
Step 7: Created new component MobileLoginForm.tsx
Result: 450 lines changed across 2 files
```

<details>
<summary>✅ Answers</summary>

**Failure A: Context Issue (Stale Context)**
- The agent viewed the file in Step 2 and found the class
- But the `old_str` doesn't match — likely the file uses `export default class` or different formatting
- The agent is working with stale/incorrect understanding of the file content
- The 3 identical retries indicate it's not adapting to the failure
- **Secondary**: Tool Misuse (retrying same failing command)

**Failure B: Environment Issue**
- The code is correct but the environment lacks a running Redis server
- The agent assumed Redis was available without checking
- **Root cause**: Environment assumption not validated before depending on it
- **Fix**: Check for Redis availability first, or use in-memory cache for dev

**Failure C: Reasoning Error (Scope Creep)**
- The task was "Fix the login button on mobile" (a CSS/styling issue)
- The agent changed: styles, layout, validation logic, API handling, and created a new component
- This is massive over-engineering — the issue was likely a simple CSS fix
- **Root cause**: Agent failed to scope its response to the actual problem
- **Fix**: Add constraints like "minimal change", "only modify styles unless functionally broken"

</details>

---

## Exercise 4.3: Evaluation Workflow Design

**Task**: Design a GitHub Actions workflow that evaluates agent output across multiple dimensions. The workflow should:
1. Run unit tests and report coverage
2. Run security scanning
3. Check diff size against thresholds
4. Verify documentation was updated
5. Generate a quality score

<details>
<summary>✅ Sample Answer</summary>

```yaml
name: Evaluate Agent Output
on:
  pull_request:
    types: [opened, synchronize]
    
jobs:
  evaluate:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      security-events: write
      
    outputs:
      quality-score: ${{ steps.score.outputs.total }}
      
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # Dimension 1: Tests & Coverage
      - name: Run tests with coverage
        id: tests
        run: |
          npm test -- --coverage --json > test-results.json 2>&1 || true
          PASS_RATE=$(jq '.numPassedTests / .numTotalTests * 100' test-results.json)
          COVERAGE=$(jq '.coverageMap.total.lines.pct' coverage/coverage-final.json)
          echo "pass_rate=$PASS_RATE" >> $GITHUB_OUTPUT
          echo "coverage=$COVERAGE" >> $GITHUB_OUTPUT

      # Dimension 2: Security
      - name: Security scan
        id: security
        uses: github/codeql-action/analyze@v3
        with:
          output: security-results
          
      - name: Count security findings
        id: sec-count
        run: |
          CRITICAL=$(jq '[.runs[].results[] | select(.level=="error")] | length' security-results/*.sarif)
          echo "critical=$CRITICAL" >> $GITHUB_OUTPUT

      # Dimension 3: Diff size
      - name: Evaluate diff size
        id: diff
        run: |
          ADDITIONS=$(git diff --stat origin/main | tail -1 | grep -oP '\d+(?= insertion)' || echo 0)
          DELETIONS=$(git diff --stat origin/main | tail -1 | grep -oP '\d+(?= deletion)' || echo 0)
          TOTAL=$((ADDITIONS + DELETIONS))
          echo "total_lines=$TOTAL" >> $GITHUB_OUTPUT
          if [ "$TOTAL" -gt 500 ]; then
            echo "size_flag=oversized" >> $GITHUB_OUTPUT
          elif [ "$TOTAL" -gt 200 ]; then
            echo "size_flag=large" >> $GITHUB_OUTPUT
          else
            echo "size_flag=acceptable" >> $GITHUB_OUTPUT
          fi

      # Dimension 4: Documentation
      - name: Check documentation
        id: docs
        run: |
          CHANGED_SRC=$(git diff --name-only origin/main -- 'src/' | wc -l)
          CHANGED_DOCS=$(git diff --name-only origin/main -- 'docs/' '*.md' | wc -l)
          if [ "$CHANGED_SRC" -gt 5 ] && [ "$CHANGED_DOCS" -eq 0 ]; then
            echo "docs_updated=false" >> $GITHUB_OUTPUT
          else
            echo "docs_updated=true" >> $GITHUB_OUTPUT
          fi

      # Dimension 5: Quality Score
      - name: Calculate quality score
        id: score
        run: |
          SCORE=0
          # Tests (max 30 points)
          SCORE=$((SCORE + $(echo "${{ steps.tests.outputs.pass_rate }} * 0.3" | bc | cut -d. -f1)))
          # Coverage (max 25 points)
          SCORE=$((SCORE + $(echo "${{ steps.tests.outputs.coverage }} * 0.25" | bc | cut -d. -f1)))
          # Security (max 25 points: 25 if 0 critical, 0 if any)
          if [ "${{ steps.sec-count.outputs.critical }}" -eq 0 ]; then SCORE=$((SCORE + 25)); fi
          # Diff size (max 10 points)
          if [ "${{ steps.diff.outputs.size_flag }}" == "acceptable" ]; then SCORE=$((SCORE + 10)); fi
          # Docs (max 10 points)
          if [ "${{ steps.docs.outputs.docs_updated }}" == "true" ]; then SCORE=$((SCORE + 10)); fi
          echo "total=$SCORE" >> $GITHUB_OUTPUT

      # Post results
      - name: Post evaluation report
        uses: actions/github-script@v7
        with:
          script: |
            const score = ${{ steps.score.outputs.total }};
            const emoji = score >= 80 ? '🟢' : score >= 60 ? '🟡' : '🔴';
            const body = `## ${emoji} Agent Output Evaluation: ${score}/100
            
            | Dimension | Score | Details |
            |-----------|-------|---------|
            | Tests | ${Math.round(${{ steps.tests.outputs.pass_rate }})}% pass | Coverage: ${{ steps.tests.outputs.coverage }}% |
            | Security | ${{ steps.sec-count.outputs.critical }} critical | CodeQL scan |
            | Diff Size | ${{ steps.diff.outputs.size_flag }} | ${{ steps.diff.outputs.total_lines }} lines |
            | Documentation | ${{ steps.docs.outputs.docs_updated }} | Updated with source changes |
            
            ${score < 70 ? '⚠️ **Below threshold** - requires manual review before merge' : '✅ Meets quality threshold'}`;
            
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body
            });
```

</details>

---

## Exercise 4.4: Failure Trace Analysis

**Task**: Given the following execution trace, identify:
1. Where the failure occurred
2. The root cause category
3. What the agent should have done differently
4. How to tune the agent to prevent this

```json
{
  "task": "Add input validation to POST /api/users",
  "trace": [
    {"step": 1, "action": "Read requirements from issue #42", "status": "success"},
    {"step": 2, "action": "Search for existing validation patterns", "status": "success", "found": "src/validation/products.ts"},
    {"step": 3, "action": "Read src/api/users/handler.ts", "status": "success"},
    {"step": 4, "action": "Read src/validation/products.ts", "status": "success"},
    {"step": 5, "action": "Create src/validation/users.ts", "status": "success"},
    {"step": 6, "action": "Edit src/api/users/handler.ts to import validation", "status": "success"},
    {"step": 7, "action": "Edit src/api/users/handler.ts to add validation call", "status": "success"},
    {"step": 8, "action": "Run tests", "status": "failure", "error": "TypeError: validateUser is not a function"},
    {"step": 9, "action": "Edit src/validation/users.ts to fix export", "status": "success"},
    {"step": 10, "action": "Run tests", "status": "failure", "error": "ValidationError: 'email' field uses deprecated EmailValidator"},
    {"step": 11, "action": "Edit src/validation/users.ts to change validator", "status": "success"},
    {"step": 12, "action": "Run tests", "status": "failure", "error": "ValidationError: 'email' field uses deprecated EmailValidator"},
    {"step": 13, "action": "Edit src/validation/users.ts to change validator (same change)", "status": "success"},
    {"step": 14, "action": "Run tests", "status": "failure", "error": "ValidationError: 'email' field uses deprecated EmailValidator"}
  ]
}
```

<details>
<summary>✅ Answer</summary>

**1. Where failure occurred:**
- Initial functional failure: Step 8 (export issue — minor)
- Persistent failure: Steps 10-14 (deprecated validator)

**2. Root cause categories:**
- Step 8: **Tool misuse** (didn't verify export syntax matched import)
- Steps 10-14: **Reasoning error** + **Context issue**
  - Agent copied the pattern from `products.ts` (Step 4) which uses the deprecated `EmailValidator`
  - Agent fails to understand WHY the test fails (deprecated validator)
  - Agent applies the same fix repeatedly (insanity loop)

**3. What agent should have done:**
- Step 8 fix: Read the created file to verify exports match imports before running tests
- Steps 10-14: 
  - Read the error message carefully
  - Search for `EmailValidator` deprecated notice in codebase
  - Find the current/recommended validator
  - Use the new validator instead of copying the old pattern

**4. Tuning to prevent:**
```markdown
# Add to custom instructions:
- When copying patterns from existing code, verify the pattern is not deprecated
- If a test fails twice with the same error after applying a fix, STOP and analyze 
  the error message more carefully. Search for related deprecation notices.
- After creating a new file, read it back to verify exports/imports are correct
  before running tests.
- Maximum 2 retries for the same error. After that, try a fundamentally different approach.
```

Also: Add a max-retry limit (2) and require the agent to change strategy after repeated failures.

</details>

---

## Exercise 4.5: Instruction Tuning

**Scenario**: An agent has the following problems. Write the specific custom instruction additions to fix each one.

| Problem | Current Behavior | Desired Behavior |
|---------|-----------------|-----------------|
| Agent creates 500-line functions | Writes everything in one function | Max 30 lines per function |
| Agent uses `any` type in TypeScript | Skips type definitions | Full type safety |
| Agent doesn't handle errors | Happy path only | Try/catch with meaningful errors |
| Agent modifies test fixtures | Changes test data to make tests pass | Tests validate actual behavior |
| Agent ignores existing patterns | Invents new approaches | Follows established patterns |

<details>
<summary>✅ Sample Answer</summary>

Add to `.github/copilot-instructions.md`:

```markdown
## Code Quality Rules (MANDATORY)

### Function Size
- Maximum function length: 30 lines (excluding type declarations)
- If a function exceeds 30 lines, extract helper functions
- Each function should do ONE thing

### Type Safety (TypeScript)
- NEVER use `any` type — use specific types or `unknown` with type guards
- All function parameters and return types must be explicitly typed
- Use generic types (`T extends Base`) over `any` when type varies
- If external library types are incomplete, create type declaration files (.d.ts)

### Error Handling
- Every async function must have try/catch or propagate errors explicitly
- Error messages must include: what failed, why, and suggested fix
- Use custom error classes for domain errors (see src/errors/)
- Never catch and swallow errors silently

### Testing Integrity
- NEVER modify test fixtures or test data to make tests pass
- If a test fails, fix the implementation, not the test
- Test fixtures represent expected/correct behavior
- Only modify tests when requirements genuinely change (and document why)

### Follow Existing Patterns
- Before writing new code, search for similar implementations in the codebase
- Use `grep` to find established patterns for: error handling, validation, API calls
- Match the style of adjacent code (same file > same directory > same project)
- If no pattern exists, propose the approach in a comment before implementing
```

</details>

---

## Exercise 4.6: Evaluation Signal Design

**Task**: You're building an evaluation dashboard for agent performance. Design the metrics you'd track, including collection method and threshold.

| Metric | Collection Method | Good | Warning | Critical |
|--------|------------------|------|---------|----------|
| Task completion rate | | | | |
| Average quality score | | | | |
| Time to completion | | | | |
| Human intervention rate | | | | |
| Rollback rate | | | | |
| Security findings | | | | |

<details>
<summary>✅ Sample Answer</summary>

| Metric | Collection Method | Good | Warning | Critical |
|--------|------------------|------|---------|----------|
| Task completion rate | Count successful vs total tasks | ≥ 90% | 70-89% | < 70% |
| Average quality score | Weighted evaluation (tests + lint + security + size) | ≥ 80/100 | 60-79 | < 60 |
| Time to completion | Measure from task start to PR creation | < 10 min | 10-30 min | > 30 min |
| Human intervention rate | Count of times humans had to fix/redo agent work | < 10% | 10-25% | > 25% |
| Rollback rate | PRs reverted after merge / total merged PRs | < 2% | 2-5% | > 5% |
| Security findings | CodeQL/Dependabot findings in agent PRs | 0 critical | 1-2 medium | Any critical |
| Retry count per task | Average retries before success | < 2 | 2-4 | > 4 |
| Diff accuracy | % of changed lines that survive human review | > 95% | 80-95% | < 80% |

**Dashboard alert rules:**
- Any metric in "Critical" → Pause agent, notify team
- 2+ metrics in "Warning" → Review agent configuration
- Trending downward for 7 days → Schedule tuning session

</details>
