---

description: "교류 (SOCIAL) 구현 작업 목록"
---

# Tasks: 교류 (SOCIAL) — 댓글·답글·공감·이웃·마을 소식

**Input**: Design documents from `/specs/004-social/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: plan.md(U14)·research.md(R18)·quickstart.md가 단위 테스트(`scripts/test-social.ts`)와 E2E(`e2e/comments.mjs`·`e2e/social.mjs`·`e2e/params.mjs`·`e2e/feed-load.mjs`)를 이 기능의 산출물로 정했으므로 검증 작업을 포함한다. 이 저장소는 테스트 프레임워크 없이 `tsx scripts/test-*.ts`와 Playwright Node 스크립트를 쓴다.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- 경로는 모두 코드 저장소 `ehgo508/blogville` 루트 기준이다 (Next.js 단일 프로젝트: `src/`, `drizzle/`, `scripts/`, `e2e/`).
- 소유 규칙(plan-context 5.2): `src/server/blog.ts`의 `paged`·`baseList`, `src/app/blog/[slug]/[postId]/page.tsx`는 post 소유, `src/components/blog/blog-header.tsx`는 blog 소유, `src/app/admin/page.tsx`는 auth 소유다. social은 표시된 함수·영역만 고친다.
- 마이그레이션 번호는 박지 않는다(`NNNN_`). 다른 브랜치가 먼저 merge되면 최신 `main`에서 `npm run db:generate`를 다시 돌린다 (data-model 6.1).

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 선행 조건 확인과 라이브러리 동작 확인 (새 패키지 없음)

- [ ] T001 선행 작업 확인: auth 단계 1(가입 통합)이 `main`에 merge되어 `e2e/helpers.mjs`의 `loginDev`가 온보딩 없이 동작하는지 확인하고, 최신 `main`에서 `004-social` 브랜치를 만든다 (plan 의존성 D-1)
- [ ] T002 `npm install` 뒤 research.md R19 표의 항목을 설치된 패키지 문서로 확인하고 결과를 PR 설명 초안에 적는다: Drizzle `select().for("key share" / "share")`, `insert().onConflictDoNothing().returning()`(충돌이면 빈 배열), select 빌더 `orderBy` 두 번 호출 가능 여부, `drizzle-kit migrate`의 트랜잭션 범위, `drizzle-kit generate --custom --name=...`, React 19 `<textarea defaultValue>` 폼 초기화, Next.js 16 Server Action `redirect()`·동적 렌더링 (`node_modules/next/dist/docs/`, `node_modules/drizzle-orm/`)
- [ ] T003 [P] 지금 DB에 답글이 있는 상태를 만들어 이전 전 값을 기록한다: `main`에서 `node e2e/blog.mjs shots` 실행 → `tester2`로 답글 2개 작성·1개 삭제·원댓글 1개 삭제 → `SELECT count(*) FROM comments WHERE parent_id IS NOT NULL`(A), `... WHERE parent_id IS NULL`(B), 깊이 2 이상 답글 수, 다른 글 부모 답글 수, `point_ledger` 행 수를 PR에 적는다 (quickstart 1절, data-model 6.2-1)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 답글 표 분리(U1~U3), 공용 규칙 모듈, 댓글 수 식. `comments.parent_id`가 사라지면 기존 댓글 조회·글 카드 댓글 수가 모두 영향을 받으므로 모든 스토리보다 먼저 끝낸다.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### 마이그레이션 (순서 고정: T004 → T005 → T006 → T007)

- [ ] T004 마이그레이션 1(구조, U1): `src/db/schema.ts`에 `replies` 블록을 추가한다 — `id` integer identity PK, `comment_id` integer NOT NULL FK → `comments.id` `ON DELETE CASCADE`, `author_id` text NOT NULL FK → `users.id` `ON DELETE CASCADE`, `content` text NOT NULL, `created_at` timestamptz NOT NULL 기본 now(), `deleted_at` timestamptz NULL, CHECK `replies_content_check` `(deleted_at IS NULL AND char_length(content) BETWEEN 1 AND 1000) OR (deleted_at IS NOT NULL AND content = '')`, INDEX `replies_comment_created_idx (comment_id, created_at)`. 같은 단계에서 `comments.content`의 기존 CHECK를 임시로 지우고 `comments.author_id`를 **text NULL** + FK `ON DELETE SET NULL`로 바꾼다(`parent_id`는 아직 남긴다). `npm run db:generate`로 `drizzle/NNNN_<replies 만들기>.sql` 생성, 맨 위에 `-- SOC-02 (2026-10-07): ...` 한국어 주석, 생성 SQL에서 `DROP NOT NULL`·FK 재생성을 확인 (data-model 6.1-1, 2.1·2.2)
- [ ] T005 마이그레이션 2(데이터, U2): `npx drizzle-kit generate --custom --name=<답글 이전>`으로 빈 파일 `drizzle/NNNN_<답글 이전>.sql`을 만들고 data-model 6.2 백필을 직접 쓴다 — (1) 이전 전 확인 쿼리 주석 (2) 재귀 CTE로 `parent_id IS NULL`인 첫 조상을 원댓글로 찾기 (3) `INSERT INTO replies (id, comment_id, author_id, content, created_at, deleted_at) OVERRIDING SYSTEM VALUE SELECT ... ON CONFLICT (id) DO NOTHING`(삭제된 행은 `content = ''`, 원댓글 `post_id`가 다른 행은 옮기지 않음) (4) `setval(pg_get_serial_sequence('replies', 'id'), ...)`로 `MAX(id) + 1` (5) 옮기지 않은 답글 `parent_id = NULL` 후 옮긴 `id`를 `comments`에서 삭제 (6) `UPDATE comments SET content = '' WHERE deleted_at IS NOT NULL` (7) 이전 후 확인 쿼리 주석. `point_ledger`는 건드리지 않는다
- [ ] T006 마이그레이션 3(구조, U3): `src/db/schema.ts`에서 `comments.parent_id`·`comments_parent_fk`를 지우고 CHECK 2개를 추가한다 — `comments_content_check` `(deleted_at IS NULL AND char_length(content) BETWEEN 1 AND 1000) OR (deleted_at IS NOT NULL AND content = '')`, `comments_author_check` `author_id IS NOT NULL OR deleted_at IS NOT NULL`. `comments_post_created_idx (post_id, created_at)`는 그대로. `npm run db:generate`로 `drizzle/NNNN_<comments 정리>.sql` 생성 + 한국어 주석 (data-model 6.1-3, FR-018)
- [ ] T007 `npm run db:migrate`로 T004~T006을 적용하고 quickstart 1절 4번 표를 확인한다: `replies` 행 수 = A, `comments` 행 수 = B, `comments.parent_id` 컬럼 없음, 두 표에서 `deleted_at IS NOT NULL AND content <> ''` 0, 옮긴 답글 `id`가 이전 전 댓글 `id`와 같음, `point_ledger` 행 수 그대로. 결과와 `shots/00-migrated.png`를 PR에 붙인다 (T003 값과 비교)

### 공용 규칙·서버 모듈

- [ ] T008 [P] `src/lib/social.ts`(새, DB 없는 순수 함수) 작성: 내용 정규화(`\r\n`·`\r` → `\n`, 앞뒤 공백 제거)와 검증(1~1000자, 실패 문구 `댓글을 적어 주세요` / `댓글은 1000자까지예요`), 삭제 권한 `canDeleteComment({ viewerId, isAdmin, authorId, blogOwnerId })`(작성자 OR 블로그 주인 OR 관리자), 오류 문구 상수(`잘못된 요청이에요`, `글을 찾을 수 없어요`, `답글을 달 댓글이 없어요`, `삭제된 댓글에는 답글을 달 수 없어요`) (FR-007, FR-008, FR-013, FR-018, research R6·R8)
- [ ] T009 [P] `scripts/test-social.ts`(새) 작성: `"  안녕 \r\n반가워  "` → `"안녕 \n반가워"`, 공백만·빈 값 → `댓글을 적어 주세요`, 1000자 통과 / 1001자 → `댓글은 1000자까지예요`, CRLF 10개 포함 1000자(LF 기준) 통과, `canDeleteComment` 작성자/블로그 주인/관리자/남/방문자 = true/true/true/false/false. `package.json`에 `"test:social": "tsx scripts/test-social.ts"`를 추가하고 `test` 체인 끝에 붙인다 (quickstart 2절)
- [ ] T010 `src/server/social.ts`(새, `import "server-only"`)에 `liveCommentCountSql(postId: AnyPgColumn | SQL): SQL<number>`(삭제 안 된 댓글 수 + 삭제 안 된 답글 수)와 `countLiveComments(): Promise<number>`(마을 전체 합계)를 만든다 (FR-016, contracts/comments.md 4절, research R12)
- [ ] T011 `src/server/blog.ts`의 `listColumns.commentCount` 식을 `liveCommentCountSql(posts.id)`로 바꾼다 — 글 카드 `💬 N`이 댓글 + 답글이 된다 (FR-016, FR-032, U10 일부, 의존 T010)
- [ ] T012 [P] `src/server/db-errors.ts`에 `foreignKeyViolation(err)`(SQLSTATE 23503 판별) 한 함수만 추가한다. 기존 `uniqueViolation`은 바꾸지 않는다 (공통 모듈 추가, research R10)
- [ ] T013 `src/app/admin/page.tsx`의 댓글 통계를 `countLiveComments()`로 바꾼다 — auth 소유 파일이므로 같은 PR에서 auth 리뷰어 승인을 받는다(요청 D-4). U3 뒤 `comments`만 세면 답글이 빠지므로 이 단계에서 함께 처리한다 (FR-016, 의존 T010)

**Checkpoint**: 답글이 `replies`로 분리되고, 댓글 수 식이 한 곳에 있다 — 스토리 작업 시작 가능

---

## Phase 3: User Story 1 - 마을 소식에서 모든 블로그의 최신 공개 글 둘러보기 (Priority: P1) 🎯 MVP

**Goal**: 방문자를 포함한 누구나 `/feed`에서 모든 블로그의 공개 글을 최신순 8개씩 보고, 카드에서 블로그 홈·글 상세로, 인기 태그로 태그 목록으로 이동한다. 화면은 이미 있으므로(post 소유 `feed-view.tsx`) social은 검증과 카드 댓글 수 정의, 44px 요청을 맡는다.

**Independent Test**: 여러 블로그에 공개·비공개 글을 섞어 만든 뒤, 로그인하지 않고 `/feed`를 열어 공개 글만 최신순 8개씩 보이는지, 카드에서 블로그 홈·글 상세로 이동되는지 확인한다.

### Tests for User Story 1

- [ ] T014 [P] [US1] `e2e/social.mjs`에 마을 소식 항목 추가: 방문자 `/feed`([🏘 마을 전체]만, [💛 이웃 새 글] 없음, 탭 제목 `마을 소식 | Blogville`), 새 회원 F의 공개 글 9개 + 비공개 1개를 방문자·F·관리자로 열어 1페이지 8개 모두 공개·최신순(`created_at DESC, id DESC`)·9번째는 2페이지·비공개 배지 없음, 카드 작성자 줄 → 블로그 홈 / 나머지 → 글 상세, `/feed?page=999` 빈 목록 문구 `아직 마을에 글이 없어요. 첫 글의 주인공이 되어 보세요! ✏️`·현재 페이지 표시 없음, 인기 태그 `#태그 N` 횟수 많은 순·같으면 이름순·최대 30개·`/tags/...` 이동, 카드 클릭 1번에 글 상세(SC-008) (US1-1~3·5·6, FR-046~051)
- [ ] T015 [P] [US1] `e2e/params.mjs`에 `/feed?page=abc`, `0`, `2.5`, `1e300`, `99999999999999999999` → HTTP 200·1페이지 항목 추가 (US1-7, FR-052, SC-007)
- [ ] T016 [P] [US1] `e2e/feed-load.mjs`(새) 작성: 공개 글 1,000개를 SQL로 넣고 캐시 없는 새 브라우저 컨텍스트에서 `/feed` 첫 화면 `load`까지 시간을 재고(기대 1초 안), 같은 데이터로 `/feed/following`(이웃 50명 중 즐겨찾기 10명)도 잰 뒤 넣은 데이터를 지운다. 인자 `<폴더> [주소]` (SC-001, NF-07, quickstart 5절)

### Implementation for User Story 1

- [ ] T017 [US1] `src/app/feed/page.tsx`·`src/components/blog/feed-view.tsx`(post 소유, 읽기만)가 spec 문구 그대로인지 대조한다: 제목 `📋 마을 소식`, 빈 문구 `아직 마을에 글이 없어요. 첫 글의 주인공이 되어 보세요! ✏️`, `아직 태그가 없어요`, 작성자 줄(캐릭터 · 닉네임 · 블로그 이름, 길면 `…`), 처음 쓴 날짜(한국 시간), 768px 미만에서 인기 태그 칸이 목록 아래, `getViewer()` 사용으로 요청마다 동적 렌더링(FR-054, research R16). 다른 점은 post에 요청으로 정리한다 (FR-046~053)
- [ ] T018 [US1] post에 요청 D-10을 보낸다: `src/components/blog/feed-view.tsx`의 탭 [🏘 마을 전체] [💛 이웃 새 글](`btn py-1.5 text-sm`)·인기 태그 칩(`py-1 text-sm`), `src/components/pagination.tsx`의 페이지 번호 링크(`py-1`) 누르는 영역을 44×44px로. `e2e/social.mjs`의 375px 탭·칩·페이지 번호 44px 항목은 반영 전까지 "D-10 대기"로 출력하게 만든다 (constitution VI, FR-004)
- [ ] T019 [US1] `e2e/social.mjs`에 375px `/feed` 항목 추가: `scrollWidth <= clientWidth`, 인기 태그 칸이 목록 아래 (US1-8, FR-053, SC-002)

**Checkpoint**: 로그인 없이 마을 소식을 둘러볼 수 있고 카드 `💬 N`이 댓글 + 답글 기준이다

---

## Phase 4: User Story 2 - 글에 댓글을 쓰고 지우기 (Priority: P1)

**Goal**: 회원은 볼 수 있는 글에 댓글을 쓰고(남의 글이면 ✨ 5 · 🪙 5, 답글과 합쳐 하루 10번), 작성자·블로그 주인·관리자는 댓글을 지운다. 지운 자리에는 `삭제된 댓글이에요`만 남고 원래 내용은 DB에서도 지운다. 방문자는 읽기만 한다.

**Independent Test**: 회원 A로 회원 B의 공개 글에 댓글을 쓰고 목록·댓글 수·보상을 확인한 뒤 지워 자리 표시와 댓글 수 감소를 확인한다. A의 다른 댓글을 블로그 주인 B로 지워 같은 자리 표시가 남는지, 방문자에게 입력칸 대신 로그인 안내가 보이는지 확인한다.

### Tests for User Story 2

- [ ] T020 [P] [US2] `e2e/comments.mjs`(새, 실행마다 새 회원 A·B·C + 관리자, `dotenv` + `pg` Pool, `check()` 모음, ❌면 `exit(1)`) 댓글 부분 작성: quickstart 4.1 표의 댓글 행 — 등록(새로고침 없이 맨 아래·입력칸 빔·`💬 댓글 N` +1·B 코인 +5), 처리 중 `등록 중...` 비활성, 내 글 댓글 보상 없음, 내 비공개 글 Q 댓글 등록, 로그인 풀린 뒤 등록·공감·[답글 등록]·[삭제] → `/` 이동·행 수 그대로, 공백 → `댓글을 적어 주세요`(내용 유지), 1001자 → `댓글은 1000자까지예요`, `postId`를 남의 비공개 글·없는 글로 → `글을 찾을 수 없어요`, 방문자 `로그인하면 댓글을 남길 수 있어요`(링크 `/`)·[답글]·[삭제] 없음, 삭제 [취소]/[확인], HTML·RSC 응답에 지운 내용·작성자 회원 ID 0건, 블로그 주인·관리자 삭제, C에게 [삭제]는 C 댓글에만, 권한 없는 삭제·이미 삭제·`99999999999` 조작 → DB 변화 0·HTTP 200, 새 회원 E 하루 댓글 11개 → 원장 10행, `<b>굵게</b> https://example.com` + 줄바꿈 글자 그대로, 글 P2 삭제 → 댓글 행 0, 키보드만 등록, 375px 가로 스크롤 0·버튼 한 줄·[삭제]·[댓글 등록]·닉네임·`로그인` 링크 ≥ 44×44px (US2-1~15, SC-002·004·005·006·008)
- [ ] T021 [P] [US2] `e2e/params.mjs`에 댓글 폼 `postId` = `abc`, `0`, `2.5`, `99999999999` → `잘못된 요청이에요`, `deleteComment` 인자 범위 밖·`"abc"` → HTTP 200 항목 추가 (US2-7, FR-003, SC-007)

### Implementation for User Story 2

- [ ] T022 [US2] `src/server/social.ts`에 `getCommentThread(postId: number, viewer: { userId: string; isAdmin: boolean } | null): Promise<CommentThread>`를 만든다 — 댓글은 오래된 순(`created_at`, `id`), 각 댓글의 `replies`도 오래된 순, 작성자는 left join(탈퇴 자리면 `author: null`), 삭제 행은 `content: ""`, `createdAtText`는 `formatDateTime`(한국 시간 24시간제 `YYYY. MM. DD. HH:MM`), `canReply`(회원이고 삭제 안 된 댓글)·`canDelete`(삭제 안 된 줄이고 `canDeleteComment` 참)를 서버가 계산, `count` = 삭제 안 된 댓글 + 삭제 안 된 답글. 화면으로 넘기는 데이터에 작성자 회원 ID(`authorId`)와 삭제된 내용을 넣지 않는다. 타입 `CommentAuthor`·`ReplyView`·`CommentView`·`CommentThread`는 contracts/comments.md 3절 그대로 (FR-011, FR-014, FR-016, FR-019, SC-006, research R13)
- [ ] T023 [US2] `src/server/blog.ts`의 `getComments`를 지우고 호출처를 `getCommentThread`로 옮긴다 (U9, 의존 T022)
- [ ] T024 [US2] `src/app/blog/actions.ts`의 `addComment(prev: CommentState, formData: FormData): Promise<CommentState>`를 고친다 — `type CommentState = { ok?: number; error?: string; content?: string }`, 판정 순서: `requireMember()`(없으면 `/`) → `parseId(postId)` 실패면 `잘못된 요청이에요`(`z.coerce.number()` 제거) → `src/lib/social.ts` 내용 검증 → 트랜잭션 안에서 글을 `FOR KEY SHARE`로 잠가 공개 글이거나 글 주인 = 나인지 확인(아니면 `글을 찾을 수 없어요`) → 댓글 insert → 글 주인 ≠ 나이면 `lockUser(나)` → `grantReward(tx, 나, "comment", 댓글ID)`(하루 10번 상한은 `grantReward`, 상한 넘으면 저장만). 모든 실패에서 받은 그대로의 `content`를 돌려준다. 성공하면 `{ ok: Date.now() }` + `revalidatePath("/", "layout")` (FR-006~010, contracts/comments.md 2.1, research R5·R6·R7)
- [ ] T025 [US2] `src/app/blog/actions.ts`의 `deleteComment(commentId: unknown): Promise<void>`를 고친다 — `requireMember()` → `parseId` 실패면 무반응 → `UPDATE comments SET deleted_at = now(), content = '' WHERE id = $1 AND deleted_at IS NULL AND (author_id = 나 OR 글의 블로그 주인 = 나 OR 나는 관리자(viewer.user.role === "admin"))` 한 문장. 답글·보상은 그대로, 0행이면 문구 없이 끝 (FR-013~015, SC-005, research R8)
- [ ] T026 [US2] `src/components/blog/comment-section.tsx`의 댓글 부분을 고친다 — props `{ postId: number; thread: CommentThread; isMember: boolean }`, `<section id="comments" aria-label="댓글">`, 댓글 줄 `id="comment-{id}"`, 제목 `💬 댓글 {count}`(0이어도, 빈 목록 문구 없음), 줄 구성 캐릭터 · 닉네임(블로그 홈 링크) · 시각 / 내용(`whitespace-pre-wrap break-words`, React 텍스트 노드) / 버튼, 삭제 자리는 `삭제된 댓글이에요`(흐린 기울임꼴)·버튼 없음, 탈퇴 자리(`author: null`)는 문구만, [삭제]는 `canDelete`인 줄에만 + 확인 창 `댓글을 삭제할까요?`, 입력칸 3줄 `따뜻한 댓글을 남겨 주세요 💬`·`maxLength={1000}`·`defaultValue={state.content ?? ""}`(안 되면 `key`로 다시 그리기), [댓글 등록] 처리 중 `등록 중...` 비활성, 오류 문구는 버튼 왼쪽 작은 빨간 글씨, 방문자는 `로그인하면 댓글을 남길 수 있어요`(`로그인` → `/`). [삭제]·[댓글 등록]·닉네임·`로그인` 링크 `min-h-11`(링크는 `inline-flex min-h-11 items-center`), 버튼 `whitespace-nowrap`, `focus-visible` 표시 (FR-004·005·009·011~014, research R7·R17, 의존 T022)
- [ ] T027 [US2] `src/app/blog/[slug]/[postId]/page.tsx`(post 소유, 댓글 영역만)에서 `getCommentThread(postId, viewer)` 결과를 `CommentSection`에 그대로 넘긴다 (U13, FR-012, FR-014, 의존 T022·T026)

**Checkpoint**: 댓글 등록·삭제·권한·보상 상한이 동작하고 지운 내용이 어디에도 남지 않는다

---

## Phase 5: User Story 3 - 글에 공감(♥)하고 취소하기 (Priority: P1)

**Goal**: 회원은 글 상세에서 공감하고 다시 눌러 취소한다. 한 회원·한 글 공감은 최대 1개(동시 요청 포함, 오류 화면 없음), 남의 글이면 글 주인이 ✨ 2 · 🪙 2(하루 20번, 같은 사람·같은 글 1번)를 받는다. 방문자는 수와 안내 문구만 본다.

**Independent Test**: 회원 A로 회원 B의 공개 글에 공감 → 취소 → 공감을 반복해 수와 하트 상태, B의 코인 변화(처음 한 번만 🪙 2)를 확인한다.

### Tests for User Story 3

- [ ] T028 [P] [US3] `e2e/social.mjs`에 공감 항목 추가: G가 H의 글에 공감(`♥ 공감 N+1`, 분홍 테두리, `aria-pressed="true"`, H 코인 +2) → 다시 누름(`♡ 공감 N`), 새로고침 안 한 다른 탭의 [♡ 공감 N] → 서버가 취소로 처리·`post_likes` 0행, 공감 → 취소 → 공감 3회 반복 → 공감 1개·`like_received` 원장 1행, H가 자기 글 공감 → 보상 없음, 방문자 → 버튼 비활성 + 버튼 아래 `로그인하면 공감할 수 있어요` 글자 보임, 두 탭 동시 첫 공감 10회 반복 → 매번 1행·오류 화면·콘솔 오류 0, 비공개 글·`abc`·`99999999999` 조작·누르는 사이 비공개 전환 → 저장 0·하트와 수 원래대로, 카드 `♥ N` = 상세 `공감 N`, DB 직접 같은 공감 두 번 → SQLSTATE 23505 (US3-1~8, FR-026, SC-003·004·007)

### Implementation for User Story 3

- [ ] T029 [US3] `src/app/blog/actions.ts`의 `toggleLike(postId: unknown): Promise<void>`를 고친다 — `requireMember()` → `parseId` 실패면 무반응 → 트랜잭션 안에서 글을 `FOR KEY SHARE`로 잠가 공개 글 또는 내 글인지 확인(아니면 무반응, `revalidatePath` 안 부름) → 내 공감이 있으면 삭제(보상 회수 없음) → 없으면 `insert(postLikes).values(...).onConflictDoNothing().returning()`; 빈 배열이면 끝(5-a), 새 행이고 글 주인 ≠ 나이면 `lockUser(글 주인)` → 원장에 `reason = 'like_received'`·`ref_id = "{글ID}:{나}"`가 없을 때만 `grantReward(tx, 글 주인, "like_received", ref_id)`. 4·5 끝에 `revalidatePath("/", "layout")` (FR-025~031, contracts/likes.md 2절, research R9)
- [ ] T030 [P] [US3] `src/components/blog/like-button.tsx`를 고친다 — props `{ postId: number; count: number; liked: boolean; canLike: boolean }` 유지, `canLike = false`면 버튼 아래에 `로그인하면 공감할 수 있어요`를 보이는 글자로 그리고 `aria-describedby`로 잇는다(`title`만으로는 `.btn:disabled`의 `pointer-events: none` 때문에 안 보임), `aria-pressed`·`useOptimistic`·처리 중 비활성 유지 (FR-027, FR-028, FR-030, research R14)
- [ ] T031 [US3] game에 요청 D-7을 보낸다: `point_ledger`에 부분 UNIQUE INDEX (`user_id`, `ref_id`) WHERE `reason = 'like_received'`(출석의 `point_ledger_attendance_uq`와 같은 방식). 만들기 전에 기존 중복 행 수를 세어 PR에 적는다. game이 받지 않으면 `specs/004-social/plan.md` Complexity Tracking에 이유를 적는다 (constitution V, FR-029)

**Checkpoint**: 공감·취소·보상 1번·동시 요청이 모두 오류 화면 없이 동작한다

---

## Phase 6: User Story 4 - 이웃을 추가하고 이웃의 새 글 모아 보기 (Priority: P2)

**Goal**: 회원은 다른 회원의 블로그 홈에서 이웃을 추가·취소하고, `/feed/following`에서 이웃 블로그의 공개 글만 본다. 최근 7일(한국 시간, 오늘 포함) 안에 쓴 즐겨찾은 이웃의 글이 맨 위에 온다.

**Independent Test**: 회원 A가 회원 B의 블로그 홈에서 이웃 추가 → 버튼·`이웃 N` 변화 확인 → 이웃 새 글에서 B의 공개 글만 보이는지 확인 → 이웃 취소 후 B의 글이 빠지는지 확인한다.

### Tests for User Story 4

- [ ] T032 [P] [US4] `scripts/test-social.ts`에 `favoriteWindowStart("2026-10-07")` → `2026-10-01`, `favoriteWindowStart("2026-03-03")` → `2026-02-25` 항목 추가 (FR-042, research R11)
- [ ] T033 [P] [US4] `e2e/social.mjs`의 이웃 항목을 고친다: [+ 이웃 추가] / [✓ 이웃] → `이웃 N` +1 / −1·확인 창 없음, 이웃 새 글에 이웃 B 공개 글(이웃 추가 전 글 포함)만·비공개·이웃 아닌 글 0, 취소 뒤 B 글 빠짐, 이웃 0명 → 두 줄 문구 + "마을 소식" 링크 `/feed`, 이웃 글 있는 회원 `/feed/following?page=999` → 같은 두 줄 문구, 내 블로그 홈 → [✏️ 글쓰기] [🎨 꾸미기] [⚙️ 관리]·자기 ID 조작 저장 0, 방문자 `/feed/following` → `/`, "온보딩 전 회원" 확인을 "없는 회원 ID / 블로그 없는 `users` 행(DB 직접) / 숫자·객체 인자 → 이웃 0·HTTP 200"으로 교체, 같은 대상 동시 추가 2개 → `follows` 1행, 추가·취소 전후 원장 변화 0, 즐겨찾은 C 3일 전·10일 전 + 일반 B 어제 → C(3일 전) → B(어제) → C(10일 전), C 6일 전은 맨 위 묶음·7일 전은 일반 순서, DB 직접 자기 자신 이웃 → 23514, 375px `/feed/following`·이웃 버튼 가로 스크롤 0·글자 한 줄·이웃 버튼 ≥ 44×44px (US4-1~11, SC-005·009)

### Implementation for User Story 4

- [ ] T034 [US4] 마이그레이션 4(구조, U4): `src/db/schema.ts`의 `follows`에 `is_favorite` **boolean NOT NULL DEFAULT false**를 추가하고 `npm run db:generate`로 `drizzle/NNNN_<follows 즐겨찾기>.sql` 생성(한국어 주석 `-- SOC-04/TOWN-08 (2026-10-07): ...`, 백필 없음), `npm run db:migrate`. PK(`follower_id`, `followee_id`), CHECK `follows_not_self_check`, INDEX `follows_followee_idx`는 그대로. 1~3과 독립이라 town이 먼저 필요하면 따로 merge해도 된다 (data-model 2.4·6.1-4, 요청 D-8)
- [ ] T035 [P] [US4] `src/lib/social.ts`에 즐겨찾기 우선 기간(7일) 상수와 `favoriteWindowStart(todayKST: string): string`(오늘 포함 한국 날짜 7일의 시작 날짜)을 추가한다 (FR-042, research R11)
- [ ] T036 [US4] `src/server/blog.ts`의 `paged(where, page, extra, orderFirst?: SQL)`와 `baseList(where, orderFirst?: SQL)`에 선택 정렬 인자를 더한다 — `paged`가 받아 `baseList`로 넘기고, `orderFirst`가 있으면 기존 정렬 앞에 붙이며 없으면 지금과 같다(post 소유 함수, post와 합의, 공통 모듈 추가 D-5) (T002의 `orderBy` 확인 결과 반영)
- [ ] T037 [US4] `src/server/blog.ts`의 `listFeed`에서 `followerId`가 있을 때 정렬을 ① `follows.is_favorite AND posts.created_at >= (${favoriteWindowStart(todayKST())}::date::timestamp AT TIME ZONE 'Asia/Seoul')`인 글 먼저 ② `created_at DESC` ③ `id DESC`로 `orderFirst`에 넘긴다. 거르기(공개 글 AND 내가 이웃 추가한 회원의 블로그)는 그대로, 인기 태그는 마을 전체 공개 글 기준 유지 (FR-042, D15, 의존 T034·T035·T036)
- [ ] T038 [US4] `src/app/blog/actions.ts`의 `toggleFollow(followeeId: unknown): Promise<void>`를 고친다 — `requireMember()` → `typeof followeeId === "string"`이고 1~64자가 아니면 무반응 → 나 자신이면 무반응 → 이미 이웃이면 행 삭제(즐겨찾기도 함께) → 아니면 `INSERT INTO follows (follower_id, followee_id) SELECT 나, owner_id FROM blogs WHERE owner_id = followeeId ON CONFLICT DO NOTHING` 한 문장(블로그 없는 대상이면 0행). 그 순간 대상이 지워진 23503은 `foreignKeyViolation(err)`로 무시하고 다른 오류는 던진다. 보상 없음, 문구·처리 중 표시 없음, 끝에 `revalidatePath("/", "layout")` (FR-035~041, contracts/follows-feed.md 2절, research R10)
- [ ] T039 [P] [US4] `src/components/blog/blog-header.tsx`(blog 소유, 이웃 버튼 부분만)의 이웃 버튼에 `min-h-11`과 `whitespace-nowrap`을 더한다. `+ 이웃 추가`(하늘색 바탕 흰 글씨) / `✓ 이웃`(흰 바탕), 회원이면서 주인이 아닐 때만 보이는 조건은 그대로 (FR-004, FR-035, research R17, 참조 D-9)
- [ ] T040 [US4] `src/app/feed/following/page.tsx`(읽기만)가 `requireMember()`로 방문자를 `/`로 보내고, 제목 `💛 이웃 새 글`, 탭 제목 `이웃 새 글 | Blogville`, 빈 문구 `아직 이웃이 없거나 이웃의 새 글이 없어요.` / `마을 소식에서 마음에 드는 블로그를 이웃으로 추가해 보세요.`("마을 소식" → `/feed`)를 쓰는지 확인한다. 다르면 post에 요청한다 (FR-042~044)

**Checkpoint**: 이웃 추가·취소와 즐겨찾은 이웃 우선 순서의 이웃 새 글이 동작한다

---

## Phase 7: User Story 5 - 댓글에 답글(1단계) 달기 (Priority: P3)

**Goal**: 회원은 삭제되지 않은 원댓글 아래 [답글]로 답글을 단다. 답글은 원댓글 바로 아래 들여쓰기로 보이고 답글에는 답글이 없다. 없는 댓글·다른 글의 댓글·삭제된 댓글을 대상으로 한 답글은 저장하지 않고 입력 내용을 남긴다.

**Independent Test**: 회원 A의 원댓글 아래에 회원 B가 답글 2개를 달아 순서·들여쓰기·버튼 유무를 확인하고, 원댓글을 지운 뒤 답글이 남는지와 댓글 수, 지운 원댓글에 보낸 답글이 거부되는지 확인한다.

### Tests for User Story 5

- [ ] T041 [P] [US5] `e2e/comments.mjs`에 답글 항목 추가: [답글] → 버튼 줄 아래·기존 답글 위 2줄 입력칸 `답글을 남겨 주세요`·버튼 [답글 취소], [답글 취소] → 닫힘·다시 열면 빔, 답글 2개 등록 → 왼쪽 선 들여쓰기·오래된 순·입력칸 닫히고 [답글], 답글 줄에 [답글] 없음·[삭제]는 답글 작성자·A·관리자에게만, 원댓글 삭제 뒤 답글 2개 그대로·`💬 댓글 2`·[답글] 없음, 입력칸 연 채 다른 브라우저에서 원댓글 삭제 → `삭제된 댓글에는 답글을 달 수 없어요`·내용 유지·저장 0, `commentId`를 없는 ID·P2 댓글 ID로 → `답글을 달 댓글이 없어요`·내용 유지·`comments`·`replies` 행 수 그대로, `99999999999`·`abc` → `잘못된 요청이에요`, 공백/1001자/그사이 글 삭제 → `댓글을 적어 주세요`/`댓글은 1000자까지예요`/`글을 찾을 수 없어요`, A가 B의 답글 삭제 → 자리 + `💬` −1, 새 회원 D 하루 댓글 6 + 답글 5 → 원장 `comment` 10행(✨ 50 · 🪙 50), A가 자기 글에 단 답글 보상 없음, 글 P2 삭제 → `replies` 행 0, DB 직접 삭제 행에 내용 넣기 → 23514, 375px [답글]·[답글 취소]·[답글 등록] ≥ 44×44px (US5-1~12, SC-004·010)
- [ ] T042 [P] [US5] `e2e/params.mjs`에 답글 폼 `commentId` = `99999999999`, `abc` → `잘못된 요청이에요`·저장 0, `deleteReply` 인자 `99999999999`, `"abc"` → HTTP 200 항목 추가 (US5-9, FR-003, SC-007)

### Implementation for User Story 5

- [ ] T043 [US5] `src/app/blog/actions.ts`에 `addReply(prev: CommentState, formData: FormData): Promise<CommentState>`(새)를 만든다 — 판정 순서: `requireMember()` → `postId`·`commentId` `parseId` 실패면 `잘못된 요청이에요` → 내용 검증 → 트랜잭션 안에서 글 `FOR KEY SHARE`로 볼 수 있는 글 확인(아니면 `글을 찾을 수 없어요`) → 원댓글 행을 `FOR SHARE`로 잠가 없거나 `post_id` ≠ 요청 글이면 `답글을 달 댓글이 없어요`(일반 댓글로도 저장 안 함), `deleted_at IS NOT NULL`(탈퇴 자리 포함)이면 `삭제된 댓글에는 답글을 달 수 없어요` → `replies` insert → 글 주인 ≠ 나이면 `lockUser(나)` → `grantReward(tx, 나, "comment", "reply:{답글ID}")`(댓글과 합쳐 하루 10번). 모든 실패에서 받은 그대로의 `content`를 돌려준다. 성공하면 `{ ok }` + `revalidatePath("/", "layout")` (FR-018, FR-021, FR-023, SC-010, contracts/comments.md 2.2, research R5)
- [ ] T044 [US5] `src/app/blog/actions.ts`에 `deleteReply(replyId: unknown): Promise<void>`(새)를 만든다 — `deleteComment`와 같은 규칙, 블로그 주인 판단은 `replies → comments → posts → blogs`, `UPDATE replies SET deleted_at = now(), content = '' WHERE ... AND deleted_at IS NULL AND (작성자 OR 블로그 주인 OR 관리자)` 한 문장, 그 밖은 무반응 (FR-024, SC-005)
- [ ] T045 [US5] `src/components/blog/comment-section.tsx`에 답글 부분을 추가한다 — `canReply`인 댓글에만 [답글], 누르면 버튼 줄 아래·기존 답글 위에 `ReplyForm`(2줄 `답글을 남겨 주세요`, `maxLength={1000}`, 숨은 `postId`·`commentId`, `defaultValue={state.content ?? ""}`) + 버튼 [답글 취소](닫고 내용 비움), 원댓글마다 따로 열리고 자동 커서 없음, [답글 등록] 처리 중 `등록 중...` 비활성, 성공하면 비우고 닫고 [답글]로, 오류 문구는 답글 입력칸 버튼 왼쪽. 답글 줄 `id="reply-{id}"`·`ml-10 border-l-2 pl-4` 들여쓰기·[답글] 없음·[삭제]는 `canDelete`일 때 확인 창 `댓글을 삭제할까요?` → `deleteReply`, 삭제 자리 `삭제된 댓글이에요`. [답글]·[답글 취소]·[답글 등록]·답글 [삭제] `min-h-11 min-w-11`·`whitespace-nowrap`·`focus-visible` (FR-019~022, FR-024, FR-004·005, research R17, 의존 T026·T043·T044)
- [ ] T046 [US5] `src/server/social.ts`에 `prepareCommentsForWithdrawal(tx: Tx, userId: string): Promise<void>`를 만든다 — (1) 그 회원 댓글 행 `FOR UPDATE` (2) 다른 회원 답글(삭제 표시된 답글 포함)이 하나도 없는 그 회원 댓글 삭제 (3) 남은 그 회원 댓글을 `deleted_at = COALESCE(deleted_at, now())`, `content = ''`. auth의 탈퇴 트랜잭션이 회원 행을 지우기 전에 부르도록 계약을 auth에 알린다 (D2, 의존성 D-3, contracts/comments.md 4절, data-model 7절)

**Checkpoint**: 모든 사용자 스토리가 독립적으로 동작한다 (1차 범위 완료)

---

## Phase 8: 2차 — 알림 발생 (game 단계 6 merge 뒤, D11)

**Purpose**: game의 `notifyActivity(tx, { recipientId, actorId, kind, postId })`(`src/server/notifications.ts`)를 공감·댓글·답글 트랜잭션 안에서 부른다. 알림함·문구·읽음 처리는 game 소유다.

**⚠️ 선행**: game 단계 6(알림 표 `notifications` + 기록 도우미)이 `main`에 merge되어 있어야 한다 (plan 의존성 D-2)

- [ ] T047 [US3] `src/app/blog/actions.ts`의 `toggleLike`에서 공감 행이 **새로 들어갔고** 글 주인 ≠ 나이면(보상 여부·하루 상한과 상관없이) 같은 트랜잭션에서 `notifyActivity(tx, { recipientId: 글 주인, actorId: 나, kind: "like", postId })`를 부른다. 취소·동시 요청으로 새 행 없음·내 글은 부르지 않는다 (FR-033, contracts/notification-triggers.md 3절)
- [ ] T048 [US2] `src/app/blog/actions.ts`의 `addComment`에서 저장 성공 AND 글 주인 ≠ 나이면 `notifyActivity(tx, { recipientId: 글 주인, actorId: 나, kind: "comment", postId })`를 부른다 (FR-055, 의존 T047과 같은 파일이므로 순서대로)
- [ ] T049 [US5] `src/app/blog/actions.ts`의 `addReply`에서 저장 성공 AND 원댓글 작성자 ≠ 나이면 `notifyActivity(tx, { recipientId: 원댓글 작성자, actorId: 나, kind: "reply", postId: 원댓글의 글 ID })`를 부른다. 글 주인에게는 남기지 않는다 (FR-056, 의존 T048)
- [ ] T050 [P] [US3] `e2e/social.mjs`에 2차 항목 추가: B가 A의 글에 공감 → A에게 공감 알림 1행, A가 자기 글 공감 → 0, 공감 → 취소 → 공감 → 공감 알림 2행 (US3-9, quickstart 7절)
- [ ] T051 [P] [US2] `e2e/comments.mjs`에 2차 항목 추가: B가 A의 글에 댓글 → A에게 1행 / A가 자기 글에 댓글 → 0, B가 C의 댓글(A의 글)에 답글 → C에게 1행·A에게 0, C가 자기 댓글에 답글 → 0, 알림 링크 `/@{slug}/{postId}#comments`로 댓글 영역 스크롤 (US2-16, US5-13, quickstart 7절)

**Checkpoint**: 공감·댓글·답글 알림이 같은 트랜잭션에서 기록된다

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: 문서, 전체 검증, 다른 spec 요청 정리

- [ ] T052 [P] `docs/02-erd.md`를 data-model 8절대로 고친다 — 1장 `comments.author_id` "탈퇴하면 NULL", 3.8 "`deleted_at`을 기록하고 내용을 비운다(CHECK)"·문구 `삭제된 댓글이에요`·탈퇴 처리 문단, 3.14 회원·댓글 줄, 3.16 `replies (comment_id, created_at)` ⏳ 제거, 4장 답글 이전 한 줄, 7장 2·6 완료, 부록 `content` "1~1000자, 삭제하면 빈 글자"·NULL 허용 목록에 `comments.author_id`. 함께 `docs/erdcloud-import.sql`의 `comments.author_id` NULL·`ON DELETE SET NULL` (`docs/erdcloud-final.sql`은 손대지 않음)
- [ ] T053 [P] `README.md` 스크립트 표에 `node e2e/comments.mjs <폴더>`, `node e2e/feed-load.mjs <폴더> [주소]` 두 줄을 추가하고 `e2e/social.mjs` 설명을 "이웃·공감·마을 소식"으로 바꾼다
- [ ] T054 정적 검사·단위 테스트: `npx tsc --noEmit`, `npx eslint`, `npm test`(끝의 `test:social` 모두 ✅) 오류 0 (quickstart 2절)
- [ ] T055 quickstart.md 3~6절을 순서대로 실행한다: `npm run db:migrate && npm run db:seed && npm run db:reset && npm run admin:create` → 빈 마을 문구 확인(`shots/01-feed-empty.png`, 비공개 글만 있을 때 포함) → `node e2e/blog.mjs shots`(출력 확인) → `comments.mjs`·`social.mjs`·`params.mjs` 종료 코드 0 → `npm run build && npm run start` 뒤 `node e2e/auth.mjs shots`·`node e2e/feed-load.mjs shots`(SC-001 값 기록)·`node e2e/nonfunctional.mjs shots`(375px) → 6절 스크린샷(댓글·답글·삭제 자리 PC/375px, 방문자 댓글·공감, 이웃 버튼 두 상태, 마을 소식·이웃 새 글 PC/375px, 이전 결과)을 PR에 첨부
- [ ] T056 다른 spec 요청 결과를 정리한다: D-4(auth 관리자 통계), D-7(game 공감 보상 부분 UNIQUE), D-10(post 탭·칩·페이지 번호 44px)을 상대가 받지 않았으면 `specs/004-social/plan.md`의 Complexity Tracking에 예외와 이유를 적는다. D-6(공감·댓글 뒤 조회수 +1)은 post D5 작업으로 해소됨을 PR에 적는다

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: auth 단계 1 merge 뒤 바로 시작. T003은 마이그레이션 전에 해야 한다
- **Foundational (Phase 2)**: Setup 완료 뒤. T004 → T005 → T006 → T007 순서 고정(중간 스키마 상태에서 각각 생성). T008·T009·T012는 마이그레이션과 병렬 가능. T010 → T011·T013. **모든 사용자 스토리를 막는다**
- **User Stories (Phase 3~7)**: 모두 Foundational 완료 뒤 시작
  - 인원이 있으면 병렬로, 아니면 우선순위 순서(US1 → US2 → US3 → US4 → US5)
- **2차 알림 (Phase 8)**: US2·US3·US5 완료 + game 단계 6 merge 뒤
- **Polish (Phase 9)**: 원하는 스토리가 모두 끝난 뒤. T055는 1차 범위(Phase 3~7) 완료 기준, 2차는 Phase 8 뒤에 7절을 추가로 돌린다

### User Story Dependencies

- **User Story 1 (P1)**: Foundational 뒤 바로. 다른 스토리에 의존하지 않는다 (카드 `💬 N`은 T011이 이미 바꿈)
- **User Story 2 (P1)**: Foundational 뒤 바로. 다른 스토리에 의존하지 않는다
- **User Story 3 (P1)**: Foundational 뒤 바로(사실상 마이그레이션과도 독립). `src/app/blog/actions.ts`를 US2·US4·US5와 함께 고치므로 같은 파일 작업은 순서대로 한다
- **User Story 4 (P2)**: Foundational 뒤. T034(마이그레이션 4)는 1~3과 독립이라 town(D-8)이 먼저 필요하면 따로 merge 가능. T036은 post와 합의 필요
- **User Story 5 (P3)**: US2의 `comment-section.tsx`(T026)·`getCommentThread`(T022) 위에 얹으므로 US2 뒤에 하는 것을 권장한다. 서버 쪽(T043·T044·T046)은 Foundational만으로 시작 가능

### Within Each User Story

- e2e 항목은 구현 전에 작성해 ❌가 나는 것을 먼저 확인한다
- 마이그레이션 → `src/lib/social.ts`(규칙) → `src/server/social.ts`(쿼리) → `src/app/blog/actions.ts`(Server Action) → 화면 → 페이지 연결
- `src/app/blog/actions.ts`는 T024·T025·T029·T038·T043·T044·T047·T048·T049가 모두 고치는 파일이라 이들 사이에는 [P]가 없다
- 스토리를 마치면 체크포인트에서 독립 검증한 뒤 다음 우선순위로 넘어간다

### Parallel Opportunities

- Phase 1: T003은 T002와 병렬
- Phase 2: T008·T009·T012는 서로, 그리고 마이그레이션(T004~T007)과 병렬
- Foundational 뒤: US1(e2e·검토 위주)·US3(`like-button.tsx`)·US4(`blog-header.tsx`, `src/lib/social.ts` 함수 추가)의 화면·규칙 작업을 다른 사람이 병렬로 할 수 있다
- 각 스토리의 e2e 작업([P])은 서로 다른 파일 또는 같은 파일의 다른 블록이면 병렬. 같은 e2e 파일(`e2e/social.mjs`: T014·T019·T028·T033)을 여러 사람이 고치면 블록별로 나눠 충돌을 피한다
- Phase 9: T052·T053 병렬

---

## Parallel Example: User Story 2

```bash
# User Story 2 검증 스크립트를 함께 작성:
Task: "e2e/comments.mjs 댓글 부분 작성 (T020)"
Task: "e2e/params.mjs 댓글 폼 postId·deleteComment 범위 밖 항목 추가 (T021)"

# 그다음 서버 데이터와 Server Action은 파일이 달라 병렬 가능:
Task: "src/server/social.ts getCommentThread (T022)"
Task: "src/app/blog/actions.ts addComment 고치기 (T024)"
```

## Parallel Example: User Story 4

```bash
Task: "scripts/test-social.ts favoriteWindowStart 항목 (T032)"
Task: "e2e/social.mjs 이웃 항목 고치기 (T033)"
Task: "src/lib/social.ts favoriteWindowStart (T035)"
Task: "src/components/blog/blog-header.tsx 이웃 버튼 44px (T039)"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: Setup 완료
2. Phase 2: Foundational 완료 (답글 표 분리·댓글 수 식 — 모든 스토리를 막음)
3. Phase 3: User Story 1 완료
4. **STOP and VALIDATE**: 로그인 없이 `/feed`를 독립 검증 (T014·T015·T019)
5. 준비되면 PR/데모

### Incremental Delivery

1. Setup + Foundational → 기반 완료 (기존 답글 이전 확인 포함)
2. US1 → 독립 검증 → 데모 (MVP)
3. US2 → 독립 검증 → 데모 (댓글·삭제 권한·보상)
4. US3 → 독립 검증 → 데모 (공감 동시성)
5. US4 → 독립 검증 → 데모 (이웃·즐겨찾기 우선 순서)
6. US5 → 독립 검증 → 데모 (답글 1단계·거부 규칙)
7. game 단계 6 merge 뒤 Phase 8 (알림 2차)
8. Phase 9 문서·전체 검증

### Parallel Team Strategy

3명 팀 기준:

1. 함께 Setup + Foundational (마이그레이션 담당 1명, `src/lib/social.ts`·`src/server/social.ts` 담당 1명)
2. Foundational 뒤:
   - 개발자 A: US2 → US5 (댓글·답글, `comment-section.tsx`)
   - 개발자 B: US3 → US4 (공감·이웃, `like-button.tsx`·`blog-header.tsx`·`listFeed`)
   - 개발자 C: US1 검증 + e2e(`feed-load.mjs`, `params.mjs`) + 다른 spec 요청(D-4·D-7·D-10)
3. `src/app/blog/actions.ts`는 A·B가 함께 고치므로 함수 단위로 나눠 순서대로 merge한다

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- 화면 문구는 spec 문구를 그대로 쓴다 (constitution III). 오류 문구는 모두 한국어 (FR-003)
- 권한·입력 검증은 서버에서 (constitution IV). 화면의 버튼 숨김은 `canDelete`·`canReply` 표시용일 뿐이다
- 저장과 보상(2차에는 알림까지)은 한 트랜잭션 (constitution V)
- 마이그레이션 SQL 맨 위에 요구사항 ID 한국어 주석을 단다
- Verify tests fail before implementing
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
