# Domain 2: Implement Tool Use and Environment Interaction (20–25%)

## 📚 References & Documentation

| Resource | Link |
|----------|------|
| MS Learn Module | [Tooling, MCP, and Agent Execution Environments](https://learn.microsoft.com/en-us/training/modules/agent-tooling-mcp-execution-environments/) |
| GitHub Docs | [Custom Agents (Copilot SDK)](https://docs.github.com/en/copilot/how-tos/copilot-sdk/use-copilot-sdk/custom-agents) |
| GitHub Docs | [Copilot Extensions](https://docs.github.com/en/copilot/building-copilot-extensions) |
| MCP Specification | [Model Context Protocol](https://modelcontextprotocol.io/) |
| MCP Servers Registry | [MCP Servers](https://github.com/modelcontextprotocol/servers) |
| GitHub Docs | [GitHub Actions](https://docs.github.com/en/actions) |
| GitHub Docs | [Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) |
| GitHub Docs | [Agent Firewall](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/customize-the-agent-firewall) |
| Copilot SDK Repo | [github/copilot-sdk](https://github.com/github/copilot-sdk) |

---

## Section 1: Select and Configure Agent Tools

### 1.1 Identify Required Tools

**Tool Categories:**

| Category | Purpose | Examples |
|----------|---------|----------|
| **Read tools** | Explore and analyze code | `grep`, `glob`, `view`, `git log` |
| **Write tools** | Modify files | `edit`, `create`, `bash` |
| **Execution tools** | Run processes | `bash`, `powershell`, GitHub Actions |
| **Communication tools** | Interact with services | HTTP clients, API calls |
| **Version control** | Manage branches/PRs | `git`, GitHub CLI (`gh`) |

**Tool Selection Criteria:**
- What actions does the agent need to perform?
- What's the minimum set of tools required (least privilege)?
- Are there security implications for certain tools?
- Can the tool produce side effects?

### 1.2 Configure Agent Tools

Using the Copilot SDK:
```typescript
import { CopilotClient, defineTool } from "@github/copilot-sdk";
import { z } from "zod";

// Define a custom tool
const deployTool = defineTool("deploy-staging", {
  description: "Deploys the current branch to staging environment",
  parameters: z.object({
    branch: z.string().describe("Branch to deploy"),
    environment: z.enum(["staging", "preview"]),
  }),
  handler: async ({ branch, environment }) => {
    // Deployment logic
    return { status: "deployed", url: `https://${environment}.example.com` };
  },
});

const session = await client.createSession({
  tools: [deployTool],
});
```

### 1.3 Configure Agent Tool Permissions

**Principle of Least Privilege:**
```typescript
const session = await client.createSession({
  customAgents: [
    {
      name: "reader",
      description: "Read-only code analysis",
      tools: ["grep", "glob", "view"],  // NO write access
      prompt: "Analyze code. Never modify files.",
    },
    {
      name: "writer",
      description: "Makes code changes",
      tools: ["view", "edit"],  // Limited write access, no bash
      prompt: "Make precise code changes as instructed.",
    },
  ],
});
```

**Tool permission patterns:**
- `tools: null` — All tools available (use sparingly)
- `tools: ["grep", "glob", "view"]` — Read-only access
- `tools: ["view", "edit", "create"]` — File modification only
- `defaultAgent.excludedTools` — Hide tools from the main agent to force delegation

---

## Section 2: Configure MCP Servers

### 2.1 Model Context Protocol (MCP) Overview

**What is MCP?**
MCP (Model Context Protocol) is a standardized protocol that allows AI agents to interact with external tools, data sources, and services through a consistent interface.

**MCP Architecture:**
```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│   Agent     │────▶│ MCP Client  │────▶│  MCP Server  │
│ (LLM-based) │     │ (Protocol)  │     │ (Tool/Data)  │
└─────────────┘     └─────────────┘     └──────────────┘
                                              │
                                    ┌─────────┼─────────┐
                                    ▼         ▼         ▼
                              [Database] [API]    [File System]
```

**MCP Server Types:**
- **Stdio servers** — Local process communication
- **HTTP/SSE servers** — Remote server communication
- **GitHub Remote MCP** — GitHub-hosted, managed MCP servers

### 2.2 Add an MCP Server as a Tool to an Agent

**Configuration in `.github/copilot/mcp.json`:**
```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost:5432/mydb"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    }
  }
}
```

**In Copilot SDK:**
```typescript
const session = await client.createSession({
  customAgents: [
    {
      name: "db-analyst",
      description: "Queries and analyzes database schemas",
      mcpServers: {
        postgres: {
          command: "npx",
          args: ["-y", "@modelcontextprotocol/server-postgres"],
          env: { DATABASE_URL: process.env.DB_URL },
        },
      },
      prompt: "You analyze database schemas and query data.",
    },
  ],
});
```

### 2.3 Configure a GitHub Remote MCP Server

GitHub provides hosted MCP servers that don't require local setup:

```json
{
  "mcpServers": {
    "github": {
      "url": "https://api.github.com/mcp",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    }
  }
}
```

**Features of GitHub Remote MCP:**
- No local installation required
- Managed by GitHub (updates automatically)
- Integrated with GitHub authentication
- Scoped to repository/organization permissions

### 2.4 Configure MCP Registries

MCP Registries allow organizations to manage approved MCP servers centrally:

```json
{
  "registry": {
    "url": "https://registry.example.com/mcp",
    "servers": ["postgres", "redis", "elasticsearch"]
  }
}
```

### 2.5 Configure MCP Allow Lists

Restrict which MCP servers agents can use:

```json
{
  "mcpAllowList": [
    "@modelcontextprotocol/server-postgres",
    "@modelcontextprotocol/server-filesystem",
    "github-mcp-server"
  ],
  "mcpDenyList": [
    "@modelcontextprotocol/server-*-experimental"
  ]
}
```

---

## Section 3: Integrate Agents Within Development Environments

### 3.1 Evaluate the Execution Context for an Agent

**Execution contexts:**
| Context | Characteristics | Use Case |
|---------|----------------|----------|
| **IDE/Editor** | Interactive, user-present | Code suggestions, refactoring |
| **CI/CD Pipeline** | Automated, event-driven | Testing, deployment, code review |
| **Cloud Agent** | Autonomous, long-running | Complex multi-step tasks |
| **CLI** | Local, developer-initiated | Quick tasks, scripting |

### 3.2 Configure Agent's Scope to a Specific Repository

```yaml
# .github/copilot-setup-steps.yml
steps:
  - name: Setup repository context
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Branch: ${{ github.ref_name }}"
      
# .github/copilot/config.yml
scope:
  repository: "owner/repo-name"
  paths:
    include: ["src/", "tests/"]
    exclude: ["node_modules/", ".env*"]
```

### 3.3 Configure Agent Invoked in CI Workflow

```yaml
# .github/workflows/agent-review.yml
name: Agent Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  agent-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - name: Run agent review
        uses: github/copilot-agent-review@v1
        with:
          scope: "changed-files"
          checks: ["security", "performance", "style"]
```

### 3.4 Configure Agent to Use Branch-Based Scope

```yaml
# Agent operates only within its own branch
agent:
  branch_pattern: "agent/*"
  base_branch: "main"
  scope:
    - create_branch: true
    - push_to_branch: true
    - create_pr: true
    - merge: false  # Requires human approval
```

### 3.5 Enable Autonomous Actions (Branches & PRs)

```yaml
# GitHub Actions workflow enabling agent to create PRs
jobs:
  agent-task:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - name: Agent creates branch and PR
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          git checkout -b agent/fix-${{ github.run_id }}
          # ... agent makes changes ...
          git add .
          git commit -m "fix: agent-generated changes"
          git push origin agent/fix-${{ github.run_id }}
          gh pr create --title "Agent: Fix identified issues" \
            --body "Automated fix by agent" \
            --base main
```

### 3.6 Handle Environment-Specific Constraints

| Constraint | Configuration |
|-----------|--------------|
| Network access | Agent firewall allow/deny lists |
| File system | Path restrictions in tool config |
| Secrets | Environment-scoped secrets, never in agent context |
| Time limits | Workflow timeout settings |
| Resource limits | Container memory/CPU limits |

---

## Section 4: Operate Agents with Safe Execution Paths

### 4.1 Implement Error Handling

```typescript
// Error handling patterns for agent tools
const safeTool = defineTool("safe-deploy", {
  description: "Deploy with error handling",
  parameters: z.object({ branch: z.string() }),
  handler: async ({ branch }) => {
    try {
      const result = await deploy(branch);
      return { status: "success", result };
    } catch (error) {
      if (error instanceof NetworkError) {
        return { status: "retryable_error", message: error.message };
      }
      if (error instanceof AuthError) {
        return { status: "escalate", message: "Auth failure - needs human" };
      }
      return { status: "fatal_error", message: error.message };
    }
  },
});
```

### 4.2 Implement Retries

```typescript
// Retry pattern with exponential backoff
async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  baseDelay: number = 1000
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;
      if (!isRetryable(error)) throw error;
      const delay = baseDelay * Math.pow(2, attempt);
      await sleep(delay);
    }
  }
  throw new Error("Max retries exceeded");
}
```

### 4.3 Implement Rollbacks

```yaml
# Rollback pattern in GitHub Actions
jobs:
  agent-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Save state before changes
        run: |
          git stash
          echo "ROLLBACK_SHA=$(git rev-parse HEAD)" >> $GITHUB_ENV
          
      - name: Agent makes changes
        id: agent-changes
        continue-on-error: true
        run: |
          # Agent execution here
          
      - name: Rollback on failure
        if: steps.agent-changes.outcome == 'failure'
        run: |
          git reset --hard ${{ env.ROLLBACK_SHA }}
          git push --force-with-lease
          echo "::warning::Agent changes rolled back"
```

### 4.4 Implement Escalation Paths

```
┌─────────────────────────────────────────────┐
│          ESCALATION HIERARCHY                │
├─────────────────────────────────────────────┤
│ Level 0: Agent retries (automated)          │
│ Level 1: Agent asks for clarification       │
│ Level 2: Notify team channel (Slack/Teams)  │
│ Level 3: Assign to human developer          │
│ Level 4: Block and require manual override  │
└─────────────────────────────────────────────┘
```

### 4.5 Implement Traceability and Accountability

Every agent action should produce:
- **Who**: Which agent/configuration performed the action
- **What**: Exact changes made (diffs, artifacts)
- **When**: Timestamps for all actions
- **Why**: Decision rationale (linked to plan/prompt)
- **How**: Tools used, tokens consumed, execution path

```yaml
# Traceability via commit metadata
git commit -m "feat: add user validation

Agent: copilot-cloud-agent
Task-ID: task-abc123
Plan-SHA: plan-def456
Tools-Used: edit, bash, grep
Execution-Time: 45s
Tokens-Used: 12500"
```

---

## 🧠 Key Memorization Points

1. **Least privilege**: Always restrict tools to the minimum set needed
2. **MCP** = Model Context Protocol, standardized tool integration
3. **MCP server types**: Stdio (local), HTTP/SSE (remote), GitHub Remote (managed)
4. **Execution contexts**: IDE, CI/CD, Cloud Agent, CLI
5. **Branch-based scope** prevents agents from modifying protected branches directly
6. **Error handling hierarchy**: retry → rollback → escalate → block
7. **Traceability** = Who + What + When + Why + How
8. **Agent firewall** controls network access in cloud agent environments
9. **Allow lists** restrict which MCP servers and tools agents can use
10. **`defaultAgent.excludedTools`** forces delegation to specialized sub-agents

---

## 📝 Self-Assessment Questions

1. How do you restrict an agent to read-only operations using the Copilot SDK?
2. What is MCP and what problem does it solve?
3. Describe the difference between stdio and HTTP MCP servers.
4. How do you configure a GitHub Remote MCP server?
5. What's the purpose of MCP allow lists?
6. How do you enable an agent to create branches and PRs autonomously?
7. Describe the escalation hierarchy for agent failures.
8. What information should be captured for agent traceability?
9. How does branch-based scope protect the main branch from agent changes?
10. What's the role of `defaultAgent.excludedTools` in agent architecture?

➡️ **Practice these concepts with exercises**: [02-exercises-tools-mcp.md](./exercises/02-exercises-tools-mcp.md)
