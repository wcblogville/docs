# Specification Quality Checklist: 광장 (TOWN)

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

- Items marked incomplete require spec updates before `/speckit-clarify` or `/speckit-plan`
- 검증 1회차: 화면 주소(`/town`, `/farm` 등), 엔진·파일·함수·테이블 이름, 쿼리 설명이 spec 본문에 없음을 검색으로 확인. 원본의 구현 방식 표는 "광장을 열 때 한 번 읽는다", "서버에서 확인" 같은 동작으로 옮김. "서버에서 확인"은 constitution 원칙 IV(권한·검증은 서버)를 따르는 요구로 남김.
- 원본 요구사항 ID TOWN-01 ~ TOWN-11 모두 FR에 연결됨(FR-001 ~ FR-061, TOWN-06은 제외 사항으로 FR-061에 명시).
- 사용자 문구와 숫자(90px, 초당 230px, 1800 × 1400, 최대 10채·10명·5마리, 코인 100, 동물 표, 돌보기 +10/+10/+5·경험치 +2)는 원본 그대로 인용.
- 2026-10-07 설계 변경(온보딩 없음, 자동 출석, 프로필 사진, 성장 아이템, 블로그 도감·전시)을 목표 동작으로 반영하고 Assumptions에 근거를 적음.
- 남은 [NEEDS CLARIFICATION] 3개(`/speckit-clarify`에서 결정 필요):
  1. FR-029 방문자에게 보여줄 "인기 블로그"의 기준 (TOWN-04)
  2. FR-036 💛 이웃 새 글에서 즐겨찾은 이웃 글을 올리는 범위(전부 / 최근 7일) (TOWN-08, SOC-04·POST-05와 협의)
  3. FR-059 집 단계를 올리는 조건(레벨 / 공개 글 수 / 코인 증축) (TOWN-11)
- 나머지 원본 열린 질문은 Assumptions의 "기본값 (원본 열린 질문)"으로 기록함.
- 2026-10-07 clarify 반영: 위 3개를 결정으로 바꾸고 spec에 Clarifications(Session 2026-10-07)를 추가함 — FR-029 인기 기준(이웃 수 많은 순, 같으면 최근 공개 글 순, 공개 글 없는 블로그 제외), FR-036 최근 7일(한국 시간) 안의 즐겨찾은 이웃 글만 위로, FR-059 집 단계 = 주인 레벨(Lv.1~9 / 10~29 / 30 이상, 증축 구매 없음). 파급으로 FR-048 성장 아이템 수치를 상점 첫 출시 초기값(필요 레벨 포함)으로 맞춤. 남은 표시 0개, 전 항목 다시 점검해 16/16 통과.
- 2026-10-07 clarify 반영 점검: Assumptions "기본값 (원본 열린 질문)" 중 결정으로 바뀐 부분(TOWN-04 인기 기준·공개 글 없는 블로그 제외, TOWN-09 성장 아이템 가격·필요 레벨·성장치)을 기본값이 아닌 clarify 결정으로 고쳐 적음. 상점 부제의 판매 분류에 배경을 넣어 상점 첫 출시 목록(D13)과 맞춤. FR-036·Assumptions의 이웃 새 글 근거를 SOC-04·POST-05로 바로잡음. Key Entities 성장 아이템에 필요 레벨 추가, US5 시나리오 1을 "상위 100곳(적으면 전부)"으로 FR-029와 맞춤. `[NEEDS CLARIFICATION]` 0개, 구현 용어 재검색 0건, FR-001 ~ FR-061 연속·원본 ID 모두 있음, 16/16 통과.
