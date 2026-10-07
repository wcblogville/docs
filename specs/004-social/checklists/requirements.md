# Specification Quality Checklist: 교류 (SOCIAL) — 댓글·답글·공감·이웃·마을 소식

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
  - 구현 용어 검색(테이블, DB, zod, React, Server Action, 트랜잭션, API, 파일 경로, `parentId`/`postId`, SQL, 쿼리, CASCADE): 0건. 원본의 함수·테이블·파일 경로는 모두 동작 설명으로 옮겼다 (예: "기본 키 (post_id, user_id)" → "(글, 회원) 한 쌍에 최대 1개, 저장 단계의 규칙으로도 거부").
  - "서버에서 확인"은 constitution IV(권한은 서버에서)의 사용자 관점 보안 요구로 보고 구현 세부로 치지 않았다.
  - 화면 문구는 원본 그대로 인용했다 (`댓글을 적어 주세요`, `댓글은 1000자까지예요`, `글을 찾을 수 없어요`, `잘못된 요청이에요`, `삭제된 댓글이에요`, `댓글을 삭제할까요?`, `로그인하면 댓글을 남길 수 있어요`, `로그인하면 공감할 수 있어요`, 이웃 새 글·마을 소식 빈 화면 문구 등).
  - 모든 FR(54개)에 원본 ID(SOC-01~05, 관련 NF·GAME-05·POST-04/05)를 붙였고, SOC-01~05 모두 하나 이상의 User Story 수용 시나리오로 확인된다.
  - 2026-10-07 결정(답글 분리 1단계 구조, 온보딩 없음)을 목표 동작으로 반영했다.
- 남은 항목: `[NEEDS CLARIFICATION]` 2개 (한도 3개 이내). `/speckit-clarify`에서 정한다.
  1. FR-013 (SOC-01): 블로그 주인·관리자의 남의 댓글 삭제 권한.
  2. FR-018 (SOC-02): 없는·다른 글의·삭제된 댓글을 대상으로 한 답글 요청을 거부할지, 현재처럼 받아 줄지.
- 나머지 원본 열린 질문은 spec의 Assumptions "기본값 (원본 열린 질문)"과 Out of Scope에 기록했다.
- 위 2개를 정하면 `/speckit-plan`으로 넘어갈 수 있다.
- 2026-10-07 clarify 반영: FR-013(삭제 권한을 댓글 작성자·그 글이 올라간 블로그의 주인·관리자로), FR-018(없는·다른 글의·삭제된 댓글 대상 답글은 저장 거부, 입력 내용 유지)을 확정해 `[NEEDS CLARIFICATION]` 0개. 다른 spec 결정 D2(탈퇴 회원 댓글, Out of Scope 문장)·D11(FR-033 공감 알림, 새 FR-055 댓글 알림·FR-056 답글 알림)·D15(FR-042 이웃 새 글 순서)를 본문에 맞췄고, D6(보상 회수 안 함)은 이미 일치해 고치지 않았다. 구현 용어 재검색 0건, 16/16 통과.
- 2026-10-07 clarify 반영 점검: (1) D2와 어긋나던 FR-014·Assumptions의 "삭제 자리는 항상 남고 작성자 닉네임·캐릭터가 보임"을 작성자·블로그 주인·관리자가 지운 경우로 좁히고, 회원 탈퇴는 AUTH-06 처리(자리 없이 함께 삭제, 다른 사람의 답글이 달린 댓글만 자리 남김, 그 자리에 닉네임·캐릭터 안 보임)를 따르게 했다 (Out of Scope·Edge Cases·Key Entities 함께 맞춤). Clarifications의 D2 줄도 결정 원문("기존 댓글 삭제 규칙대로", constitution VII)에 맞췄다. (2) D9와 어긋나던 Assumptions "답글 오류 문구는 댓글과 같다"에 답글 대상 댓글 오류 예외를 적었다. (3) D4와 어긋나던 Out of Scope "검색은 내 블로그 안에서"를 "회원이 자기 블로그 홈에서 마을 전체 공개 글·블로그를 검색, 마을 소식·광장에는 검색창 없음"으로 고쳤다. (4) FR-042에 최신순 기준(마을 소식과 같음)과 페이지를 넘겨도 순서가 이어짐을 더했다 (003-post FR-035와 같음). (5) GAME-08 알림과 헷갈리지 않게 이웃 버튼의 "완료 알림"을 "완료 안내"로 바꿨다. FR 56개(FR-055·FR-056은 번호를 바꾸지 않으려고 각 절 끝에 둠) 모두 원본 ID 있음, 마커 0개, 구현 용어 0건, 16/16 통과.
