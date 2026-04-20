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
