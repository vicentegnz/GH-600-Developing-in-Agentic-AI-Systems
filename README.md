# 🎯 GH-600: Developing in Agentic AI Systems — Complete Study Plan

> 📖 **[Read the full study guide as a website →](https://vicentegnz.github.io/GH-600-Developing-in-Agentic-AI-Systems/)**

## Certification Overview

| Field | Details |
|-------|---------|
| **Exam** | GH-600: Developing in Agentic AI Systems |
| **Passing Score** | 700 / 1000 |
| **Format** | Multiple choice, case studies, drag-and-drop |
| **Audience** | Developers operating, integrating, and governing AI agents in production SDLC |
| **Study Guide** | [Official Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/gh-600) |
| **Exam Sandbox** | [Try the exam environment](https://aka.ms/examdemo) |

---

## 📚 Skills Measured (Weighted Domains)

| # | Domain | Weight | Study File |
|---|--------|--------|------------|
| 1 | Prepare Agent Architecture and SDLC Processes | 15–20% | [01-agent-architecture-sdlc.md](docs/01-agent-architecture-sdlc.md) |
| 2 | Implement Tool Use and Environment Interaction | 20–25% | [02-tool-use-environment.md](docs/02-tool-use-environment.md) |
| 3 | Manage Memory, State, and Execution | 10–15% | [03-memory-state-execution.md](docs/03-memory-state-execution.md) |
| 4 | Perform Evaluation, Error Analysis, and Tuning | 15–20% | [04-evaluation-error-tuning.md](docs/04-evaluation-error-tuning.md) |
| 5 | Orchestrate Multi-Agent Coordination | 15–20% | [05-multi-agent-coordination.md](docs/05-multi-agent-coordination.md) |
| 6 | Implement Guardrails and Accountability | 10–15% | [06-guardrails-accountability.md](docs/06-guardrails-accountability.md) |

---

## 📖 Official Learning Paths (Microsoft Learn)

| Module | Link |
|--------|------|
| Foundations of Agentic AI in GitHub | [Learn Module](https://learn.microsoft.com/en-us/training/modules/foundations-agentic-ai/) |
| Designing Agent Architecture and SDLC Integration | [Learn Module](https://learn.microsoft.com/en-us/training/modules/design-agent-architecture-integration/) |
| Tooling, MCP, and Agent Execution Environments | [Learn Module](https://learn.microsoft.com/en-us/training/modules/agent-tooling-mcp-execution-environments/) |

---

## 📘 Official GitHub Documentation References

| Topic | Documentation Link |
|-------|-------------------|
| Prepare Agent Architecture | [GitHub Docs - Prepare for Custom Agents](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-for-organization/prepare-for-custom-agents) |
| Custom Agents (SDK) | [GitHub Docs - Custom Agents](https://docs.github.com/en/copilot/how-tos/copilot-sdk/use-copilot-sdk/custom-agents) |
| Copilot Memory | [GitHub Docs - Copilot Memory](https://docs.github.com/en/copilot/concepts/agents/copilot-memory) |
| Managing Copilot Memory | [GitHub Docs - Managing Memory](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/copilot-memory) |
| Implementation Planner | [GitHub Docs - Implementation Planner](https://docs.github.com/en/copilot/tutorials/customization-library/custom-agents/implementation-planner) |
| Cloud Agent Guardrails | [GitHub Docs - Build Guardrails](https://docs.github.com/en/copilot/tutorials/cloud-agent/build-guardrails) |
| Risks and Mitigations | [GitHub Docs - Risks & Mitigations](https://docs.github.com/en/copilot/concepts/agents/cloud-agent/risks-and-mitigations) |
| Agent Firewall | [GitHub Docs - Agent Firewall](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/customize-the-agent-firewall) |
| Copilot SDK Repository | [github/copilot-sdk](https://github.com/github/copilot-sdk) |
| MCP Specification | [Model Context Protocol](https://modelcontextprotocol.io/) |
| GitHub Actions | [GitHub Docs - Actions](https://docs.github.com/en/actions) |
| Branch Protection & Rulesets | [GitHub Docs - Rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets) |
| CODEOWNERS | [GitHub Docs - CODEOWNERS](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) |

---

## � Study Plan (~35–40 h total)

> Complete activities at your own pace. Each item shows an estimated time investment.

### Phase 1: Foundations & Architecture (Domains 1 + 6) — ~8 h

| Est. Time | Focus | Activity |
|-----------|-------|----------|
| 1.5 h | Agentic AI Foundations | Complete MS Learn "Foundations of Agentic AI" module |
| 1 h | Agent vs Assistant distinctions | Study [01-agent-architecture-sdlc.md](docs/01-agent-architecture-sdlc.md) Sections 1-2 |
| 1 h | SDLC Integration & Planning | Study Section 3 + complete exercises |
| 1 h | Observability & Control | Study Section 4 + complete exercises |
| 1.5 h | Guardrails & Accountability | Study [06-guardrails-accountability.md](docs/06-guardrails-accountability.md) |
| 2 h | Exercises & Review | Complete all Phase 1 exercises + review key concepts |

### Phase 2: Tools, MCP & Environments (Domain 2) — ~9 h

| Est. Time | Focus | Activity |
|-----------|-------|----------|
| 1.5 h | Tool Selection & Config | Complete MS Learn "Tooling, MCP" module |
| 1 h | MCP Servers | Study [02-tool-use-environment.md](docs/02-tool-use-environment.md) Sections 1-2 |
| 1 h | Agent Execution Environments | Study Sections 3-4 |
| 1 h | Error Handling & Safe Execution | Study Section 4 + exercises |
| 2.5 h | Hands-on Lab | Build a custom MCP server configuration |
| 2 h | Exercises & Review | Complete all Phase 2 exercises + review configurations and patterns |

### Phase 3: Memory, Evaluation & Multi-Agent (Domains 3, 4, 5) — ~13 h

| Est. Time | Focus | Activity |
|-----------|-------|----------|
| 1.5 h | Memory Strategies | Study [03-memory-state-execution.md](docs/03-memory-state-execution.md) |
| 1.5 h | State & Context Drift | Study + complete memory exercises |
| 1.5 h | Evaluation & Success Criteria | Study [04-evaluation-error-tuning.md](docs/04-evaluation-error-tuning.md) Sections 1-2 |
| 1.5 h | Failure Analysis & Tuning | Study Sections 3 + complete exercises |
| 1.5 h | Multi-Agent Orchestration | Study [05-multi-agent-coordination.md](docs/05-multi-agent-coordination.md) Sections 1-2 |
| 1.5 h | Multi-Agent Failures & Lifecycle | Study Sections 3-4 + exercises |
| 4 h | Review | Review all Phase 3 material |

### Phase 4: Practice & Mastery — ~8 h

| Est. Time | Focus | Activity |
|-----------|-------|----------|
| 1 h | Full Review Domain 1 + 2 | Re-read notes, redo failed exercises |
| 1 h | Full Review Domain 3 + 4 | Re-read notes, redo failed exercises |
| 1 h | Full Review Domain 5 + 6 | Re-read notes, redo failed exercises |
| 2 h | Practice Exam | Complete [practice-questions.md](docs/practice-exam/practice-questions.md) |
| 1.5 h | Gap Analysis | Review wrong answers, study weak areas |
| 1.5 h | Final Review | Focus on highest-weighted domains (2, 4, 5) |

---

## 🏋️ Exercises Index

| Exercise File | Domain Coverage |
|---------------|-----------------|
| [01 - Architecture Exercises](docs/exercises/01-exercises-architecture.md) | Agent Architecture & SDLC |
| [02 - Tools & MCP Exercises](docs/exercises/02-exercises-tools-mcp.md) | Tool Use & Environment |
| [03 - Memory & State Exercises](docs/exercises/03-exercises-memory-state.md) | Memory, State, Execution |
| [04 - Evaluation Exercises](docs/exercises/04-exercises-evaluation.md) | Evaluation, Error Analysis, Tuning |
| [05 - Multi-Agent Exercises](docs/exercises/05-exercises-multi-agent.md) | Multi-Agent Coordination |
| [06 - Guardrails Exercises](docs/exercises/06-exercises-guardrails.md) | Guardrails & Accountability |
| [Practice Exam Questions](docs/practice-exam/practice-questions.md) | All Domains |

---

## 🏋️ Exercises Index

| Exercise File | Domain Coverage |
|---------------|----------------|
| [01 - Architecture Exercises](./exercises/01-exercises-architecture.md) | Agent Architecture & SDLC |
| [02 - Tools & MCP Exercises](./exercises/02-exercises-tools-mcp.md) | Tool Use & Environment |
| [03 - Memory & State Exercises](./exercises/03-exercises-memory-state.md) | Memory, State, Execution |
| [04 - Evaluation Exercises](./exercises/04-exercises-evaluation.md) | Evaluation, Error Analysis, Tuning |
| [05 - Multi-Agent Exercises](./exercises/05-exercises-multi-agent.md) | Multi-Agent Coordination |
| [06 - Guardrails Exercises](./exercises/06-exercises-guardrails.md) | Guardrails & Accountability |
| [Practice Exam Questions](./practice-exam/practice-questions.md) | All Domains |

---

## 💡 Study Tips

1. **Focus on the highest-weighted domains first** — Domains 2 (20-25%), 4 (15-20%), and 5 (15-20%) together make up 50-65% of the exam.
2. **Hands-on practice is essential** — Set up a GitHub repository and actually configure agents, MCP servers, and workflows.
3. **Understand the "why"** — The exam tests understanding of trade-offs, not just configuration steps.
4. **Think in terms of the contributor model** — Agents are treated like junior developers: their PRs need review, their access is scoped, their outputs are auditable.
5. **Know the anti-patterns** — Questions often present scenarios and ask you to identify what's wrong.

---

## 🔗 Additional Resources

- [GitHub Blog](https://github.blog/) — Latest announcements on Copilot and agent features
- [GitHub Community Discussions](https://github.com/orgs/community/discussions/categories/github-learn)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [GitHub Copilot SDK Repository](https://github.com/github/copilot-sdk)
- [Exam Scoring & Reports](https://learn.microsoft.com/en-us/credentials/certifications/exam-scoring-reports)
- [Request Exam Accommodations](https://learn.microsoft.com/en-us/credentials/certifications/request-accommodations)
