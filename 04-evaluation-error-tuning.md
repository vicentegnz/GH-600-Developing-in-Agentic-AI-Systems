# Domain 4: Perform Evaluation, Error Analysis, and Tuning (15–20%)

## 📚 References & Documentation

| Resource | Link |
|----------|------|
| GitHub Docs | [Implementation Planner](https://docs.github.com/en/copilot/tutorials/customization-library/custom-agents/implementation-planner) |
| GitHub Docs | [Copilot Code Review](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review) |
| GitHub Docs | [Code Scanning](https://docs.github.com/en/code-security/code-scanning) |
| GitHub Docs | [Dependabot](https://docs.github.com/en/code-security/dependabot) |
| GitHub Docs | [Secret Scanning](https://docs.github.com/en/code-security/secret-scanning) |
| MS Learn | [Foundations of Agentic AI](https://learn.microsoft.com/en-us/training/modules/foundations-agentic-ai/) |
| MS Learn | [Designing Agent Architecture](https://learn.microsoft.com/en-us/training/modules/design-agent-architecture-integration/) |

---

## Section 1: Define Success Criteria and Evaluation Signals

### 1.1 Specify Expected Outcomes and Operational Constraints

**Well-defined success criteria template:**

```yaml
evaluation:
  task: "Generate API endpoint for user registration"
  
  expected_outcomes:
    functional:
      - "Endpoint accepts POST /api/users with email and password"
      - "Returns 201 with user ID on success"
      - "Returns 400 with validation errors on bad input"
      - "Returns 409 if email already exists"
    quality:
      - "Code follows repository conventions"
      - "No new lint errors introduced"
      - "Test coverage ≥ 80% for new code"
    security:
      - "Password is hashed before storage"
      - "Input is validated and sanitized"
      - "No secrets in code or logs"
      
  operational_constraints:
    - max_execution_time: "5 minutes"
    - max_files_modified: 10
    - no_changes_to: ["src/core/", "migrations/"]
    - required_artifacts: ["tests", "documentation"]
```

### 1.2 Identify Qualitative and Quantitative Evaluation Signals

**Quantitative Signals (Measurable):**

| Signal | Metric | Tool |
|--------|--------|------|
| Test pass rate | % of tests passing | `npm test`, `pytest` |
| Code coverage | % lines/branches covered | Istanbul, Coverage.py |
| Lint score | # of lint errors/warnings | ESLint, Pylint |
| Security findings | # of vulnerabilities | CodeQL, Dependabot |
| Build success | Pass/Fail | GitHub Actions |
| Execution time | Seconds/minutes | Workflow timing |
| Token usage | # tokens consumed | SDK metrics |
| Diff size | Lines added/removed | `git diff --stat` |

**Qualitative Signals (Judgment-based):**

| Signal | Evaluation Method |
|--------|-------------------|
| Code readability | Human review, style guide compliance |
| Architectural fit | Pattern matching against conventions |
| Naming quality | Consistency with existing codebase |
| Documentation clarity | Human assessment |
| Solution elegance | Simplicity vs. complexity ratio |
| Intent alignment | Does output match what was requested? |

### 1.3 Align Evaluation Criteria with Development Intent

**The Intent-Output Gap:**
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Developer    │     │ Agent        │     │ Actual       │
│ Intent       │────▶│ Interpretation│────▶│ Output       │
│ (What they   │     │ (What agent  │     │ (What was    │
│  wanted)     │     │  understood) │     │  produced)   │
└──────────────┘     └──────────────┘     └──────────────┘
        │                                        │
        └────────── ALIGNMENT GAP ───────────────┘
```

**Reducing the gap:**
- Clear, specific prompts with examples
- Success criteria defined BEFORE execution
- Incremental validation (check at each step)
- Feedback loops that refine agent behavior

### 1.4 Generate Evaluation Signals Using Automated Scanning Tools

**GitHub-native scanning tools:**

| Tool | What it scans | Evaluation signal |
|------|--------------|-------------------|
| **CodeQL** | Source code for vulnerabilities | Security quality |
| **Dependabot** | Dependencies for known CVEs | Supply chain security |
| **Secret Scanning** | Code for exposed secrets | Credential safety |
| **Copilot Code Review** | PRs for bugs, style, security | Code quality |
| **GitHub Actions** | CI/CD pipeline results | Build/test health |

**Integrating scanners into agent evaluation:**
```yaml
# .github/workflows/evaluate-agent-output.yml
name: Evaluate Agent Output
on: pull_request

jobs:
  evaluate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run tests
        run: npm test -- --coverage
        
      - name: Run linter
        run: npm run lint
        
      - name: Security scan
        uses: github/codeql-action/analyze@v3
        
      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "::error::Coverage $COVERAGE% below 80% threshold"
            exit 1
          fi
          
      - name: Evaluate diff size
        run: |
          LINES=$(git diff --stat origin/main | tail -1 | grep -oP '\d+(?= insertion)')
          if [ "$LINES" -gt 500 ]; then
            echo "::warning::Large diff ($LINES lines) - may need manual review"
          fi
```

---

## Section 2: Analyze Agent Failures and Identify Root Causes

### 2.1 Identify Failures Using Logs, Plans, Traces, and Artifacts

**Failure identification sources:**

| Source | What to look for |
|--------|-----------------|
| **Agent logs** | Error messages, unexpected tool calls, timeout events |
| **Execution plan** | Steps that diverge from original plan |
| **Tool traces** | Failed tool invocations, wrong parameters |
| **Git history** | Unexpected or partial commits |
| **Workflow artifacts** | Missing expected outputs |
| **PR description** | Incomplete or incoherent summaries |
| **Test results** | Failed tests that should pass |

**Example trace analysis:**
```json
{
  "trace": [
    { "step": 1, "tool": "grep", "status": "success", "duration": "0.3s" },
    { "step": 2, "tool": "view", "status": "success", "duration": "0.1s" },
    { "step": 3, "tool": "edit", "status": "failure", "error": "old_str not found" },
    { "step": 4, "tool": "edit", "status": "failure", "error": "old_str not found" },
    { "step": 5, "tool": "edit", "status": "failure", "error": "old_str not found" }
  ],
  "diagnosis": "Agent is trying to edit content that doesn't match the actual file. Likely stale context or wrong file version."
}
```

### 2.2 Classify Root Causes

**Root Cause Taxonomy:**

| Category | Description | Examples |
|----------|-------------|----------|
| **Reasoning errors** | Agent's logic is flawed | Wrong algorithm choice, incorrect assumptions, misunderstood requirements |
| **Tool misuse** | Agent uses tools incorrectly | Wrong parameters, calling tools in wrong order, using write tool for read |
| **Context issues** | Agent has wrong/incomplete information | Stale files, missing dependencies, incomplete project understanding |
| **Environment issues** | External factors cause failure | Network timeouts, permission errors, resource limits |
| **Instruction issues** | Prompts/instructions are ambiguous | Vague requirements, contradictory instructions, missing constraints |

**Root Cause Analysis Framework:**
```
1. WHAT failed? → Identify the specific failure point
2. WHEN did it fail? → Determine the step/timing
3. WHY did it fail? → Classify into root cause category
4. HOW to fix? → Define corrective action
5. HOW to prevent? → Define preventive measure
```

**Common patterns by root cause:**

```
Reasoning Error Indicators:
  - Agent produces logically inconsistent output
  - Agent contradicts its own plan
  - Agent makes decisions that don't follow from the evidence
  
Tool Misuse Indicators:
  - Repeated failed tool calls with same parameters
  - Using edit tool on files that don't exist
  - Calling tools in wrong sequence
  
Context Issue Indicators:
  - Agent references files/functions that don't exist
  - Agent assumes outdated API signatures
  - Agent doesn't account for recent changes

Environment Issue Indicators:
  - Timeout errors
  - Permission denied errors
  - Network connectivity failures
```

---

## Section 3: Tune Agent Behavior Based on Evaluation Results

### 3.1 Revise Instructions, Workflows, or Constraints

**Instruction Tuning Strategies:**

| Problem Observed | Tuning Action |
|-----------------|---------------|
| Agent modifies wrong files | Add explicit path constraints |
| Agent produces verbose code | Add "prefer concise solutions" instruction |
| Agent ignores conventions | Add specific examples in custom instructions |
| Agent over-engineers solutions | Add "minimal viable change" constraint |
| Agent misunderstands domain terms | Add glossary/definitions section |

**Before tuning:**
```markdown
# Custom Instructions
Write good code that follows best practices.
```

**After tuning (specific, actionable):**
```markdown
# Custom Instructions

## Code Style
- Maximum function length: 20 lines
- Use early returns over nested conditionals
- All public methods must have JSDoc comments

## Architecture Rules
- API handlers must not contain business logic
- Use dependency injection, never import singletons directly
- All database calls go through repository interfaces

## Testing Requirements
- Every new function needs at least one test
- Use test names: "should [expected behavior] when [condition]"
- Mock external services, never call real APIs in tests
```

### 3.2 Refine Memory Usage

| Problem | Memory Tuning |
|---------|---------------|
| Agent forgets project conventions | Add to Copilot Memory or custom instructions |
| Agent context window overflows | Reduce loaded files, summarize large contexts |
| Agent uses outdated information | Set memory expiration, implement freshness checks |
| Agent applies wrong conventions | Scope memory to specific directories/file types |

### 3.3 Refine Tool Usage and Tool Access

| Problem | Tool Tuning |
|---------|-------------|
| Agent uses bash for simple edits | Restrict to `edit` tool, remove `bash` |
| Agent creates files instead of editing | Remove `create` tool, keep `edit` |
| Agent makes network calls unnecessarily | Restrict network tools via firewall |
| Agent doesn't use available tools | Improve tool descriptions for better matching |
| Agent uses tools too broadly | Add per-agent tool restrictions |

**Iterative tuning cycle:**
```
1. Run agent on task
2. Evaluate output against success criteria
3. Identify gaps/failures
4. Classify root cause
5. Apply targeted fix (instruction/memory/tool adjustment)
6. Re-run and compare
7. Repeat until success criteria are met
```

---

## 🧠 Key Memorization Points

1. **Success criteria** = Expected outcomes (functional + quality + security) + Operational constraints
2. **Quantitative signals**: Test pass rate, coverage, lint score, security findings, diff size
3. **Qualitative signals**: Readability, architectural fit, naming quality, intent alignment
4. **Root cause categories**: Reasoning errors, tool misuse, context issues, environment issues, instruction issues
5. **Evaluation tools**: CodeQL, Dependabot, Secret Scanning, Copilot Code Review, CI/CD
6. **Intent-Output Gap**: The difference between what developer wanted and what agent produced
7. **Tuning hierarchy**: Instructions → Memory → Tools → Workflow constraints
8. **Iterative tuning**: Run → Evaluate → Identify gaps → Fix → Re-run → Compare
9. **Trace analysis**: Step-by-step review of tool calls to identify failure point
10. **Automated scanning** integrates into PR checks for continuous evaluation

---

## 📝 Self-Assessment Questions

1. What's the difference between quantitative and qualitative evaluation signals?
2. How do you integrate CodeQL into agent evaluation workflows?
3. Name the five root cause categories for agent failures.
4. What indicators suggest a "reasoning error" vs. "tool misuse"?
5. Describe the iterative tuning cycle for agent behavior.
6. How do you align evaluation criteria with development intent?
7. What artifacts help you identify where an agent failed?
8. Give three examples of instruction tuning to fix specific agent problems.
9. When should you refine tool access vs. refine instructions?
10. How do you generate evaluation signals using automated scanning tools?

➡️ **Practice these concepts with exercises**: [04-exercises-evaluation.md](./exercises/04-exercises-evaluation.md)
