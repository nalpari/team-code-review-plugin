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
| ⚡ **Performance** | (경로에 `repository`, `service`, `query`, `dao` 포함 AND 반복문/쿼리 변경) / 키워드 `SELECT`, `JOIN`, `forEach`, `for (`, `while (`, `N+1`, `cache` 변경 / 반복문 내부 `await` 또는 `Promise.all(`, `for await` 패턴 / 100줄 이상 단일 함수 변경 |
| 📊 **Data Steward** | 경로에 `migration`, `schema`, `prisma`, `sequelize`, `alembic`, `flyway`, `*.sql` 포함 / 키워드 `ALTER TABLE`, `DROP`, `CREATE INDEX`, `nullable`, `constraint`, `@Entity`, `@Column` 변경 |
| 💰 **Business Analyst** | 경로에 `domain/`, `usecase/`, `order`, `payment`, `billing`, `inventory`, `settlement`, `tax`, `discount`, `calculation` 포함 / 키워드 `BigDecimal`, `Money`, `Amount`, `calculate`, `compute`, 조건분기 3중 이상 |
| 🌐 **Frontend Expert** (API 계약 관점) | (경로에 `controller`, `api/`, `routes/`, `*.dto.ts`, `*.schema.ts` 포함 AND 응답/요청 스펙 변경) / 키워드 `ResponseEntity`, `DTO`, `interface`, `type ` 변경 / OpenAPI·GraphQL 스키마 파일 변경 |

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

## Phase 2: 병렬 에이전트 실행

활성화된 에이전트를 하나의 메시지에서 동시에 호출한다(`Agent` 도구 병렬). 모든 에이전트는 research-only — 파일 수정 금지.

**공통 컨텍스트 치환**: 각 전문가 에이전트 프롬프트의 `[공통 컨텍스트]` 자리에는 Phase 1-4에서 패키징한 4개 항목(PR 제목·설명, 필터링된 diff, 언어/프레임워크/도메인, 감지된 전문가 목록과 매칭 근거)을 그대로 삽입한다.

**선택적 파일 I/O:**

- 필터 후 diff ≤ 1500줄: 각 에이전트는 반환값으로만 결과 전달. 프롬프트에 파일 저장 지시를 포함하지 않는다.
- 필터 후 diff > 1500줄: 각 에이전트 프롬프트 말미에 `결과를 /tmp/boston-review-<agent-id>-findings.txt에 저장하세요.` 문장을 추가한다. `<agent-id>`는 `coderabbit`, `security`, `performance`, `data`, `business`, `frontend` 중 하나. 오케스트레이터가 Phase 3 시작 시 모두 읽음.

**공통 템플릿**: 5개 전문가 에이전트는 2-2 Security와 동일한 프롬프트 구조를 따른다. CodeRabbit은 `subagent_type: "coderabbit:code-reviewer"`를 사용하므로 구조가 다르다.

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
  - 총 이슈 수 요약"
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
  - 잘된 점 1~2개"
)
```

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
  - 잘된 점 1~2개"
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
  - 잘된 점 1~2개"
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
  - 잘된 점 1~2개"
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
  - 잘된 점 1~2개"
)
```

---

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

## Phase 4: 사용자 확인 (PR 리뷰)

리포트를 먼저 **파일로 저장**한 뒤 게시 여부를 묻는다. 셸 이스케이프(백틱·따옴표 이중 escape) 로 인한 렌더 깨짐을 방지하기 위함.

1. `Write` 도구로 리포트를 `/tmp/boston-review-report.md`에 저장한다.
   - 마크다운 원문을 **있는 그대로** 기록한다. 백틱(`` ` ``), 큰따옴표(`"`), 달러(`$`), 백슬래시(`\`)를 **절대 이스케이프하지 말 것**. `--body-file`은 파일 내용을 그대로 업로드하므로 셸 해석이 없다.
   - 코드 블록은 표준 펜스(``` ```ts ```)로 작성하고, 리스트 항목 내부에 둘 때는 펜스를 블록 바로 다음 줄부터 시작한다(펜스와 텍스트를 같은 줄에 붙이지 않는다).
2. 저장 후 사용자에게 확인 요청:

```
리뷰 결과를 /tmp/boston-review-report.md에 저장했습니다. PR 코멘트로 게시할까요?

[리포트 본문 표시]

- 'yes' 또는 '게시': PR 코멘트 게시
- 'edit' 또는 '수정': 수정 부분 지시
- 'no' 또는 '취소': 게시 취소
```

- yes/게시 → Phase 5 진행
- edit/수정 → 파일을 수정(`Edit` 도구)하고 다시 확인
- no/취소 → Phase 5 건너뛰고 Phase 6(정리)로

파일/스니펫 리뷰는 Phase 4~5를 건너뛰고 리포트를 직접 출력한다.

---

## Phase 5: PR 코멘트 게시

Phase 4에서 저장한 파일을 `--body-file`로 업로드한다. **HEREDOC이나 `--body "..."` 인라인 문자열을 사용하지 말 것** — 셸 따옴표 중첩 때문에 백틱·따옴표가 이스케이프되어 코드 블록이 ``` \`\`\`ts ``` 처럼 그대로 보이는 증상이 발생한다.

```bash
gh pr comment <PR_NUMBER> --repo <OWNER/REPO> --body-file /tmp/boston-review-report.md
```

---

## Phase 6: 임시 파일 정리

```bash
rm -f /tmp/boston-review-*.txt /tmp/boston-review-report.md
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
- **CodeRabbit subagent 사용 불가 (`coderabbit:code-reviewer` 미설치)**: baseline 상실을 리포트 상단에 경고로 표기하고, 감지된 전문가만으로 진행. 전문가도 0개면 사용자에게 Chicago 또는 수동 리뷰 권장
- **매우 큰 diff (필터 후 3000줄 초과)**: 영향도 높은 파일 우선. 제외 파일을 리포트에 명시
- **이슈 없음**: 긍정 리뷰 게시, 잘된 점 강조
- **사용자가 게시 거부**: 임시 파일 정리 후 종료
- **자동 감지 결과 0개**: CodeRabbit 단독 실행 (경량 리뷰 기본 케이스)
- **자동 감지 결과 5개 전부**: CodeRabbit + 전문가 5명 병렬 실행 (복잡 PR)
- **사용자가 특정 역할만 지정**: 자동 감지 우회, 지정 에이전트만 실행
