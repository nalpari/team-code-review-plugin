# Chicago Code Review

Multi-agent code review plugin with 6+ expert personas analyzing code in parallel.

## Features

- **6 Core Expert Personas** analyzing code simultaneously:
  - **🏗️ Architect**: SOLID principles, coupling, cohesion, layer dependencies
  - **🔒 Security Engineer**: OWASP Top 10, auth/authz, input validation, secrets exposure
  - **⚡ Performance Engineer**: N+1 queries, memory leaks, algorithmic complexity, caching
  - **🧪 QA Engineer**: Edge cases, null handling, test coverage, idempotency
  - **📖 Code Craftsman**: Naming, readability, DRY, Clean Code principles
  - **🔄 DevOps Engineer**: Logging, monitoring, deployment safety, rollback capability
- **4 Optional Personas** (auto-activated by code context):
  - **💰 Business Analyst**: Domain logic accuracy, ubiquitous language
  - **🌐 Frontend Expert**: API contract changes, TypeScript types, UX states
  - **📊 Data Steward**: Schema changes, migration safety, data integrity
  - **👶 Junior Developer**: Implicit knowledge detection, documentation gaps
- **Parallel Subagent Execution**: No tmux or team agents required
- **Trade-off Analysis**: Conflicting opinions between experts are reconciled
- **Severity-Based Prioritization**: 🔴 CRITICAL / 🟠 HIGH / 🟡 MEDIUM / 🟢 LOW
- **User Confirmation**: Review results presented for approval before posting as PR comments

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated (for PR reviews)
- Access to Claude Sonnet 4.6 model

## Installation

```bash
claude plugin add https://github.com/nalpari/team-code-review-plugin.git --name chicago-code-review
```

## Usage

```text
# PR review
/chicago-code-review #42
/chicago-code-review owner/repo#42

# File or code review
/chicago-code-review (then provide file paths or code)

# Specific personas only
"Security와 Performance 관점으로만 리뷰해줘"
```

## Workflow

```text
Phase 0: Collect Code Input (PR diff / file / snippet)
    ↓
Phase 1: Orchestrator Context Analysis & Role Selection
    ↓
Phase 2: Parallel Subagent Execution (6-10 experts)
    ↓
Phase 3: Collect Results
    ↓
Phase 4: Orchestrator Synthesis & Conflict Resolution
    ↓
Phase 5: User Confirmation (PR review only)
    ↓
Phase 6: Post PR Comment (if confirmed)
    ↓
Phase 7: Cleanup
```

## Differences from other review plugins

| Feature | team-code-review | montreal-code-review | chicago-code-review |
|---------|-----------------|---------------------|---------------------|
| Expert personas | 2 | 4 | 6 core + 4 optional |
| Agent architecture | Subagents | Team agents (tmux) | Subagents (no tmux) |
| Discussion rounds | Cross-check agents | Leader-reviewer dialogue | Orchestrator synthesis |
| Trade-off analysis | No | No | Yes |
| Input types | PR only | PR only | PR / file / snippet |
| User confirmation | No | Yes | Yes |

## License

MIT
