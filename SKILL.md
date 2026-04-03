---
name: montreal-code-review
description: "Multi-agent team code review with 4 specialized reviewers (Opus deep analysis + Sonnet pattern review + Sonnet adversarial review via Codex + Sonnet CodeRabbit review) orchestrated by a leader who discusses findings with each reviewer before finalizing. Use this whenever the user wants a thorough multi-model code review, says '/montreal-review', mentions 'montreal review', wants adversarial or devil's advocate review, needs cross-validated PR analysis, or wants a comprehensive review with discussion rounds before posting comments. Also trigger when user asks for a 'multi-perspective review', or emphasizes wanting different viewpoints on their code."
---

# Montreal Code Review

A multi-agent code review system where a leader orchestrates four specialized reviewers, engages in discussion with each reviewer about their findings, and presents the consolidated results to the user for approval before posting.

## Team Structure

- **Leader** (you, the main agent): Creates the agent team, orchestrates the review, discusses findings with each reviewer, synthesizes the final report, and asks the user for confirmation before posting
- **Reviewer 1** (Opus): Deep architectural analysis — focuses on correctness, logic flaws, and security
- **Reviewer 2** (Sonnet): Code quality and pattern analysis — focuses on design, readability, and performance
- **Reviewer 3** (Sonnet + Codex): Adversarial review — uses `/codex:adversarial-review` skill if available, otherwise conducts adversarial analysis directly to find edge cases, attack surfaces, and overlooked failure modes
- **Reviewer 4** (Sonnet + CodeRabbit): CodeRabbit review — uses `/coderabbit:review` skill to perform an AI-powered code review with CodeRabbit's methodology

## Review Focus Areas

Each reviewer has a specialized lens:

| Reviewer | Model | Primary Focus | Secondary Focus |
|----------|-------|---------------|-----------------|
| Reviewer 1 | Opus 4.6 | Correctness, logic, security | Architecture, data flow |
| Reviewer 2 | Sonnet 4.6 | Design patterns, readability | Performance, maintainability |
| Reviewer 3 | Sonnet 4.6 + Codex | Edge cases, failure modes | Attack surfaces, adversarial inputs |
| Reviewer 4 | Sonnet 4.6 + CodeRabbit | CodeRabbit AI review | Best practices, code smells, suggestions |

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
3. If the filtered diff exceeds ~3000 lines, further trim to the most impactful files (prioritize: auth, API routes, middleware, components, utilities, data models) and note which files were excluded
4. Save the focused diff to `/tmp/montreal-review-diff-focused.txt`

### Phase 1: Create Agent Team

Create a review team using `TeamCreate`. You (the leader) orchestrate the entire process.

```text
TeamCreate(
  team_name: "montreal-review",
  description: "Montreal Code Review team for PR #{pr_number}"
)
```

### Phase 2: Spawn Reviewers and Assign Tasks

**Step 2-1: Create review tasks** for each reviewer:

```text
TaskCreate(subject: "Review PR #{pr_number} — deep correctness & security analysis", description: "Read /tmp/montreal-review-diff-focused.txt and analyze for logic bugs, security vulnerabilities, and architecture issues.")
TaskCreate(subject: "Review PR #{pr_number} — code quality & pattern analysis", description: "Read /tmp/montreal-review-diff-focused.txt and analyze for design patterns, readability, and performance issues.")
TaskCreate(subject: "Review PR #{pr_number} — adversarial analysis", description: "Read /tmp/montreal-review-diff-focused.txt and analyze for edge cases, attack surfaces, and overlooked failure modes.")
TaskCreate(subject: "Review PR #{pr_number} — CodeRabbit review", description: "Use /coderabbit:review skill to review the PR and analyze for best practices, code smells, and actionable suggestions.")
```

**Step 2-2: Spawn 4 reviewers as teammates** — launch all four simultaneously in a single message:

```text
Agent(
  name: "reviewer-1-opus",
  model: "opus",
  team_name: "montreal-review",
  prompt: "You are Reviewer 1 (Opus) on the montreal-review team. Research-only — do NOT edit files.

  1. Check TaskList and claim the correctness & security analysis task (set owner to your name)
  2. Read the diff at /tmp/montreal-review-diff-focused.txt

  PR Title: {title}
  PR Description: {body}
  Base: {base} → Head: {head}

  Your specialty is deep correctness and security analysis:
  1. **Logic & Correctness**: Bugs, incorrect algorithms, missing edge cases, race conditions, null/undefined handling, off-by-one errors
  2. **Security**: Injection vulnerabilities, auth bypasses, data exposure, insecure defaults, OWASP top 10
  3. **Architecture**: Coupling issues, abstraction leaks, broken invariants, data flow problems

  For each finding provide:
  - File path and exact line number(s) from the diff (e.g., `src/auth.ts:42-45`)
  - Severity: CRITICAL / WARNING / INFO
  - Clear description of the issue
  - Suggested fix or approach

  **IMPORTANT — MUST-FIX items**: For every CRITICAL finding, you MUST additionally provide:
  - The exact file path and line number(s) where the fix is needed (e.g., `src/auth.ts:42`)
  - A concrete, actionable description of what code must change and how (not just what is wrong)
  - Mark these findings with a `[MUST-FIX]` prefix so the leader can extract them for the final report

  Also note 1-2 positive aspects of the code.
  Save your findings to /tmp/montreal-review-r1-findings.txt
  Mark your task as completed when done.
  Then wait for further instructions from the team lead — do not shut down."
)

Agent(
  name: "reviewer-2-sonnet",
  model: "sonnet",
  team_name: "montreal-review",
  prompt: "You are Reviewer 2 (Sonnet) on the montreal-review team. Research-only — do NOT edit files.

  1. Check TaskList and claim the code quality & pattern analysis task (set owner to your name)
  2. Read the diff at /tmp/montreal-review-diff-focused.txt

  PR Title: {title}
  PR Description: {body}
  Base: {base} → Head: {head}

  Your specialty is code quality and design analysis:
  1. **Design Patterns**: SOLID violations, code smells, unnecessary complexity, duplication, naming conventions
  2. **Readability**: Unclear logic, missing context, confusing control flow, poor abstraction boundaries
  3. **Performance**: Algorithmic complexity, memory leaks, unnecessary computations, N+1 queries, bundle size impact

  For each finding provide:
  - File path and exact line number(s) from the diff (e.g., `src/utils.ts:18-22`)
  - Severity: CRITICAL / WARNING / INFO
  - Clear description of the issue
  - Suggested fix or approach

  **IMPORTANT — MUST-FIX items**: For every CRITICAL finding, you MUST additionally provide:
  - The exact file path and line number(s) where the fix is needed (e.g., `src/utils.ts:18`)
  - A concrete, actionable description of what code must change and how (not just what is wrong)
  - Mark these findings with a `[MUST-FIX]` prefix so the leader can extract them for the final report

  Also note 1-2 positive aspects of the code.
  Save your findings to /tmp/montreal-review-r2-findings.txt
  Mark your task as completed when done.
  Then wait for further instructions from the team lead — do not shut down."
)

Agent(
  name: "reviewer-3-sonnet",
  model: "sonnet",
  team_name: "montreal-review",
  prompt: "You are Reviewer 3 (Sonnet) on the montreal-review team. Research-only — do NOT edit files.

  1. Check TaskList and claim the adversarial analysis task (set owner to your name)
  2. Read the diff at /tmp/montreal-review-diff-focused.txt

  PR Title: {title}
  PR Description: {body}
  Base: {base} → Head: {head}

  If the /codex:adversarial-review skill is available, invoke it and follow its methodology. Otherwise, conduct the adversarial analysis directly using the approach below. Your specialty is adversarial analysis — think like an attacker or a malicious user:

  1. **Edge Cases & Failure Modes**: What inputs break this? What happens under extreme load? What if dependencies fail? What if the network is unreliable?
  2. **Attack Surfaces**: Can this be exploited? Are there injection points? Can auth be bypassed? Is there data leakage?
  3. **Overlooked Scenarios**: What did the developer probably not think about? Concurrency issues? Timezone problems? Unicode edge cases? Empty/null states?

  For each finding provide:
  - File path and exact line number(s) from the diff (e.g., `src/api.ts:55-60`)
  - Severity: CRITICAL / WARNING / INFO
  - Attack vector or failure scenario description
  - Suggested mitigation

  **IMPORTANT — MUST-FIX items**: For every CRITICAL finding, you MUST additionally provide:
  - The exact file path and line number(s) where the fix is needed (e.g., `src/api.ts:55`)
  - A concrete, actionable description of what code must change and how (not just what is wrong)
  - Mark these findings with a `[MUST-FIX]` prefix so the leader can extract them for the final report

  Be creative and adversarial in your thinking.
  Save your findings to /tmp/montreal-review-r3-findings.txt
  Mark your task as completed when done.
  Then wait for further instructions from the team lead — do not shut down."
)

Agent(
  name: "reviewer-4-sonnet",
  model: "sonnet",
  team_name: "montreal-review",
  prompt: "You are Reviewer 4 (Sonnet) on the montreal-review team. Research-only — do NOT edit files.

  1. Check TaskList and claim the CodeRabbit review task (set owner to your name)
  2. Invoke the /coderabbit:review skill to review PR #{pr_number} in {owner}/{repo}

  PR Title: {title}
  PR Description: {body}
  Base: {base} → Head: {head}

  Your specialty is CodeRabbit-powered AI review. Use the /coderabbit:review skill to analyze the PR. The skill will provide its own methodology and analysis. After the CodeRabbit review completes, consolidate the results into the standard format:

  For each finding provide:
  - File path and exact line number(s) from the diff (e.g., `src/config.ts:30-35`)
  - Severity: CRITICAL / WARNING / INFO
  - Clear description of the issue
  - Suggested fix or approach

  **IMPORTANT — MUST-FIX items**: For every CRITICAL finding, you MUST additionally provide:
  - The exact file path and line number(s) where the fix is needed (e.g., `src/config.ts:30`)
  - A concrete, actionable description of what code must change and how (not just what is wrong)
  - Mark these findings with a `[MUST-FIX]` prefix so the leader can extract them for the final report

  Also note 1-2 positive aspects of the code.
  Save your findings to /tmp/montreal-review-r4-findings.txt
  Mark your task as completed when done.
  Then wait for further instructions from the team lead — do not shut down."
)
```

### Phase 3: Collect Reports

Wait for all four reviewers to complete their tasks. The leader monitors `TaskList` to track progress. Teammates automatically send a notification when they complete work, so there is no need to poll — just wait for the notifications. If a reviewer has not completed after 5 minutes, proceed with the available reports and note the missing reviewer in the final comment.

Once all review tasks are marked completed, the findings are available at:

- `/tmp/montreal-review-r1-findings.txt` — Reviewer 1 (Opus) findings
- `/tmp/montreal-review-r2-findings.txt` — Reviewer 2 (Sonnet) findings
- `/tmp/montreal-review-r3-findings.txt` — Reviewer 3 (Sonnet/Codex) findings
- `/tmp/montreal-review-r4-findings.txt` — Reviewer 4 (Sonnet/CodeRabbit) findings

### Phase 4: Leader-Reviewer Discussion Rounds

This is the critical differentiator — instead of just merging reports, the leader discusses findings with each reviewer. The reviewers are still alive as team members, so the leader sends follow-up messages using `SendMessage` to engage in back-and-forth dialogue.

**Discussion with Reviewer 1 (Opus):**

```text
SendMessage(
  to: "reviewer-1-opus",
  message: "Thank you for your review. I've also received findings from three other reviewers.

  Here is a summary of the other reviewers' findings that overlap or contrast with yours:
  {summarize relevant findings from R2, R3, and R4 that relate to R1's areas}

  Questions:
  1. Do you agree with these overlapping/contrasting findings?
  2. Are there any findings from the others that you think are false positives? Why?
  3. Given the other perspectives, would you add or modify any of your findings?
  4. Which of your CRITICAL findings do you have the highest confidence in?

  Please respond with your updated assessment."
)
```

**Discussion with Reviewer 2 (Sonnet):**

```text
SendMessage(
  to: "reviewer-2-sonnet",
  message: "Thank you for your review. I've also received findings from three other reviewers.

  Here is a summary of the other reviewers' findings that overlap or contrast with yours:
  {summarize relevant findings from R1, R3, and R4 that relate to R2's areas}

  Questions:
  1. Do you agree with these overlapping/contrasting findings?
  2. Are there any findings from the others that you think are false positives? Why?
  3. Given the other perspectives, would you add or modify any of your findings?
  4. Which of your findings do you consider most impactful for code maintainability?

  Please respond with your updated assessment."
)
```

**Discussion with Reviewer 3 (Sonnet/Codex):**

```text
SendMessage(
  to: "reviewer-3-sonnet",
  message: "Thank you for your adversarial review. I've also received findings from three other reviewers.

  Here is a summary of the other reviewers' findings that overlap or contrast with yours:
  {summarize relevant findings from R1, R2, and R4 that relate to R3's areas}

  Questions:
  1. Do any of the other reviewers' findings reveal additional attack surfaces you didn't consider?
  2. Are there any findings from the others that you think underestimate the risk? Why?
  3. Given the other perspectives, would you add or escalate any of your findings?
  4. What is the single most dangerous scenario you identified?

  Please respond with your updated assessment."
)
```

**Discussion with Reviewer 4 (Sonnet/CodeRabbit):**

```text
SendMessage(
  to: "reviewer-4-sonnet",
  message: "Thank you for your CodeRabbit review. I've also received findings from three other reviewers.

  Here is a summary of the other reviewers' findings that overlap or contrast with yours:
  {summarize relevant findings from R1, R2, and R3 that relate to R4's areas}

  Questions:
  1. Do you agree with these overlapping/contrasting findings?
  2. Did CodeRabbit catch anything that the other reviewers missed entirely?
  3. Are there any findings from the others that CodeRabbit's analysis would classify differently in severity?
  4. Which of your findings do you consider the most actionable for the PR author?

  Please respond with your updated assessment."
)
```

Wait for all discussion responses. Initiate a second round only if a reviewer's response raises a new CRITICAL finding not present in any initial report. Limit to at most one additional round per reviewer. The leader now has:
- 4 initial review reports
- 4 discussion-refined assessments

### Phase 5: Leader Synthesis

Synthesize all eight documents (4 initial reports + 4 discussion responses) using this decision matrix:

| Situation | Action | Confidence Tag |
|-----------|--------|---------------|
| 3+ reviewers agree | Include with highest confidence | `[Consensus]` |
| 2 out of 4 agree | Include with high confidence | `[Majority]` |
| 1 found it, confirmed in discussion | Include with moderate confidence | `[Single + Validated]` |
| 1 found it, others disagreed in discussion | Include only if evidence is compelling; note the disagreement | `[Disputed]` |
| Flagged as false positive by 2+ reviewers | Exclude unless strong override reason | — |
| Adversarial-only finding (R3) with no overlap | Include if the attack scenario is realistic and specific | `[Adversarial]` |
| CodeRabbit-only finding (R4) with no overlap | Include if the suggestion is actionable and specific | `[CodeRabbit]` |

**Confidence tagging**: Each finding in the final report gets a confidence indicator:
- `[Consensus]` — 3+ reviewers agree
- `[Majority]` — 2 out of 4 agree
- `[Single + Validated]` — One reviewer found it, validated in discussion
- `[Disputed]` — Disagreement exists, included with reasoning
- `[Adversarial]` — Adversarial reviewer only, accepted by leader judgment
- `[CodeRabbit]` — CodeRabbit reviewer only, accepted by leader judgment

Prioritize findings: CRITICAL > WARNING > INFO.

**Extracting MUST-FIX items**: After synthesis, collect all CRITICAL findings into a dedicated "필수 수정 사항" section. For each MUST-FIX item, ensure the following information is present:
1. **Sequential number** (e.g., `MF-1`, `MF-2`, ...)
2. **Exact file path and line number(s)** (e.g., `src/auth.ts:42-45`)
3. **Current problematic code** — quote the specific line(s) from the diff
4. **What must change** — a concrete, actionable description of the required fix
5. **Why it must change** — brief explanation of the risk (data loss, security vulnerability, runtime error, etc.)
6. **Confidence tag** — from the synthesis decision matrix

This section appears at the top of the review (right after the summary) so the PR author can immediately see what blocks the merge.

### Phase 6: User Confirmation

Before posting the review as a PR comment, present the synthesized review to the user and ask for confirmation.

Display the formatted review and ask:

```text
"리뷰 결과가 준비되었습니다. 아래 내용을 PR 코멘트로 게시할까요?

[전체 리뷰 내용 표시]

- 'yes' 또는 '게시': 코멘트를 PR에 게시합니다
- 'edit' 또는 '수정': 수정할 부분을 알려주세요
- 'no' 또는 '취소': 게시를 취소합니다"
```

Wait for the user's response before proceeding:
- If the user says yes/게시: proceed to Phase 7
- If the user says edit/수정: apply their requested changes and show the updated review again
- If the user says no/취소: skip Phase 7 and go directly to cleanup

### Phase 7: Post PR Comment

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
> **Reviewers**: Claude Opus 4.6, Claude Sonnet 4.6, Claude Sonnet 4.6 (+ Codex Adversarial), Claude Sonnet 4.6 (+ CodeRabbit)
> **Review method**: Independent parallel review → Leader-reviewer discussion → Consensus synthesis

---

## :memo: 요약
{1-3 문장으로 전체 변경 사항의 의도와 리뷰 결과 요약}

## :rotating_light: 필수 수정 사항 (머지 전 반드시 수정)

> 아래 항목은 머지를 차단하는 CRITICAL 이슈입니다. 각 항목의 파일 경로와 라인 번호를 확인하고 수정해 주세요.

{CRITICAL 이슈가 없으면 이 섹션 전체를 `:white_check_mark: 필수 수정 사항 없음 — 머지 가능합니다.`로 대체}

### MF-1: {이슈 제목} `[신뢰도 태그]`
- **위치**: `{파일경로}:{라인번호}` (예: `src/auth.ts:42-45`)
- **현재 코드**:
  ```
  {diff에서 해당 라인의 문제 코드 인용}
  ```
- **수정 내용**: {구체적으로 어떻게 변경해야 하는지 서술}
- **사유**: {왜 수정해야 하는지 — 런타임 에러, 데이터 손실, 보안 취약점 등}

### MF-2: ...

---

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

## :rabbit2: CodeRabbit 리뷰

| 우선순위 | 신뢰도 | 파일 | 위치 | 설명 | 제안 |
|----------|--------|------|------|------|------|

## :white_check_mark: 잘된 점
{긍정적 피드백 1-3개}

## :handshake: 리뷰어 논의 요약
{리더-리뷰어 논의에서 도출된 주요 합의/이견 사항}
{허위 양성으로 판정된 항목과 그 이유}

---
:robot: *Generated by Montreal Code Review (Opus 4.6 + Sonnet 4.6 + Sonnet 4.6/Codex + Sonnet 4.6/CodeRabbit)*
```

## Severity Guide

- **CRITICAL**: Must fix before merge. Runtime errors, data loss, security vulnerabilities, exploitable attack surfaces.
- **WARNING**: Should fix. Logic issues, poor patterns, maintenance burden, performance concerns, realistic edge cases.
- **INFO**: Nice to have. Style suggestions, minor improvements, alternative approaches, theoretical edge cases.

## Edge Cases

- **No PR found**: Ask the user which PR to review, or offer to review the current branch diff against main/master.
- **Any reviewer fails**: Log the error, proceed with the remaining reviewers, and note in the final comment which reviewers completed analysis. The discussion phase adapts — skip discussion with the failed reviewer.
- **Codex adversarial-review skill not available**: Reviewer 3 (Sonnet) falls back to manual adversarial analysis without the Codex skill. Note this in the final report.
- **CodeRabbit review skill not available**: Reviewer 4 (Sonnet) falls back to manual code review focusing on best practices, code smells, and actionable suggestions. Note this in the final report.
- **Very large diff (>3000 lines after filtering)**: Prioritize source files. Note excluded files in the review.
- **No issues found**: Still post a positive review confirming the code looks good, highlighting the positive aspects.
- **User declines to post**: Respect the decision. Clean up temporary files and end gracefully.
- **Discussion reveals new critical issue**: The leader may re-engage a specific reviewer for deeper investigation before finalizing. Limit to at most one additional round per reviewer to prevent unbounded discussion loops.

## Agent Team Lifecycle Management

After all phases are complete (whether the review was posted or cancelled), the leader must gracefully shut down the team:

**Step 1: Shutdown all teammates** — send shutdown requests to each reviewer:

```text
SendMessage(to: "reviewer-1-opus", message: {"type": "shutdown_request"})
SendMessage(to: "reviewer-2-sonnet", message: {"type": "shutdown_request"})
SendMessage(to: "reviewer-3-sonnet", message: {"type": "shutdown_request"})
SendMessage(to: "reviewer-4-sonnet", message: {"type": "shutdown_request"})
```

Wait for each reviewer to acknowledge the shutdown or timeout after 30 seconds before proceeding.

**Step 2: Delete the team** — once all teammates have shut down:

```text
TeamDelete()
```

This removes the team config (`~/.claude/teams/montreal-review/`) and task list (`~/.claude/tasks/montreal-review/`).

**Step 3: Clean up temporary files:**

```bash
rm -f /tmp/montreal-review-*.txt
```
