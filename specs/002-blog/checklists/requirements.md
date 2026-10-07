# Specification Quality Checklist: 블로그 (BLOG)

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-10-07
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [ ] No [NEEDS CLARIFICATION] markers remain
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

### 검증 기록

**1회차**
- 구현 세부 사항: 원본의 파일 경로, 함수 이름, 테이블·제약 이름, 쿠키 이름, 라이브러리 이름(`blogs_owner_id_unique`, `parseId`, `bv_visitor`, `revalidatePath` 등)은 모두 동작으로 바꿔 적었다 ("데이터 규칙으로 거부", "숫자만 적힌 1~2147483647 정수만 받는다", "브라우저에 남기는 무작위 방문자 표시"). 남은 기술 용어는 사용자에게 보이는 것(`/@주소` 주소 형식, 404 화면, px 크기)과 constitution 원칙 IV(권한 검사는 서버에서)뿐이라 통과.
- 실패 1건: User Story 6에 [NEEDS CLARIFICATION]을 Given/When/Then 시나리오 안에 넣어 시나리오가 테스트할 수 없는 형태였다 → 시나리오에서 빼고 FR-050으로 옮김.
- 2026-10-07 결정 반영 확인: 온보딩 관련 흐름("온보딩 전 회원", `/onboarding` 재진입)은 목표 동작에서 제외, 블로그 자동 생성(FR-001), 주소·닉네임 수정(FR-018, FR-019), 2단계 카테고리(FR-033~041), 프로필·전시·도감(FR-028~031) 포함.
- 요구사항 ID 확인: BLOG-01~07과 "블로그 홈 화면(BLOG-01~05 공통)"이 모두 FR에 출처 ID로 붙어 있다.

**2회차**
- 위 수정 뒤 전 항목 다시 확인. "No [NEEDS CLARIFICATION] markers remain"만 미통과(의도적, 3개 이하 제한 준수).
- 남은 [NEEDS CLARIFICATION] 3개:
  1. 가입 아이디로 기본 주소 `/@{아이디}`를 만들 수 없을 때(예약어와 같음, 다른 회원이 주소를 그 값으로 바꿔 이미 씀)의 처리 — Edge Cases
  2. 블로그 주소를 바꾼 뒤 예전 주소를 다른 회원이 바로 가져갈 수 있는지(사칭 위험) — Edge Cases
  3. BLOG-07 검색을 누가 어느 블로그에서 쓸 수 있고 검색 대상이 무엇인지 — FR-050
- 나머지 원본 열린 질문은 spec Assumptions의 "기본값 (원본 열린 질문)"에 기록했다.
- 다음 단계: `/speckit-clarify`로 위 3개를 정한 뒤 `/speckit-plan`.
