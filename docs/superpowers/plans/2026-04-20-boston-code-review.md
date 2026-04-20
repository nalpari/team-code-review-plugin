# Boston Code Review Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** CodeRabbit 단일 baseline + 5개 전문가 에이전트 자동 감지 구조의 경량 멀티 에이전트 코드리뷰 플러그인 `boston-code-review`를 `plugins/` 디렉터리에 신설한다.

**Architecture:** 기존 `chicago-code-review`, `montreal-code-review`와 동일한 플러그인 디렉터리 구조를 따른다. `.claude-plugin/plugin.json` 매니페스트 + `skills/boston-code-review/SKILL.md` 스킬 파일 + `README.md` 설치·사용 안내로 구성하고, 최상위 `.claude-plugin/marketplace.json`에 플러그인 엔트리를 추가한다. SKILL.md는 승인된 스펙(`docs/superpowers/specs/2026-04-20-boston-code-review-design.md`)의 Phase 0~6 흐름을 300줄 이하로 압축해 기술한다.

**Tech Stack:** Markdown(SKILL.md, README.md), JSON(manifest), Bash/Git/gh CLI(검증). 언어 런타임·테스트 프레임워크 없음(스킬 콘텐츠·문서 작업).

**승인된 스펙:** [docs/superpowers/specs/2026-04-20-boston-code-review-design.md](../specs/2026-04-20-boston-code-review-design.md)

---

## File Structure

**신규 생성:**
- `plugins/boston-code-review/.claude-plugin/plugin.json` — 플러그인 매니페스트 (name, description, version, author)
- `plugins/boston-code-review/README.md` — 기능 요약·설치 명령·Workflow 다이어그램·플러그인 비교표·라이선스
- `plugins/boston-code-review/skills/boston-code-review/SKILL.md` — 스킬 본문 (frontmatter + Phase 0~6 + 자동 감지 규칙 + 리포트 포맷 + 엣지 케이스)

**수정:**
- `.claude-plugin/marketplace.json` — `plugins` 배열에 `boston-code-review` 엔트리 추가

**파일 책임 경계:**
- `plugin.json`: 플러그인 메타데이터만. 스킬 내용 없음.
- `SKILL.md`: 오케스트레이터 실행 로직의 전부. 300줄 이하. 단일 스킬 진입점.
- `README.md`: 외부 사용자 대상 설명. SKILL.md 내용을 중복하지 않고 요약만.
- `marketplace.json`: 마켓플레이스 인덱스. 플러그인 발견 경로만 제공.

---

## Task 1: 플러그인 디렉터리 스캐폴드 생성

**Files:**
- Create: `plugins/boston-code-review/.claude-plugin/plugin.json`
- Create: `plugins/boston-code-review/skills/boston-code-review/` (디렉터리)

- [ ] **Step 1-1: 디렉터리 구조 생성**

Run:
```bash
mkdir -p plugins/boston-code-review/.claude-plugin
mkdir -p plugins/boston-code-review/skills/boston-code-review
```

Verify:
```bash
ls -la plugins/boston-code-review/
ls -la plugins/boston-code-review/.claude-plugin/
ls -la plugins/boston-code-review/skills/boston-code-review/
```

Expected: 세 디렉터리 모두 존재, `.claude-plugin/`와 `skills/boston-code-review/`는 비어 있음.

- [ ] **Step 1-2: `plugin.json` 매니페스트 작성**

파일: `plugins/boston-code-review/.claude-plugin/plugin.json`

```json
{
  "name": "boston-code-review",
  "description": "Lightweight multi-agent code review with CodeRabbit baseline + up to 5 auto-detected specialists (Security, Performance, Data Steward, Business Analyst, Frontend). Orchestrator activates specialists only when diff patterns match, reducing token cost by ~80% on small PRs while preserving coverage on complex changes.",
  "version": "1.0.0",
  "author": {
    "name": "nalpari",
    "url": "https://github.com/nalpari"
  }
}
```

- [ ] **Step 1-3: JSON 유효성 검증**

Run:
```bash
python -c "import json; json.load(open('plugins/boston-code-review/.claude-plugin/plugin.json'))" && echo "JSON OK"
```

Expected: `JSON OK` 출력, 오류 없음.

- [ ] **Step 1-4: 커밋**

```bash
git add plugins/boston-code-review/.claude-plugin/plugin.json
git commit -m "feat(boston): scaffold plugin manifest"
```

---

## Task 2: 마켓플레이스 매니페스트에 플러그인 등록

**Files:**
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 2-1: 현재 마켓플레이스 매니페스트 확인**

Run:
```bash
cat .claude-plugin/marketplace.json
```

Expected: 3개 플러그인(`team-code-review`, `montreal-code-review`, `chicago-code-review`) 엔트리 존재.

- [ ] **Step 2-2: `boston-code-review` 엔트리 추가**

파일: `.claude-plugin/marketplace.json`

`chicago-code-review` 엔트리 뒤에 boston 엔트리 추가 (배열 마지막 원소의 닫는 `}`와 `]` 사이).

변경 후 전체 파일:

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "nalpari-plugins",
  "description": "Claude Code plugins by nalpari",
  "owner": {
    "name": "nalpari"
  },
  "plugins": [
    {
      "name": "team-code-review",
      "description": "Multi-agent team code review that spawns parallel reviewers (Claude Opus + Claude Sonnet) for thorough PR analysis with cross-validation",
      "source": "./plugins/team-code-review",
      "category": "development",
      "homepage": "https://github.com/nalpari/team-code-review-plugin"
    },
    {
      "name": "montreal-code-review",
      "description": "Multi-agent team code review with 4 specialized reviewers (Opus + Sonnet + Sonnet/Codex adversarial + Sonnet/CodeRabbit) orchestrated by a leader with discussion rounds and user confirmation",
      "source": "./plugins/montreal-code-review",
      "category": "development",
      "homepage": "https://github.com/nalpari/team-code-review-plugin"
    },
    {
      "name": "chicago-code-review",
      "description": "Multi-agent code review with 5+ expert personas (Security, Performance, QA, Craftsman, CodeRabbit) analyzing code in parallel. Orchestrator validates and filters findings before producing unified report with severity-based prioritization.",
      "source": "./plugins/chicago-code-review",
      "category": "development",
      "homepage": "https://github.com/nalpari/team-code-review-plugin"
    },
    {
      "name": "boston-code-review",
      "description": "Lightweight multi-agent code review with CodeRabbit baseline + up to 5 auto-detected specialists (Security, Performance, Data Steward, Business Analyst, Frontend). Token-efficient alternative to chicago-code-review; activates specialists only when diff patterns match.",
      "source": "./plugins/boston-code-review",
      "category": "development",
      "homepage": "https://github.com/nalpari/team-code-review-plugin"
    }
  ]
}
```

- [ ] **Step 2-3: JSON 유효성 및 엔트리 수 검증**

Run:
```bash
python -c "
import json
m = json.load(open('.claude-plugin/marketplace.json'))
names = [p['name'] for p in m['plugins']]
assert 'boston-code-review' in names, f'boston-code-review missing. Found: {names}'
assert len(names) == 4, f'Expected 4 plugins, got {len(names)}'
print('OK:', names)
"
```

Expected: `OK: ['team-code-review', 'montreal-code-review', 'chicago-code-review', 'boston-code-review']`

- [ ] **Step 2-4: 커밋**

```bash
git add .claude-plugin/marketplace.json
git commit -m "feat(marketplace): register boston-code-review plugin"
```

---

## Task 3: SKILL.md 본문 작성 (Phase 0~2)

**Files:**
- Create: `plugins/boston-code-review/skills/boston-code-review/SKILL.md`

이 SKILL.md는 오케스트레이터(메인 Claude)가 실행할 전체 흐름을 기술한다. Task 3~5에 걸쳐 단일 파일을 점진적으로 작성한다. 파일은 한 번에 만들고, 각 Task에서 해당 섹션만 검증한다.

- [ ] **Step 3-1: frontmatter + 서두 + Phase 0 작성**

파일: `plugins/boston-code-review/skills/boston-code-review/SKILL.md`

아래 내용을 **파일 처음부터** 작성한다 (이후 Step에서 덧붙임).

````markdown
---
name: boston-code-review
description: >
  토큰 효율을 우선하는 경량 멀티 에이전트 코드리뷰 스킬.
  CodeRabbit을 baseline으로 항상 실행하고, diff 패턴을 스캔해 필요한 전문가
  (Security / Performance / Data Steward / Business / Frontend)만 자동 투입한다.
  "boston review", "경량 코드 리뷰", "토큰 아끼는 리뷰", "빠른 PR 리뷰" 등
  종합 품질 점검을 저비용으로 수행하려는 요청에 사용한다. Chicago가 필요한 깊이의
  다관점 리뷰가 요구될 때는 chicago-code-review를 대신 사용한다.
---

# Boston Code Review Skill

CodeRabbit 한 명을 baseline으로 항상 실행하고, 오케스트레이터가 diff를 스캔해 조건이
충족된 전문가만 병렬 투입하는 경량 리뷰 스킬.

---

## 전체 흐름

```
코드 입력 (PR diff / 파일 / 스니펫)
        │
        ▼
  [오케스트레이터]
  - 컨텍스트 파악 (언어, 프레임워크, 도메인)
  - 자동 감지로 활성화할 전문가 선택
  - 공통 컨텍스트 패키징
        │
        ├──▶ 🐰 CodeRabbit (항상)       ─┐
        ├──▶ 🔒 Security    (조건부)    ─┤
        ├──▶ ⚡ Performance (조건부)    ─┼──▶ [오케스트레이터] 최종 검증 & 통합
        ├──▶ 📊 Data        (조건부)    ─┤            │
        ├──▶ 💰 Business    (조건부)    ─┤            ▼
        └──▶ 🌐 Frontend    (조건부)    ─┘      최종 리뷰 리포트
```

---

## Phase 0: 코드 입력 수집

### PR 기반 리뷰

사용자가 PR 번호, URL, 또는 현재 브랜치를 제공하는 경우:

```bash
gh pr view <PR_NUMBER> --repo <OWNER/REPO> --json number,title,body,baseRefName,headRefName
gh pr diff <PR_NUMBER> --repo <OWNER/REPO> > /tmp/boston-review-full-diff.txt
```

**대용량 diff 필터링:**

1. 락파일 / 자동 생성 파일 제외: `pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`, `*.generated.*`, `*.min.js`, `*.min.css`
2. 소스 디렉터리 우선: `src/`, `lib/`, `app/`, 설정, 마이그레이션
3. 필터 후 3000줄 초과 시 영향도 높은 파일 우선. 제외 파일은 리포트에 명시
4. 최종 diff를 `/tmp/boston-review-diff-focused.txt`에 저장

### 파일/스니펫 리뷰

PR이 아닌 파일 경로나 코드 스니펫이 제공된 경우:

- 파일 경로: `Read` 도구로 내용 수집
- 스니펫: 사용자 제공 코드를 그대로 사용
- `/tmp/boston-review-code-input.txt`에 저장

---
````

- [ ] **Step 3-2: Phase 1 (오케스트레이터 컨텍스트 분석 + 자동 감지 규칙) 작성**

위 파일 끝에 이어서 추가:

````markdown
## Phase 1: 오케스트레이터 — 컨텍스트 분석 + 자동 감지

### 1-1. 컨텍스트 분석

코드를 읽고 다음을 파악한다:

- **언어/프레임워크**: 예) Spring Boot/Kotlin, Next.js/TypeScript, Django/Python
- **도메인**: 예) ERP, 커머스, API 서버 (CLAUDE.md 또는 파일 경로 단서 활용)
- **변경 범위**: 신규 기능, 버그픽스, 리팩토링, 스키마 변경

### 1-2. 전문가 자동 감지 규칙

각 조건은 OR로 평가한다. 하나라도 충족하면 해당 전문가를 활성화한다. CodeRabbit은 항상 활성화.

| 전문가 | 활성화 조건 |
|---|---|
| 🔒 **Security** | 경로에 `auth`, `login`, `jwt`, `token`, `crypto`, `password`, `security`, `middleware`, `cors`, `csrf` 포함 / 키워드 `hash`, `encrypt`, `decrypt`, `sign`, `verify`, `secret`, `sanitize`, `escape`, `permission`, `role` 변경 / 환경변수·시크릿 파일 변경 |
| ⚡ **Performance** | 경로에 `repository`, `service`, `query`, `dao` 포함 AND 반복문/쿼리 변경 / 키워드 `SELECT`, `JOIN`, `forEach`, `for (`, `while (`, `N+1`, `cache`, `await` 다수 / 100줄 이상 단일 함수 변경 |
| 📊 **Data Steward** | 경로에 `migration`, `schema`, `prisma`, `sequelize`, `alembic`, `flyway`, `*.sql` 포함 / 키워드 `ALTER TABLE`, `DROP`, `CREATE INDEX`, `nullable`, `constraint`, `@Entity`, `@Column` 변경 |
| 💰 **Business Analyst** | 경로에 `domain/`, `usecase/`, `order`, `payment`, `billing`, `inventory`, `settlement`, `tax`, `discount`, `calculation` 포함 / 키워드 `BigDecimal`, `Money`, `Amount`, `calculate`, `compute`, 조건분기 3중 이상 |
| 🌐 **Frontend Expert** | 경로에 `controller`, `api/`, `routes/`, `*.dto.ts`, `*.schema.ts` 포함 AND 응답/요청 스펙 변경 / 키워드 `ResponseEntity`, `DTO`, `interface`, `type ` 변경 / OpenAPI·GraphQL 스키마 파일 변경 |

감지된 전문가 목록과 매칭 근거(어느 파일/키워드로 발화했는지)를 1줄로 기록한다.

### 1-3. 사용자 수동 override

사용자가 `"Security만으로 리뷰"`, `"Performance 빼고"` 등을 명시하면 자동 감지를 우회하고 지정 리스트만 실행한다. CodeRabbit도 해제 가능.

### 1-4. 공통 컨텍스트 패키징

모든 에이전트에 동일하게 전달:

- PR 제목/설명 (PR 리뷰인 경우)
- 필터링된 diff 또는 원본 코드
- 언어/프레임워크/도메인
- 감지된 전문가 목록과 매칭 근거

---
````

- [ ] **Step 3-3: Phase 2 서두 + CodeRabbit + Security 에이전트 프롬프트 작성**

이어서 추가:

````markdown
## Phase 2: 병렬 에이전트 실행

활성화된 에이전트를 하나의 메시지에서 동시에 호출한다(`Agent` 도구 병렬). 모든 에이전트는 research-only — 파일 수정 금지.

**선택적 파일 I/O:**

- 필터 후 diff ≤ 1500줄: 각 에이전트는 반환값으로만 결과 전달
- 필터 후 diff > 1500줄: 각 에이전트가 `/tmp/boston-review-{agent}-findings.txt`에 결과 기록. 오케스트레이터가 Phase 3 시작 시 모두 읽음

각 에이전트는 자신 담당 관점에만 집중하고 다른 관점(보안, 성능 등) 코멘트는 금지.

### 2-1. CodeRabbit (baseline, 항상 실행)

```
Agent(
  name: "coderabbit",
  subagent_type: "coderabbit:code-reviewer",
  prompt: "CodeRabbit AI 리뷰를 수행하세요. Research-only — 파일을 수정하지 마세요.

  **SCOPE**: PR diff에서 추가(+) 또는 수정된 라인에만 집중하세요.

  **분석 대상**: /tmp/boston-review-diff-focused.txt (또는 code-input.txt)

  **프로젝트 컨텍스트**: [언어/프레임워크, 도메인, 변경 목적]

  **관점**:
  - 버그·로직 오류 탐지
  - 보안 취약점 패턴
  - 코드 품질·모범 사례 위반
  - 잠재적 런타임 에러
  - 의존성·호환성 문제

  **출력 형식**:
  - 각 이슈는 심각도 🔴🟠🟡🟢 + 파일:라인 + 설명 + 개선안
  - 이슈가 없으면 '✅ 이상 없음'
  - 총 이슈 수 요약
  [large diff 모드면: /tmp/boston-review-coderabbit-findings.txt에 저장]"
)
```

### 2-2. Security (조건부)

```
Agent(
  name: "security",
  model: "sonnet",
  prompt: "당신은 The Guardian입니다. Research-only — 파일을 수정하지 마세요.

  페르소나: OWASP Top 10 전문, 모든 입력값을 잠재적 공격으로 간주.

  [공통 컨텍스트]

  **SCOPE**: PR diff의 추가/수정 라인에만 집중.

  **관점**:
  - OWASP Top 10 (Injection, Broken Auth, XSS, IDOR 등)
  - 인증/인가 누락 또는 우회
  - 민감 정보 노출 (PII 로그, 하드코딩 시크릿)
  - 입력값 검증, 신뢰 경계
  - JWT/세션 검증, 암호화 알고리즘 적절성

  **출력**:
  - 심각도 🔴🟠🟡🟢 + 위치 + 공격 시나리오 1줄 + 수정 가이드
  - 잘된 점 1~2개
  [large diff 모드면: /tmp/boston-review-security-findings.txt에 저장]"
)
```

---
````

- [ ] **Step 3-4: 파일 존재 확인**

Run:
```bash
wc -l plugins/boston-code-review/skills/boston-code-review/SKILL.md
grep -c "^## Phase" plugins/boston-code-review/skills/boston-code-review/SKILL.md
```

Expected: 줄 수 약 130~170줄, Phase 헤더 3개(Phase 0, 1, 2).

- [ ] **Step 3-5: 커밋**

```bash
git add plugins/boston-code-review/skills/boston-code-review/SKILL.md
git commit -m "feat(boston): add SKILL.md Phase 0-2 (input, context, coderabbit, security agents)"
```

---

## Task 4: SKILL.md 본문 작성 (Phase 2 나머지 + Phase 3)

**Files:**
- Modify: `plugins/boston-code-review/skills/boston-code-review/SKILL.md`

- [ ] **Step 4-1: Performance / Data Steward / Business / Frontend 에이전트 프롬프트 추가**

SKILL.md 끝(Task 3 마지막 `---` 뒤)에 이어서 추가:

````markdown
### 2-3. Performance (조건부)

```
Agent(
  name: "performance",
  model: "sonnet",
  prompt: "당신은 The Optimizer입니다. Research-only — 파일을 수정하지 마세요.

  페르소나: DB 튜닝과 프로파일링에 집착하는 성능 전문가.

  [공통 컨텍스트]

  **SCOPE**: PR diff의 추가/수정 라인에만 집중.

  **관점**:
  - N+1 쿼리 (ORM lazy loading, 루프 내 쿼리)
  - 중복 DB 조회, 캐시 미적용
  - 반복문 내 무거운 연산 (I/O, 암호화)
  - 인덱스 미활용 쿼리, 메모리 누수
  - 비동기 처리 필요 구간

  **출력**:
  - 병목 지점 + Big-O 분석 (가능 시)
  - 개선 전/후 코드, 예상 효과
  - 잘된 점 1~2개
  [large diff 모드면: /tmp/boston-review-performance-findings.txt에 저장]"
)
```

### 2-4. Data Steward (조건부)

```
Agent(
  name: "data-steward",
  model: "sonnet",
  prompt: "당신은 Data Steward입니다. Research-only — 파일을 수정하지 마세요.

  페르소나: 데이터 무결성과 마이그레이션 안전성에 집착.

  [공통 컨텍스트]

  **SCOPE**: PR diff의 추가/수정 라인에만 집중.

  **관점**:
  - 스키마 변경 영향도
  - 인덱스 설계 적절성
  - 데이터 유실 위험 (롤백 가능성)
  - 정합성 보장

  **출력**:
  - 위험 항목 + 롤백 방안
  - 잘된 점 1~2개
  [large diff 모드면: /tmp/boston-review-data-findings.txt에 저장]"
)
```

### 2-5. Business Analyst (조건부)

```
Agent(
  name: "business-analyst",
  model: "sonnet",
  prompt: "당신은 The Translator입니다. Research-only — 파일을 수정하지 마세요.

  페르소나: 요구사항과 구현 사이의 간극을 찾아내는 통역사.

  [공통 컨텍스트]

  **SCOPE**: PR diff의 추가/수정 라인에만 집중.

  **관점**:
  - 요구사항·구현 일치
  - 비즈니스 규칙 정확성 (계산식·상태 전이·예외)
  - 도메인 용어의 코드 반영 (Ubiquitous Language)

  **출력**:
  - 불일치 지점 + 근거 + 보정 제안
  - 잘된 점 1~2개
  [large diff 모드면: /tmp/boston-review-business-findings.txt에 저장]"
)
```

### 2-6. Frontend Expert (조건부)

```
Agent(
  name: "frontend-expert",
  model: "sonnet",
  prompt: "당신은 The UX Guardian입니다. Research-only — 파일을 수정하지 마세요.

  페르소나: API 계약과 프론트 연동의 사각지대를 찾아내는 전문가.

  [공통 컨텍스트]

  **SCOPE**: PR diff의 추가/수정 라인에만 집중.

  **관점**:
  - API 응답 스펙 변경의 프론트 영향
  - TypeScript 타입 정의 일치
  - 에러 메시지 UX, 로딩/에러 상태 처리

  **출력**:
  - 영향 항목 + 연동 리스크 + 방어 코드 제안
  - 잘된 점 1~2개
  [large diff 모드면: /tmp/boston-review-frontend-findings.txt에 저장]"
)
```

---
````

- [ ] **Step 4-2: Phase 3 (최종 검증 & 통합) 추가**

이어서:

````markdown
## Phase 3: 오케스트레이터 최종 검증 & 통합

서브에이전트 결과를 그대로 전달하지 않는다. 반드시 전체를 종합 검토·필터링한 뒤 최종 리포트를 작성한다.

### 3-1. 중복 제거

동일 파일·라인에 대해 여러 에이전트가 유사한 이슈를 보고한 경우, 가장 상세한 것을 대표로 선택하고 다른 에이전트의 관점을 병기한다.

### 3-2. 오탐 필터링

프레임워크가 이미 처리하는 사안, 프로젝트 컨텍스트와 맞지 않는 지적은 제거한다.

### 3-3. 심각도 재조정

프로젝트 특성에 맞게 조정. 조정 시 원 심각도와 사유를 병기한다.

### 3-4. 충돌 분석

에이전트 간 상반된 의견을 트레이드오프 섹션에 정리한다.

예: "Performance는 인덱스 추가 권고, Data Steward는 삽입 비용 증가 우려
→ 현재 읽기:쓰기 비율 10:1이므로 인덱스 추가가 더 유리"

### 3-5. 실행 가능성 및 컨벤션 교차 확인

- 제안된 수정이 실제로 적용 가능한지, 다른 코드와 충돌하지 않는지 확인
- 프로젝트의 기존 패턴·컨벤션과 일치하는지 확인

### 3-6. 심각도 집계

- 🔴 CRITICAL: N건
- 🟠 HIGH: N건
- 🟡 MEDIUM: N건
- 🟢 LOW: N건

---
````

- [ ] **Step 4-3: 줄 수 및 Phase 헤더 확인**

Run:
```bash
wc -l plugins/boston-code-review/skills/boston-code-review/SKILL.md
grep -c "^## Phase" plugins/boston-code-review/skills/boston-code-review/SKILL.md
```

Expected: 줄 수 약 230~270, Phase 헤더 4개(Phase 0, 1, 2, 3).

- [ ] **Step 4-4: 커밋**

```bash
git add plugins/boston-code-review/skills/boston-code-review/SKILL.md
git commit -m "feat(boston): add SKILL.md Phase 2 specialists and Phase 3 orchestrator validation"
```

---

## Task 5: SKILL.md 본문 작성 (Phase 4~6 + 리포트 포맷 + 엣지 케이스)

**Files:**
- Modify: `plugins/boston-code-review/skills/boston-code-review/SKILL.md`

- [ ] **Step 5-1: Phase 4~6 + 심각도 레이블 + 리포트 포맷 + 엣지 케이스 추가**

SKILL.md 끝에 이어서 추가:

````markdown
## Phase 4: 사용자 확인 (PR 리뷰)

PR 코멘트로 게시하기 전 사용자에게 확인 요청:

```
리뷰 결과가 준비되었습니다. 아래 내용을 PR 코멘트로 게시할까요?

[리포트 표시]

- 'yes' 또는 '게시': PR 코멘트 게시
- 'edit' 또는 '수정': 수정 부분 지시
- 'no' 또는 '취소': 게시 취소
```

- yes/게시 → Phase 5 진행
- edit/수정 → 수정 후 다시 확인
- no/취소 → Phase 5 건너뛰고 Phase 6(정리)로

파일/스니펫 리뷰는 Phase 4~5를 건너뛰고 리포트를 직접 출력한다.

---

## Phase 5: PR 코멘트 게시

```bash
gh pr comment <PR_NUMBER> --repo <OWNER/REPO> --body "$(cat <<'COMMENT_EOF'
<review content>
COMMENT_EOF
)"
```

---

## Phase 6: 임시 파일 정리

```bash
rm -f /tmp/boston-review-*.txt
```

---

## 공통 심각도 레이블

| 레이블 | 의미 | 처리 기준 |
|---|---|---|
| 🔴 CRITICAL | 보안 취약점, 데이터 유실 | 배포 블로킹, 즉시 수정 |
| 🟠 HIGH | 명백한 버그, 심각한 성능 문제 | 이번 PR에서 수정 권장 |
| 🟡 MEDIUM | 설계 개선, 테스트 추가 필요 | 다음 스프린트 |
| 🟢 LOW | 스타일, 가독성, 제안 | 선택적 반영 |

---

## 최종 리포트 포맷

```markdown
# 🔍 Boston Code Review

## 📋 요약
- **대상**: PR #{n} {title}  (또는 파일명)
- **참여 에이전트**: CodeRabbit [+ Security + Performance ...]
- **자동 감지 근거**: {매칭된 파일/키워드 1줄}
- **전체 평가**: {오케스트레이터 1줄 총평}
- **머지 판단**: ✅ 승인 / ⚠️ 조건부 승인 / ❌ 수정 필요
- 🔴 N / 🟠 N / 🟡 N / 🟢 N

## 🚨 즉시 수정 필요 (CRITICAL / HIGH)

### 1. [이슈 제목]  `🔴 CRITICAL`  — [출처 에이전트]
- **위치**: `파일:라인`
- **설명**: ...
- **수정 방안**: ...

(해당 이슈 없으면: "✅ 즉시 수정 항목 없음 — 코드가 양호합니다.")

## 🟡 개선 제안 (MEDIUM / LOW)

중복 제거된 이슈를 한 줄씩 flat 리스트로 나열:
- `파일:라인` — 제목 `🟡` [에이전트] — 1줄 설명
- `파일:라인` — 제목 `🟢` [에이전트] — 1줄 설명

## 🎯 오케스트레이터 노트

- **검증 요약**: 오탐 N건 필터링, 심각도 조정 N건
- **트레이드오프**: {충돌 있을 때만 서술, 없으면 생략}
- **잘된 점**: {1~3개, 간결하게}
- **액션**: [즉시] ... / [이번 PR] ... / [다음 스프린트] ...

---
🤖 Generated by Boston Code Review (CodeRabbit baseline + auto-detected specialists)
```

---

## 엣지 케이스

- **PR을 찾을 수 없음**: 사용자에게 PR 지정 요청 또는 현재 브랜치 대비 main/master diff 리뷰 제안
- **특정 에이전트 실패**: 나머지 결과로 진행. 리포트에 실패 에이전트 명시
- **매우 큰 diff (필터 후 3000줄 초과)**: 영향도 높은 파일 우선. 제외 파일을 리포트에 명시
- **이슈 없음**: 긍정 리뷰 게시, 잘된 점 강조
- **사용자가 게시 거부**: 임시 파일 정리 후 종료
- **자동 감지 결과 0개**: CodeRabbit 단독 실행 (경량 리뷰 기본 케이스)
- **자동 감지 결과 5개 전부**: CodeRabbit + 전문가 5명 병렬 실행 (복잡 PR)
- **사용자가 특정 역할만 지정**: 자동 감지 우회, 지정 에이전트만 실행
````

- [ ] **Step 5-2: 전체 줄 수 및 섹션 구조 확인**

Run:
```bash
wc -l plugins/boston-code-review/skills/boston-code-review/SKILL.md
grep "^## " plugins/boston-code-review/skills/boston-code-review/SKILL.md
```

Expected:
- 줄 수: 250~300줄 (스펙 목표치 ≤300 충족)
- 섹션: `전체 흐름`, `Phase 0`, `Phase 1`, `Phase 2`, `Phase 3`, `Phase 4`, `Phase 5`, `Phase 6`, `공통 심각도 레이블`, `최종 리포트 포맷`, `엣지 케이스`

- [ ] **Step 5-3: 스펙 커버리지 확인**

Run:
```bash
# 자동 감지 규칙 5개 전문가 모두 존재
grep -E "^\| (🔒|⚡|📊|💰|🌐)" plugins/boston-code-review/skills/boston-code-review/SKILL.md | wc -l
# Agent 블록 6개 (CodeRabbit + 5 specialists)
grep -c "^Agent(" plugins/boston-code-review/skills/boston-code-review/SKILL.md
```

Expected: 전문가 표 5줄, Agent 블록 6개.

- [ ] **Step 5-4: frontmatter 유효성 검증**

Run:
```bash
python -c "
import sys
content = open('plugins/boston-code-review/skills/boston-code-review/SKILL.md').read()
assert content.startswith('---\n'), 'frontmatter missing'
end = content.find('\n---\n', 4)
assert end > 0, 'frontmatter not closed'
fm = content[4:end]
assert 'name: boston-code-review' in fm
assert 'description:' in fm
print('frontmatter OK')
"
```

Expected: `frontmatter OK`.

- [ ] **Step 5-5: 커밋**

```bash
git add plugins/boston-code-review/skills/boston-code-review/SKILL.md
git commit -m "feat(boston): complete SKILL.md with Phase 4-6, report format, and edge cases"
```

---

## Task 6: README.md 작성

**Files:**
- Create: `plugins/boston-code-review/README.md`

- [ ] **Step 6-1: README.md 작성**

파일: `plugins/boston-code-review/README.md`

```markdown
# Boston Code Review

CodeRabbit을 baseline으로 항상 실행하고, diff 패턴 스캔으로 필요한 전문가만 자동 투입하는 경량 멀티 에이전트 코드리뷰 플러그인.

`chicago-code-review`의 토큰 효율형 대안으로 설계되었으며, 소형 PR에서 Chicago 대비 약 80% 토큰 절감을 목표로 한다.

## Features

- **Baseline 에이전트 (항상 실행)**: 🐰 CodeRabbit — 버그·보안 패턴·품질·모범 사례를 단일 에이전트로 포괄
- **자동 감지 전문가 풀 (조건부 실행, 최대 5명)**:
  - 🔒 **Security**: 인증/인가·시크릿·입력 검증 관련 변경 시
  - ⚡ **Performance**: 쿼리·반복문·대형 함수 변경 시
  - 📊 **Data Steward**: 스키마·마이그레이션 변경 시
  - 💰 **Business Analyst**: 도메인 로직·계산식 변경 시
  - 🌐 **Frontend Expert**: API 계약·DTO·타입 변경 시
- **오케스트레이터 최종 검증**: 중복 제거, 오탐 필터링, 심각도 재조정, 충돌 분석
- **선택적 파일 I/O**: 1500줄 초과 diff에서만 중간 파일 사용
- **사용자 확인**: PR 코멘트 게시 전 승인 요청

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- [GitHub CLI](https://cli.github.com/) (`gh`) authenticated (for PR reviews)
- CodeRabbit subagent (`coderabbit:code-reviewer`) 사용 가능한 환경
- Access to Claude Sonnet 4.6 model

## Installation

Claude Code 세션 내에서 다음 슬래시 명령을 실행하세요:

```text
/plugin marketplace add https://github.com/nalpari/team-code-review-plugin.git
/plugin install boston-code-review@nalpari-plugins
```

> 참고: `nalpari-plugins`는 이 저장소의 `.claude-plugin/marketplace.json`에 정의된 marketplace 이름입니다.

## Usage

```text
# PR 리뷰
/boston-code-review #42
/boston-code-review owner/repo#42

# 파일/스니펫 리뷰
/boston-code-review (이후 파일 경로나 코드 제공)

# 특정 에이전트만
"Security와 Performance 관점으로만 리뷰해줘"
```

## Workflow

```text
Phase 0: 코드 입력 수집 (PR diff / 파일 / 스니펫)
    ↓
Phase 1: 오케스트레이터 컨텍스트 분석 + 자동 감지
    ↓
Phase 2: 병렬 에이전트 실행 (CodeRabbit + 감지된 전문가 0~5명)
    ↓
Phase 3: 오케스트레이터 최종 검증 (중복/오탐/심각도/충돌)
    ↓
Phase 4: 사용자 확인 (PR 리뷰)
    ↓
Phase 5: PR 코멘트 게시
    ↓
Phase 6: 임시 파일 정리
```

## Differences from other review plugins

| Feature | team-code-review | montreal-code-review | chicago-code-review | boston-code-review |
|---|---|---|---|---|
| Baseline 에이전트 | 2 (Opus + Sonnet) | 4 (Opus + Sonnet x3) | 5 core | 1 (CodeRabbit) |
| 조건부 전문가 | 없음 | 없음 | 4 옵션 | 5 자동 감지 |
| 토큰 효율 | 중 | 중 | 낮음 | 높음 |
| 입력 타입 | PR only | PR only | PR / 파일 / 스니펫 | PR / 파일 / 스니펫 |
| 사용자 확인 | 자동 게시 | 확인 후 게시 | 확인 후 게시 | 확인 후 게시 |
| 최적 용도 | 빠른 2-모델 교차검증 | 적대적 관점 포함 종합 | 깊은 다관점 리뷰 | 저비용 기본 리뷰 |

## License

MIT
```

- [ ] **Step 6-2: 파일 유효성 확인**

Run:
```bash
wc -l plugins/boston-code-review/README.md
grep -c "^## " plugins/boston-code-review/README.md
```

Expected: 줄 수 약 60~90, 섹션 헤더 6~7개 (Features, Prerequisites, Installation, Usage, Workflow, Differences, License).

- [ ] **Step 6-3: 커밋**

```bash
git add plugins/boston-code-review/README.md
git commit -m "docs(boston): add README with installation, usage, and plugin comparison"
```

---

## Task 7: 통합 검증 (로컬 구조 점검)

**Files:**
- None (읽기 전용 검증)

- [ ] **Step 7-1: 플러그인 디렉터리 트리 최종 확인**

Run:
```bash
find plugins/boston-code-review -type f | sort
```

Expected 출력:
```
plugins/boston-code-review/.claude-plugin/plugin.json
plugins/boston-code-review/README.md
plugins/boston-code-review/skills/boston-code-review/SKILL.md
```

세 파일 모두 존재해야 한다. 다른 파일이 섞여 있으면 정리.

- [ ] **Step 7-2: JSON 파일 전수 유효성 검증**

Run:
```bash
python -c "
import json
for path in ['.claude-plugin/marketplace.json', 'plugins/boston-code-review/.claude-plugin/plugin.json']:
    json.load(open(path))
    print('OK:', path)
"
```

Expected: 두 파일 모두 `OK:` 출력.

- [ ] **Step 7-3: 기존 플러그인 구조와 비교**

Run:
```bash
for p in chicago-code-review montreal-code-review team-code-review boston-code-review; do
  echo "=== $p ==="
  find plugins/$p -type f | sort
done
```

Expected: 네 플러그인 모두 동일한 3-파일 구조 (`plugin.json`, `README.md`, `skills/<name>/SKILL.md`).

- [ ] **Step 7-4: SKILL.md frontmatter 일관성 확인**

Run:
```bash
for p in chicago-code-review montreal-code-review team-code-review boston-code-review; do
  echo "=== $p ==="
  head -3 plugins/$p/skills/$p/SKILL.md
done
```

Expected: 모두 `---\nname: <plugin-name>\n...` 형식으로 시작.

- [ ] **Step 7-5: 빈 커밋 체크 및 전체 diff 확인**

Run:
```bash
git log --oneline main..HEAD
git diff main..HEAD --stat
```

Expected: Task 1~6에서 만든 6개 커밋 표시, 변경 파일은 4개(`.claude-plugin/marketplace.json`, `plugins/boston-code-review/.claude-plugin/plugin.json`, `plugins/boston-code-review/README.md`, `plugins/boston-code-review/skills/boston-code-review/SKILL.md`).

---

## Task 8: 실제 플러그인 로드 smoke test (수동)

**Files:**
- None (수동 검증)

이 Task는 **수동 실행**이다. 로컬에서 Claude Code 세션을 열고 플러그인이 실제로 인식되는지 확인한다.

- [ ] **Step 8-1: 현재 브랜치를 origin에 푸시**

```bash
git push -u origin HEAD
```

Expected: 원격에 브랜치 생성.

- [ ] **Step 8-2: 새 Claude Code 세션에서 플러그인 마켓플레이스 재로드**

별도 터미널에서 Claude Code 세션 시작 후:

```text
/plugin marketplace add https://github.com/nalpari/team-code-review-plugin.git
```

이미 추가되어 있으면:

```text
/plugin marketplace update nalpari-plugins
```

Expected: `boston-code-review` 플러그인이 목록에 노출됨.

- [ ] **Step 8-3: 플러그인 설치 및 활성화 확인**

```text
/plugin install boston-code-review@nalpari-plugins
```

Expected: 설치 성공 메시지. 슬래시 자동 완성에서 `/boston-code-review` 등장.

- [ ] **Step 8-4: 스킬 트리거 테스트 (작은 PR 또는 샘플 파일)**

Claude Code 세션에서:

```text
boston review를 해줘. 대상: <작은 테스트 파일 경로>
```

Expected 동작:
- 오케스트레이터가 Phase 0 입력 수집
- Phase 1에서 전문가 자동 감지 결과 및 근거 출력 (예: `CodeRabbit만 활성화. 자동 감지: 없음`)
- Phase 2에서 CodeRabbit 단독 또는 CodeRabbit + 감지된 전문가 병렬 실행
- Phase 3 리포트가 요약/즉시 수정/개선 제안/오케스트레이터 노트 구조로 출력

실패 시 Task 3~5의 SKILL.md 내용을 재점검하고 수정 후 재커밋.

- [ ] **Step 8-5: 검증 결과 스펙 성공 기준과 비교**

스펙 `성공 기준` 섹션의 항목을 점검:

1. ✅ 소형 PR 자동 감지 0~1개로 CodeRabbit 단독 실행 가능한가?
2. ✅ 복잡 PR에서 전문가 3개 이상 자동 활성화되는가?
3. ✅ `wc -l SKILL.md` 결과가 300 이하인가?
4. ⚠️ 팀 직관 80% 일치 — 실사용 후 판정 (이번 Task에서는 체크 불가, 별도 피드백 세션 필요)
5. ✅ 기존 Chicago / Montreal / team-code-review 플러그인이 동시에 동작하는가?

1, 2, 3, 5가 통과하면 구현 종료. 4는 실운영 단계에서 피드백 수집.

---

## Task 9: Pull Request 생성

**Files:**
- None

- [ ] **Step 9-1: PR 제목과 본문 준비**

제목: `feat: add boston-code-review lightweight plugin`

본문 (아래 HEREDOC 사용):

```bash
gh pr create --title "feat: add boston-code-review lightweight plugin" --body "$(cat <<'EOF'
## Summary

- CodeRabbit을 baseline으로 항상 실행하고, 5개 전문가(Security/Performance/Data Steward/Business/Frontend)를 diff 패턴으로 자동 감지·투입하는 경량 코드리뷰 플러그인 추가
- 기존 `chicago-code-review`의 토큰 과소비 이슈에 대한 저비용 대안
- 소형 PR 기준 ~80% 토큰 절감 목표

## Changes

- `plugins/boston-code-review/` 신설 (plugin.json, README.md, SKILL.md)
- `.claude-plugin/marketplace.json`에 boston-code-review 엔트리 등록

## Design Document

See [docs/superpowers/specs/2026-04-20-boston-code-review-design.md](docs/superpowers/specs/2026-04-20-boston-code-review-design.md)

## Test plan

- [x] JSON 매니페스트 유효성 검증
- [x] SKILL.md 300줄 이하 확인
- [x] 4개 플러그인 구조 일관성 확인
- [ ] Claude Code 세션에서 `/plugin install boston-code-review@nalpari-plugins` 성공 확인
- [ ] 샘플 파일 대상 스킬 호출하여 Phase 1 자동 감지 동작 확인
- [ ] 샘플 PR 대상 전체 Phase 0~6 흐름 동작 확인

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 9-2: PR URL 확인**

Run:
```bash
gh pr view --json url -q .url
```

Expected: 새 PR URL 출력. 이 URL을 사용자에게 보고.

---

## 완료 조건

- [ ] Task 1~9 모든 Step 체크박스 완료
- [ ] `wc -l plugins/boston-code-review/skills/boston-code-review/SKILL.md` ≤ 300
- [ ] `python -c "import json; json.load(open('.claude-plugin/marketplace.json'))"` 오류 없음
- [ ] 로컬 Claude Code 세션에서 `/boston-code-review` 스킬 트리거 확인
- [ ] PR 생성 및 URL 보고
