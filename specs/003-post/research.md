# Research: 글 (POST)

**Feature**: `003-post` | **Date**: 2026-10-07 | **Plan**: [plan.md](plan.md)

Technical Context의 모르는 것과 이 기능의 기술 선택을 정리한다. 근거는 코드 저장소 `main` `feb4c05`에서 읽은 코드와 문서다.
코드 위치는 코드 저장소 기준 상대 경로로 쓴다.

**확인 방법 표시**

- **코드 확인**: 코드 저장소에서 직접 읽은 사실.
- **추측**: 이 체크아웃에는 `node_modules`가 없어 라이브러리 문서·타입을 열 수 없었다. 기억이나 기존 코드의 쓰임새에 기댄 판단이며, **구현 전에 설치된 패키지 문서로 확인**한다 (확인할 곳을 함께 적었다).

## 요약

| # | 주제 | 결정 (한 줄) | 관련 |
|---|---|---|---|
| R1 | 조회수 하루 1번 저장 | 새 표 `post_views (post_id, date, visitor_id)` + `posts.view_count`는 계속 쌓는 숫자 | FR-046, SC-012 |
| R2 | 조회수 세는 시점 | 서버 렌더가 아니라 화면이 열린 뒤 부르는 Server Action `recordPostView` | FR-046, US7-5 |
| R3 | 방문자 쿠키 | BLOG-06의 `bv_visitor`를 같이 쓰고 도우미 `src/server/visitor.ts`를 새로 둔다 | FR-046 |
| R4 | 대분류 삭제와 CHECK 충돌 | `posts`에 `BEFORE UPDATE OF category_id` 트리거로 소분류를 함께 비운다 | FR-030, ERD 3.14·3.18 |
| R5 | `ON DELETE SET NULL (subcategory_id)` | schema.ts에 FK를 적고 생성된 SQL을 손으로 고친다 | ERD 3.18 |
| R6 | 첨부를 글에 붙이는 규칙 | `savePost` 트랜잭션 안에서 첨부 행을 잠그고 "붙일 수 있는 것"만 남긴다 | FR-047, FR-054 |
| R7 | 떨어진 시각 | `attachments.detached_at` + `BEFORE UPDATE OF post_id` 트리거 | FR-059 |
| R8 | 붙여 넣기 다시 올리기 | 판정 Server Action + 서버에서 파일 복사로 새 첨부 | FR-047, US8-10, US9-9 |
| R9 | 첨부 주소 권한·캐시 | 붙은 글의 공개 범위·올린 사람 확인, `private, no-cache` + ETag | FR-029, FR-059, SC-004 |
| R10 | 정리 작업 | `npm run posts:cleanup` 스크립트. 배포 예약 방법은 NF-08과 함께 | FR-059, SC-011 |
| R11 | 본문 200,000자와 요청 크기 | 설정은 그대로(기본 1MB). 아주 큰 본문은 브라우저가 같은 스키마로 먼저 막는다 | FR-006, Assumptions |
| R12 | 조작된 형식 값 | ID·공개 설정의 모든 오류 문구를 `잘못된 요청이에요`로 | FR-018, Edge Cases |
| R13 | 발행 안내 한 번 | `?new=1` + 주인 + 10분 안 + 원장으로 보상 판단 + 주소에서 `?new` 지우기 | FR-013 |
| R14 | 임시 저장 | `localStorage` `blogville:draft:{userId}`, 2초 debounce, 로그아웃 때 지움 | FR-060~064 |
| R15 | 대분류·소분류 저장 규칙 | 남의·없는 대분류는 조용히 없음, 다른 대분류의 소분류는 거부, 없는 소분류는 대분류만 | FR-030~032 |
| R16 | 카테고리 배지 | `대분류 › 소분류`, 상세 배지는 가장 아래 단계 목록(`?category=…&sub=…`, blog R-13)으로 | FR-033 |
| R17 | 정화·글자 수 | 규칙은 그대로, `known`만 "붙일 수 있는 첨부"로 좁힌다 | FR-007, FR-011, SC-003, SC-005 |
| R18 | 기존 데이터 이전 | `attachments.post_id`를 본문 주소로 채운다 (가장 먼저 쓴 내 글) | ERD 7장 3 |
| R19 | 프로필 사진 예외 | auth의 `profiles.photo_key` 하나의 조건으로 세 곳에서 쓴다 | FR-047, FR-059 |
| R20 | 테스트 방식 | 순수 규칙은 `scripts/test-post.ts`, 흐름은 새 e2e 6개 | constitution III |
| R21 | 누르는 영역 44×44px | post 소유 화면의 작은 버튼·링크를 `min-h-11`로 (blog·social 요청 포함) | constitution VI, FR-066 |

NEEDS CLARIFICATION은 모두 아래에서 결정했다. 팀 확인이 필요한 것은 [plan.md](plan.md)의 "남은 문제"에 모았다.

---

## R1. 조회수 "같은 브라우저 하루 1번"을 어디에 저장하나

**Decision**: 새 표 `post_views`(`post_id`, `date`(한국 날짜), `visitor_id` uuid) 복합 PK를 둔다. `posts.view_count`는 그대로 쌓는 숫자로 두고,
`post_views`에 행이 **새로 들어갔을 때만** 같은 트랜잭션에서 `view_count = view_count + 1`을 한다. 이틀 지난 `post_views` 행은 정리 작업(R10)이 지운다.

**Rationale**

- BLOG-06 방문자 수가 이미 같은 모양(`blog_visits`의 PK (`blog_id`, `date`, `visitor_id`), `onConflictDoNothing()`)으로 동작하고
  `e2e/visits.mjs`로 검증되어 있다 (코드 확인: `src/app/blog/actions.ts`의 `recordBlogVisit`). spec도 "BLOG-06과 같은 기준"이다.
- "하루 1번"을 복합 PK가 지키므로 동시에 여러 번 열어도 한 번만 센다 (constitution V). 증가는 한 줄 `UPDATE ... + 1`이라 여러 사람이 동시에 열어도 빠지는 수가 없다 (FR-046, 지금 `incrementViewCount`와 같은 방식).
- 목록 카드는 지금처럼 `view_count` 컬럼을 읽으므로 목록 쿼리 비용이 늘지 않는다 (SC-002).
- 조회 기록은 "오늘 이미 셌나"를 판단할 때만 필요하다. 어제 행까지만 남기면 충분해 쌓이지 않는다 (constitution VII). 블로그 방문자 표와 달리 날짜별 통계 화면이 없다 (spec 기본값: 조회수 통계 없음).

**Alternatives considered**

- 쿠키에 오늘 본 글을 적기 (원본 POST-06 제안 `seen_글ID`): 글마다 쿠키가 늘거나 한 쿠키가 4KB를 넘는다. 동시 요청에서 한 번만 세는 것을 DB가 보장하지 못한다.
- `view_count`를 없애고 `post_views` 행 수를 센다: 목록마다 세는 서브쿼리가 늘고, 오래된 행을 지울 수 없으며, 지금까지 쌓인 숫자를 옮길 수 없다.
- 회원 ID로 구분: 로그인하지 않은 방문자도 세야 하고, spec은 "같은 회원이라도 브라우저가 다르면 따로 센다"이다.

## R2. 조회수를 언제 세나: 서버 렌더 → 브라우저에서 부르는 Server Action

**Decision**: 글 상세(`src/app/blog/[slug]/[postId]/page.tsx`)의 서버 렌더에서 `incrementViewCount()`를 없앤다. 새 클라이언트 컴포넌트
`ViewCount`(`src/components/blog/view-count.tsx`)가 화면이 열린 뒤 `useEffect`(의존성 `postId`)에서 Server Action `recordPostView(postId)`를
**한 번** 부르고, 돌려받은 숫자로 `👀 N`을 바꾼다. 서버 렌더는 첫 숫자만 계산한다: 주인이 아니고, 이 브라우저(쿠키)로 오늘 센 기록이 없으면 `저장값 + 1`, 아니면 저장값.

**Rationale**

- Next.js 16 Server Component는 쿠키를 읽을 수만 있고 새로 만들 수 없다. 그래서 `recordBlogVisit`도 Server Action에서 쿠키를 만든다 (코드 확인: 그 함수 주석 "Next.js 16 cookies 규칙").
  첫 방문 브라우저는 쿠키가 없으므로 서버 렌더에서는 셀 수 없다.
- 렌더 중에 세면 공감·댓글 뒤 `revalidatePath("/", "layout")`로 화면을 다시 그릴 때마다 또 센다 (US7-5 위반). 원본 POST-06 열린 질문이 이 문제와 prefetch 부작용을 이미 지적했다.
- spec 기본값 "글 상세가 브라우저에서 실제로 열렸을 때만 센다(화면을 열지 않는 요청은 세지 않는다)"와 그대로 맞는다.
- `VisitCount`(`src/components/blog/visit-count.tsx`)가 같은 방식(처음 숫자로 시작 → Server Action 결과로 바꿈, 의존성 하나)이라 팀에 익숙하다.
- 첫 숫자를 미리 `+1`로 그리면 "방금 연 것까지 반영"(FR-046)이 깜빡임 없이 보이고, Server Action 결과가 오면 그 값으로 맞춘다.

**Alternatives considered**

- Route Handler `POST /api/views`: Server Action이 해 주는 출처(Origin) 확인을 직접 해야 한다 (`src/app/api/uploads/route.ts`처럼). 이점이 없다.
- 서버 렌더에서 쿠키가 있을 때만 세기: 첫 방문을 못 세고, 재렌더 문제가 남는다.

**추측·확인할 것**: 쿠키를 바꾸는 Server Action 뒤에 Next.js가 현재 화면을 다시 그린다는 점(`VisitCount` 주석에 적힌 동작)과 `cookies()` 읽기·쓰기 규칙을
`node_modules/next/dist/docs/01-app/02-guides/server-actions.md`와 `cookies` API 문서로 확인한다.

## R3. 방문자 쿠키를 같이 쓰고, 첫 방문 경쟁을 피한다

**Decision**

- BLOG-06의 `bv_visitor` 쿠키(무작위 UUID, `HttpOnly`, `SameSite=Lax`, `Path=/`, 1년, 배포에서 `Secure`)를 그대로 쓴다.
- 쿠키 이름·속성·UUID 형식 검사를 새 서버 모듈 `src/server/visitor.ts`에 둔다: `readVisitorId()`(서버 렌더용, 읽기만), `ensureVisitorId()`(Server Action용, 없으면 만듦).
  `recordPostView`가 이 모듈을 쓴다. blog의 `recordBlogVisit`는 그대로 두고, blog가 원하면 이 도우미로 바꾼다 (비차단 요청, plan 의존성).
- 글 상세에는 이미 `RecordVisit`(blog)가 있다. 첫 방문 브라우저에서 두 Server Action이 서로 다른 UUID로 쿠키를 만들면, 조회 기록은 A로 남고 쿠키는 B가 되어
  새로고침 때 한 번 더 셀 수 있다. Next.js 클라이언트는 Server Action을 **한 번에 하나씩** 보내므로 앞 요청의 `Set-Cookie`가 뒤 요청에 실린다는 동작에 기댄다.
  `e2e/post-views.mjs`의 "새 브라우저 첫 방문 → 새로고침 → 1만 오름"으로 지킨다. 문서 확인 결과 그렇지 않으면, `ViewCount`가 한 `useEffect` 안에서
  `recordBlogVisit` → `recordPostView`를 차례로 부르도록 바꾸고 상세 화면의 `RecordVisit`를 뺀다 (blog와 합의).

**Rationale**: spec이 "BLOG-06 방문자 수와 같은 기준(같은 브라우저)"이다. 식별 쿠키를 하나 더 두면 "같은 브라우저"의 정의가 둘로 갈리고 저장하는 값이 늘어난다 (constitution VII).

**Alternatives considered**: 조회수 전용 쿠키(`bv_post_viewer`): 경쟁은 없지만 식별 쿠키가 둘이 된다.

**추측·확인할 것**: "Server Action을 한 번에 하나씩 보낸다"는 Next.js 문서의 Server Functions 안내에 "현재 구현"으로 적힌 동작으로 기억한다.
`node_modules/next/dist/docs/` 안 server actions 문서에서 확인한다.

## R4. 대분류를 지우면 CHECK 위반이 난다 → `posts` 트리거로 소분류도 비운다

**사실 (코드 확인 + 추측)**

- ERD 3.18은 `posts`에 복합 FK (`category_id`, `subcategory_id`) → `subcategories` (`category_id`, `id`) `ON DELETE SET NULL (subcategory_id)`와
  CHECK (`subcategory_id IS NULL OR category_id IS NOT NULL`)를 정했다. `posts.category_id` → `categories`는 지금 `ON DELETE SET NULL`이다 (코드 확인: `drizzle/0000_init.sql`).
- 대분류를 지우면 `categories`의 AFTER DELETE FK 트리거 두 개가 돈다: ① `posts.category_id`를 NULL로 (0000_init에서 생성) ② `subcategories` CASCADE 삭제 (blog 마이그레이션에서 생성).
  PostgreSQL은 같은 표·같은 이벤트의 트리거를 **이름순**으로 실행하고 FK 트리거 이름은 생성 순서의 OID를 쓴다(`RI_ConstraintTrigger_a_<oid>`) → ①이 먼저 돌아
  소분류가 남아 있는 글에서 (`category_id` NULL, `subcategory_id` 있음)이 되어 **CHECK 위반 오류**가 난다. (추측: 실행 순서는 PostgreSQL 트리거 문서의 이름순 규칙에 근거. 구현 때 SQL로 재현해 확인한다.)
- 회원 삭제도 같다. `blogs`를 지우면 `categories` CASCADE(0000_init 186행, 먼저 생성)가 `posts` CASCADE(197행)보다 먼저 돌아, 지워지기 전의 글에 ①이 걸린다 (코드 확인: 0000_init의 FK 생성 순서).
  탈퇴(AUTH-06)·관리자 회원 삭제가 소분류 글이 있는 회원에서 실패할 수 있다.
- blog research R-11은 같은 문제를 보고 `deleteCategory`가 같은 트랜잭션에서 글의 두 칸을 먼저 비운 뒤 대분류를 지우기로 했다 (트리거는 "규칙이 숨는다"며 고르지 않음).
  이 방법은 블로그 관리의 삭제만 막는다. 회원 삭제의 CASCADE 경로와, `deleteCategory`가 글을 비운 뒤 대분류 행을 지우기 전에 다른 탭의 `savePost`가 같은 대분류·소분류로 글을 저장한 경우(저장이 대분류 행을 `FOR KEY SHARE`로 잡아 삭제가 기다렸다가 그 글에 ①을 건다)는 남는다.

**Decision**: `posts`에 행 트리거 `posts_clear_subcategory`(`BEFORE UPDATE OF category_id`)를 둔다: 새 `category_id`가 NULL이면 `subcategory_id`도 NULL로 바꾼다.
FK 동작이 내부에서 하는 `UPDATE`에도 사용자 트리거가 돈다. CHECK와 복합 FK는 ERD대로 둔다. INSERT에는 걸지 않아, 앱 실수로 (NULL, 소분류)를 넣으면 CHECK가 거부한다.

**Rationale**

- ERD 3.14 "대분류 삭제 → 글은 남기고 `category_id`·`subcategory_id`를 비움"을 **모든 경로**(블로그 관리 삭제, 회원 탈퇴, 관리자 회원 삭제, 삭제와 겹친 저장)에서 DB가 지킨다 (constitution V).
- auth의 탈퇴·관리자 회원 삭제 코드를 바꾸지 않아도 된다 (공통 모듈 소유 규칙). blog R-11의 앱 처리와 함께 써도 충돌하지 않는다 (먼저 비워졌으면 트리거가 할 일이 없다).
- "규칙이 숨는다"는 걱정은 트리거 이름·이유를 마이그레이션 주석과 ERD 3.14·3.18에 적어 줄인다. 어느 방식으로 맞출지는 blog와 팀 확인 (plan 남은 문제 2).

**Alternatives considered**

- CHECK를 없애고 앱에서만 막기: "소분류만 있는 글"을 DB가 막지 못한다 (V 약화). 복합 FK는 칸 하나가 NULL이면 검사하지 않는다 (ERD 3.18).
- blog가 대분류를 지우기 전에 글을 먼저 비우기: 회원 삭제의 CASCADE 경로는 막지 못한다.
- `posts.category_id` FK를 다시 만들어 트리거 순서를 바꾸기: OID·이름순에 기대는 숨은 규칙이라 다음 사람이 알 수 없다.
- CHECK를 deferrable constraint trigger로: 더 복잡하다.

ERD 3.18과 3.14에 이 트리거를 적는다 (post가 `posts` 담당).

## R5. 복합 FK의 `ON DELETE SET NULL (subcategory_id)`를 Drizzle로 표현하기

**Decision**: `src/db/schema.ts`의 `posts`에 `foreignKey({ name: "posts_subcategory_fk", columns: [categoryId, subcategoryId], foreignColumns: [subcategories.categoryId, subcategories.id] }).onDelete("set null")`과
CHECK `posts_subcategory_check`를 적는다. `npm run db:generate`가 만든 SQL의 `ON DELETE set null`을 `ON DELETE SET NULL ("subcategory_id")`로 손으로 고치고,
R4 트리거 함수·트리거도 같은 파일 끝에 직접 쓴다. 파일 맨 위 한국어 주석에 "손으로 고친 줄"과 요구사항 ID를 적는다 (관례: `drizzle/0004_attachments.sql`).

**Rationale**

- Drizzle의 `onDelete()`는 동작 이름만 받고 컬럼 목록을 받지 않는다 (추측: drizzle-orm 0.45 `pg-core` 타입 기억. 구현 전 `node_modules/drizzle-orm/pg-core/foreign-keys.d.ts` 확인).
  그대로 두면 소분류를 지울 때 두 컬럼이 모두 NULL이 되어 글이 대분류까지 잃는다 (blog US5-8 위반).
- 이 저장소는 `db:generate` + `db:migrate`만 쓰고 `push`·`introspect`를 쓰지 않는다 (코드 확인: `package.json`). 스냅숏(`set null`)과 실제 DB(컬럼 지정)의 작은 차이는
  다음 `generate`에서 차이로 잡히지 않는다 (generate는 스키마와 스냅숏을 비교한다).
- `ON DELETE SET NULL (컬럼)`은 PostgreSQL 15 이상 문법이다. README 설치 안내는 17이다.

**Alternatives considered**: FK를 schema.ts에서 빼고 직접 쓴 마이그레이션에만 두기 (schema.ts = ERD라는 저장소 규칙과 어긋난다), drizzle 올리기 (의존성 변경, 지원 여부도 모름).

## R6. 첨부를 글에 붙이는 시점과 규칙

**Decision**: `savePost`(`src/app/write/actions.ts`)의 트랜잭션 안, `lockUser` 뒤에서

1. 본문의 첨부 키(`attachmentKeysIn`)에 해당하는 `attachments` 행을 **키 순서로**(`ORDER BY key`) `FOR UPDATE`로 읽는다.
2. **붙일 수 있는 것**만 고른다: 내가 올렸고(`user_id` = 나), (어느 글에도 붙지 않았거나 이 글에 붙어 있고), 지금 프로필 사진이 아닌 것 (R19).
3. 그것만 `known`으로 `sanitizePostHtml(html, known)`에 넘긴다. 나머지 첨부 주소(남의 것, 다른 글의 것, 없는 키)는 정화가 본문에서 뺀다 (지금 `exclusiveFilter` 동작 그대로).
4. 글을 넣거나 고친 뒤, 남은 키에 `post_id` = 이 글, 이 글에 붙어 있었지만 본문에서 빠진 첨부는 `post_id` = NULL.

공통 처리는 `src/server/posts.ts`의 `linkAttachments(tx, …)`로 둔다.

**Rationale**

- FR-047·FR-054·D7: "내가 올렸고 아직 어느 글에도 붙지 않았거나 이미 이 글에 붙은 것만", "한 첨부는 한 글에만". `post_id` 컬럼이 하나라 한 첨부가 두 글에 붙는 일은 구조상 없다.
- 같은 회원의 저장은 `lockUser`(advisory lock)로 이미 줄을 선다 (코드 확인: `src/server/points.ts`). 정리 작업(R10)과의 경쟁은 행 잠금으로 막는다:
  정리 작업은 지울 행을 `FOR UPDATE SKIP LOCKED`로 고르므로 저장이 잡은 행은 이번 실행에서 건너뛴다(다음 실행 때 다시 본다). 정리 작업이 먼저 잡은 행은 저장이 기다렸다가 지워진 뒤에는 못 보고 주소를 뺀다.
  저장이 여러 행을 키 순서로 잠그고 정리 작업은 잠긴 행을 기다리지 않으므로, 두 쪽이 서로 다른 순서로 행을 기다리다 교착(deadlock → 저장 500)하는 일이 없다.
- 첨부는 글자 수에 들어가지 않으므로(코드 확인: `htmlToText`, `scripts/test-text-length.ts`) 빈 본문 확인(`본문을 적어 주세요`)은 트랜잭션 전에 첨부 정보 없이 해도 결과가 같다. 그래서 검사 순서(FR-009)를 지킨다.

**Alternatives considered**: 다대다 표(한 첨부를 여러 글에서 함께 쓰기는 spec Out of Scope), 올릴 때 글 ID를 받기(새 글은 아직 ID가 없다), 정화 뒤 별도 쿼리로 붙이기만 하고 잠그지 않기(정리 작업과 경쟁).

## R7. "떨어진 시각"을 기록한다

**Decision**: `attachments.detached_at timestamptz NULL`을 더하고, 행 트리거 `attachments_track_detached`(`BEFORE UPDATE OF post_id`)가
`post_id`가 값 → NULL로 바뀌면 `detached_at = now()`, NULL → 값이면 `detached_at = NULL`로 맞춘다 (`IS DISTINCT FROM`으로 실제로 바뀔 때만).
정리 기준은 `post_id IS NULL AND COALESCE(detached_at, created_at) < now() - interval '1 day'`.

**Rationale**

- FR-059 "올린 지(또는 떨어진 지) 하루가 지나면". 오래전에 올린 첨부를 글에서 빼자마자 지우면 안 된다.
- 글 삭제의 `ON DELETE SET NULL`은 DB가 하므로 앱이 시각을 넣을 수 없다. 트리거면 글쓴이 삭제(`deletePost`), 관리자 삭제(`adminDeletePost`, auth 파일), 본문에서 빼기를
  **한 규칙**으로 처리하고 다른 spec 파일을 바꾸지 않는다.

**Alternatives considered**: `created_at`만 보기 (spec과 다르게 일찍 지운다), 앱에서 `detached_at` 쓰기 (FK 동작·관리자 삭제 경로가 빠진다. auth에 요청이 생긴다).

## R8. 내 다른 글의 첨부를 붙여 넣으면 "다시 올리기"

**Decision**

- 에디터 `handlePaste`(`src/components/editor/rich-editor.tsx`)가 붙여 넣는 HTML에 `/files/키`가 있으면 기본 붙여 넣기를 막고, `useAttachmentUpload`에 새로 둘 흐름으로 처리한다:
  1. Server Action `classifyPastedAttachments(keys, postId?)`가 키마다 판정한다.
     - **keep**: 내 것, 그리고 (어느 글에도 없음 또는 이 글에 붙음), 그리고 프로필 사진 아님 → 그대로 넣는다.
     - **reupload**: 내 것, 그리고 (다른 글에 붙음 또는 지금 프로필 사진) → 다시 올린다.
     - **drop**: 남의 것, 없는 키 → 넣지 않는다.
  2. reupload 키마다 Server Action `reuploadAttachment(key, postId?)`: 서버가 저장소 파일을 새 무작위 키로 **복사**하고 같은 `name`·`mime`·`size`·`kind`로 새 행(`post_id` NULL)을 만든다.
  3. 키를 새 키로 바꾸고 drop 노드를 뺀 HTML을 붙여 넣은 자리(트랜잭션 매핑으로 따라감, 지금 `upload`와 같은 방식)에 넣는다.
- 진행 중에는 기존 올리기와 같이 `올리는 중... (i/n)`(n = 다시 올릴 개수), [🖼 사진]·[📎 파일] 막기, 발행 버튼 `첨부를 올리는 중...`, 다른 파일을 넣으면 `다른 파일을 올리는 중이에요. 끝난 뒤 다시 넣어 주세요` (FR-049).
- 실패한 것은 넣지 않고 `파일 이름: 올리지 못했어요. 다시 시도해 주세요` (FR-047, FR-050 문구 형식).

**Rationale**

- 사용자가 보는 결과(새 첨부, 원래 이름·형식·크기 같음, 올리는 중 표시, 원래 글은 그대로)가 spec과 같다. 브라우저가 최대 30MB를 내려받았다가 다시 올리지 않는다. 원본은 올릴 때 이미 형식·크기·파일 앞부분 검사를 통과했다.
- drop을 붙여 넣는 순간에 하는 것은 FR-054(다른 사이트 사진은 에디터에도 넣지 않음)와 같은 방식이다. 저장 때 빠질 사진이 화면에 보였다가 사라지지 않는다.
  spec은 남의 첨부를 "저장할 때 본문에서 뺀다"고만 적었으므로 저장 때 안전망(R6)은 그대로 둔다.
- 붙여 넣기 판정은 서버만 할 수 있다 (첨부의 주인·붙은 글은 DB에 있다). Server Action은 출처 확인이 내장되어 있다.

**Alternatives considered**: 브라우저가 `/files/키`를 받아 `/api/uploads`로 다시 POST (전송이 두 배, 큰 파일에서 느림), 저장할 때 서버가 몰래 복사 (편집 화면과 저장 결과가 달라진다).

**남는 한계**: 다른 탭의 글에서 사진을 **끌어다 놓기**(붙여 넣기가 아님)는 다시 올리지 않는다. 저장할 때 붙일 수 없는 주소는 안전망이 뺀다 (spec은 붙여 넣기만 적었다).

## R9. 첨부 주소(`GET /files/[key]`)의 권한과 캐시

**Decision**

- 첨부 행과 붙은 글(`posts.visibility`, `blogs.owner_id`)을 한 번에 읽는다 (`src/server/attachments.ts`의 `findReadableAttachment`). 판단은 순수 함수 `attachmentAccess`(`src/lib/attachments.ts`)로:
  - 붙은 글이 공개 → 누구나
  - 붙은 글이 비공개 → 그 블로그 주인만 (관리자 포함 다른 사람은 안 됨, spec 기본값 "관리자도 비공개 글 본문을 볼 수 없음")
  - 어느 글에도 없음 → 올린 사람만. 단 지금 프로필 사진이면 누구나 (R19)
  - 안 되면 없는 첨부와 같은 404 `파일을 찾을 수 없어요` (있는지 숨김, FR-023과 같은 원칙)
- 머리글: `Cache-Control: private, no-cache`, `ETag: "키"`. `If-None-Match`가 같으면 **권한을 확인한 뒤** 304. 나머지 머리글(`Content-Type`, `nosniff`, CSP `sandbox`, `Content-Disposition`)은 그대로.

**Rationale**

- FR-029, FR-059, SC-004. 지금은 누구에게나 `public, max-age=31536000, immutable`로 준다 (코드 확인: `src/app/files/[key]/route.ts`). 공개였던 글을 비공개로 바꾼 뒤에도 공유 캐시가 1년 동안 내줄 수 있다.
- 키가 같으면 내용이 바뀌지 않으므로 ETag 재확인은 파일을 읽지 않는 가벼운 요청이다.

**Alternatives considered**: 서명된 임시 주소 (배포 저장소가 정해지지 않았다, 복잡), 공개 글 첨부만 긴 캐시 (공개 범위를 바꾼 뒤 늦게 반영된다).

## R10. 정리 작업을 어떻게 돌리나

**Decision**

- 핵심 함수 `cleanupPostData()`(`src/server/attachments.ts`)와 스크립트 `scripts/cleanup-posts.ts`, npm `posts:cleanup`(`tsx --conditions=react-server`, `test:sanitize`와 같은 방식으로 `server-only` 모듈을 부른다).
  `.env.local`을 먼저 읽고 DB 모듈을 불러온다 (`src/db/index.ts`가 불러올 때 `DATABASE_URL`을 읽는다).
- 하는 일:
  1. 붙지 않은 지 하루 넘은 첨부 행 `DELETE ... WHERE key IN (SELECT key ... FOR UPDATE SKIP LOCKED) RETURNING key` (프로필 사진 제외, R19, 잠금은 R6) → 저장소 파일 지우기 (없으면 넘어감)
  2. 저장소에 있는데 DB 행이 없는 파일 중 1시간 넘은 것 지우기: 회원 삭제 CASCADE로 행만 지워진 파일(ERD 3.14 "저장소의 파일은 정리 작업이 지운다", auth 요청), 저장 실패로 남은 파일.
     1시간은 `/api/uploads`·다시 올리기(R8)가 파일을 먼저 쓰고 행을 나중에 넣는 사이를 건드리지 않기 위한 여유다.
     그래서 `copyAttachment`는 복사본의 수정 시각을 지금으로 맞춘다 (복사가 원본의 옛 수정 시각을 그대로 옮기는 환경이 있다. 예: macOS `copyfile`. 추측, 구현 때 확인).
  3. 이틀 지난 `post_views` 행 지우기 (R1)
  4. 지운 개수를 한국어로 출력. `--dry-run`이면 세기만 한다.
- `src/server/storage.ts`에 `copyAttachment(from, to)`, `deleteAttachment(key)`, `listStoredFiles()`를 더한다. 배포 저장소가 바뀌어도 이 파일만 바꾼다 (지금 파일 주석의 원칙).
- 스크립트가 Next.js 밖에서 불러오므로 `cleanupPostData()`가 있는 `src/server/attachments.ts`는 `next/*`, `src/server/dal.ts`, `src/lib/auth.ts`를 import하지 않는다 (보는 사람은 Route Handler·Server Action이 구해서 인자로 넘긴다).
  `src/db/index.ts`와 `src/server/storage.ts`가 불러올 때 `DATABASE_URL`·`UPLOAD_DIR`을 읽으므로, 스크립트는 `dotenv`로 `.env.local`을 읽은 **뒤** 이 모듈을 동적 import한다.
- 개발에서는 손으로 돌리고, e2e(`e2e/attachment-links.mjs`)가 직접 실행해 SC-011을 확인한다. **배포 환경에서 하루 1번 예약 실행하는 방법은 NF-08(배포 서비스)과 함께 정한다** (남은 문제).

**Rationale**: 배포 서비스가 정해지지 않아(NF-08 ⬜) cron 기반을 고를 수 없다. 스크립트는 어떤 예약 방식에도 붙일 수 있고 테스트에서 바로 부를 수 있다.

**Alternatives considered**: 업로드 요청 안에서 그 회원의 오래된 첨부 지우기 (안 올리는 회원·탈퇴 회원의 파일은 남는다), Next.js `after()` (서버리스에서 실행 보장이 없다), Route Handler + 외부 cron (배포 결정 전이라 미룸. 정해지면 같은 함수를 부르면 된다).

## R11. 본문 200,000자 검사보다 요청 크기 상한이 먼저 걸리지 않게

**사실**

- Server Action 요청 본문의 기본 상한은 1MB이고 `next.config.ts`에 `serverActions` 설정이 없다 (코드 확인: `next.config.ts`. 1MB는 `src/app/api/uploads/route.ts` 주석과 같은 이해. 추측: Next 16 기본값을 `node_modules/next/dist/docs`의 `serverActions.bodySizeLimit`에서 확인).
- zod의 `.max(200_000)`은 UTF-16 단위로 센다. 한 단위는 UTF-8로 최대 3바이트(BMP 글자 3바이트, 이모지는 2단위 4바이트)라, 200,000자를 조금 넘는 본문도 약 600KB로 상한 아래다.
  즉 지금 설정으로도 "조금 넘는" 본문은 서버 검사에 닿아 `글이 너무 길어요`가 나온다.
- 문제는 화면에서 아주 큰 내용(예: 40만 자 한글)을 붙여 넣고 발행하는 경우다. 1MB를 넘으면 서버 함수가 불리기 전에 막혀 오류 화면이 나온다.

**Decision**

- `next.config.ts`는 바꾸지 않는다.
- 서버 검사 스키마를 `src/lib/post-rules.ts`(`postInputSchema`)로 옮겨 서버와 브라우저가 같이 쓴다. `PostForm`은 본문 HTML의 UTF-8 크기가 900,000바이트를 넘으면(= 반드시 300,000자 이상)
  요청을 보내지 않고 브라우저에서 같은 스키마를 돌려 **서버와 같은 순서의 첫 오류**(`제목을 적어 주세요` / `제목은 100자까지예요` / `글이 너무 길어요` …)를 같은 자리에 보인다. 입력값은 그대로다.

**Rationale**: spec Assumptions "요청 크기 상한에 걸려 이 문구가 나오지 않는 일이 없게". 서버 검사는 그대로라 조작 요청은 서버가 막는다 (constitution IV). 문구와 순서를 한 곳에서 관리한다.

**Alternatives considered**: `bodySizeLimit`을 키우기 (상한은 어딘가 남고 큰 요청을 받는 폭만 늘어난다), 본문을 Route Handler로 보내기 (`useActionState`와 Server Action의 출처 확인을 버려야 한다).

## R12. 조작된 형식 값에도 한국어 문구

**사실 (코드 확인)**: 지금 스키마의 `postId`·`categoryId`는 `z.coerce.number().int().positive()`, `visibility`는 `z.enum([...])`이고 문구 인자가 없다.
`abc`·`0`·`1.5`·`x`를 보내면 zod 기본 문구(영어)가 `issues[0].message`로 화면에 나온다. 또 `coerce`는 `1e1`을 10으로 받는다.

**Decision**: `postId`·`categoryId`·`subcategoryId`는 문자열로 받아 빈 값이면 없음, 아니면 `parseId`(`src/lib/ids.ts`, 숫자만 적힌 1~2147483647)를 통과해야 하고 실패하면 `잘못된 요청이에요`.
`visibility`도 `public`/`private`가 아니면 `잘못된 요청이에요`. 검사 순서는 [contracts/write-actions.md](contracts/write-actions.md)에 적는다.

**Rationale**: spec Edge Cases·Assumptions(POST-02·POST-03 "영어 기본 문구 대신 `잘못된 요청이에요`"), FR-018, NF-19, constitution III. `parseId` 규칙은 주소 숫자 검사(#21)와 같다.

## R13. 발행 안내를 주인에게 한 번만

**Decision**

- `savePost`는 새 글이면 `/@주소/글ID?new=1`로 보낸다 (보상 여부를 주소에 담지 않는다). 수정은 지금처럼 쿼리 없음.
- 글 상세는 `?new`가 있고, 보는 사람이 주인이고, 글이 만들어진 지 10분 안일 때만 안내를 그린다. 보상 여부는 원장에서 판단한다:
  `point_ledger`에 (주인, `reason = 'post'`, `ref_id` = 글 ID) 행이 있으면 받음 (`grantReward(tx, userId, "post", id)`가 `ref_id = String(id)`로 남긴다, 코드 확인).
- 안내는 새 클라이언트 컴포넌트 `PublishNotice`(`src/components/blog/publish-notice.tsx`)가 그리고, 처음 그린 뒤 `window.history.replaceState`로 주소에서 `?new`를 지운다.
  새로고침해도 다시 보이지 않는다. 같은 컴포넌트가 임시 글을 지운다 (R14).

**Rationale**: FR-013 기본값 "글 주인에게, 발행 직후 한 번만. 새로고침하거나 다른 사람이 같은 주소를 열어도 다시 보이지 않는다". 지금은 주소의 `?new=reward`만 보고 보는 사람을 확인하지 않아,
누가 열어도 보이고 주소를 고치면 사실과 다른 안내가 나온다 (코드 확인).

**Alternatives considered**: 한 번 쓰는 쿠키 (Server Component에서 지울 수 없고, Server Action으로 지우면 화면을 다시 그려 안내가 곧바로 사라진다), `sessionStorage` 표시 (새로고침 때 잠깐 보였다 사라진다).

**추측·확인할 것**: Next.js 16 App Router가 `window.history.replaceState`를 라우터 상태와 맞춰 준다는 점을 `node_modules/next/dist/docs`의 linking and navigating 문서에서 확인한다.

**영향**: `e2e/write-count.mjs`(33행)와 `e2e/farm.mjs`(94행)가 주소의 `new=reward`로 보상을 판단한다. 안내 문구(`✨ 경험치 30 · 🪙 30 코인을 받았어요`)로 판단하도록 같은 작업에서 고친다.
`e2e/write-count.mjs` 34행은 글 ID를 `/(\d+)\?/`로 읽어 주소에 `?`가 있어야 한다. `?new`가 지워지면 실패하므로 `new URL(page.url()).pathname`에서 읽게 고친다 (`e2e/attachments.mjs` 99행 방식). `e2e/blog.mjs` 51행(`replace("?new=1", "")`)은 그대로 동작한다.

## R14. 임시 저장 (POST-08)

**Decision**: 원본 `docs/01-requirements.md` POST-08의 구현 제안을 따른다.

- 저장 위치: `localStorage`, 키 `blogville:draft:{userId}`, 값 JSON `{ title, categoryId, subcategoryId, visibility, contentHtml, tags, savedAt }`. 읽기·쓰기·지우기는 새 모듈 `src/lib/draft.ts`가 `try/catch`로 감싼다 (FR-064).
- 저장: 새 글 화면에서만. 제목·카테고리·공개 설정·본문·태그 중 하나가 바뀔 때마다 타이머를 다시 걸어 2초 뒤 저장 (debounce). 제목(앞뒤 공백 제외)과 본문 글자가 모두 비면 저장하지 않는다.
  저장하면 아래 상자에 `임시 저장됨 14:05` (`Intl.DateTimeFormat("ko-KR", { timeZone: "Asia/Seoul", hour: "2-digit", minute: "2-digit", hour12: false })`), 처음에는 `작성 중인 글은 이 브라우저에 자동 저장됩니다.`.
- 불러오기: 화면이 열린 뒤(`useEffect`) 한 번, `window.confirm("작성 중이던 글이 있어요. 불러올까요?")`. 확인이면 저장된 값을 초기값으로 폼을 `key`를 바꿔 다시 그린다
  (`RichEditor`는 `initialHtml`을 에디터를 만들 때 한 번만 쓴다, 코드 확인). 지워진 대분류·소분류는 `카테고리 없음`/대분류만으로 채운다. 개발 모드의 effect 두 번 실행에 대비해 ref로 한 번만 묻는다.
- 발행: [발행하기]를 누르면 남은 예약을 바로 저장하고 그 뒤 저장을 멈춘다. 오류로 돌아오면 다시 시작한다 (FR-063 "발행이 실패하면 지우지 않는다").
  성공하면 글 상세의 `PublishNotice`가 지운다 (R13). `PostForm`이 사라질 때 타이머를 취소해, 남은 예약이 지운 임시 글을 다시 만들지 않는다.
- 로그아웃: `SignOutButton`(`src/components/sign-out-button.tsx`, auth 소유)에 선택 prop `userId`를 더해 로그아웃 전에 그 회원의 임시 글을 지운다.
  `site-header.tsx`(town 소유)가 `viewer.userId`를 넘긴다. 둘 다 "추가만"이다 (공통 모듈 소유 규칙).
  지금 버튼은 브라우저에서 `authClient.signOut()`을 부르지만(코드 확인), auth 단계 1이 Server Action 폼(`<form action={signOut}>`, auth contracts `auth-entry.md` 5절)으로 바꾼다. 그 뒤에는 폼의 `onSubmit`(브라우저, 요청을 보내기 전)에서 지운다.
  그래서 버튼은 브라우저 컴포넌트로 남아야 한다 (auth와 합의). React 19에서 `action`과 `onSubmit`을 함께 쓸 때 `onSubmit`이 먼저 도는지는 구현 전에 확인한다 (추측).
- 아래 상자의 글자 수 `N`은 `toLocaleString("ko-KR")`로 고정한다 (지금은 브라우저 언어를 따라 `1.234`가 될 수 있다, FR-010).

**Rationale**: FR-060~064, spec 기본값(브라우저에만, DB 변경 없음, 수정 화면은 없음, 로그아웃 때 지움). 로그인 유지 2시간 결정으로 가치가 커졌다.

**Alternatives considered**: 서버 저장 (Out of Scope), `sessionStorage` (탭을 닫으면 사라진다), 로그아웃 때 모든 회원의 임시 글 지우기 (다른 회원이 로그인 만료 뒤 되찾을 임시 글까지 지운다. spec은 "그 회원의 임시 글").

**한계**: 하루 넘게 둔 임시 글의 사진·파일은 붙지 않은 첨부라 정리 작업(R10)이 지울 수 있다. 불러온 뒤 발행하면 없는 주소는 저장 때 빠진다.

## R15. 대분류·소분류 선택과 서버 확인

**Decision**

- 글쓰기·수정 화면은 새 함수 `getCategoryOptions(blogId)`(`src/server/posts.ts`)로 대분류와 그 아래 소분류를 받는다. 순서는 블로그 관리와 같이 `position`, `id`.
- 화면: [대분류 ▼](`카테고리 없음` + 대분류) [소분류 ▼](고른 대분류의 소분류, 맨 위 `소분류 없음`). 대분류를 바꾸면 소분류는 풀린다. 대분류가 없으면 소분류 칸은 비활성.
  `소분류 없음`은 "고르지 않음"을 나타낼 첫 항목이 필요해 대분류의 `카테고리 없음`을 본떠 둔 문구인데 spec에는 없다 (FR-065). 팀 확인 전까지 이 문구로 두고 plan 남은 문제 8로 올린다.
- 서버 저장 규칙 (트랜잭션 안, 대분류·소분류 행은 `FOR KEY SHARE`로 읽어 저장 직전에 지워져 FK 오류(500)가 나지 않게):

  | 보낸 값 | 결과 |
  |---|---|
  | 대분류 없음 | 둘 다 NULL (소분류 값은 무시) |
  | 대분류가 내 블로그 것이 아니거나 없음 (쓰는 사이 지워짐 포함) | 둘 다 NULL, 오류 없음 |
  | 대분류 OK, 소분류 없음 | 대분류만 |
  | 대분류 OK, 소분류가 그 대분류 소속 | 둘 다 저장 |
  | 대분류 OK, 소분류가 **다른 대분류**(다른 블로그 포함)에 속해 존재 | `잘못된 요청이에요`, 저장 안 함, 입력값 유지 |
  | 대분류 OK, 소분류 번호의 행이 없음 (쓰는 사이 지워짐) | 대분류만, 오류 없음 |

**Rationale**: FR-030~032, US5-5·US5-6, FR-062("불러온 카테고리가 지워졌으면 카테고리 없음")와 같은 너그러움. 마지막 줄(지워진 소분류)은 spec에 직접 적혀 있지 않아
대분류 규칙(FR-032 "쓰는 사이 지워진 경우 포함 → 오류 없이")에 맞췄다 (남은 문제에 낮은 우선으로 적음).

**Alternatives considered**: blog가 만들 트리 함수를 같이 쓰기 (블로그 홈용은 글 수를 함께 세어 모양이 다르다), 지워진 소분류도 오류 (쓰는 사이 다른 탭에서 지운 경우 사용자가 원인을 알 수 없다).

## R16. 카테고리 배지와 목록

**Decision**

- 배지 글자: 대분류만이면 `대분류`, 소분류가 있으면 `대분류 › 소분류` (spec Assumptions POST-03). 노랑, `🔒 비공개` 앞 (지금 자리 그대로).
- `src/server/blog.ts`의 `listColumns`·`baseList`·`getPost`에 `subcategories` LEFT JOIN으로 `subcategoryId`·`subcategoryName`을 더한다 (post 소유 함수).
- 글 상세 배지 링크: 소분류가 있으면 `/@주소?category=대분류ID&sub=소분류ID`, 없으면 `/@주소?category=대분류ID`. 블로그 홈 주소 규칙은 blog research R-13이 정했다 (`sub`가 올바른 번호면 소분류로 거르고 `category`는 무시).
  두 값을 함께 넣으면 blog 단계 4(블로그 홈 트리·거르기) 전에는 지금 블로그 홈이 `sub`를 모른 채 대분류로 거르고, 단계 4 뒤에는 소분류로 거른다. 그래서 post 소유 상세 화면을 단계 4 때 다시 고치지 않아도 된다. 목록 카드의 배지는 따로 눌리지 않는다 (FR-033, 지금 그대로).
- `listBlogPosts`에 선택 인자 `subcategoryId`를 더한다 (`WHERE subcategory_id = ?`). 대분류로 거를 때는 지금처럼 `category_id = ?`라 소분류 글도 포함된다 (Assumptions POST-03, ERD 3.18). blog 단계 4가 이 인자를 쓴다.

**Rationale**: 배지는 post(FR-033), 블로그 홈 트리·거르기 화면은 blog(BLOG-05) 범위다. 함수 소유는 post라 blog가 바로 쓸 수 있게 인자를 먼저 열어 둔다.

## R17. 정화와 글자 수

**Decision**: 허용 태그·속성·글자 수 규칙은 바꾸지 않는다. `sanitizePostHtml(html, known)`에 넘기는 `known`을 R6의 "붙일 수 있는 첨부"로 좁히는 것만 바뀐다 (`src/server/sanitize.ts` 코드 변경 없음).

**Rationale**: 지금 코드가 FR-007·FR-011·FR-054·FR-055·SC-003·SC-005를 이미 만족한다 (코드 확인: `scripts/test-sanitize.ts`의 XSS 6가지, `scripts/test-text-length.ts` 24경우, `e2e/write-count.mjs`).
허용 태그의 `h1`(에디터는 H2·H3만)은 원본 문서 정리 항목이라 이 plan에서 바꾸지 않는다.

## R18. 기존 데이터 이전 (백필)

**Decision**

- `attachments.post_id`: 첨부마다 **그 첨부를 올린 회원의 블로그 글 중 본문에 `/files/키`가 든 가장 먼저 쓴 글**(`created_at`, `id` 순)로 채운다. 직접 쓴 데이터 SQL 마이그레이션으로 남긴다 (`drizzle/0002_give_all_starters.sql`처럼 다시 실행해도 같은 결과: `post_id IS NULL`인 행만 고친다).
- 남는 참조(같은 키가 다른 글 본문에도 있음, 남의 첨부가 본문에 있음)는 고치지 않고 확인 쿼리로 개수만 본다. 그 글은 다음에 저장할 때 안전망(R6)이 주소를 뺀다.
- `detached_at`은 모두 NULL로 시작한다 (붙지 않은 첨부는 `created_at` 기준으로 정리된다).
- `posts.subcategory_id`는 NULL로 시작한다 (ERD 7장 6-3 "기존 글은 대분류만"). `post_views`는 빈 표, `view_count`는 지금 숫자 그대로 (표 도입 전 조회 포함).

**Rationale**: 배포 전(NF-08 ⬜)이라 기존 데이터는 개발 데이터뿐이다 (코드 확인: README "배포할 때는…", 요구사항 NF-08). 파일 복사가 필요한 중복 참조를 SQL만으로 풀 수 없고, 개발 데이터에서 그 비용을 들일 이유가 없다.

**Alternatives considered**: 중복 참조마다 파일을 복사해 새 첨부를 만드는 일회성 스크립트 (배포 데이터가 생긴 뒤라면 필요하지만 지금은 과하다).

## R19. 프로필 사진 예외 (auth의 `profiles.photo_key`)

**Decision**: "지금 프로필 사진으로 쓰는 첨부"는 `profiles.photo_key = attachments.key`인 행이 있는가 하나의 조건이다. post의 세 곳이 이 조건을 쓴다:
저장할 때 본문에서 빼기(R6), `/files` 누구나 보기(R9), 정리 작업 제외(R10). 붙여 넣기에서는 reupload(R8).
`profiles.photo_key`는 auth가 추가한다 (ERD 7장 3, 테이블 담당 규칙). post가 먼저 구현되면 이 조건 없이 내보내고, auth의 마이그레이션과 같은 PR(또는 바로 뒤)에서 조건을 켠다.
올리는 화면이 아직 어느 spec에도 없어(plan-context 5.3) 그 사이 영향받는 데이터가 없다.

## R20. 테스트 방식

**Decision**

- 순수 규칙은 `src/lib`에 두고 단위 테스트한다 (저장소 관례, `scripts/test-*.ts`): `src/lib/post-rules.ts`(`postInputSchema` 한국어 문구·순서, `parseTags` FR-041, 본문 크기 판단),
  `src/lib/attachments.ts`에 더할 `attachmentAccess`(R9 표)·`pasteAction`(R8 판정). → `scripts/test-post.ts`, npm `test:post`, `test` 체인 끝에 붙인다.
- 흐름은 새 e2e 6개(실행마다 새 회원, `check()` + `process.exit(1)`, pg로 준비·확인): `e2e/post-write.mjs`, `e2e/post-lists.mjs`, `e2e/post-categories.mjs`, `e2e/post-views.mjs`, `e2e/attachment-links.mjs`, `e2e/post-drafts.mjs`.
- "다음 날", "하루 지난 첨부"는 pg로 `post_views.date`·`attachments.created_at`·`detached_at`을 하루 전으로 바꿔 흉내 낸다 (`e2e/visits.mjs`의 "어제" 방식).
- 소분류는 blog의 관리 화면이 없을 수 있으므로 e2e에서 pg로 `subcategories` 행을 직접 넣는다.
- 바뀌는 기존 e2e: `e2e/blog.mjs`·`e2e/params.mjs`(카테고리 칸 이름 `카테고리` → `대분류`. Playwright `getByLabel`은 부분 일치라 이름에 `카테고리`가 없으면 찾지 못한다), `e2e/write-count.mjs`·`e2e/farm.mjs`(R13).
- `e2e/nonfunctional.mjs`(공통 모듈 추가): NF-07 측정 경로에 `/tags/{태그}`를 더한다. 지금은 `/feed`와 `/@normal01`만 재서 SC-002의 태그별 글 목록이 빠진다.

**Rationale**: constitution III(수용 기준 = 시험 시나리오), 저장소 관례(plan-context 6장). CI가 없어 PR 작성자가 직접 돌리고 결과를 PR에 적는다.

## R21. 누르는 영역 44×44px (constitution VI)

**사실 (코드 확인)**: post 소유 화면의 버튼·링크가 44×44px보다 작다.

| 곳 | 파일 | 지금 |
|---|---|---|
| 페이지 번호 | `src/components/pagination.tsx` | `min-w-9 px-2.5 py-1` (약 36×32px) |
| 에디터 도구·[🖼 사진]·[📎 파일]·안내 [✕] | `src/components/editor/rich-editor.tsx` | `px-2.5 py-1 text-sm`, [✕]는 글자만 |
| 공개 토글 | `src/components/editor/post-form.tsx` | `px-3 py-1.5 text-sm` |
| 상세 카테고리 배지·`#태그`·[수정]·[삭제] | `src/app/blog/[slug]/[postId]/page.tsx`, `src/components/blog/delete-post-button.tsx` | `py-0.5 text-xs` / `py-1 text-sm` / 글자만 |
| 마을 소식 탭·인기 태그 칩 | `src/components/blog/feed-view.tsx` | `btn py-1.5 text-sm` / `py-1 text-sm` |
| 카드 작성자 줄 | `src/components/blog/post-card.tsx` | 캐릭터 26px 줄 |

**Decision**: 모두 `min-h-11`(필요하면 `min-w-11`, `inline-flex items-center`)로 키우고 글자는 `whitespace-nowrap`. 모양(색·둥근 정도)과 문구·동작은 그대로다.
에디터 도구 모음은 375px에서 여러 줄로 감긴다 (가로 스크롤 없음). `e2e/post-lists.mjs`가 375px에서 이 요소들의 `boundingBox()`가 44×44px 이상인지 잰다.

**Rationale**: constitution VI("버튼과 링크의 누르는 영역은 최소 44×44px"), FR-066. 파일 소유가 post라 blog(페이지 번호, blog FR-059)와 social(D-10: 탭·인기 태그·페이지 번호)이 post에 요청했다.

**Alternatives considered**: 보이는 크기는 두고 투명한 여백(`::before`)만 키우기 — 이웃한 작은 칩끼리 누르는 영역이 겹친다. 에디터 도구를 "더 보기" 메뉴로 접기 — 새 화면 요소와 문구가 생긴다.
