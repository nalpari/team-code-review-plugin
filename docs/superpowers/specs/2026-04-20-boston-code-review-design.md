# Boston Code Review — 경량 멀티 에이전트 코드리뷰 스킬 설계

**날짜**: 2026-04-20
**작성**: brainstorming 세션 결과물
**상태**: 설계 승인 대기

---

## 배경

기존 `chicago-code-review` 플러그인은 5개의 필수 에이전트(Security / Performance / QA / Craftsman / CodeRabbit)와 최대 4개의 선택 에이전트를 병렬 실행하여 다관점 리뷰를 수행한다. 이 구조는 리뷰 품질은 우수하지만, 다음과 같은 토큰 중복이 팀 내 피드백으로 제기되었다:

1. 모든 에이전트가 동일한 diff를 개별적으로 읽어 컨텍스트 중복 발생
2. CodeRabbit이 이미 보안·버그·품질 영역을 포괄함에도 Security / QA / Craftsman 에이전트가 상시 실행되어 관점 중복
3. 각 에이전트가 `/tmp/*-findings.txt`에 결과를 기록하고 오케스트레이터가 재차 읽는 파일 I/O 오버헤드
4. `SKILL.md` 자체가 647줄로 스킬 로드 시점의 컨텍스트 소비가 큼
5. 단순 PR에도 풀 세트 5 에이전트가 항상 실행되어, PR 성격별 토큰 적응이 없음

팀내 사용 빈도가 높은 만큼 토큰 소비 최적화 효과가 크다는 판단하에, Chicago를 대체하지 않고 **독립된 경량 플러그인 `boston-code-review`**를 신설한다.

---

## 목표

1. **기본 케이스 토큰 5배 이상 절감** — 소형 PR에서 Chicago 대비 10~20% 토큰 수준을 목표로 한다.
2. **정확도 유지** — 위험 신호가 있는 PR(인증/마이그레이션/도메인 로직 등)에서는 전문가 에이전트를 자동 투입하여 Chicago에 준하는 커버리지를 확보한다.
3. **사용자 개입 최소화** — 활성화할 에이전트를 사용자가 직접 고르지 않아도 오케스트레이터가 diff 패턴으로 자동 결정한다.
4. **Chicago 보존** — Chicago는 그대로 유지하여 종합 리뷰가 필요한 경우 선택 가능하게 한다.

---

## 비목표

- Chicago의 9개 역할을 그대로 옮기지 않는다 (QA / Craftsman / Junior Developer는 CodeRabbit이 커버한다고 판단)
- UI 생성이나 프론트엔드 전용 심화 리뷰는 범위에서 제외
- 리뷰 품질 평가 지표의 정량적 측정 자동화는 범위에서 제외 (추후 별도 과제)

---

## 아키텍처 개요

**핵심 구조**: CodeRabbit 단일 에이전트를 baseline으로 항상 실행하고, 오케스트레이터가 diff를 스캔하여 5개 전문가 풀 중 조건 충족분만 자동 병렬 투입한다.

```
코드 입력 (PR diff / 파일 / 스니펫)
        │
        ▼
  [오케스트레이터]
  - 컨텍스트 파악 (언어, 프레임워크, 도메인)
  - 파일 패턴 + 키워드 스캔으로 전문가 자동 감지
  - 공통 컨텍스트 패키징
        │
        ├──▶ 🐰 CodeRabbit (항상 실행, baseline)         ─┐
        ├──▶ 🔒 Security        (조건부, 자동 감지)       ─┤
        ├──▶ ⚡ Performance     (조건부, 자동 감지)       ─┼──▶ [오케스트레이터] 최종 검증 & 통합
        ├──▶ 📊 Data Steward    (조건부, 자동 감지)       ─┤                       │
        ├──▶ 💰 Business        (조건부, 자동 감지)       ─┤                       ▼
        └──▶ 🌐 Frontend        (조건부, 자동 감지)       ─┘                 최종 리뷰 리포트
```

**에이전트 구성**:
- **Baseline (1명)**: CodeRabbit — 보안·버그·품질·모범 사례 포괄
- **전문가 풀 (최대 5명)**: Security / Performance / Data Steward / Business Analyst / Frontend Expert
- **실행 범위**: CodeRabbit + 자동 감지된 전문가 0~5명 (캡 5)
- **실제 예상 실행 수**: 단순 PR 1명, 평균 PR 2~3명, 복잡 PR 4~6명

---

## 토큰 절감 전략

| 절감 포인트 | 구현 방식 |
|---|---|
| 기본 에이전트 수 축소 | 5 → 1 (CodeRabbit만 baseline) |
| 조건부 전문가 투입 | 파일 경로 + 키워드 기반 자동 감지, 불필요 실행 제거 |
| 파일 I/O 선택적 적용 | 필터 후 diff 1500줄 이하는 반환값만, 초과 시에만 파일 기록 |
| SKILL.md 압축 | 647줄 → 약 250~300줄 목표 (템플릿 단순화, 중복 설명 제거) |
| 리포트 구조 축약 | 에이전트별 섹션 5개 → 이슈 flat 리스트 + 오케스트레이터 노트 |

**토큰 예산 추정 (보수적 가정치)**:
- Chicago 현재: PR당 ~50~80k 토큰
- Boston 목표: 소형 PR ~10~15k, 복잡 PR ~30~40k

---

## Phase 흐름

### Phase 0: 코드 입력 수집

Chicago와 동일한 입력 처리 로직을 재사용한다.

- **PR 리뷰**: `gh pr view` + `gh pr diff` 로 메타데이터와 diff 수집
- **파일/스니펫 리뷰**: `Read` 도구 또는 사용자 제공 코드 그대로 사용
- **대형 diff 필터링**:
  1. 락파일 / 자동생성 파일 제외 (`pnpm-lock.yaml`, `*.min.js` 등)
  2. 소스 디렉터리 우선 (`src/`, `lib/`, `app/`, 마이그레이션)
  3. 필터 후 3000줄 초과 시 영향도 높은 파일 우선 선택, 제외 파일은 리포트에 명시
- 최종 diff 저장 경로: `/tmp/boston-review-diff-focused.txt`

### Phase 1: 오케스트레이터 — 컨텍스트 분석 + 자동 전문가 감지

#### 1-1. 컨텍스트 분석

- 언어 / 프레임워크 식별
- 도메인 추정 (CLAUDE.md 또는 파일 경로 단서 활용)
- 변경 범위 분류 (신규 기능 / 버그픽스 / 리팩토링 / 스키마 변경)

#### 1-2. 전문가 자동 감지 규칙

각 조건은 OR로 평가한다. 하나라도 충족하면 해당 전문가를 활성화.

| 전문가 | 활성화 조건 (OR) |
|---|---|
| 🔒 **Security** | 경로: `auth`, `login`, `jwt`, `token`, `crypto`, `password`, `security`, `middleware`, `cors`, `csrf` 포함 / 키워드: `hash`, `encrypt`, `decrypt`, `sign`, `verify`, `secret`, `sanitize`, `escape`, `permission`, `role` 변경 / 환경변수·시크릿 파일 변경 |
| ⚡ **Performance** | 경로: `repository`, `service`, `query`, `dao` 포함 AND 반복문/쿼리 변경 / 키워드: `SELECT`, `JOIN`, `forEach`, `for (`, `while (`, `N+1`, `cache`, `await` 다수 / 100줄 이상 단일 함수 변경 |
| 📊 **Data Steward** | 경로: `migration`, `schema`, `prisma`, `sequelize`, `alembic`, `flyway`, `*.sql` / 키워드: `ALTER TABLE`, `DROP`, `CREATE INDEX`, `nullable`, `constraint`, `@Entity`, `@Column` 변경 |
| 💰 **Business Analyst** | 경로: `domain/`, `usecase/`, `order`, `payment`, `billing`, `inventory`, `settlement`, `tax`, `discount`, `calculation` / 키워드: `BigDecimal`, `Money`, `Amount`, `calculate`, `compute`, 조건분기 3중 이상 |
| 🌐 **Frontend Expert** | 경로: `controller`, `api/`, `routes/`, `*.dto.ts`, `*.schema.ts` AND 응답/요청 스펙 변경 / 키워드: `ResponseEntity`, `DTO`, `interface`, `type ` 변경 / OpenAPI·GraphQL 스키마 파일 변경 |

#### 1-3. 사용자 수동 override

- `"Security만으로"`, `"Performance 빼고"` 등 명시적 요청 시 자동 감지 우회
- 사용자 지정 리스트 그대로 사용 (CodeRabbit도 해제 가능)

#### 1-4. 공통 컨텍스트 패키징

모든 에이전트에 전달할 정보:
- PR 제목 / 설명 (해당 시)
- 필터링된 diff 또는 원본 코드
- 언어 / 프레임워크 / 도메인
- 감지된 전문가 목록과 매칭 근거 (디버깅용)

### Phase 2: 병렬 에이전트 실행

- 활성화된 에이전트를 **하나의 메시지에서 동시 호출** (Agent 도구 병렬)
- 모든 에이전트는 research-only. 파일 수정 금지
- **선택적 파일 I/O**:
  - 필터 후 diff ≤ 1500줄: 각 에이전트는 반환값으로만 결과 전달
  - 필터 후 diff > 1500줄: 각 에이전트가 `/tmp/boston-review-{agent}-findings.txt`에 결과 기록. 오케스트레이터가 Phase 3 시작 시 모두 읽음
- 각 에이전트는 자신 담당 관점에만 집중하고 타 관점(보안, 성능 등) 코멘트는 금지

#### 2-1. CodeRabbit (baseline, 항상 실행)

```
Agent(
  name: "coderabbit",
  subagent_type: "coderabbit:code-reviewer",
  prompt: "CodeRabbit AI 리뷰를 수행하세요. Research-only.

  SCOPE: PR diff에서 추가(+) 또는 수정된 라인에만 집중.

  분석 대상: /tmp/boston-review-diff-focused.txt (또는 code-input.txt)
  프로젝트 컨텍스트: [언어/프레임워크, 도메인, 변경 목적]

  관점: 버그·로직 오류, 보안 취약점 패턴, 코드 품질, 런타임 에러, 의존성·호환성.

  출력: 심각도 🔴🟠🟡🟢 + 파일:라인 + 설명 + 개선안. 총 이슈 수 요약.
  [large diff 모드면: /tmp/boston-review-coderabbit-findings.txt에 저장]"
)
```

#### 2-2. Security (조건부)

```
Agent(
  name: "security",
  model: "sonnet",
  prompt: "당신은 The Guardian입니다. Research-only.

  페르소나: OWASP Top 10 전문, 입력값을 잠재적 공격으로 간주.

  [공통 컨텍스트]

  관점: OWASP Top 10, 인증/인가 누락·우회, 민감정보 노출, 입력값 검증, JWT/세션 취약점, 암호화 알고리즘 적절성.

  출력: 심각도 + 위치 + 공격 시나리오 1줄 + 수정 가이드. 잘된 점 1~2개.
  [large diff 모드면: /tmp/boston-review-security-findings.txt에 저장]"
)
```

#### 2-3. Performance (조건부)

```
Agent(
  name: "performance",
  model: "sonnet",
  prompt: "당신은 The Optimizer입니다. Research-only.

  [공통 컨텍스트]

  관점: N+1 쿼리, 중복 DB 조회, 캐시 미적용, 반복문 내 무거운 연산, 인덱스 미활용, 메모리 누수, 비동기 필요 구간.

  출력: 병목 + Big-O 분석(가능 시) + 개선 전/후 코드 + 예상 효과. 잘된 점 1~2개.
  [large diff 모드면: /tmp/boston-review-performance-findings.txt에 저장]"
)
```

#### 2-4. Data Steward (조건부)

```
Agent(
  name: "data-steward",
  model: "sonnet",
  prompt: "당신은 Data Steward입니다. Research-only.

  [공통 컨텍스트]

  관점: 스키마 변경 영향도, 인덱스 설계, 데이터 유실 위험, 마이그레이션 롤백 가능성, 정합성.

  출력: 위험 항목 + 롤백 방안. 잘된 점 1~2개.
  [large diff 모드면: /tmp/boston-review-data-findings.txt에 저장]"
)
```

#### 2-5. Business Analyst (조건부)

```
Agent(
  name: "business-analyst",
  model: "sonnet",
  prompt: "당신은 The Translator입니다. Research-only.

  [공통 컨텍스트]

  관점: 요구사항·구현 일치, 비즈니스 규칙 정확성(계산식·상태 전이·예외), 도메인 용어의 코드 반영.

  출력: 불일치 지점 + 근거 + 보정 제안. 잘된 점 1~2개.
  [large diff 모드면: /tmp/boston-review-business-findings.txt에 저장]"
)
```

#### 2-6. Frontend Expert (조건부)

```
Agent(
  name: "frontend-expert",
  model: "sonnet",
  prompt: "당신은 The UX Guardian입니다. Research-only.

  [공통 컨텍스트]

  관점: API 스펙 변경의 프론트 영향, TS 타입 정의 일치, 에러 메시지 UX, 로딩/에러 상태 처리.

  출력: 영향 항목 + 연동 리스크 + 방어 코드 제안. 잘된 점 1~2개.
  [large diff 모드면: /tmp/boston-review-frontend-findings.txt에 저장]"
)
```

### Phase 3: 오케스트레이터 최종 검증 & 통합

Chicago의 검증 단계를 유지하되, 리포트 포맷을 축약한다.

- **중복 제거**: 동일 파일·라인의 중복 이슈는 가장 상세한 것을 대표로 선택하고 다른 관점을 병기
- **오탐 필터링**: 프레임워크가 이미 처리하는 사안, 프로젝트 컨텍스트에 맞지 않는 지적 제거
- **심각도 재조정**: 프로젝트 특성에 맞춰 조정. 조정 시 원 심각도와 사유 병기
- **충돌 분석**: 에이전트 간 상반된 의견을 트레이드오프 섹션에 정리
- **실행 가능성 검증**: 제안된 수정이 실제 적용 가능한지, 다른 코드와 충돌하지 않는지 확인
- **프로젝트 컨벤션 부합 여부** 교차 확인

### Phase 4: 사용자 확인 (PR 리뷰인 경우)

Chicago와 동일.

```
리뷰 결과가 준비되었습니다. PR 코멘트로 게시할까요?
[리포트 표시]
- yes/게시: 게시 진행
- edit/수정: 수정 반영 후 재확인
- no/취소: 게시 취소
```

파일/스니펫 리뷰는 Phase 4~5를 건너뛰고 리포트를 직접 출력한다.

### Phase 5: PR 코멘트 게시

```bash
gh pr comment <PR_NUMBER> --repo <OWNER/REPO> --body "$(cat <<'COMMENT_EOF'
<review content>
COMMENT_EOF
)"
```

### Phase 6: 임시 파일 정리

```bash
rm -f /tmp/boston-review-*.txt
```

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

중복 제거된 이슈를 한 줄씩 flat 리스트로 나열한다.
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

## 공통 심각도 레이블

Chicago와 동일한 레이블을 사용한다.

| 레이블 | 의미 | 처리 기준 |
|---|---|---|
| 🔴 CRITICAL | 보안 취약점, 데이터 유실 | 배포 블로킹, 즉시 수정 |
| 🟠 HIGH | 명백한 버그, 심각한 성능 문제 | 이번 PR에서 수정 권장 |
| 🟡 MEDIUM | 설계 개선, 테스트 추가 필요 | 다음 스프린트 |
| 🟢 LOW | 스타일, 가독성, 제안 | 선택적 반영 |

---

## 파일 구조

```
plugins/boston-code-review/
├── README.md
└── skills/
    └── boston-code-review/
        └── SKILL.md
```

- 기존 `plugins/chicago-code-review/`, `plugins/montreal-code-review/`, `plugins/team-code-review/`와 독립된 플러그인
- 마켓플레이스 manifest에 등록 (기존 플러그인 등록 방식 재사용)

---

## 엣지 케이스

- **PR을 찾을 수 없음**: 사용자에게 지정 요청, 또는 현재 브랜치 대비 main/master diff 리뷰 제안
- **특정 에이전트 실패**: 나머지 결과로 진행. 리포트에 실패 에이전트 명시
- **매우 큰 diff (3000줄 초과, 필터 후)**: 소스 우선 선택. 제외 파일 리포트에 명시
- **이슈가 없음**: 긍정 리뷰 게시, 잘된 점 강조
- **사용자가 게시를 거부**: 임시 파일 정리 후 종료
- **자동 감지 결과 0개**: CodeRabbit 단독 실행. 이 경우가 경량 리뷰의 기본 케이스
- **자동 감지 결과 5개 전부**: CodeRabbit + 전문가 5명 병렬 실행. 복잡 PR에서만 발생 예상

---

## 성공 기준

1. 소형 PR (diff ≤ 300줄, 자동 감지 0~1개) 기준으로 Chicago 대비 토큰 80% 이상 절감
2. 복잡 PR (diff > 1000줄, 자동 감지 3개 이상) 기준으로 Chicago 대비 토큰 40~50% 절감 + 주요 이슈 커버리지 90% 이상 유지
3. SKILL.md 파일 크기 300줄 이하
4. 사용자 수동 override 없이도 자동 감지 결과가 팀 직관과 80% 이상 일치
5. 기존 Chicago / Montreal / team-code-review 플러그인과 충돌 없이 독립 동작

---

## 향후 확장 (범위 외)

- 자동 감지 규칙의 언어별 세분화 (Python / Go / Rust 키워드 추가)
- 리뷰 결과의 품질 지표 자동 측정 (false positive rate 트래킹)
- 토큰 사용량 리포트 자동 생성
- 팀 규칙/컨벤션 파일 연동으로 조직 맞춤 감지 규칙
