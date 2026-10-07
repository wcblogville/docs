# Specification Quality Checklist: 캐릭터 / 성장 (GAME)

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-10-07
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [x] No [NEEDS CLARIFICATION] markers remain
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details)
- [x] All acceptance scenarios are defined
- [x] Edge cases are identified
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification

## Notes

- 검증 1회차 (2026-10-07):
  - 구현 세부: 원본의 파일 경로, 함수 이름, 테이블 · 컬럼 이름, 라이브러리, 화면 주소, 잠금 방식은 모두 동작 문장으로 바꿨다
    (예: "회원 단위 잠금 트랜잭션" → "동시에 요청이 와도 한 번만 처리", "`attendances.session_id`" → "출석이 일어난 로그인 상태").
    남은 "원장", "로그인 상태(세션)"는 원본 7장 용어 정의에 있는 도메인 용어라 유지했다.
  - 화면 문구와 보상 숫자는 원본에서 그대로 인용했다 (FR-015, FR-019, FR-020, FR-026~028, FR-035, FR-038~043).
  - 요구사항 ID: GAME-01~08 모두 FR에 연결, GAME-09는 보류라 Assumptions의 Out of Scope에 적었다. NF-04 · 05 · 06 · 15 · 16 · 19 연결.
  - 2026-10-07 결정 반영: 온보딩 없음(가입 화면에서 캐릭터 선택), 자동 출석 · 1~7일차 주기, 7일 보너스 중단, GAME-09 보류.
- 검증 2회차: 1회차에서 하루 최대 경험치(원본 190)가 새 출석 규칙과 맞지 않아 Assumptions 의존성에 210으로 보정해 적었다. 그 밖의 실패 항목 없음.
- 남은 항목: `[NEEDS CLARIFICATION]` 2개 (FR-020 출석 일차별 보상 숫자 확정, FR-045 알림함 2단계 공감 · 댓글 알림의 범위(GAME vs SOC)).
  `/speckit-clarify`에서 정한 뒤 체크한다. 나머지 원본 열린 질문은 Assumptions에 "기본값 (원본 열린 질문)"으로 정리했다.
- 2026-10-07 clarify 반영: FR-020 출석 일차별 보상 제안값 확정(하루 최대 경험치 210), FR-045 알림함 2단계(공감 · 댓글 · 답글 알림)를 GAME-08 범위로 확정, 다른 spec 결정 D6(보상 회수 안 함 → FR-016 확정) · D12(이미 산 캐릭터 계속 장착 → FR-005) 반영. 하루 최대 경험치 210 문장은 Assumptions 의존성에서 FR-020으로 옮겼다. `[NEEDS CLARIFICATION]` 0개, 모든 항목 통과.
- 2026-10-07 clarify 반영 점검: 2단계 알림 표가 행동한 회원의 닉네임을 보여주게 되어, 탈퇴한 회원의 댓글 · 답글을 함께 지우는 결정(D2, AUTH-06)과 맞추려고 FR-046 · 알림 엔티티 · Edge Cases에 "행동한 회원이 탈퇴하면 그 알림도 지운다"를 넣었다. FR-045에 답글 알림은 글 주인이 아니라 원댓글 작성자가 받는다고 밝혀 SOC 영역 spec과 맞췄고, SC-011에 코인을 넣어 FR-016과 맞췄다. 다시 확인한 결과 모든 항목 통과.
- Items marked incomplete require spec updates before `/speckit-clarify` or `/speckit-plan`
