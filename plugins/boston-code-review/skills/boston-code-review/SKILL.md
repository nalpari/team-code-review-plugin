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

- 필터 후 diff ≤ 1500줄: 각 에이전트는 반환값으로만 결과 전달
- 필터 후 diff > 1500줄: 각 에이전트가 `/tmp/boston-review-{agent}-findings.txt`에 결과 기록. 오케스트레이터가 Phase 3 시작 시 모두 읽음

**전문가 에이전트 공통 템플릿**: Security·Performance·Data Steward·Business·Frontend 5개 전문가 에이전트는 모두 동일한 프롬프트 구조(`페르소나 → [공통 컨텍스트] → **SCOPE** → **관점** → **출력**`)를 따른다. CodeRabbit은 별도 subagent_type을 사용하므로 구조가 다르다.

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
