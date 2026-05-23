# GH-600 Practice Exam Questions

> **Instructions**: 50 questions covering all 6 domains. Choose the BEST answer.
> Passing score: 70% (35/50 correct)
> Time limit: 90 minutes (simulated)

---

## Domain 1: Agent Architecture & SDLC (Questions 1-9)

### Question 1
What distinguishes an AI **agent** from an AI **assistant** in the SDLC context?

A) Agents use larger language models  
B) Agents can plan, reason, and act autonomously within defined boundaries  
C) Agents are always faster than assistants  
D) Agents don't require any human oversight  

<details><summary>Answer</summary>**B** — Agents are distinguished by their ability to autonomously plan, reason, and take actions, while assistants respond to prompts and require humans to act on suggestions.</details>

---

### Question 2
Which of the following is an anti-pattern in agent systems?

A) Requiring PR review for agent-generated changes  
B) Scoping agent tools to the minimum necessary set  
C) Allowing an agent unlimited retries without a timeout  
D) Producing structured execution plans before acting  

<details><summary>Answer</summary>**C** — Unlimited retries without timeout is the "infinite loops" anti-pattern. The agent could loop indefinitely without making progress.</details>

---

### Question 3
What is the primary purpose of separating agent PLANNING from EXECUTION?

A) To reduce token usage  
B) To enable human review of the plan before any changes are made  
C) To make the agent run faster  
D) To reduce the number of API calls  

<details><summary>Answer</summary>**B** — Separating planning from execution allows humans to review, approve, or modify the agent's proposed approach before any irreversible actions occur.</details>

---

### Question 4
According to the "contributor model," how should agent-generated work be treated?

A) As pre-approved since it comes from AI  
B) Like work from a junior developer — subject to standard review processes  
C) With more scrutiny than human code, requiring 3+ reviewers  
D) It should bypass review to maintain velocity  

<details><summary>Answer</summary>**B** — The contributor model treats agents like junior developers: their work goes through standard PR review, follows branch protection rules, and is subject to the same quality checks as human-written code.</details>

---

### Question 5
Which is the BEST set of success criteria for an agent task?

A) "Make the code better"  
B) "All tests pass, coverage > 80%, no new lint errors, PR created with description"  
C) "Finish within 5 minutes"  
D) "Use fewer than 10,000 tokens"  

<details><summary>Answer</summary>**B** — Good success criteria are specific, measurable, and cover functional requirements (tests pass), quality (coverage, lint), and deliverables (PR with description).</details>

---

### Question 6
An agent configured with `approval_required: false` and `scope: entire-organization` exhibits which anti-patterns?

A) Opaque reasoning and silent failure  
B) Unbounded autonomy, scope creep, and trust escalation  
C) Context pollution and infinite loops  
D) Only scope creep  

<details><summary>Answer</summary>**B** — No approval required = unbounded autonomy. Entire org scope = scope creep. The combination represents trust escalation (agent has more privilege than it needs).</details>

---

### Question 7
Which GitHub feature ensures that security-critical file changes are reviewed by the appropriate team?

A) GitHub Actions  
B) Branch protection rules  
C) CODEOWNERS  
D) Dependabot  

<details><summary>Answer</summary>**C** — CODEOWNERS assigns specific teams/users as required reviewers for particular file paths, ensuring that security-critical paths (e.g., `/src/auth/`) require security team review.</details>

---

### Question 8
What should an agent produce to enable observability of its autonomous actions?

A) Only the final code changes  
B) Structured plans, execution logs, decision rationale, and metrics  
C) A summary email after all work is complete  
D) Only test results  

<details><summary>Answer</summary>**B** — Full observability requires multiple artifact types: structured plans (what it intends to do), logs (what it did), decision rationale (why), and metrics (how long, what tools).</details>

---

### Question 9
Which approach BEST configures human intervention without slowing delivery?

A) Require approval for every single change  
B) Use tiered review: auto-merge low-risk changes, require review for high-risk  
C) Remove all human review to maximize speed  
D) Have humans review only after deployment  

<details><summary>Answer</summary>**B** — Tiered review right-sizes oversight: low-risk changes auto-merge (preserving velocity), while high-risk changes require human review (preserving safety).</details>

---

## Domain 2: Tool Use & Environment (Questions 10-21)

### Question 10
What principle should guide agent tool assignment?

A) Give agents all available tools for maximum flexibility  
B) Least privilege — minimum tools necessary for the task  
C) Only give agents read tools  
D) Match tools to agent's preferred language  

<details><summary>Answer</summary>**B** — Least privilege ensures agents can only perform actions necessary for their assigned task, reducing risk of unintended side effects.</details>

---

### Question 11
What is the Model Context Protocol (MCP)?

A) A protocol for encrypting agent communications  
B) A standardized protocol for AI agents to interact with external tools and services  
C) A way to limit agent memory usage  
D) GitHub's internal API protocol  

<details><summary>Answer</summary>**B** — MCP provides a consistent, standardized interface for AI agents to access external tools, data sources, and services regardless of the specific implementation.</details>

---

### Question 12
What distinguishes a GitHub Remote MCP server from a local stdio MCP server?

A) Remote servers are faster  
B) Remote servers are hosted by GitHub, require no local setup, and integrate with GitHub auth  
C) Local servers are more secure  
D) Remote servers can only read data  

<details><summary>Answer</summary>**B** — GitHub Remote MCP servers are managed by GitHub, don't require local installation, update automatically, and integrate with GitHub's authentication system.</details>

---

### Question 13
What does `defaultAgent.excludedTools` accomplish?

A) Removes tools from the system entirely  
B) Hides tools from the main agent to force delegation to specialized sub-agents  
C) Blocks tools for all agents  
D) Adds tools to the deny list  

<details><summary>Answer</summary>**B** — Tools in `excludedTools` remain registered and available to sub-agents but are hidden from the main agent's tool list, forcing it to delegate to specialized agents for those capabilities.</details>

---

### Question 14
How should an agent be scoped to work only within a specific repository?

A) Use a broader token and trust the agent's instructions  
B) Configure repository-scoped tokens, path restrictions, and branch scope  
C) Give it organization-wide access but ask it to stay in one repo  
D) Only run the agent on a specific machine  

<details><summary>Answer</summary>**B** — Proper scoping requires technical enforcement: repository-scoped tokens (can't access other repos), path restrictions (can only read/write specific directories), and branch scope (can only create/push to designated branches).</details>

---

### Question 15
What is the correct order of an error handling escalation hierarchy?

A) Escalate → Retry → Rollback → Block  
B) Retry → Rollback → Escalate → Block  
C) Block → Escalate → Retry → Rollback  
D) Rollback → Retry → Block → Escalate  

<details><summary>Answer</summary>**B** — The escalation hierarchy goes from least disruptive to most: first retry (automated), then rollback (undo), then escalate (notify humans), then block (halt execution).</details>

---

### Question 16
Which MCP server configuration restricts agent access to only the `/workspace/docs` directory?

A) `"args": ["-y", "@modelcontextprotocol/server-filesystem"]`  
B) `"args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace/docs"]`  
C) `"env": {"PATH": "/workspace/docs"}`  
D) `"args": ["--restrict", "/workspace/docs"]`  

<details><summary>Answer</summary>**B** — The filesystem MCP server accepts a path argument that restricts its scope to only that directory and its children.</details>

---

### Question 17
What information must agent traceability capture?

A) Only the final output  
B) Who (agent), What (changes), When (timestamps), Why (rationale), How (tools used)  
C) Just the execution time  
D) Only the tools that were used  

<details><summary>Answer</summary>**B** — Complete traceability requires all five dimensions: which agent acted, what it changed, when, why (linked to plan/intent), and how (which tools and execution path).</details>

---

### Question 18
An agent needs to create PRs autonomously. What workflow permissions are required?

A) `contents: read` only  
B) `contents: write` and `pull-requests: write`  
C) `admin: write`  
D) `contents: read` and `pull-requests: read`  

<details><summary>Answer</summary>**B** — Creating a PR requires `contents: write` (to push the branch) and `pull-requests: write` (to create the PR). Read-only permissions are insufficient, and admin is excessive.</details>

---

### Question 19
What does the agent firewall control?

A) Which files the agent can read  
B) Network access — which URLs the agent can reach  
C) Which tools the agent can use  
D) CPU and memory limits  

<details><summary>Answer</summary>**B** — The agent firewall specifically controls network access, defining which external URLs and services the agent can communicate with (allowlist/blocklist approach).</details>

---

### Question 20
An MCP allow list serves what purpose?

A) Lists servers the agent must use  
B) Restricts which MCP servers an agent is permitted to connect to  
C) Defines server priorities  
D) Specifies server health checks  

<details><summary>Answer</summary>**B** — Allow lists restrict which MCP servers agents can use, preventing connection to unauthorized or potentially dangerous external services.</details>

---

### Question 21
What is the BEST approach for an agent handling environment-specific constraints?

A) Hard-code environment URLs in agent configuration  
B) Use environment variables, scoped secrets, and configurable firewall rules  
C) Give the agent access to all environments  
D) Let the agent detect the environment automatically  

<details><summary>Answer</summary>**B** — Environment-specific constraints should use proper infrastructure patterns: environment variables for configuration, scoped secrets for credentials, and firewall rules for network boundaries.</details>

---

## Domain 3: Memory, State & Execution (Questions 22-28)

### Question 22
Which memory type is used for repository conventions that should persist across all agent sessions?

A) Short-term (working memory)  
B) Long-term (Copilot Memory or custom instructions)  
C) External (README file)  
D) Session memory  

<details><summary>Answer</summary>**B** — Repository conventions that apply to all future work are best stored as long-term memory via Copilot Memory (repo facts) or custom instructions (`.github/copilot-instructions.md`).</details>

---

### Question 23
What is "context drift" in agent systems?

A) When an agent forgets its instructions  
B) When an agent's understanding diverges from the actual state of the codebase  
C) When context window fills up  
D) When the agent changes topics  

<details><summary>Answer</summary>**B** — Context drift occurs when the agent's loaded/cached understanding of the codebase becomes stale because external changes (other developers' pushes, concurrent modifications) have altered the actual state.</details>

---

### Question 24
Copilot Memory facts automatically expire after how long of non-use?

A) 7 days  
B) 14 days  
C) 28 days  
D) Never (permanent)  

<details><summary>Answer</summary>**C** — Copilot Memory entries that go unused are automatically deleted after 28 days. The timer resets when Copilot successfully validates and uses an entry.</details>

---

### Question 25
How should an agent resume work after an interruption?

A) Start the entire task from scratch  
B) Load last state artifact, verify completed steps, continue from next pending step  
C) Ask the user what to do  
D) Retry the last failed step only  

<details><summary>Answer</summary>**B** — The correct resume pattern is: load the last known state, verify which steps actually completed (don't trust state alone), identify the next pending step, and continue without re-executing completed work.</details>

---

### Question 26
What prevents conflicting context between multiple tools working on the same codebase?

A) Running all tools on the same machine  
B) Single source of truth, locking mechanisms, and concurrency controls  
C) Using the same AI model for all tools  
D) Disabling parallel execution entirely  

<details><summary>Answer</summary>**B** — Conflict prevention requires designating authoritative sources, implementing locking during modifications, and using concurrency controls (e.g., `concurrency` groups in GitHub Actions).</details>

---

### Question 27
Which approach BEST prevents stale context in a long-running agent task?

A) Load everything at the start and never refresh  
B) Re-read files before modifying them and check for upstream changes periodically  
C) Limit task duration to 1 minute  
D) Never cache any context  

<details><summary>Answer</summary>**B** — Freshness checks before modification (re-read target files) combined with periodic upstream checks (`git fetch` + diff) provides the best balance of performance and accuracy.</details>

---

### Question 28
Where is the BEST place to persist agent state for cross-session continuity?

A) In the agent's conversation history  
B) As durable artifacts: git commits, PR descriptions, workflow artifacts  
C) In local memory only  
D) In environment variables  

<details><summary>Answer</summary>**B** — Durable artifacts (committed files, PR descriptions, GitHub Actions artifacts) survive across sessions and can be loaded by any future agent instance.</details>

---

## Domain 4: Evaluation & Tuning (Questions 29-36)

### Question 29
Which is a QUANTITATIVE evaluation signal for agent output?

A) "The code is readable"  
B) "Test coverage increased from 60% to 85%"  
C) "The solution is elegant"  
D) "Good architectural fit"  

<details><summary>Answer</summary>**B** — Quantitative signals are objectively measurable numbers. Coverage percentage is measurable; readability, elegance, and architectural fit are qualitative (judgment-based).</details>

---

### Question 30
An agent repeatedly tries the same `edit` command with identical parameters but it fails each time. What root cause category is this?

A) Environment issue  
B) Reasoning error  
C) Tool misuse combined with context issue  
D) Instruction issue  

<details><summary>Answer</summary>**C** — The agent is misusing the tool (retrying the same failing command) AND has a context issue (the `old_str` doesn't match actual file content, indicating stale or incorrect understanding).</details>

---

### Question 31
What is the "Intent-Output Gap"?

A) The time between request and response  
B) The difference between what the developer wanted and what the agent produced  
C) The gap in test coverage  
D) Missing documentation  

<details><summary>Answer</summary>**B** — The Intent-Output Gap is the misalignment between the developer's actual intent and what the agent interprets and produces. Reducing this gap is a key goal of evaluation and tuning.</details>

---

### Question 32
Which automated scanning tool detects exposed secrets in code?

A) CodeQL  
B) Dependabot  
C) Secret Scanning  
D) ESLint  

<details><summary>Answer</summary>**C** — GitHub Secret Scanning specifically detects exposed credentials, API keys, and tokens in code. CodeQL finds code vulnerabilities, Dependabot checks dependencies.</details>

---

### Question 33
An agent produces correct code but uses 500 lines where 50 would suffice. What tuning action is MOST appropriate?

A) Restrict the agent's tool access  
B) Add "prefer minimal, concise solutions" to custom instructions  
C) Reduce the context window  
D) Switch to a different AI model  

<details><summary>Answer</summary>**B** — This is an instruction tuning issue. The agent needs explicit guidance about conciseness. Adding constraints like "prefer minimal solutions" or "maximum function length: 30 lines" directly addresses over-engineering.</details>

---

### Question 34
After an agent fails, you find it's using a deprecated API. Which root cause category is this?

A) Tool misuse  
B) Context issue (stale information)  
C) Reasoning error  
D) Environment issue  

<details><summary>Answer</summary>**B** — Using a deprecated API indicates the agent has outdated information about the codebase. Either its training data is stale, or it copied an old pattern without checking for deprecation notices.</details>

---

### Question 35
What is the correct iterative tuning cycle?

A) Deploy → Hope it works → Deploy again  
B) Run → Evaluate → Identify gaps → Fix (instruction/memory/tool) → Re-run → Compare  
C) Write tests → Run tests → Fix tests  
D) Change model → Retry → Change model again  

<details><summary>Answer</summary>**B** — The iterative tuning cycle: run the agent, evaluate output against criteria, identify gaps/failures, classify root cause, apply targeted fix, re-run, and compare results to previous run.</details>

---

### Question 36
When should you refine tool access instead of refining instructions?

A) When the agent produces verbose output  
B) When the agent uses tools it shouldn't (e.g., bash for simple edits)  
C) When the agent's naming is inconsistent  
D) When the agent ignores conventions  

<details><summary>Answer</summary>**B** — Tool refinement is appropriate when the problem is which tools the agent uses, not how it reasons. If it uses `bash` when it should use `edit`, restricting tool access is more reliable than asking it nicely in instructions.</details>

---

## Domain 5: Multi-Agent Coordination (Questions 37-44)

### Question 37
Which orchestration pattern uses a central coordinator that delegates tasks to specialized workers?

A) Sequential Pipeline  
B) Parallel Fan-Out  
C) Hierarchical  
D) Event-Driven  

<details><summary>Answer</summary>**C** — The Hierarchical pattern has a central orchestrator agent that decomposes tasks and delegates to specialized worker agents, collecting and integrating their results.</details>

---

### Question 38
What is the PRIMARY purpose of agent isolation in parallel execution?

A) To make agents run faster  
B) To prevent agents from producing conflicting changes  
C) To reduce token usage  
D) To use different AI models  

<details><summary>Answer</summary>**B** — Isolation prevents conflicts when multiple agents work simultaneously. Without isolation, agents might modify the same files, create incompatible changes, or overwrite each other's work.</details>

---

### Question 39
Two agents both modified the same shared type file. This is what type of multi-agent conflict?

A) Resource contention  
B) Duplicated effort  
C) Overlapping edits  
D) Contradictory outputs  

<details><summary>Answer</summary>**C** — Overlapping edits occur when multiple agents modify the same file. This leads to merge conflicts that must be resolved before integration.</details>

---

### Question 40
An agent has been running for 30 minutes with no new output. This state is called:

A) Failed  
B) Partial  
C) Stalled  
D) Degraded  

<details><summary>Answer</summary>**C** — A stalled agent is one that is not making progress (no new output after a timeout period). It differs from failed (explicit error) or degraded (poor quality output).</details>

---

### Question 41
What is the correct recovery hierarchy for multi-agent failures?

A) Retry → Fallback → Isolate → Escalate → Rollback  
B) Rollback → Retry → Escalate → Fallback  
C) Escalate immediately  
D) Always rollback everything  

<details><summary>Answer</summary>**A** — Start with least disruptive: retry, try alternative (fallback), remove failed agent (isolate), notify human (escalate), undo changes (rollback).</details>

---

### Question 42
A handoff record between agents should document:

A) Only the data being passed  
B) From/To agents, context passed, decisions made, and expected outcomes  
C) Just the timestamp  
D) Only the tools available  

<details><summary>Answer</summary>**B** — Complete handoff documentation includes: which agents are involved, what context is transferred, what decisions were already made, and what outcomes the receiving agent should produce.</details>

---

### Question 43
When retiring an agent from a multi-agent workflow, what must you preserve?

A) Nothing — just delete it  
B) Audit trail, archived configuration, and documentation of why it was retired  
C) Only the source code  
D) Only the agent's memory  

<details><summary>Answer</summary>**B** — Retirement requires preserving: archived configuration (for audit), documentation of reason/date (for compliance), all past outputs (for accountability), and updated workflow references.</details>

---

### Question 44
What is the safest way to update an agent in a production workflow?

A) Replace it immediately  
B) Blue/green deployment: run new version alongside old, gradually shift traffic  
C) Turn off the old one, turn on the new one  
D) Let both run permanently  

<details><summary>Answer</summary>**B** — Blue/green (or canary) deployment runs both versions, gradually shifts traffic to the new version while monitoring quality, with instant rollback capability if the new version degrades.</details>

---

## Domain 6: Guardrails & Accountability (Questions 45-50)

### Question 45
At which autonomy level does an agent act and then a human reviews the result?

A) Level 0 (Disabled)  
B) Level 1 (Suggest)  
C) Level 2 (Act + Review)  
D) Level 4 (Full Autonomous)  

<details><summary>Answer</summary>**C** — Level 2 (Act + Review) means the agent performs the action and a human reviews it afterward. Level 1 = suggest only, Level 3 = act + notify, Level 4 = fully autonomous.</details>

---

### Question 46
An irreversible action (like deleting a database) should be assigned which autonomy level?

A) Level 4 (Full Autonomous)  
B) Level 3 (Act + Notify)  
C) Level 2 (Act + Review)  
D) Level 0 (Disabled) or Level 1 (Suggest only)  

<details><summary>Answer</summary>**D** — Irreversible actions with high impact should be at Level 0 (agent cannot perform it) or Level 1 (agent can only suggest it, human must execute).</details>

---

### Question 47
What is the MOST reliable layer for enforcing policy (hardest for an agent to bypass)?

A) Instruction layer (agent prompt)  
B) Tool layer (permission restrictions)  
C) Platform layer (branch protection, rulesets)  
D) All layers are equally reliable  

<details><summary>Answer</summary>**C** — Platform-level controls (branch protection, rulesets, CODEOWNERS) are enforced by GitHub itself and cannot be bypassed regardless of what instructions or tools the agent has.</details>

---

### Question 48
Which strategy preserves execution velocity while maintaining guardrails?

A) Remove all approval requirements  
B) Pre-approved patterns that skip review for known-safe, repetitive tasks  
C) Require 5 reviewers for everything  
D) Only run agents during business hours  

<details><summary>Answer</summary>**B** — Pre-approved patterns (like auto-formatting, removing unused imports) skip individual review because they're objectively safe and verifiable by automated checks, preserving velocity without sacrificing safety.</details>

---

### Question 49
Environment protection rules in GitHub are BEST used for:

A) Protecting source code files  
B) Gating deployments with required reviewers and wait timers  
C) Encrypting agent communications  
D) Limiting agent memory usage  

<details><summary>Answer</summary>**B** — Environment protection rules specifically gate deployment workflows with required reviewers, wait timers, and branch policies, ensuring that deploying to production requires explicit human authorization.</details>

---

### Question 50
Which Responsible AI principle requires that agent reasoning and actions be inspectable?

A) Fairness  
B) Safety  
C) Transparency  
D) Accountability  

<details><summary>Answer</summary>**C** — Transparency requires that agent reasoning, decisions, and actions can be inspected and understood by humans. This includes producing structured plans, decision logs, and execution traces.</details>

---

## Scoring

Count your correct answers:

| Score | Result |
|-------|--------|
| 45-50 | 🏆 Excellent — Ready for the exam |
| 35-44 | ✅ Passing — Review weak areas |
| 25-34 | ⚠️ Needs work — Focus on lowest domains |
| < 25 | 🔄 Review all material before retaking |

### Score by Domain:
- Domain 1 (Q1-9): ___/9
- Domain 2 (Q10-21): ___/12
- Domain 3 (Q22-28): ___/7
- Domain 4 (Q29-36): ___/8
- Domain 5 (Q37-44): ___/8
- Domain 6 (Q45-50): ___/6

**Focus your remaining study time on your lowest-scoring domain.**
