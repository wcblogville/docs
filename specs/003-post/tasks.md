---

description: "글 (POST) 영역 구현 작업 목록"
---

# Tasks: 글 (POST) — 글쓰기·공개 범위·분류·목록·조회수·첨부·임시 저장

**Input**: Design documents from `/specs/003-post/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: plan.md 변경 단위 16과 quickstart.md가 검증 산출물(`scripts/test-post.ts`, 새 e2e 6개, 기존 e2e 수정)을 명시하므로 각 User Story 단계에 테스트 작업을 넣었다. 저장소 관례상 테스트 프레임워크는 없고 `tsx`로 돌리는 `scripts/test-*.ts`와 `@playwright/test`의 `chromium`을 쓰는 `e2e/*.mjs`다.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- 모든 코드 경로는 **코드 저장소 `ehgo508/blogville`** 기준 상대 경로다 (기준 `main` `feb4c05`). 문서 경로(`specs/…`)만 이 문서 저장소 기준이다.
- 단일 Next.js 풀스택 앱: 화면·Server Action·Route Handler는 `src/app/`, 서버 전용 처리는 `src/server/`(`import "server-only"`), 화면·서버 공용 순수 규칙은 `src/lib/`, 화면 조각은 `src/components/`, 마이그레이션은 `drizzle/`, 단위 테스트는 `scripts/test-*.ts`, E2E는 `e2e/*.mjs`.
- 마이그레이션 번호는 박지 않는다. 만들 때 최신 `main`에서 `npm run db:generate`로 다음 번호(0007부터)를 받는다. 각 SQL 맨 위에 요구사항 ID를 단 한국어 주석, 직접 쓴 문장 사이에는 `--> statement-breakpoint`(함수 본문 `$$ … $$` 안은 제외).
- 다른 spec 소유 파일(`src/components/sign-out-button.tsx`(auth), `src/components/site-header.tsx`(town), `package.json`, `e2e/nonfunctional.mjs`, `e2e/farm.mjs`)은 "추가만" 한다.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 구현 전에 라이브러리 전제 확인과 스크립트 자리 마련

- [ ] T001 설치된 패키지 문서·SQL 재현으로 research "추측" 전제를 확인하고 결과를 PR 설명에 적는다 (plan 남은 문제 6): Next.js 16.3.8 Server Action 순차 처리·요청 본문 기본 상한 1MB·쿠키 변경 뒤 재렌더·`history.replaceState`, React 19 `<form action>`+`onSubmit` 순서, Drizzle 0.45 `onDelete` 컬럼 목록 미지원·`drizzle-kit generate --custom`·`select().for("key share")`, PostgreSQL FK 트리거 실행 순서. 다르면 research.md의 대안을 택한다 (`node_modules/next/dist/docs`, `node_modules/drizzle-orm`)
- [ ] T002 `package.json` `scripts`에 `"test:post": "tsx scripts/test-post.ts"`를 `test` 체인 끝(`test:game` → `test:ids` → `test:sanitize` → `test:post`)에, `"posts:cleanup": "tsx --conditions=react-server scripts/cleanup-posts.ts"`를 추가한다 (공통 모듈 추가만)
- [ ] T003 [P] `scripts/test-post.ts` 뼈대를 만든다: 기존 `scripts/test-*.ts` 관례대로 `✅`/`❌` 출력, 실패가 있으면 `process.exit(1)`. 묶음별 함수(`postInputSchema`, 본문 크기 판단, `parseTags`, `attachmentAccess`, `pasteAction`) 자리만 둔다

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 모든 User Story가 쓰는 공용 입력 규칙과 blog를 기다리지 않는 DB 변경(마이그레이션 A·B·C)

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T004 `src/lib/post-rules.ts`(새)에 공용 스키마 `postInputSchema`(zod ^4)를 만든다. 검사 순서와 문구는 contracts/write-actions.md §2 표 그대로: ① `postId`가 빈 값이거나 숫자만 적힌 1~2147483647 → `잘못된 요청이에요` ② 제목 앞뒤 공백 제거 후 1자 이상 → `제목을 적어 주세요` ③ 제목 100자 이하 → `제목은 100자까지예요` ④ 본문 HTML "200,000자 이하 (UTF-16 단위)" → `글이 너무 길어요` ⑤ `categoryId`·`subcategoryId`가 빈 값이거나 숫자만 적힌 1~2147483647 (`0`, `abc`, `1.5`, `99999999999`, `1e1` 거부) → `잘못된 요청이에요` ⑥ 태그 칸 300자 이하 → `태그는 모두 합쳐 300자까지예요` ⑦ `visibility`가 `public`/`private` → `잘못된 요청이에요`. 첫 오류 하나만 돌려주는 도우미와, 본문 HTML의 UTF-8 크기가 900,000바이트를 넘는지 판단하는 함수도 둔다 (FR-006, FR-009, FR-018, R11, R12)
- [ ] T005 `src/app/write/actions.ts`의 `parseTags`를 `src/lib/post-rules.ts`로 옮기고(동작 그대로: 쉼표·`#`·줄바꿈으로 나눔, 공백은 구분자 아님, 앞뒤 공백 제거·가운데 공백 한 칸·영문 소문자, 빈 것과 20자 초과 버림, 중복 제거(먼저 적은 것 유지), 앞 10개) `actions.ts`는 import로 바꾼다 (FR-041, R20)
- [ ] T006 [P] `scripts/test-post.ts`에 `postInputSchema`·본문 크기 판단 묶음을 채운다: 빈 제목·공백 제목, 101자, 200,001자, 태그 301자, 공개 설정 `x`, 카테고리 `0`·`abc`·`1.5`·`99999999999`·`1e1`, 여러 오류일 때 정해진 순서의 첫 번째, 900,000바이트 경계 (quickstart §1)
- [ ] T007 `src/db/schema.ts`(post 담당 블록)에 `attachments.postId`(integer NULL, FK → `posts` `ON DELETE SET NULL`), `attachments.detachedAt`(timestamptz NULL, "글에서 떨어진 시각. 붙어 있거나 한 번도 안 붙었으면 NULL"), 인덱스 (`post_id`)를 더한다. 기존 `key` CHECK `^[a-f0-9]{32}$`, `name` 1~255, `size` > 0, 인덱스 (`user_id`, `created_at`)는 그대로 (FR-047, FR-059, data-model §3.1)
- [ ] T008 `npm run db:generate`로 마이그레이션 A `drizzle/<번호>_attachment_post.sql`을 만들고, 같은 파일 끝에 트리거 함수·트리거 `attachments_track_detached`를 직접 쓴다: `BEFORE UPDATE OF post_id`, 값→NULL이면 `detached_at = now()`, NULL→값이면 `detached_at = NULL` (실제로 바뀔 때만) (R7, data-model §7 A)
- [ ] T009 마이그레이션 B `drizzle/<번호>_attachment_backfill.sql`을 `drizzle-kit generate --custom`(T001에서 확인한 방법)으로 만들고 데이터 이전 SQL을 쓴다: `post_id IS NULL`인 첨부마다 `blogs.owner_id = a.user_id`인 블로그 글 중 `content_html`에 `'/files/' || a.key`가 든 글을 `created_at`, `id` 순으로 찾아 첫 글 `id`를 넣고, 없으면 NULL로 둔다. `post_id IS NULL` 행만 고쳐 다시 돌려도 결과가 같게 한다 (data-model §7.1, R18) (T008 다음)
- [ ] T010 [P] `src/db/schema.ts`에 `postViews` 표를 더하고 `npm run db:generate`로 마이그레이션 C `drizzle/<번호>_post_views.sql`을 만든다: `post_id` integer NOT NULL FK → `posts` `ON DELETE CASCADE`, `date` date NOT NULL(한국 날짜), `visitor_id` uuid NOT NULL, `created_at` timestamptz NOT NULL 기본 now(), PK (`post_id`, `date`, `visitor_id`). IP·회원 ID 컬럼 없음 (FR-046, data-model §4)
- [ ] T011 `src/server/posts.ts`(새)에 `linkAttachments(tx, userId, postId, html)`를 만든다: 본문의 `/files/키`를 모아 첨부 행을 **키 순서로** `FOR UPDATE` 잠그고, data-model §3.3의 5조건(① 행 있음 ② `user_id` = 저장하는 회원 ③ `post_id IS NULL` 또는 `post_id = P`(새 글은 `post_id IS NULL`만) ④ 어느 `profiles.photo_key`도 K가 아님(auth 컬럼이 들어온 뒤 켬, R19) ⑤ `<img>`는 `image`, 파일 카드는 `file`)을 모두 만족하는 키만 정화의 `known`으로 넘긴다. 저장 뒤 남은 키는 `post_id` = 이 글, 이 글에 붙어 있다가 빠진 첨부는 `post_id` = NULL로 바꾼다 (FR-047, FR-054, R6, R17)
- [ ] T012 `src/app/write/actions.ts`의 `savePost`를 공용 스키마와 첨부 붙이기로 바꾼다: T004의 `postInputSchema`로 검사 1~7, 정화 뒤 본문 글자가 비면 `본문을 적어 주세요`(8), 트랜잭션(`lockUser`) 안에서 T011 `linkAttachments`로 `known`을 좁혀 `src/server/sanitize.ts`(변경 없음)에 넘긴다. 실패 반환은 `{ error, values: { title, categoryId, subcategoryId, tags } }`. 글 저장·보상(`grantReward`)·`growForPost`·태그·첨부 붙이기가 한 트랜잭션 (FR-006~009, FR-012, FR-047, constitution V) (T004, T005, T007, T011 다음)

**Checkpoint**: 공용 규칙·첨부 연결·조회 기록 표가 준비됨 — User Story 작업 시작 가능

---

## Phase 3: User Story 1 - 글을 써서 발행하고 보상 안내를 받는다 (Priority: P1) 🎯 MVP

**Goal**: 회원이 글을 발행하면 글 상세로 이동해 주인에게만, 발행 직후 한 번 보상 여부 안내를 보고, 아래 상자에서 글자 수·보상 여부를 미리 본다 (POST-01, GAME-05)

**Independent Test**: 회원으로 100자 이상 공개 글을 발행해 글 상세에서 `🎉 글을 발행했어요! ✨ 경험치 30 · 🪙 30 코인을 받았어요`와 서식이 보이고, 새로고침하면 안내가 사라지는지 확인한다 (`e2e/post-write.mjs`)

### Tests for User Story 1

- [ ] T013 [P] [US1] `e2e/post-write.mjs`(새) US1 부분을 쓴다: 실행마다 새 회원 A·B를 `e2e/helpers.mjs`의 `loginDev`로 만들고, US1-1~10(보상 안내 두 문구, 서식 12가지 보존, 위험 본문 6가지 제거, 빈 본문·공백 제목 오류와 입력값 유지, 49+50자=`100자 · 저장하면 ✨ 경험치 30 · 🪙 30 보상 (하루 3번까지)`, 50+줄바꿈+49자=`99자 · 100자 이상 쓰면 보상을 받아요`, 본문 안내 문구, 로그아웃 시 `/write` → `/`), 새로고침·다른 회원에게 안내 안 보임, 실패 시 종료 코드 1 (quickstart §4)
- [ ] T014 [P] [US1] `e2e/write-count.mjs`의 보상 판단을 주소 `new=reward` 대신 안내 문구로(33행), 글 ID를 주소 경로에서 읽게(34행) 고친다 (`?new`가 지워지므로)
- [ ] T015 [P] [US1] `e2e/farm.mjs`(town 파일, 깨지는 줄만) 94행의 발행 보상 판단을 안내 문구로 읽게 고친다

### Implementation for User Story 1

- [ ] T016 [US1] `src/app/write/actions.ts`의 `savePost` 성공 이동을 바꾼다: 새 글 → `revalidatePath("/", "layout")` 후 `redirect("/@{주소}/{글ID}?new=1")`(보상 여부를 주소에 담지 않음), 수정 → `redirect("/@{주소}/{글ID}")` (FR-013, R13)
- [ ] T017 [P] [US1] `src/server/posts.ts`에 `getPublishNotice(viewerId, post)`를 더한다: 보는 사람이 주인 + 글이 만들어진 지 10분 안 + `point_ledger`에 (주인, `reason = 'post'`, `ref_id` = 글 ID) 행이 있으면 `reward`, 행이 없으면 `noReward`, 그 밖은 `null` (FR-013, R13)
- [ ] T018 [P] [US1] `src/components/blog/publish-notice.tsx`(새, `"use client"`)를 만든다: `reward`면 `🎉 글을 발행했어요! ✨ 경험치 30 · 🪙 30 코인을 받았어요`, `noReward`면 `🎉 글을 발행했어요! (비공개 글, 짧은 글, 또는 오늘 글쓰기 보상 3번을 다 받아서 보상은 없어요)`를 맨 위에 보이고, 처음 그린 뒤 `history.replaceState`로 주소에서 `?new`를 지운다 (임시 글 지우기는 US10 T080에서 더함)
- [ ] T019 [US1] `src/app/blog/[slug]/[postId]/page.tsx`에서 `?new`가 있을 때만 `getPublishNotice`를 불러 `PublishNotice`를 그리게 한다 (기존 `?new=` 문자열만 보고 누구에게나 보이던 처리 제거) (T017, T018 다음)
- [ ] T020 [US1] `src/components/editor/post-form.tsx`를 고친다: 아래 상자 글자 수를 `toLocaleString("ko-KR")`로 고정(브라우저 언어 무시, FR-010), 본문 HTML의 UTF-8 크기가 900,000바이트를 넘으면 요청을 보내지 않고 같은 `postInputSchema`로 검사 1~7을 돌려 첫 문구를 [발행하기] 왼쪽 빨간 글씨 자리에 보인다 (R11). 서버 실패 반환 `values`로 칸을 되돌린다 (FR-009)
- [ ] T021 [P] [US1] `src/components/editor/rich-editor.tsx`의 도구 버튼과 [🖼 사진]·[📎 파일]을 `min-h-11 min-w-11`·`whitespace-nowrap`으로 키운다 (지금 `px-2.5 py-1 text-sm`, 동작·문구 그대로) (FR-066, constitution VI, R21)

**Checkpoint**: 글쓰기·발행·발행 안내가 독립적으로 동작한다 (MVP)

---

## Phase 4: User Story 2 - 내 글을 고치고 지운다 (Priority: P1)

**Goal**: 글 주인만 수정·삭제할 수 있고, 수정에는 보상·발행 안내가 없으며 삭제는 보상을 회수하지 않는다 (POST-01)

**Independent Test**: 글 하나를 [수정]해 제목을 바꾸고 [삭제]해 블로그 홈으로 돌아오는지, 다른 회원으로 수정 주소를 열면 404인지 확인한다 (`e2e/post-write.mjs`)

### Tests for User Story 2

- [ ] T022 [US2] `e2e/post-write.mjs`에 US2-1~9를 더한다: 수정 화면 저장값 채움·[수정 완료]·`글을 고치고 있어요`, 수정 뒤 안내 없음·작성 시각/목록 순서 그대로, 삭제 확인 창 문구 `이 글을 삭제할까요? 댓글과 공감도 함께 지워져요.`와 취소, 남의 글·없는 글·범위 밖 번호 수정 화면 404, 남의 글 수정 조작 → `잠깐 문제가 생겼어요`, 남의 글 삭제 조작 → 0행, 삭제 뒤 경험치·코인·레벨 그대로, 지운 글도 하루 3번에 포함 (T013 다음, 같은 파일)

### Implementation for User Story 2

- [ ] T023 [US2] `src/app/write/actions.ts`의 `savePost` 수정 경로를 확인·정리한다: `posts` UPDATE는 `blog_id` = 내 블로그 조건, 0행이면 예외를 던져 `src/app/error.tsx`(`잠깐 문제가 생겼어요` / [다시 시도] [광장으로 돌아가기])로 보낸다. 보상·발행 안내 없음, `created_at` 그대로 (FR-015, FR-017)
- [ ] T024 [US2] `src/app/write/actions.ts`의 `deletePost`는 코드를 바꾸지 않되 계약을 확인한다: `parseId` 실패면 아무것도 안 함, `DELETE FROM posts WHERE id = ? AND blog_id = 내 블로그`, 태그 연결·댓글·답글·공감·조회 기록 CASCADE, 첨부는 FK `SET NULL` + 트리거로 `detached_at` 기록, 보상 회수 없음, `redirect("/@{내 주소}")` (FR-016, contracts/write-actions.md §3)
- [ ] T025 [P] [US2] `src/components/blog/delete-post-button.tsx`의 [삭제]를 `min-h-11`·`whitespace-nowrap`으로 키운다 (동작 그대로) (R21)
- [ ] T026 [US2] `src/app/blog/[slug]/[postId]/page.tsx`의 [수정] 링크를 `min-h-11`·`whitespace-nowrap`으로 키우고, [수정]·[삭제]가 주인에게만 보이는지 유지한다 (FR-021, R21)

**Checkpoint**: US1·US2가 모두 독립적으로 동작한다

---

## Phase 5: User Story 3 - 글을 공개 또는 비공개로 둔다 (Priority: P1)

**Goal**: 비공개 글과 그 첨부는 주인만 볼 수 있고 다른 사람에게는 존재조차 알리지 않는다 (POST-02, 2026-10-07 첨부 공개 범위 결정)

**Independent Test**: 비공개 글(사진 포함)을 발행한 뒤 다른 회원·로그아웃 상태로 목록 4곳·글 상세·첨부 주소를 열어 어디에도 보이지 않는지 확인한다

### Tests for User Story 3

- [ ] T027 [P] [US3] `scripts/test-post.ts`에 `attachmentAccess` 묶음을 채운다: 공개 글 첨부 누구나, 비공개 글 첨부 주인만(관리자 거부), 붙지 않은 첨부 올린 사람만, 붙지 않은 첨부 + 프로필 사진 누구나 (FR-029, FR-059)
- [ ] T028 [US3] `e2e/post-write.mjs`에 US3-1~7을 더한다: 기본 [🌍 공개], 비공개 `N자 · 비공개 글은 보상이 없어요`, 비공개 글 상세 404 화면(🧭 `길을 잃었어요` / `찾는 블로그나 글이 없거나, 비공개 글이에요.`), 탭 제목 `글 | Blogville`(주인 포함), 비공개로 바뀐 글에 공감·댓글 → `글을 찾을 수 없어요`, 공개↔비공개 왕복 뒤 보상 없음 (T022 다음, 같은 파일)

### Implementation for User Story 3

- [ ] T029 [P] [US3] `src/lib/attachments.ts`에 순수 함수 `attachmentAccess({ postVisibility, postOwnerId, uploaderId, isProfilePhoto, viewerId })`를 추가한다 (data-model §3.4 표 그대로, 관리자 예외 없음)
- [ ] T030 [P] [US3] `src/server/attachments.ts`(새, `next/*`·`src/server/dal.ts`·`src/lib/auth.ts` import 금지)에 `findReadableAttachment(key)`를 만든다: 첨부 행 + 붙은 글의 `visibility` + 그 블로그 `owner_id` (+ auth의 `profiles.photo_key`가 들어온 뒤 프로필 사진 여부)를 한 번에 읽는다
- [ ] T031 [US3] `src/app/files/[key]/route.ts`를 contracts/attachments-http.md §2 순서로 바꾼다: 키 형식 확인 → `findReadableAttachment` → 공개 글·프로필 사진이면 세션을 읽지 않고, 아니면 `getViewer()` → `attachmentAccess` → 불가·행 없음·파일 없음은 모두 404 `파일을 찾을 수 없어요`(`text/plain; charset=utf-8`) → `If-None-Match`가 `"{key}"`면 304 → 200. `Cache-Control: private, no-cache`(지금 `public, max-age=31536000, immutable`), `ETag: "{key}"` 추가, `Content-Type`·`Content-Length`·`Content-Disposition`(RFC 8187)·`X-Content-Type-Options: nosniff`·CSP는 그대로 (FR-029, FR-057, R9) (T029, T030 다음)
- [ ] T032 [P] [US3] `src/components/editor/post-form.tsx`의 공개 토글 [🌍 공개]/[🔒 비공개]를 `min-h-11`·`whitespace-nowrap`으로 키운다 (지금 `px-3 py-1.5`, 기본 `public`·진한 배경 표시 그대로) (FR-022, R21) (T020 다음, 같은 파일)

**Checkpoint**: 비공개 글과 첨부가 주인 밖으로 새지 않는다 (SC-004)

---

## Phase 6: User Story 4 - 글 목록을 최신순으로 페이지마다 본다 (Priority: P1)

**Goal**: 블로그 홈·마을 소식·태그별 글·이웃 새 글 목록이 한 페이지 8개 최신순, 375px에서 가로 스크롤 없이 44×44px 누르는 영역으로 동작한다 (POST-05, TOWN-08)

**Independent Test**: 글 9개 블로그 홈에서 1페이지 8개·2페이지 1개, 페이지 번호 `1 … 8 9 10 11 12 … 20`, 잘못된 페이지 값이 1페이지로 가는지 확인한다 (`e2e/post-lists.mjs`)

### Tests for User Story 4

- [ ] T033 [P] [US4] `e2e/post-lists.mjs`(새)를 쓴다: 새 회원 + 글 9개로 US4-1~9(8개씩, 8개 이하 페이지 번호 숨김, 생략 표시, `0`·`-1`·`abc`·`2.5`·`012`·`1e1`·`99999999999999999999` → 1페이지, 카테고리 유지, 수정 뒤 순서·날짜 그대로, 작성자 줄 유무, 빈 화면 문구 4가지, 로그아웃 시 이웃 새 글 → `/`), 비공개 글이 다른 사람 목록·이전/다음 글·인기 태그 수에서 빠짐(US3-4·5), 375px에서 가로 스크롤 0과 페이지 번호·탭·인기 태그 칩·작성자 줄의 누르는 영역 44×44px 측정. US4-10(즐겨찾기 7일 우선)은 social 단계 5 전이면 건너뛴다
- [ ] T034 [P] [US4] `e2e/nonfunctional.mjs`(공통 모듈 추가만)의 NF-07 측정 경로에 `/tags/{태그}` 한 줄을 더한다 (SC-002)

### Implementation for User Story 4

- [ ] T035 [P] [US4] `src/components/pagination.tsx`의 페이지 번호를 `min-h-11 min-w-11`로 키운다 (지금 `min-w-9 py-1`, `Pagination`·`parsePage` 동작 그대로, blog FR-059·social D-10 요청) (FR-038, R21)
- [ ] T036 [P] [US4] `src/components/blog/feed-view.tsx`의 마을 소식 탭(`btn py-1.5`)과 인기 태그 칩(`py-1`)을 `min-h-11`·`whitespace-nowrap`으로 키운다 (social D-10 요청) (FR-045, R21)
- [ ] T037 [P] [US4] `src/components/blog/post-card.tsx`의 작성자 줄 `{캐릭터} {닉네임} · {블로그 이름}` 링크를 `min-h-11`로 키운다 (긴 블로그 이름 한 줄 `…` 유지, 작성자 줄은 블로그 홈, 나머지는 글 상세 링크 그대로) (FR-037, R21)
- [ ] T038 [US4] `src/server/blog.ts`의 post 소유 목록 함수(`paged`·`baseList`·`listFeed`)에 다른 spec이 요청한 선택 인자 자리를 확인한다: social D-5 `orderFirst`, blog B9 `search`는 "없으면 지금 동작"으로 먼저 들어간 쪽을 따른다. 정렬 작성 시각 최신순(같으면 나중에 쓴 글 먼저)·8개·비공개 제외 규칙은 유지 (FR-034, FR-035, plan 의존성 표)

**Checkpoint**: P1 스토리 4개(US1~US4)가 모두 독립적으로 동작한다

---

## Phase 7: User Story 5 - 글을 대분류·소분류로 분류한다 (Priority: P2)

**Goal**: 글쓰기에서 [대분류 ▼] [소분류 ▼]를 고르고, 배지 `대분류 › 소분류`를 눌러 그 카테고리 글을 모아 본다 (POST-03, 2026-10-07 카테고리 2단계 결정)

**Independent Test**: 대분류 "개발" 아래 소분류 "Git"이 있는 블로그에서 "개발 + Git"으로 발행해 상세 배지 `개발 › Git`과 블로그 홈 카테고리 목록에서 보이는지 확인한다 (`e2e/post-categories.mjs`)

**선행**: blog 단계 2의 `subcategories` 마이그레이션(UK (`category_id`, `id`) 포함)

### Tests for User Story 5

- [ ] T039 [P] [US5] `e2e/post-categories.mjs`(새)를 쓴다: pg로 소분류를 넣어 준비하고 US5-1~9(`카테고리 없음` 기본·대분류 순서, 소분류는 고른 대분류 것만·선택 사항, 대분류 바꾸면 소분류 풀림, 배지 없음, 다른 대분류 소분류 조작 → `잘못된 요청이에요`와 입력값 유지, 남의·없는·지워진 대분류 → `카테고리 없음`, 배지 링크, 이름 변경 반영, `99999999999` → `잘못된 요청이에요`), 대분류 삭제·소분류 삭제 뒤 글이 남는지(Edge Cases)
- [ ] T040 [P] [US5] `e2e/blog.mjs`, `e2e/params.mjs`의 카테고리 칸 이름을 `대분류`로 고친다

### Implementation for User Story 5

- [ ] T041 [US5] `src/db/schema.ts`(post 담당 블록)에 `posts.subcategoryId`(integer NULL), 복합 FK `posts_subcategory_fk` (`category_id`, `subcategory_id`) → `subcategories` (`category_id`, `id`), CHECK `posts_subcategory_check` "`subcategory_id IS NULL OR category_id IS NOT NULL`"을 더한다 (FR-030, data-model §2.1)
- [ ] T042 [US5] `npm run db:generate`로 마이그레이션 D `drizzle/<번호>_post_subcategory.sql`을 만들고 손질한다: FK의 `ON DELETE` 줄을 `SET NULL ("subcategory_id")`로 고치고(R5, PostgreSQL 15 이상), 트리거 함수·트리거 `posts_clear_subcategory`(`BEFORE UPDATE OF category_id`, 새 `category_id`가 NULL이면 `subcategory_id`도 NULL)를 직접 쓴다. 기존 글은 이전할 데이터 없음 (R4, data-model §7 D) (T041 다음)
- [ ] T043 [US5] `src/server/posts.ts`에 `getCategoryOptions(blogId)`(대분류를 블로그 관리 순서로, 각 대분류 아래 소분류 순서대로)와 `resolveCategory(tx, blogId, categoryId, subcategoryId)`를 더한다: `FOR KEY SHARE`로 읽고, 남의·없는 대분류 → 둘 다 NULL(오류 없음), 소분류 행이 없으면 대분류만, 다른 대분류 소속 소분류 → `잘못된 요청이에요` (FR-032, R15)
- [ ] T044 [US5] `src/app/write/actions.ts`의 `savePost`가 `subcategoryId`를 받아 트랜잭션 안에서 `resolveCategory`를 부르고(검사 9), INSERT/UPDATE에 `subcategory_id`를 넣으며, 실패 반환 `values`에 `subcategoryId`를 담게 한다 (FR-032) (T043 다음)
- [ ] T045 [US5] `src/components/editor/post-form.tsx`의 `카테고리` 한 칸을 [대분류 ▼] [소분류 ▼] 두 칸으로 바꾼다: 대분류 첫 항목 `카테고리 없음`, 소분류 첫 항목 `소분류 없음`(plan 남은 문제 8, 팀 확인 대상), 대분류를 바꾸면 소분류를 풀고 새 대분류의 소분류만 보인다. 375px에서 두 칸이 줄바꿈되어 가로 스크롤이 없게 한다 (FR-030, FR-031, FR-066) (T032 다음, 같은 파일)
- [ ] T046 [P] [US5] `src/app/write/page.tsx`가 `getCategoryOptions`로 카테고리 트리를 넘기게 한다 (새 글 기본값 `카테고리 없음`) (FR-031)
- [ ] T047 [P] [US5] `src/app/write/[postId]/page.tsx`가 카테고리 트리와 저장된 대분류·소분류를 넘기게 한다 (지워졌으면 그만큼 비워 보임) (FR-031)
- [ ] T048 [US5] `src/server/blog.ts`의 post 소유 함수(`listColumns`·`baseList`·`getPost`)에 `subcategories` LEFT JOIN(PK)으로 `subcategoryName`을 더하고, `listBlogPosts`에 선택 인자 `subcategoryId`를 더한다(blog 단계 4가 쓴다. 대분류로 거르면 그 대분류 아래 소분류 글도 포함) (FR-033, Assumptions, R16)
- [ ] T049 [P] [US5] `src/components/blog/post-card.tsx`의 카테고리 배지를 `대분류` 또는 `대분류 › 소분류`로 보인다 (노랑, `🔒 비공개` 앞, 목록 카드 배지는 따로 눌리지 않음) (FR-033) (T037 다음, 같은 파일)
- [ ] T050 [US5] `src/app/blog/[slug]/[postId]/page.tsx`의 상세 배지를 `대분류 › 소분류`로 보이고 링크를 소분류가 있으면 `/@{주소}?category={대분류ID}&sub={소분류ID}`, 없으면 `/@{주소}?category={대분류ID}`로, 누르는 영역 `min-h-11`로 바꾼다 (FR-033, R16, R21) (T048 다음)

**Checkpoint**: 카테고리 2단계가 글쓰기·배지·목록에서 동작한다

---

## Phase 8: User Story 6 - 태그를 달고 태그별로 모아 본다 (Priority: P2)

**Goal**: 태그 칸 규칙대로 태그를 달고 `#태그`·인기 태그로 마을 전체 공개 글을 모아 본다 (POST-04)

**Independent Test**: 태그 칸에 `git, #회고, Git`을 적고 발행해 `#git` `#회고` 두 개만 달리고, `#git`을 눌러 태그별 글 목록이 열리는지 확인한다

### Tests for User Story 6

- [ ] T051 [P] [US6] `scripts/test-post.ts`에 `parseTags` 묶음을 채운다: `git, #회고, Git` → `git`, `회고` / 21자 버림 / 11개 → 앞 10개 / `c#` → `c` / 가운데 공백 한 칸 / 줄바꿈 구분 (FR-041, US6-1·2)
- [ ] T052 [US6] `e2e/post-lists.mjs`에 US6-3~8을 더한다: 태그 고치기·비우기 뒤 상세, 태그 칸 300자 입력 제한과 조작 시 `태그는 모두 합쳐 300자까지예요`, 로그아웃 태그별 목록·빈 문구 `이 태그가 달린 글이 없어요.`, 인기 태그 정렬·최대 30개·`아직 태그가 없어요`, `100%` 태그의 목록과 탭 제목 `#100% | Blogville`, 비공개 글 태그 제외 (T033 다음, 같은 파일)

### Implementation for User Story 6

- [ ] T053 [US6] `src/app/blog/[slug]/[postId]/page.tsx`의 본문 아래(공감 버튼 위) `#태그` 배지를 이름순으로 유지하고 누르는 영역을 `min-h-11`·`whitespace-nowrap`으로 키운다 (지금 `py-1`, 링크 `/tags/{인코딩한 태그}` 그대로) (FR-043, R21)
- [ ] T054 [US6] `src/app/tags/[name]/page.tsx`와 `src/server/blog.ts`의 `getAllTags`는 코드를 바꾸지 않고 계약(제목 `🏷 #태그`, 탭 제목 `#태그 | Blogville`, 공개 글만, 마을 소식과 같은 카드·페이지 규칙, 대문자 주소는 소문자로 바꾸지 않음)을 확인한다 (FR-044, FR-045)

**Checkpoint**: 태그 달기·태그별 목록·인기 태그가 동작한다

---

## Phase 9: User Story 7 - 글 조회수를 센다 (Priority: P2)

**Goal**: 주인이 아닌 사람이 글 상세를 열면 같은 브라우저 기준 글마다 한국 시간 하루 1번 조회수를 올린다 (POST-06, BLOG-06과 같은 기준)

**Independent Test**: 방문자로 남의 공개 글을 열어 `👀 N`이 1 오르고, 새로고침해도 더 오르지 않으며, 주인이 열면 오르지 않는지 확인한다 (`e2e/post-views.mjs`)

### Tests for User Story 7

- [ ] T055 [P] [US7] `e2e/post-views.mjs`(새, `blog.mjs` 다음 실행)를 쓴다: US7-1~8(처음 열면 +1이 화면에 반영, 주인 안 오름, 404 경우 안 오름, `updated_at` 그대로, 공감·댓글 뒤 안 오름, 새로고침·다른 탭 안 오름, pg로 `post_views.date`를 어제로 바꾼 뒤 다시 +1, 글 A·B 따로), SC-012(같은 브라우저 10번 열어도 +1)
- [ ] T056 [P] [US7] `e2e/visits.mjs`를 회귀로 돌려 `bv_visitor` 쿠키를 같이 써도 방문자 수가 그대로인지 확인하고, 깨지면 고친다

### Implementation for User Story 7

- [ ] T057 [P] [US7] `src/server/visitor.ts`(새)를 만든다: `readVisitorId()`(쿠키 `bv_visitor`가 UUID 형식이면 값, 아니면 `null`), `ensureVisitorId()`(없거나 형식이 틀리면 새 UUID로 쿠키 설정: `HttpOnly`, `SameSite=Lax`, `Path=/`, `Max-Age` 1년, 배포 `Secure`) (FR-046, R3)
- [ ] T058 [US7] `src/server/posts.ts`에 `recordView(postId, visitorId)`를 더한다: 트랜잭션에서 `post_views` (`post_id`, `todayKST()`(`src/lib/game.ts`), `visitor_id`) `INSERT ... ON CONFLICT DO NOTHING RETURNING` → 들어갔을 때만 `posts.view_count + 1`(`updated_at`은 원래 값을 다시 넣어 그대로), 현재 `view_count`를 돌려준다 (FR-046, data-model §4) (T010 다음)
- [ ] T059 [US7] `src/app/blog/[slug]/[postId]/actions.ts`(새, `"use server"`)에 `recordPostView(postId)`를 만든다: 로그인 불필요, `parseId` 실패·없는 글·비공개 글 → `null`, 보는 사람이 그 블로그 주인 → `{ viewCount: 저장값 }`, 그 밖은 `ensureVisitorId()` 후 `recordView` → `{ viewCount }`. IP·회원 ID 저장 안 함 (contracts/post-pages.md §2) (T057, T058 다음)
- [ ] T060 [US7] `src/components/blog/view-count.tsx`(새, `"use client"`)를 만든다: 서버가 정한 처음 값(주인이 아니고 오늘 이 브라우저로 센 기록이 없으면 저장값 + 1, 아니면 저장값)으로 `👀 N`(천 단위 쉼표 없음)을 그리고, 주인이 아니면 처음 그릴 때 한 번만(의존성 글ID) `recordPostView`를 불러 돌려받은 값으로 바꾼다. 공감·댓글 뒤 재렌더에서는 다시 부르지 않는다 (US7-5) (T059 다음)
- [ ] T061 [US7] `src/server/blog.ts`에서 `incrementViewCount`를 지우고, `src/app/blog/[slug]/[postId]/page.tsx`가 서버 렌더에서 조회수를 바꾸지 않고 `readVisitorId()`와 `post_views` 조회로 처음 값을 정해 `ViewCount`를 그리게 한다 (FR-021, FR-046, R2) (T060 다음)

**Checkpoint**: 조회수가 하루 1번 기준으로 정확히 센다

---

## Phase 10: User Story 8 - 글에 사진을 넣는다 (Priority: P3)

**Goal**: 사진은 그 글에 붙고, 내 다른 글의 사진을 붙여 넣으면 새 첨부로 다시 올리며, 어느 글에도 붙지 않은 첨부는 하루 뒤 정리된다 (POST-07, 2026-10-07 첨부 결정)

**Independent Test**: 사진 두 장을 넣어 발행한 뒤 상세와 수정 화면에 그대로 있고, 그 사진을 새 글에 붙여 넣어 발행해도 원래 글 사진이 그대로인지 확인한다 (`e2e/attachment-links.mjs`)

### Tests for User Story 8

- [ ] T062 [P] [US8] `scripts/test-post.ts`에 `pasteAction` 묶음을 채운다: 내 것·안 붙음 keep, 내 것·이 글 keep, 내 것·다른 글 reupload, 내 것·프로필 사진 reupload, 남의 것·없는 키 drop (FR-047, D7)
- [ ] T063 [P] [US8] `e2e/attachment-links.mjs`(새, 실행마다 새 회원 A·B)를 쓴다: US3-8(비공개 글 첨부 남이 열면 404), US8-2(`올리는 중... (1/3)`·버튼 막힘·`첨부를 올리는 중...`), US8-6(일부만 문제), US8-8(다른 사이트·내장 데이터·없는 첨부·남의 첨부·다른 글 첨부 주소가 저장 때 빠지고 원래 글 첨부 그대로), US8-10(내 다른 글 사진 붙여 넣기 → 다시 올리기), 글 삭제 뒤 `detached_at` 기록, pg로 시각을 하루 전으로 바꾼 뒤 `npm run posts:cleanup`으로 행·파일 삭제(SC-011), SC-013(한 첨부가 두 글에 붙은 경우 0)
- [ ] T064 [P] [US8] `e2e/attachments.mjs`를 회귀로 돌려 올리기 동작(US8-1·3~5·7·9)이 그대로인지 확인하고, `/files` 캐시 머리글 변경으로 깨지는 부분만 고친다

### Implementation for User Story 8

- [ ] T065 [P] [US8] `src/server/storage.ts`(추가만)에 `copyAttachment(fromKey, toKey)`(같은 저장소 안 복사, 대상이 있으면 실패, 복사본 수정 시각 = 지금), `deleteAttachment(key)`(없으면 조용히 넘어감), `listStoredFiles()`(`{ key, modifiedAt }[]`, 키 형식에 맞는 이름만)를 더한다 (contracts/attachments-http.md §4, R10)
- [ ] T066 [P] [US8] `src/lib/attachments.ts`에 순수 함수 `pasteAction({ uploaderId, postId, isProfilePhoto }, viewerId, currentPostId)` → `"keep" | "reupload" | "drop"`을 추가한다 (contracts/write-actions.md §4 판정 표)
- [ ] T067 [US8] `src/server/attachments.ts`에 붙여 넣기 판정(키 목록의 행 읽기 + `pasteAction`)과 다시 올리기(새 무작위 키 16바이트 → 32자 16진수, `copyAttachment`, `attachments` INSERT: `user_id` = 나, 같은 `kind`·`name`·`mime`·`size`, `post_id` NULL)를 더한다 (R8) (T065, T066 다음)
- [ ] T068 [US8] `src/app/write/actions.ts`에 Server Action `classifyPastedAttachments(keys, postId?)`(권한 `requireMember()`, `keys` 1~50개 각 `^[a-f0-9]{32}$`, 잘못되면 `{ error: "잘못된 요청이에요" }`, 반환 `{ items: { key, action }[] }`)와 `reuploadAttachment(key, postId?)`(서버에서 다시 판정해 `reupload`일 때만, 성공 `{ ok: { key, url: "/files/새키", kind, name, size } }`, 실패 `{ error: "올리지 못했어요. 다시 시도해 주세요" }`)를 더한다 (FR-047, contracts/write-actions.md §4·§5) (T067 다음)
- [ ] T069 [US8] `src/components/editor/use-attachment-upload.ts`에 다시 올리기 흐름을 더한다: 기존 올리기와 같은 `올리는 중... (i/n)` 표시, [🖼 사진]·[📎 파일]·발행 버튼 막기, 올리는 중 다른 파일을 넣으면 `다른 파일을 올리는 중이에요. 끝난 뒤 다시 넣어 주세요`, 실패 시 `파일 이름: 올리지 못했어요. 다시 시도해 주세요` (FR-049, FR-050)
- [ ] T070 [US8] `src/components/editor/rich-editor.tsx`의 붙여 넣기 처리를 바꾼다: 붙여 넣는 HTML에 `/files/키`가 있으면 `classifyPastedAttachments`를 불러 keep은 그대로, reupload는 T069 흐름으로 새 주소로 바꿔 넣고, drop(남의 첨부·없는 키)은 넣지 않는다 (plan 남은 문제 4, 팀 확인 대상). 다른 사이트·내장 데이터 사진 거르기는 그대로 (FR-047, FR-054) (T021 다음, 같은 파일, T068·T069 다음)
- [ ] T071 [US8] `src/server/attachments.ts`에 `cleanupPostData({ dryRun })`를 만들고 `scripts/cleanup-posts.ts`(새)에서 `.env.local`을 읽은 뒤 동적 import로 부른다: ① `post_id IS NULL AND COALESCE(detached_at, created_at) < now() - interval '1 day'`이고 프로필 사진이 아닌 행을 `FOR UPDATE SKIP LOCKED`로 골라 `DELETE ... RETURNING key` → `deleteAttachment` ② 저장소에 있지만 행이 없고 수정 시각이 1시간 넘은 파일 삭제 ③ `post_views`에서 `date < 어제(한국)` 삭제. 한국어 한 줄씩 지운 수 출력, `--dry-run`은 세기만, 성공 0·오류 1 (FR-059, SC-011, contracts/attachments-http.md §3) (T065 다음)

**Checkpoint**: 사진이 글에 붙고, 붙여 넣기·정리 작업이 동작한다

---

## Phase 11: User Story 9 - 글에 파일을 붙이고 원래 이름으로 내려받는다 (Priority: P3)

**Goal**: 파일 카드도 사진과 같은 붙이기·다시 올리기·권한 규칙을 따르고, 원래 이름으로 내려받는다 (POST-09)

**Independent Test**: PDF·HWP를 함께 끌어다 놓고 발행한 뒤 파일 카드를 눌러 `보고서 최종본.pdf` 같은 원래 이름으로 바이트 단위까지 같게 내려받아지는지 확인한다

### Tests for User Story 9

- [ ] T072 [US9] `e2e/attachment-links.mjs`에 US9-2([📎 파일]로 사진을 고르면 사진으로), US9-6(카드 이름·크기 조작 → 처음 올린 값으로 저장·내려받기), US9-9(내 다른 글 파일 카드 붙여 넣기 → 같은 원래 이름·크기의 새 첨부, 원래 글 카드도 내려받아짐)를 더한다 (T063 다음, 같은 파일)
- [ ] T073 [P] [US9] `scripts/test-sanitize.ts`에 저장 때 `known`을 좁혀 넘긴 경우의 파일 카드 경우(붙일 수 없는 키의 카드는 빠짐, 카드 이름·크기는 처음 올린 값으로 다시 씀)를 더한다 (FR-055, R17)

### Implementation for User Story 9

- [ ] T074 [US9] `src/components/editor/rich-editor.tsx`의 자체 `FileCard` 확장이 붙여 넣기 다시 올리기 뒤 새 주소·같은 원래 이름·크기로 카드 속성을 바꾸는지 확인하고, 사진만 처리하고 있으면 파일 카드도 같은 흐름에 넣는다 (FR-047, FR-055) (T070 다음, 같은 파일)
- [ ] T075 [US9] `src/app/files/[key]/route.ts`의 파일 응답(`Content-Disposition: attachment; filename="ASCII 대체"; filename*=UTF-8''{인코딩한 원래 이름}`)이 권한 변경(T031) 뒤에도 한글·괄호·따옴표 이름으로 원래 내용 그대로 내려주는지 확인한다 (FR-056, FR-057, SC-007)

**Checkpoint**: 사진·파일 첨부 모두 글에 붙고 원래 이름으로 내려받아진다

---

## Phase 12: User Story 10 - 쓰던 새 글을 임시 저장하고 다시 불러온다 (Priority: P3)

**Goal**: 새 글 쓰기 중 입력이 2초 멈추면 이 브라우저에 회원별로 임시 저장하고, 다시 열 때 불러올지 묻는다 (POST-08, 미구현 목표)

**Independent Test**: 새 글에 제목·본문을 쓰고 2초 기다린 뒤 새로고침해서 불러오기를 고르면 모든 칸이 서식 포함 그대로 채워지는지 확인한다 (`e2e/post-drafts.mjs`)

**선행**: US1의 `PublishNotice`(T018), auth 단계 1의 `SignOutButton` Server Action 폼

### Tests for User Story 10

- [ ] T076 [P] [US10] `e2e/post-drafts.mjs`(새)를 쓴다: US10-1~9(`작성 중인 글은 이 브라우저에 자동 저장됩니다.` → `임시 저장됨 HH:mm`(한국 시간), 불러오기 질문 `작성 중이던 글이 있어요. 불러올까요?` 확인·취소, 제목·본문 비면 저장 안 함, 2초 안 발행 성공 뒤 질문 없음, 발행 실패 뒤 남음, 회원 B에게 A 임시 글 안 보임, `localStorage` 막은 브라우저에서 발행 정상, 수정 화면에 임시 저장 없음), 로그아웃 때 지움, SC-010

### Implementation for User Story 10

- [ ] T077 [P] [US10] `src/lib/draft.ts`(새)를 만든다: 키 `blogville:draft:{userId}`, 값 `{ title: string, categoryId: number | null, subcategoryId: number | null, visibility: "public" | "private", contentHtml: string, tags: string, savedAt: string(ISO) }`, `readDraft`·`writeDraft`·`clearDraft`를 모두 `try/catch`로 감싸 저장을 못 쓰면 조용히 건너뛴다 (FR-064, contracts/write-actions.md §6)
- [ ] T078 [US10] `src/app/write/page.tsx`가 임시 글 주인 `userId`를 `PostForm`에 넘기게 한다 (수정 화면 `src/app/write/[postId]/page.tsx`는 넘기지 않음) (T046 다음, 같은 파일)
- [ ] T079 [US10] `src/components/editor/post-form.tsx`에 임시 저장을 넣는다 (새 글 화면에서만): 제목·카테고리·공개 설정·본문·태그 중 하나가 바뀐 뒤 2초 멈추면 `writeDraft`(제목 앞뒤 공백 제외와 본문 글자가 모두 비면 쓰지 않음), 아래 상자에 처음 `작성 중인 글은 이 브라우저에 자동 저장됩니다.`, 저장 뒤 `임시 저장됨 HH:mm`(한국 시간), 열 때 값이 있으면 1번 `작성 중이던 글이 있어요. 불러올까요?`를 묻고 확인 시 서식 그대로 채움(지워진 카테고리는 `카테고리 없음`), 발행 버튼을 누르면 남은 예약을 바로 쓰고 멈춰 발행 뒤 임시 글을 다시 만들지 않는다 (FR-060~064) (T045, T077 다음, 같은 파일)
- [ ] T080 [US10] `src/components/blog/publish-notice.tsx`가 주인일 때 발행 성공 뒤 `clearDraft(userId)`를 부르게 한다 (발행 실패 시에는 지우지 않음) (FR-063) (T018, T077 다음)
- [ ] T081 [P] [US10] `src/components/sign-out-button.tsx`(auth 소유, 추가만)에 선택 prop `userId`를 더해, 로그아웃 요청을 보내기 전 브라우저(`<form action={signOut}>`의 `onSubmit`)에서 `clearDraft(userId)`를 부른다. 컴포넌트는 `"use client"`로 남긴다 (Assumptions, R14) (T077 다음)
- [ ] T082 [P] [US10] `src/components/site-header.tsx`(town 소유, 추가만)가 `SignOutButton`에 `viewer.userId`를 넘기게 한다 (T081 다음)

**Checkpoint**: 모든 User Story가 독립적으로 동작한다

---

## Phase 13: Polish & Cross-Cutting Concerns

**Purpose**: 문서 갱신, 품질 관문, quickstart 전체 검증

- [ ] T083 [P] `docs/02-erd.md`(post 담당 부분)를 data-model.md §8 표대로 고친다: 1장 관계도 `post_views`·`attachments.detached_at`, 2장, 3.7 복합 PK, 3.9 첨부(붙이는 규칙·누가 여나·트리거·`npm run posts:cleanup`·다시 올리기), 3.14 삭제 규칙, 3.16 `attachments (post_id)` ⏳ 지움, 3.17 `view_count` 반정규화 설명, 3.18 트리거 `posts_clear_subcategory`와 손으로 고친 `SET NULL (subcategory_id)`, 4장 이전 B, 7장 3·6-3 완료, 부록 NULL 허용 `attachments.detached_at` (plan 남은 문제 2는 팀 확인 후)
- [ ] T084 [P] `docs/01-requirements.md`의 POST-01~09 구현 방식·상태를 갱신하고 변경 이력 한 줄을 더한다 (constitution II)
- [ ] T085 [P] `README.md`, `CLAUDE.md`의 스크립트 표에 `test:post`, `posts:cleanup`을, 규칙에 첨부(글에 붙음·비공개 첨부 주인만·하루 뒤 정리)와 조회수(같은 브라우저 하루 1번) 한 줄씩을 더한다
- [ ] T086 `npx tsc --noEmit`, `npx eslint`, `npm test`(`test:game` → `test:ids` → `test:sanitize` → `test:post`)를 돌려 오류 0·모두 `✅`인지 확인한다 (quickstart §1)
- [ ] T087 quickstart.md §2(마이그레이션 적용 뒤 제약·트리거·표 확인, 이전 B 결과 data-model §7.2 개수)와 §3 E2E 순서(`blog.mjs` → `post-write` → `post-lists` → `post-categories` → `post-views` → `attachment-links` → `post-drafts`, 회귀 `write-count`·`attachments`·`params`·`visits`·`social`·`farm`)를 실행하고, `npm run posts:cleanup -- --dry-run`과 실제 실행을 손으로 돌린다
- [ ] T088 quickstart.md §5 SC 확인 표를 채운다: SC-001(처음 쓰는 회원 3분)은 수동 측정, SC-002는 `e2e/nonfunctional.mjs`, 나머지는 e2e 결과. §6 375px·PC 스크린샷(글쓰기, 사진·파일 카드가 있는 글 상세, 목록 4곳)을 남기고 §7대로 결과를 PR에 기록한다 (CI 없음)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 의존 없음. T001은 다른 작업의 전제(특히 T009 `--custom`, T042 `SET NULL (컬럼)`, T018 `history.replaceState`, T081 `onSubmit` 순서)를 확인하므로 가장 먼저 한다.
- **Foundational (Phase 2)**: Setup 뒤. 모든 User Story를 막는다. 마이그레이션 A(T008) → B(T009) 순서 필수, C(T010)는 독립.
- **User Stories (Phase 3~12)**: Foundational 뒤. 우선순위 순서 P1(US1~US4) → P2(US5~US7) → P3(US8~US10)로 진행하거나 인원이 있으면 병렬.
- **Polish (Phase 13)**: 원하는 User Story가 모두 끝난 뒤.

### 외부 spec 의존 (공통 맥락 5.1, 이 기능은 단계 3)

- **auth 단계 1** (온보딩 제거·`loginDev`): 모든 e2e 실행을 막는다 (코드 작업은 막지 않음). `SignOutButton` Server Action 폼 → T081.
- **blog 단계 2** (`subcategories` + UK (`category_id`, `id`)): US5 전체(T039~T050)를 막는다. 나머지 스토리는 기다리지 않는다.
- **auth `profiles.photo_key`**: T011·T030·T071의 프로필 사진 조건만. 없이 먼저 내보내고 컬럼이 들어오면 켠다.
- **social 단계 5** (D15, `replies` 분리): US4-10과 카드 댓글 수 확인만 (T033에서 건너뜀 허용).

### User Story Dependencies

- **US1 (P1)**: Foundational 뒤 바로. 다른 스토리에 의존 없음.
- **US2 (P1)**: Foundational 뒤. US1과 같은 `e2e/post-write.mjs`에 이어 쓰므로 T022는 T013 다음.
- **US3 (P1)**: Foundational(첨부 연결 T011·T012) 뒤. `post-form.tsx` 공유로 T032는 T020 다음.
- **US4 (P1)**: Foundational 뒤. 독립.
- **US5 (P2)**: blog 단계 2 뒤. `post-form.tsx`(T045)는 T032 다음, `post-card.tsx`(T049)는 T037 다음.
- **US6 (P2)**: Foundational(T005 `parseTags` 이동) 뒤. `e2e/post-lists.mjs`(T052)는 T033 다음.
- **US7 (P2)**: 마이그레이션 C(T010) 뒤. 독립.
- **US8 (P3)**: Foundational 뒤. `rich-editor.tsx`(T070)는 T021 다음.
- **US9 (P3)**: US8의 다시 올리기·정리(T065~T071) 뒤 (같은 흐름을 파일 카드에 적용).
- **US10 (P3)**: US1의 `PublishNotice`(T018) 뒤, `post-form.tsx`(T079)는 T045 다음, auth 단계 1의 로그아웃 폼 뒤.

### 같은 파일을 여러 스토리가 만지는 곳 (병렬 금지, 순서대로)

- `src/components/editor/post-form.tsx`: T020(US1) → T032(US3) → T045(US5) → T079(US10)
- `src/app/blog/[slug]/[postId]/page.tsx`: T019(US1) → T026(US2) → T050(US5) → T053(US6) → T061(US7)
- `src/app/write/actions.ts`: T005 → T012 → T016(US1) → T023·T024(US2) → T044(US5) → T068(US8)
- `src/components/editor/rich-editor.tsx`: T021(US1) → T070(US8) → T074(US9)
- `src/server/posts.ts`: T011 → T017(US1) → T043(US5) → T058(US7)
- `src/server/attachments.ts`: T030(US3) → T067(US8) → T071(US8)
- `src/db/schema.ts`: T007 → T010 → T041(US5)
- `scripts/test-post.ts`: T003 → T006 → T027(US3) → T051(US6) → T062(US8) (서로 다른 묶음 함수라 충돌은 작지만 순서대로 합친다)

### Within Each User Story

- 테스트 작업은 구현과 같은 PR에서 쓰고, 구현 뒤 통과를 확인한다 (저장소 관례: 프레임워크 없는 스크립트·e2e).
- 스키마·마이그레이션 → 서버 함수(`src/server/*`) → Server Action·Route Handler → 화면 컴포넌트 → 화면 라우트 순.
- 스토리를 마치고 Checkpoint에서 독립 검증 뒤 다음 우선순위로.

### Parallel Opportunities

- Setup: T003은 T002와 병렬.
- Foundational: T006(테스트)·T010(마이그레이션 C)은 T007~T009와 병렬.
- US1: T013·T014·T015(e2e 세 파일), T017·T018(서로 다른 새 파일), T021.
- US3: T027·T029·T030, T032.
- US4: T033·T034·T035·T036·T037 모두 서로 다른 파일.
- US5: T039·T040, T046·T047, T049.
- US7: T055·T056·T057.
- US8: T062·T063·T064·T065·T066.
- US10: T076·T077, T081·T082.
- Polish: T083·T084·T085.
- 인원이 있으면 Foundational 뒤 US1·US3·US4·US7을 서로 다른 사람이 동시에 진행할 수 있다 (위 "같은 파일" 순서만 지킨다).

---

## Parallel Example: User Story 1

```bash
# US1 테스트 파일 세 개를 함께:
Task: "e2e/post-write.mjs(새) US1 부분 작성"
Task: "e2e/write-count.mjs 33·34행을 안내 문구·주소 경로 기준으로 수정"
Task: "e2e/farm.mjs 94행을 안내 문구 기준으로 수정"

# 서로 다른 새 파일 두 개를 함께:
Task: "src/server/posts.ts에 getPublishNotice 추가"
Task: "src/components/blog/publish-notice.tsx 새로 만들기"
```

## Parallel Example: User Story 4

```bash
# 누르는 영역 44×44px 작업 세 개와 e2e를 함께 (모두 다른 파일):
Task: "src/components/pagination.tsx 페이지 번호 min-h-11 min-w-11"
Task: "src/components/blog/feed-view.tsx 탭·인기 태그 칩 min-h-11"
Task: "src/components/blog/post-card.tsx 작성자 줄 min-h-11"
Task: "e2e/post-lists.mjs 새로 작성"
```

## Parallel Example: User Story 8

```bash
# 저장소 함수·순수 규칙·테스트를 함께:
Task: "src/server/storage.ts에 copyAttachment·deleteAttachment·listStoredFiles 추가"
Task: "src/lib/attachments.ts에 pasteAction 추가"
Task: "scripts/test-post.ts에 pasteAction 묶음"
Task: "e2e/attachment-links.mjs 새로 작성"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: Setup 완료 (T001 라이브러리 전제 확인 포함)
2. Phase 2: Foundational 완료 (CRITICAL - 공용 스키마, 마이그레이션 A·B·C, 첨부 붙이기)
3. Phase 3: User Story 1 완료
4. **STOP and VALIDATE**: `e2e/post-write.mjs` US1 부분과 `e2e/write-count.mjs`로 독립 검증
5. PR로 내보낸다 (CI 없음 → 결과를 PR에 기록)

### Incremental Delivery

1. Setup + Foundational → 기반 준비
2. US1 → 검증 → PR (MVP)
3. US2 → US3 → US4 (P1 완성: 글쓰기·수정·공개 범위·목록)
4. US5 (blog 단계 2 뒤) → US6 → US7 (P2)
5. US8 → US9 → US10 (P3)
6. 각 스토리는 이전 스토리를 깨지 않고 가치를 더한다. 마지막에 Phase 13로 문서·전체 검증

### Parallel Team Strategy (3명)

1. 셋이 Setup + Foundational을 함께 마친다
2. 그 뒤:
   - 개발자 A: US1 → US2 → US10 (글쓰기 화면·`post-form.tsx` 줄)
   - 개발자 B: US3 → US8 → US9 (첨부·`/files`·`rich-editor.tsx` 줄)
   - 개발자 C: US4 → US7 → US6, blog 단계 2가 끝나면 US5
3. "같은 파일을 여러 스토리가 만지는 곳" 순서를 지키고, 나중에 merge하는 쪽이 최신 `main`에 맞춘다

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- 화면·오류 문구는 spec 문구를 한 글자도 바꾸지 않는다 (FR-065, NF-19). 새 문구는 `소분류 없음` 하나뿐이며 팀 확인 대상(plan 남은 문제 8)
- 코드 주석·마이그레이션 주석에 요구사항 ID(POST-01~09, FR 번호)를 단다 (constitution II)
- 비밀값은 환경 변수 이름만 적는다. `npm run db:reset`은 로컬 DB에서만
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
