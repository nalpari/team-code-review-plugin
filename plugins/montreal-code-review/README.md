# Montreal Code Review

Multi-agent team code review plugin with 3 specialized reviewers orchestrated by a leader agent.

## Features

- **3 Specialized Reviewers**: Each reviewer uses a different model and focuses on different aspects
  - **Reviewer 1 (Opus 4.6)**: Deep correctness, logic, and security analysis
  - **Reviewer 2 (Sonnet 4.6)**: Code quality, design patterns, and performance
  - **Reviewer 3 (Haiku 4.5 + Codex)**: Adversarial review — edge cases, attack surfaces, failure modes
- **Leader-Reviewer Discussion**: The leader discusses findings with each reviewer, cross-referencing results for validation
- **Consensus-Based Synthesis**: Findings are tagged with confidence levels based on reviewer agreement
- **User Confirmation**: Review results are presented for approval before posting as PR comments

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated
- Access to Claude Opus 4.6, Sonnet 4.6, and Haiku 4.5 models
- (Optional) [Codex adversarial-review skill](https://github.com/anthropics/codex) for enhanced adversarial analysis in Reviewer 3

## Installation

```bash
claude plugin marketplace add https://github.com/nalpari/team-code-review-plugin.git
claude plugin install montreal-code-review
```

## Usage

```text
/montreal-review
/montreal-review #42
/montreal-review owner/repo#42
```

## Workflow

```text
Phase 0: Gather PR Context
    ↓
Phase 1: Launch 3 Reviewers in Parallel
    ↓ (Opus, Sonnet, Haiku+Codex)
Phase 2: Collect Reports
    ↓
Phase 3: Leader Discusses with Each Reviewer
    ↓ (cross-reference, validate, refine)
Phase 4: Synthesize with Confidence Tags
    ↓ ([Consensus], [Majority], [Single + Validated], [Disputed])
Phase 5: User Confirmation
    ↓ (approve / edit / cancel)
Phase 6: Post PR Comment & Cleanup
```

## Differences from team-code-review

| Feature | team-code-review | montreal-code-review |
|---------|-----------------|---------------------|
| Reviewers | 2 (Opus + Sonnet) | 3 (Opus + Sonnet + Haiku/Codex) |
| Adversarial review | No | Yes (Codex adversarial-review) |
| Cross-validation | Leader spawns cross-check agents | Leader discusses with original reviewers |
| User confirmation | No (auto-posts) | Yes (asks before posting) |
| Confidence tags | No | Yes ([Consensus], [Majority], etc.) |
| Security section | No dedicated section | Dedicated section |

## License

MIT
