---
name: montreal-code-review
description: "Multi-agent team code review with 3 specialized reviewers (Opus deep analysis + Sonnet pattern review + Haiku adversarial review via Codex) orchestrated by a leader who discusses findings with each reviewer before finalizing. Use this whenever the user wants a thorough multi-model code review, says '/montreal-review', mentions 'montreal review', wants adversarial or devil's advocate review, needs cross-validated PR analysis, or wants a comprehensive review with discussion rounds before posting comments. Also trigger when user asks for a '3-model review', 'multi-perspective review', or emphasizes wanting different viewpoints on their code."
---

# Montreal Code Review

A multi-agent code review system where a leader orchestrates three specialized reviewers, engages in discussion with each reviewer about their findings, and presents the consolidated results to the user for approval before posting.

## Team Structure

- **Leader** (you, the main agent): Creates the agent team, orchestrates the review, discusses findings with each reviewer, synthesizes the final report, and asks the user for confirmation before posting
- **Reviewer 1** (Opus): Deep architectural analysis — focuses on correctness, logic flaws, and security
- **Reviewer 2** (Sonnet): Code quality and pattern analysis — focuses on design, readability, and performance
- **Reviewer 3** (Haiku + Codex): Adversarial review — uses `/codex:adversarial-review` skill to find edge cases, attack surfaces, and overlooked failure modes

## Review Focus Areas

Each reviewer has a specialized lens:

| Reviewer | Model | Primary Focus | Secondary Focus |
|----------|-------|---------------|-----------------|
| Reviewer 1 | Opus 4.6 | Correctness, logic, security | Architecture, data flow |
| Reviewer 2 | Sonnet 4.6 | Design patterns, readability | Performance, maintainability |
| Reviewer 3 | Haiku 4.5 + Codex | Edge cases, failure modes | Attack surfaces, adversarial inputs |

## Workflow

Execute these phases in strict order. The key difference from a simple parallel review: the leader actively discusses findings with each reviewer to refine and validate results before synthesis.

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
gh pr diff <PR_NUMBER> --repo <OWNER/REPO> > /tmp/montreal-review-full-diff.txt
```

**Handling large diffs:**

1. Exclude lock files and auto-generated content: filter out `pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `*.generated.*`, `*.min.js`, `*.min.css`
2. Focus on source code: prioritize `src/`, `lib/`, `app/`, config files, and migrations
3. If the filtered diff exceeds ~3000 lines, further trim to the most impactful files and note which files were excluded
4. Save the focused diff to `/tmp/montreal-review-diff-focused.txt`

### Phase 1: Parallel Independent Review — Launch All 3 Reviewers

Launch all three reviewers simultaneously using the Agent tool. Each reviewer works independently with no knowledge of the others.

**Reviewer 1 (Opus — Deep Analysis):**

```
Agent(
  name: "reviewer-1-opus",
  model: "opus",
  run_in_background: true,
  prompt: "You are Reviewer 1 (Opus) in a 3-person code review team. Research-only — do NOT edit files.

  Read the diff at /tmp/montreal-review-diff-focused.txt

  PR Title: {title}
  PR Description: {body}
  Base: {base} → Head: {head}

  Your specialty is deep correctness and security analysis:
  1. **Logic & Correctness**: Bugs, incorrect algorithms, missing edge cases, race conditions, null/undefined handling, off-by-one errors
  2. **Security**: Injection vulnerabilities, auth bypasses, data exposure, insecure defaults, OWASP top 10
  3. **Architecture**: Coupling issues, abstraction leaks, broken invariants, data flow problems

  For each finding provide:
  - File path and approximate line number
  - Severity: CRITICAL / WARNING / INFO
  - Clear description of the issue
  - Suggested fix or approach

  Also note 1-2 positive aspects of the code.
  Output as structured markdown with clear sections."
)
```

**Reviewer 2 (Sonnet — Quality & Patterns):**

```
Agent(
  name: "reviewer-2-sonnet",
  model: "sonnet",
  run_in_background: true,
  prompt: "You are Reviewer 2 (Sonnet) in a 3-person code review team. Research-only — do NOT edit files.

  Read the diff at /tmp/montreal-review-diff-focused.txt

  PR Title: {title}
  PR Description: {body}
  Base: {base} → Head: {head}

  Your specialty is code quality and design analysis:
  1. **Design Patterns**: SOLID violations, code smells, unnecessary complexity, duplication, naming conventions
  2. **Readability**: Unclear logic, missing context, confusing control flow, poor abstraction boundaries
  3. **Performance**: Algorithmic complexity, memory leaks, unnecessary computations, N+1 queries, bundle size impact

  For each finding provide:
  - File path and approximate line number
  - Severity: CRITICAL / WARNING / INFO
  - Clear description of the issue
  - Suggested fix or approach

  Also note 1-2 positive aspects of the code.
  Output as structured markdown with clear sections."
)
```

**Reviewer 3 (Haiku + Codex Adversarial Review):**

```
Agent(
  name: "reviewer-3-haiku",
  model: "haiku",
  run_in_background: true,
  prompt: "You are Reviewer 3 (Haiku) in a 3-person code review team. Research-only — do NOT edit files.

  Read the diff at /tmp/montreal-review-diff-focused.txt

  PR Title: {title}
  PR Description: {body}
  Base: {base} → Head: {head}

  You must invoke the /codex:adversarial-review skill and follow its methodology to conduct your review. Your specialty is adversarial analysis — think like an attacker or a malicious user:

  1. **Edge Cases & Failure Modes**: What inputs break this? What happens under extreme load? What if dependencies fail? What if the network is unreliable?
  2. **Attack Surfaces**: Can this be exploited? Are there injection points? Can auth be bypassed? Is there data leakage?
  3. **Overlooked Scenarios**: What did the developer probably not think about? Concurrency issues? Timezone problems? Unicode edge cases? Empty/null states?

  For each finding provide:
  - File path and approximate line number
  - Severity: CRITICAL / WARNING / INFO
  - Attack vector or failure scenario description
  - Suggested mitigation

  Be creative and adversarial in your thinking. Output as structured markdown."
)
```

### Phase 2: Collect Reports and Save

Wait for all three reviewers to complete. As each reviewer finishes, save their findings:

- `/tmp/montreal-review-r1-findings.txt` — Reviewer 1 (Opus) findings
- `/tmp/montreal-review-r2-findings.txt` — Reviewer 2 (Sonnet) findings
- `/tmp/montreal-review-r3-findings.txt` — Reviewer 3 (Haiku/Codex) findings

### Phase 3: Leader-Reviewer Discussion Rounds

This is the critical differentiator — instead of just merging reports, the leader discusses findings with each reviewer. The purpose is to validate findings, resolve ambiguities, and deepen understanding.

For each reviewer, the leader sends a follow-up message using `SendMessage` to the named agent. This creates a back-and-forth dialogue.

**Discussion with Reviewer 1 (Opus):**

```
SendMessage(
  to: "reviewer-1-opus",
  message: "Thank you for your review. I've also received findings from two other reviewers.

  Here is a summary of the other reviewers' findings that overlap or contrast with yours:
  {summarize relevant findings from R2 and R3 that relate to R1's areas}

  Questions:
  1. Do you agree with these overlapping/contrasting findings?
  2. Are there any findings from the others that you think are false positives? Why?
  3. Given the other perspectives, would you add or modify any of your findings?
  4. Which of your CRITICAL findings do you have the highest confidence in?

  Please respond with your updated assessment."
)
```

**Discussion with Reviewer 2 (Sonnet):**

```
SendMessage(
  to: "reviewer-2-sonnet",
  message: "Thank you for your review. I've also received findings from two other reviewers.

  Here is a summary of the other reviewers' findings that overlap or contrast with yours:
  {summarize relevant findings from R1 and R3 that relate to R2's areas}

  Questions:
  1. Do you agree with these overlapping/contrasting findings?
  2. Are there any findings from the others that you think are false positives? Why?
  3. Given the other perspectives, would you add or modify any of your findings?
  4. Which of your findings do you consider most impactful for code maintainability?

  Please respond with your updated assessment."
)
```

**Discussion with Reviewer 3 (Haiku/Codex):**

```
SendMessage(
  to: "reviewer-3-haiku",
  message: "Thank you for your adversarial review. I've also received findings from two other reviewers.

  Here is a summary of the other reviewers' findings that overlap or contrast with yours:
  {summarize relevant findings from R1 and R2 that relate to R3's areas}

  Questions:
  1. Do any of the other reviewers' findings reveal additional attack surfaces you didn't consider?
  2. Are there any findings from the others that you think underestimate the risk? Why?
  3. Given the other perspectives, would you add or escalate any of your findings?
  4. What is the single most dangerous scenario you identified?

  Please respond with your updated assessment."
)
```

Wait for all discussion responses. The leader now has:
- 3 initial review reports
- 3 discussion-refined assessments

### Phase 4: Leader Synthesis

Synthesize all six documents (3 initial reports + 3 discussion responses) using this decision matrix:

| Situation | Action |
|-----------|--------|
| All 3 reviewers agree | Include with highest confidence |
| 2 out of 3 agree | Include with high confidence |
| 1 found it, confirmed in discussion | Include with moderate confidence |
| 1 found it, others disagreed in discussion | Include only if evidence is compelling; note the disagreement |
| Flagged as false positive by 2+ reviewers | Exclude unless strong override reason |
| Adversarial-only finding (R3) with no overlap | Include if the attack scenario is realistic and specific |

**Confidence tagging**: Each finding in the final report gets a confidence indicator:
- `[Consensus]` — All reviewers agree
- `[Majority]` — 2 out of 3 agree
- `[Single + Validated]` — One reviewer found it, validated in discussion
- `[Disputed]` — Disagreement exists, included with reasoning

Prioritize findings: CRITICAL > WARNING > INFO.

### Phase 5: User Confirmation

Before posting the review as a PR comment, present the synthesized review to the user and ask for confirmation.

Display the formatted review and ask:

```
"리뷰 결과가 준비되었습니다. 아래 내용을 PR 코멘트로 게시할까요?

[전체 리뷰 내용 표시]

- 'yes' 또는 '게시': 코멘트를 PR에 게시합니다
- 'edit' 또는 '수정': 수정할 부분을 알려주세요
- 'no' 또는 '취소': 게시를 취소합니다"
```

Wait for the user's response before proceeding:
- If the user says yes/게시: proceed to Phase 6
- If the user says edit/수정: apply their requested changes and show the updated review again
- If the user says no/취소: skip Phase 6 and go directly to cleanup

### Phase 6: Post PR Comment

Only execute this phase after user confirmation.

```bash
gh pr comment <PR_NUMBER> --repo <OWNER/REPO> --body "$(cat <<'COMMENT_EOF'
<review content>
COMMENT_EOF
)"
```

## PR Comment Template

Adapt sections based on actual findings — omit empty sections.

```markdown
# :mag: Montreal Code Review

> **PR**: #{pr_number} {pr_title}
> **Reviewers**: Claude Opus 4.6, Claude Sonnet 4.6, Claude Haiku 4.5 (+ Codex Adversarial)
> **Review method**: Independent parallel review → Leader-reviewer discussion → Consensus synthesis

---

## :memo: 요약
{1-3 문장으로 전체 변경 사항의 의도와 리뷰 결과 요약}

## :bug: 버그 / 로직 오류

| 심각도 | 신뢰도 | 파일 | 위치 | 설명 | 제안 |
|--------|--------|------|------|------|------|

## :triangular_ruler: 코드 품질 / 패턴

| 우선순위 | 신뢰도 | 파일 | 위치 | 설명 | 제안 |
|----------|--------|------|------|------|------|

## :zap: 성능

| 영향도 | 신뢰도 | 파일 | 위치 | 설명 | 제안 |
|--------|--------|------|------|------|------|

## :shield: 보안 / 적대적 분석

| 심각도 | 신뢰도 | 파일 | 위치 | 공격 시나리오 | 완화 방안 |
|--------|--------|------|------|--------------|----------|

## :white_check_mark: 잘된 점
{긍정적 피드백 1-3개}

## :handshake: 리뷰어 논의 요약
{리더-리뷰어 논의에서 도출된 주요 합의/이견 사항}
{허위 양성으로 판정된 항목과 그 이유}

---
:robot: *Generated by Montreal Code Review (Opus 4.6 + Sonnet 4.6 + Haiku 4.5/Codex)*
```

## Severity Guide

- **CRITICAL**: Must fix before merge. Runtime errors, data loss, security vulnerabilities, exploitable attack surfaces.
- **WARNING**: Should fix. Logic issues, poor patterns, maintenance burden, performance concerns, realistic edge cases.
- **INFO**: Nice to have. Style suggestions, minor improvements, alternative approaches, theoretical edge cases.

## Edge Cases

- **No PR found**: Ask the user which PR to review, or offer to review the current branch diff against main/master.
- **Any reviewer fails**: Log the error, proceed with the remaining reviewers, and note in the final comment which reviewers completed analysis. The discussion phase adapts — skip discussion with the failed reviewer.
- **Codex adversarial-review skill not available**: Reviewer 3 (Haiku) falls back to manual adversarial analysis without the Codex skill. Note this in the final report.
- **Very large diff (>3000 lines after filtering)**: Prioritize source files. Note excluded files in the review.
- **No issues found**: Still post a positive review confirming the code looks good, highlighting the positive aspects.
- **User declines to post**: Respect the decision. Clean up temporary files and end gracefully.
- **Discussion reveals new critical issue**: The leader may re-engage a specific reviewer for deeper investigation before finalizing.

## Agent Lifecycle Management

After all phases are complete (whether the review was posted or cancelled):

1. All named agents (`reviewer-1-opus`, `reviewer-2-sonnet`, `reviewer-3-haiku`) have completed their work through the discussion rounds
2. No further messages will be sent to them
3. Clean up all temporary files:

```bash
rm -f /tmp/montreal-review-*.txt
```

The agents are naturally released when the conversation moves on — there is no explicit "kill" needed, but the leader confirms all agent interactions are finalized before cleanup.
