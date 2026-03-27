# Team Code Review Plugin

Multi-agent team code review plugin for Claude Code. Spawns parallel reviewers (Claude Opus + Gemini) that independently analyze your PR, cross-validate each other's findings, and post a consolidated review comment.

## Features

- **Parallel multi-model review**: Claude Opus 4.6 + Gemini 2.5 Flash review independently
- **Cross-validation**: Each reviewer verifies the other's findings to reduce false positives
- **Consolidated output**: Leader agent synthesizes all findings into a single, structured PR comment
- **Korean review comments**: Final review is posted in Korean with clear severity levels
- **Large diff handling**: Automatically filters lock files and prioritizes source code

## Review Focus Areas

| Area | What it checks |
|------|---------------|
| Bugs / Logic Errors | Runtime errors, missing edge cases, race conditions, null handling |
| Code Quality / Patterns | SOLID violations, code smells, duplication, naming |
| Performance | Algorithmic complexity, memory leaks, unnecessary computations |

## Installation

```bash
npx skills add nalpari/team-code-review-plugin@team-code-review
```

## Usage

In Claude Code, use any of the following:

- `/team-code-review` or `/team-review`
- "team review PR #123"
- "review this PR with multiple models"
- "do a thorough code review of nalpari/repo#8"

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- [GitHub CLI (`gh`)](https://cli.github.com/) authenticated
- (Optional) [Gemini CLI (`gemini`)](https://github.com/google-gemini/gemini-cli) installed for multi-model review. Falls back to Claude Sonnet if unavailable.

## How It Works

```
Phase 0: Gather PR diff
    |
Phase 1: Parallel independent review
    |-- Reviewer 1 (Claude Opus 4.6)
    |-- Reviewer 2 (Gemini 2.5 Flash)
    |
Phase 2: Cross-validation
    |-- Reviewer 1 checks Reviewer 2's findings
    |-- Reviewer 2 checks Reviewer 1's findings
    |
Phase 3: Leader synthesizes with consensus matrix
    |
Phase 4: Post consolidated PR comment
```

## License

MIT
