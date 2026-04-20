# Montreal Code Review

Multi-agent team code review plugin with 3 specialized reviewers orchestrated by a leader agent.

## Features

- **4 Specialized Reviewers**: Each reviewer focuses on different aspects
  - **Reviewer 1 (Opus 4.6)**: Deep correctness, logic, and security analysis
  - **Reviewer 2 (Sonnet 4.6)**: Code quality, design patterns, and performance
  - **Reviewer 3 (Sonnet 4.6 + Codex)**: Adversarial review — edge cases, attack surfaces, failure modes
  - **Reviewer 4 (Sonnet 4.6 + CodeRabbit)**: CodeRabbit AI review — best practices, code smells, actionable suggestions
- **Leader-Reviewer Discussion**: The leader discusses findings with each reviewer, cross-referencing results for validation
- **Consensus-Based Synthesis**: Findings are tagged with confidence levels based on reviewer agreement
- **User Confirmation**: Review results are presented for approval before posting as PR comments

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated
- Access to Claude Opus 4.6 and Sonnet 4.6 models
- (Optional) [Codex adversarial-review skill](https://github.com/anthropics/codex) for enhanced adversarial analysis in Reviewer 3

## Installation

Claude Code 세션 내에서 다음 슬래시 명령을 실행하세요:

```text
/plugin marketplace add https://github.com/nalpari/team-code-review-plugin.git
/plugin install montreal-code-review@nalpari-plugins
```

> 참고: `nalpari-plugins`는 이 저장소의 `.claude-plugin/marketplace.json`에 정의된 marketplace 이름입니다.

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
Phase 1: Create Agent Team (TeamCreate)
    ↓
Phase 2: Spawn Reviewers & Assign Tasks
    ↓ (Opus, Sonnet, Sonnet+Codex, Sonnet+CodeRabbit as teammates)
Phase 3: Collect Reports
    ↓
Phase 4: Leader Discusses with Each Reviewer (SendMessage)
    ↓ (cross-reference, validate, refine)
Phase 5: Synthesize with Confidence Tags
    ↓ ([Consensus], [Majority], [Single + Validated], [Disputed])
Phase 6: User Confirmation
    ↓ (approve / edit / cancel)
Phase 7: Post PR Comment
    ↓
Cleanup: Shutdown Teammates & TeamDelete
```

## Differences from team-code-review

| Feature | team-code-review | montreal-code-review |
|---------|-----------------|---------------------|
| Reviewers | 2 (Opus + Sonnet) | 4 (Opus + Sonnet + Sonnet/Codex + Sonnet/CodeRabbit) |
| Adversarial review | No | Yes (Codex adversarial-review) |
| Agent architecture | Individual subagents | Agent team (TeamCreate/TeamDelete) |
| Cross-validation | Leader spawns cross-check agents | Leader discusses with team members |
| User confirmation | No (auto-posts) | Yes (asks before posting) |
| Confidence tags | No | Yes ([Consensus], [Majority], etc.) |
| Security section | No dedicated section | Dedicated section in output |

## License

MIT
