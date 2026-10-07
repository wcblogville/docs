# Specification Quality Checklist: 글 (POST) — 글쓰기·공개 범위·분류·목록·조회수·첨부·임시 저장

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
- 검증 1회차 (2026-10-07):
  - 구현 세부: 원본의 파일 경로, 함수·라이브러리 이름, 테이블·컬럼 이름, HTTP 상태 코드, 요청 머리글을 모두 빼고 동작으로 옮겼다.
    남은 `PNG·JPG`, `JSON`, `MP4` 등은 사용자가 고르는 파일 형식이고, `404`는 사용자에게 보이는 "찾을 수 없음 화면"의 이름으로만 썼다.
  - `[NEEDS CLARIFICATION]` 표시가 마커 3개 외에 참조 문장 2곳(FR-046, Edge Cases)에도 있어 셈이 헷갈렸다 → "확인 질문"으로 바꿈.
- 검증 2회차: 위 항목 모두 통과. 남은 실패 항목은 의도된 `[NEEDS CLARIFICATION]` 3개뿐이다 (한도 3개 이내).
  1. US7 시나리오 6 / FR-046 (POST-06): 같은 사람의 반복 조회를 매번 셀지, 일정 기간 1번만 셀지
  2. FR-016 (POST-01): 삭제한 글의 보상을 회수할지 (GAME-05와 함께 결정)
  3. FR-047 (POST-07·POST-09): 이미 내 다른 글에 붙은 첨부를 다른 글 본문에 넣을 때의 처리 (다시 올림 / 뺌 / 옮김)
- 나머지 원본 열린 질문은 spec Assumptions에 "기본값 (원본 열린 질문)"으로 정리했다. `/speckit-clarify`에서 위 3개를 정한 뒤 `/speckit-plan`으로 간다.
- 2026-10-07 설계 변경(온보딩 없음, 카테고리 2단계, 첨부는 글에 붙음, 비공개 글 첨부는 주인만, 로그인 유지 2시간)을 목표 동작으로 반영했음을 확인했다.
- 2026-10-07 clarify 반영: 조회수는 같은 브라우저 기준 글마다 한국 시간 하루 1번(POST-06), 글을 지워도 보상 회수 없음(POST-01·GAME-05), 내 다른 글의 첨부를 붙여 넣으면 새로 다시 올림·저장 시 이 글에 붙일 수 없는 첨부는 뺌(POST-07·POST-09), 이웃 새 글은 최근 7일 안 즐겨찾은 이웃 글을 맨 위에(TOWN-08 파급). `[NEEDS CLARIFICATION]` 0개, 16/16 항목 통과.
- 2026-10-07 clarify 반영 점검: 글 삭제 때 회수하지 않는 공감 보상의 받는 사람을 "다른 회원"에서 "글 주인"으로 바로잡음(US2 시나리오 8, FR-016, GAME-05·SOC-03과 맞춤). Key Entities 첨부를 FR-047과 맞춤(다시 올리는 것은 내 다른 글의 첨부만, 남의 첨부·다른 글의 첨부는 저장 때 뺌). FR-047의 "붙일 수 있는 첨부"에서 지금 쓰는 프로필 사진을 빼고, FR-059·SC-011·Assumptions에 프로필 사진 예외(정리하지 않음, 누구나 봄, BLOG-04)를 넣음. 조회수는 BLOG-06과 같은 기준이 되도록 FR-046에 "브라우저에서 실제로 열었을 때"를 넣고, Assumptions의 검색 로봇 기본값을 이 기준에 맞춰 고침. US7 시나리오 1의 조건을 "아직 열지 않은 글"에서 "아직 조회수가 오르지 않은 글"로 고침. `[NEEDS CLARIFICATION]` 0개, 구현 용어 재검색 0건, FR-001 ~ FR-066 연속·원본 ID 모두 있음, 16/16 통과.
