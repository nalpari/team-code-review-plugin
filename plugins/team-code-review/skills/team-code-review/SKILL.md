---
name: team-code-review
description: "Multi-agent team code review that spawns parallel reviewers (Claude Opus + Gemini) for thorough PR analysis. Use this whenever the user wants a code review of a PR, asks for multi-model or multi-perspective review, says '/team-review', mentions 'team review', or wants a comprehensive quality check before merging. Also trigger when the user emphasizes thoroughness, wants cross-validated review, or asks multiple models to review their code."
---

# Team Code Review

A multi-agent code review system that forms a reviewer team with different AI models, cross-checks findings between them, and posts a consolidated review as a PR comment.

## Team Structure

- **Leader** (you, the main agent): Orchestrates the review, synthesizes findings, posts the final comment
- **Reviewer 1**: Claude Opus 4.6 subagent — deep architectural and logic analysis
- **Reviewer 2**: Gemini (via `gemini` CLI) — fresh perspective on bugs, patterns, and performance

## Review Focus Areas

Each reviewer analyzes the PR diff for:
1. **Bugs / Logic Errors**: potential bugs, edge case omissions, race conditions, incorrect logic
2. **Code Quality / Patterns**: SOLID violations, code smells, duplication, naming, readability
3. **Performance**: algorithmic complexity, memory leaks, unnecessary computations, N+1 queries

## Workflow

Execute these phases in order. The key insight: independent parallel review first, then cross-validation, then synthesis. This catches issues that a single reviewer would miss.

### Phase 0: Gather PR Context

Determine the PR to review. The user may provide:
- PR number + repo: `nalpari/tech-blog#8`
- PR URL: `https://github.com/owner/repo/pull/8`
- Just a PR number (if in a git repo): `#8`
- Nothing (detect from current branch): `gh pr view --json number`

Fetch metadata and diff:

```bash
# Metadata
gh pr view <PR_NUMBER> --repo <OWNER/REPO> --json number,title,body,baseRefName,headRefName

# Full diff
gh pr diff <PR_NUMBER> --repo <OWNER/REPO> > /tmp/team-review-full-diff.txt
```

**Handling large diffs** (this is important — real PRs are often big):

1. Exclude lock files and auto-generated content: filter out `pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `*.generated.*`
2. Focus on source code: prioritize `src/`, `lib/`, `app/`, config files, and migrations
3. If the filtered diff exceeds ~3000 lines, further trim to the most impactful files (auth, API routes, components, utilities) and note which files were excluded
4. Save the focused diff to `/tmp/team-review-diff-focused.txt`

### Phase 1: Parallel Independent Review

Launch both reviewers simultaneously — they have no dependencies on each other.

**Reviewer 1 (Claude Opus 4.6)** — use the Agent tool:

```
Agent(
  model: "opus",
  run_in_background: true,
  prompt: "You are Reviewer 1 in a multi-agent code review team. This is research-only — do NOT edit files.

  Step 1: Read the diff file at /tmp/team-review-diff-focused.txt
  Step 2: Produce your review.

  PR Title: {title}
  PR Description: {body}
  Base: {base} → Head: {head}

  Focus areas:
  1. Bugs / Logic Errors — potential bugs, missing edge cases, race conditions, null handling
  2. Code Quality / Patterns — SOLID violations, code smells, duplication, naming
  3. Performance — algorithmic complexity, memory leaks, unnecessary computations

  For each finding: file path, line number (approx OK), severity (CRITICAL/WARNING/INFO), description, suggested fix.
  Note 1-2 positive aspects. Output as structured markdown."
)
```

**Reviewer 2 (Gemini)** — invoke via `gemini` CLI:

The `gemini -p` flag requires a prompt string argument. For large diffs, pipe the diff content via stdin and pass the review instructions via `-p`:

```bash
cat /tmp/team-review-diff-focused.txt | gemini -p "You are Reviewer 2 in a code review team. Review the PR diff provided via stdin.

PR Title: {title}
PR Description: {body}
Base: {base} → Head: {head}

Focus: 1) Bugs/Logic Errors 2) Code Quality/Patterns 3) Performance
For each finding: file path, severity (CRITICAL/WARNING/INFO), description, suggested fix.
Note 1-2 positives. Output as structured markdown." \
  -m gemini-2.5-flash --yolo 2>/dev/null \
  | grep -v "^YOLO mode\|^Loaded cached\|^Registering\|^Server '\|^Scheduling\|^Executing\|^MCP context" \
  > /tmp/team-review-gemini-result.txt
```

**Important Gemini CLI notes** (learned from testing):
- `-p` MUST have a string value — `gemini -p "your prompt here"`, not just piping to stdin
- Use `2>/dev/null` to suppress stderr, and pipe through `grep -v` to remove CLI noise (YOLO mode messages, MCP server registration logs)
- The default Gemini model is `gemini-2.5-flash`. If the user specifies a different model, validate it first with a simple test: `gemini -p "test" -m <model> 2>&1 | head -3`
- If Gemini fails (model not found, quota exceeded), proceed with Reviewer 1 only and note this in the final comment

### Phase 2: Cross-Check

Each reviewer examines the other's findings. This catches false positives, validates real issues, and surfaces missed items.

First, save both reviews to structured text files:
- `/tmp/team-review-r1-findings.txt` — Reviewer 1's findings (from Agent result)
- `/tmp/team-review-r2-findings.txt` — Reviewer 2's findings (from Gemini result)

Run both cross-checks in parallel:

**Reviewer 1 cross-checks Reviewer 2** — spawn Opus subagent:

```
Agent(
  model: "opus",
  run_in_background: true,
  prompt: "You are Reviewer 1 (Claude Opus). Cross-check Reviewer 2 (Gemini)'s findings. Research-only — do NOT edit files.

  Read: /tmp/team-review-r1-findings.txt (your findings)
  Read: /tmp/team-review-r2-findings.txt (Gemini's findings)

  For each Gemini finding: AGREE, DISAGREE, or ADD CONTEXT? Brief explanation.
  Did Gemini catch anything you missed?
  Are any of Gemini's findings false positives? Explain why.
  Any new issues from combined perspective?

  Output as structured markdown."
)
```

**Reviewer 2 cross-checks Reviewer 1** — via Gemini CLI:

```bash
gemini -p "You are Reviewer 2 (Gemini). Cross-check Reviewer 1 (Claude Opus)'s findings.

Your findings:
$(cat /tmp/team-review-r2-findings.txt)

Claude Opus's findings:
$(cat /tmp/team-review-r1-findings.txt)

For each of Claude's findings: AGREE, DISAGREE, or ADD CONTEXT?
Did Claude catch anything you missed?
Are any of Claude's findings false positives?
Any new issues from both perspectives?

Output as structured markdown." \
  -m gemini-2.5-flash --yolo 2>/dev/null \
  | grep -v "^YOLO mode\|^Loaded cached\|^Registering\|^Server '\|^Scheduling\|^Executing\|^MCP context" \
  > /tmp/team-review-gemini-crosscheck-result.txt
```

### Phase 3: Leader Synthesis

You now have four documents. Synthesize them using this decision matrix:

| Situation | Action |
|-----------|--------|
| Both agree on an issue | Include with high confidence |
| One found it, other confirmed in cross-check | Include |
| One found it, other disagreed | Include only if evidence is strong; note the disagreement |
| Only one found it, not addressed in cross-check | Include if specific and well-reasoned |
| Flagged as false positive by cross-checker | Exclude unless you have strong reason to override |

Prioritize findings by severity: CRITICAL first, then WARNING, then INFO.

### Phase 4: Post PR Comment

Format the final review in Korean and post as a PR comment:

```bash
gh pr comment <PR_NUMBER> --repo <OWNER/REPO> --body "$(cat <<'COMMENT_EOF'
<review content>
COMMENT_EOF
)"
```

## PR Comment Template

Adapt sections based on actual findings — omit empty sections.

```markdown
# :mag: Team Code Review

> **PR**: #{pr_number} {pr_title}
> **Reviewers**: Claude Opus 4.6, Gemini 2.5 Flash
> **Review method**: Independent parallel review + cross-validation

---

## :memo: 요약
{1-3 문장으로 전체 변경 사항의 의도와 리뷰 결과 요약}

## :bug: 버그 / 로직 오류

| 심각도 | 파일 | 위치 | 설명 | 제안 |
|--------|------|------|------|------|

## :triangular_ruler: 코드 품질 / 패턴

| 우선순위 | 파일 | 위치 | 설명 | 제안 |
|----------|------|------|------|------|

## :zap: 성능

| 영향도 | 파일 | 위치 | 설명 | 제안 |
|--------|------|------|------|------|

## :white_check_mark: 잘된 점
{긍정적 피드백 1-3개}

## :handshake: 리뷰어 합의
{크로스 체크에서 특히 중요했던 합의/이견 사항. 허위 양성으로 판정된 항목도 기록}

---
:robot: *Generated by Team Code Review (Claude Opus 4.6 + Gemini 2.5 Flash)*
```

## Severity Guide

- **CRITICAL**: Must fix before merge. Runtime errors, data loss, security vulnerabilities.
- **WARNING**: Should fix. Logic issues, poor patterns, maintenance burden, performance concerns.
- **INFO**: Nice to have. Style suggestions, minor improvements, alternative approaches.

## Edge Cases

- **No PR found**: Ask the user which PR to review, or offer to review the current branch diff against main/master.
- **Gemini CLI fails**: Log the error, proceed with Reviewer 1 only, and note in the final comment that only one model reviewed. Common failures: model not found (`ModelNotFoundError`), quota exceeded (`exhausted capacity`).
- **Gemini CLI not installed**: Fall back to a second Claude subagent with `model: "sonnet"` as Reviewer 2.
- **Very large diff (>3000 lines after filtering)**: Prioritize source files over docs/config. Note excluded files in the review.
- **No issues found**: Still post a positive review confirming the code looks good, highlighting the positive aspects.
- **Cross-repo PR**: Always use `--repo owner/repo` with `gh` commands when the current directory is not the target repo.

## Cleanup

After posting the comment, remove temporary files:
```bash
rm -f /tmp/team-review-*.txt
```
