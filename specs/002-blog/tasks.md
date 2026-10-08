---

description: "블로그 (BLOG) 구현 작업 목록"
---

# Tasks: 블로그 (BLOG)

**Input**: Design documents from `/specs/002-blog/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: spec SC-010("자동 테스트로 100% 재현")과 plan·quickstart가 `npm run test:blog`(단위)와 새 e2e 6개를 요구하므로 테스트 작업을 넣는다. 순수 규칙(`src/lib/blog.ts`)은 테스트를 먼저 쓰고 실패를 확인한 뒤 구현한다. e2e는 각 스토리 구현과 함께 쓰고 스토리 체크포인트에서 통과시킨다.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- 경로는 **코드 저장소 `ehgo508/blogville`** 루트 기준이다 (Next.js 단일 프로젝트: `src/`, `drizzle/`, `scripts/`, `e2e/`, `docs/`).
- `[추가]`라고 적은 작업은 다른 spec 소유 파일에 끼워 넣기만 한다 (공통 맥락 5.2). 먼저 들어간 쪽의 모양을 그대로 쓴다.
- *(plan 임시)* 문구는 spec에 없어 plan이 임시로 정한 값이다 (plan 남은 문제 1).
- 선행 spec 작업(auth 단계 1, post 단계 3, town 단계 11)이 없으면 해당 작업은 건너뛰고 PR에 이유를 적는다.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 구현 전 확인과 테스트 골격

- [x] T001 설치된 패키지 문서로 R-26 목록을 확인하고 결과를 PR 메모에 적는다: Next.js 16 Server Action Origin 확인·`next/form`, React 19 `useActionState` 폼 초기화, Drizzle `.for("update")`·복합 FK `.onDelete("set null")` 생성 SQL, zod 4 문자열 길이 단위, Better Auth username 소문자 처리 (`node_modules/next/dist/docs/`, `node_modules/drizzle-orm/`, `node_modules/zod/` 참조)
- [x] T002 마이그레이션 전 개발 DB 데이터 점검(research R-28)을 `psql`/`npm run db:studio`로 실행한다: 예약어 16개와 같은 `blogs.slug`(관리자 `notice` 제외), 다른 회원 `users.username`과 같은 slug, 다른 회원 아이디와 대소문자 무시로 같은 `profiles.nickname`, `char_length(blogs.description) > 160` — 모두 0건이어야 하고 결과를 PR 메모에 적는다
- [x] T003 [P] 단위 테스트 파일 골격을 만든다: `scripts/test-blog.ts` (`tsx`로 실행, 기존 `scripts/test-*.ts`의 자체 `expect` 관례, 실패 시 종료 코드 1)
- [x] T004 [P] `package.json`의 `scripts`에 `"test:blog": "tsx scripts/test-blog.ts"`를 추가하고 `test` 체인 끝에 `&& npm run test:blog`를 붙인다 [추가]
- [x] T005 [P] 순수 규칙 모듈 빈 파일과 export 이름을 만든다: `src/lib/blog.ts` (`SLUG_RE`, `charCount`, `toLikePattern`, `parseSearchQuery`, `buildCategoryTree`, `swapPosition`, `ROOF_COLORS`)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 여러 스토리가 함께 쓰는 순수 규칙, 폼 결과 형식, 공통 마이그레이션

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Tests for Foundational (먼저 쓰고 실패 확인)

- [x] T006 `scripts/test-blog.ts`에 순수 규칙 시험을 쓴다 (quickstart 1절 표): 주소 정규화 `  My_Blog ` → `my_blog`와 형식 `^[a-z0-9_]{3,20}$` 통과/거부, 예약어 16개(`admin` `api` `town` `feed` `shop` `closet` `write` `settings` `blog` `onboarding` `farm` `attendance` `tags` `wallet` `files` `notice`)와 `Notice` → 예약어, `charCount("😀") === 1`, `toLikePattern("100%_\\")`가 `%`·`_`·`\`를 이스케이프, `parseSearchQuery("  ")` → 빈 검색어·51자 → 무시, `buildCategoryTree` 정렬 `position` → `id`·소분류가 제 대분류 아래, `swapPosition` 맨 위 ▲·맨 아래 ▼는 그대로·결과 0..n-1 빈틈 없음

### Implementation for Foundational

- [x] T007 `src/lib/blog.ts`에 `SLUG_RE = /^[a-z0-9_]{3,20}$/`과 `charCount`(앞뒤 공백 제거 뒤 코드 포인트 수, `[...s].length`)를 구현한다. 정규화·예약어 판정은 auth의 `src/lib/names.ts`(`normalizeName`, `isReservedName`, `RESERVED_NAMES`)를 import해 쓰고, 그 모듈이 아직 없으면 auth와 합의한 16개 목록 값만 PR에 적는다 (research R-02, R-07)
- [x] T008 [P] `src/lib/blog.ts`에 `toLikePattern(q)`(`\` → `\\`, `%` → `\%`, `_` → `\_` 뒤 `%…%`)와 `parseSearchQuery(raw)`(앞뒤 공백 제거, 0자 → `{ empty: true }`, 51자 이상 → `null` 무시, 1~50자 → 검색어)를 구현한다 (FR-053, research R-16·R-17)
- [x] T009 [P] `src/lib/blog.ts`에 `buildCategoryTree(categories, subcategories)`(대분류·소분류 모두 `position` → `id` 정렬, 소분류는 `categoryId`로 묶음)와 `swapPosition(ids, id, direction)`(`-1`/`1`만, 끝이면 그대로, 0부터 다시 매긴 `{id, position}[]`)를 구현한다 (FR-039·040, research R-12·R-14)
- [x] T010 [P] `src/lib/blog.ts`에 `ROOF_COLORS = ["red","orange","yellow","green","sky","blue","purple","brown"] as const`를 정의한다 (TOWN-07 요청, research R-22)
- [x] T011 `src/app/settings/blog/actions.ts`의 `FormState`를 `{ error?: string; ok?: number; values?: Record<string, string> }`로 바꾸고, 길이 검사를 `charCount`로 바꾼다 (contracts/blog-settings.md 0절, research R-07·R-08)
- [x] T012 `src/db/schema.ts`의 `blogs` 블록에 `roofColor: text("roof_color")`(NULL 허용)와 CHECK `blogs_roof_color_check`(NULL 또는 `ROOF_COLORS` 8색), CHECK `blogs_description_check`(`char_length(description) <= 160`)를 추가한다 (B-M2, data-model 2.1)
- [x] T013 `npm run db:generate`로 `drizzle/NNNN_blog_roof_description.sql`과 `drizzle/meta/` 스냅숏을 만들고 SQL 맨 위에 `-- BLOG-03, TOWN-07 (2026-10-07): 소개 160자 CHECK, 지붕 색 칸` 주석을 단 뒤 `npm run db:migrate`로 적용한다. T002에서 160자 초과 소개가 있으면 먼저 그 행을 고친다. town 단계 11 전에 merge한다 (research R-22)
- [x] T014 `npm test`(`test:blog` 포함)와 `npx tsc --noEmit`, `npx eslint`가 통과하는지 확인한다

**Checkpoint**: 순수 규칙·폼 결과 형식·B-M2 준비 완료 — 사용자 스토리 작업 시작 가능

---

## Phase 3: User Story 1 - 가입하면 내 블로그가 자동으로 생긴다 (Priority: P1) 🎯 MVP

**Goal**: 가입 직후 `{아이디}의 블로그`·`/@{아이디}`·빈 소개·초원 배경·대분류 "일상"을 가진 블로그가 정확히 1개 있고, 내 집·`내 블로그로 →`로 갈 수 있다. 가입 트랜잭션 자체는 auth가 구현하고(research R-27), blog는 기본값 제공과 검증을 맡는다.

**Independent Test**: 새 아이디로 가입한 뒤 `/@{아이디}`를 열어 블로그 홈이 보이는지, DB에 블로그가 정확히 1개인지, 기본값(이름·소개·배경·기본 카테고리)이 맞는지 확인한다 (`node e2e/blog-home.mjs <폴더>`).

### Tests for User Story 1

- [x] T015 [US1] `e2e/blog-home.mjs`를 새로 만들고 US1 시나리오를 쓴다 (실행마다 새 회원, `pg`로 DB 확인): 가입 직후 `blogs` 1개·`slug` = 아이디·`title` = `{아이디}의 블로그`·`description` = `''`·배경 초원·대분류 "일상" 1개·광장 이동(US1-1·8, SC-001), 광장 내 집과 `/settings/blog`의 `내 블로그로 →`가 `/@{아이디}`(US1-2, FR-007), 같은 회원 블로그 INSERT를 `blogs_owner_id_unique`가 거부·동시 두 요청도 1개(US1-4), 블로그 홈·관리·꾸미기에 블로그 만들기·지우기 버튼 없음(US1-5), `npm run admin:create` 두 번 실행해도 `/@notice` 1개·이름 `Blogville 공지사항`·소개 `마을 소식과 업데이트를 알려드려요`·대분류 "공지"(US1-6), 아이디 `town`·`notice`·`Admin` 가입 → `이 아이디는 쓸 수 없어요`·회원·블로그 0(US1-7, SC-012)

### Implementation for User Story 1

- [x] T016 [US1] auth와 가입 블로그 기본값을 확정한다: 이름 `{아이디}의 블로그`, 주소 = 아이디(대체값 없음), 소개 `''`, 배경 초원, 대분류 "일상" 1개 (FR-001·002, research R-27). 값이 상수로 필요하면 `src/lib/blog.ts`에 `defaultBlogFor(username)`을 두고 auth 가입 트랜잭션이 import한다
- [x] T017 [US1] `scripts/create-admin.ts`가 관리자 블로그 `/@notice`(이름 `Blogville 공지사항`, 소개 `마을 소식과 업데이트를 알려드려요`, 대분류 "공지")를 만들고 다시 실행해도 늘지 않는지 확인하고, 어긋나면 고친다 (FR-006)
- [x] T018 [US1] 블로그 만들기·지우기 기능이 어디에도 없는지 `src/app/`, `src/components/blog/`를 검색해 확인한다 (FR-005, US1-5). 회원 삭제 CASCADE 경로는 T058에서 확인한다
- [x] T019 [US1] auth 단계 1 merge 뒤 `node e2e/blog-home.mjs <폴더>`의 US1 줄이 모두 `✅`인지 확인한다. 원자성(US1-3, SC-002)은 auth 검증 결과를 PR에 함께 적는다

**Checkpoint**: 가입 = 블로그 보유가 확인됨

---

## Phase 4: User Story 2 - 누구나 `/@주소`로 블로그 홈을 본다 (Priority: P1)

**Goal**: 방문자·다른 회원·주인이 `/@주소`로 블로그 홈을 보고, 보는 사람별 버튼·비공개 글·404·탭 제목·375px 규칙이 지켜진다. 대부분 이미 동작하므로 44px 누르는 영역을 보강하고 검증한다.

**Independent Test**: 공개 글·비공개 글·카테고리가 있는 블로그를 방문자·다른 회원·주인 세 컨텍스트로 열어 버튼·글·숫자·404를 확인한다.

### Tests for User Story 2

- [x] T020 [US2] `e2e/blog-home.mjs`에 US2 시나리오를 더한다: 세 컨텍스트별 버튼(없음 / 이웃 / [✏️ 글쓰기] [🎨 꾸미기] [⚙️ 관리]), 비공개 글은 주인에게만 `🔒 비공개` 배지·카테고리 글 수 포함, 탭 제목 `{블로그 이름} | Blogville`(US2-1·3·4), `/@Notice`·없는 주소·다른 블로그 글 ID·남의 비공개 글·`abc`·`0`·`012`·`1e1`·`2147483648` → HTTP 404 `길을 잃었어요`(US2-2), 대분류 글 9개 이상에서 선택 → 노란 배경·2페이지 유지(US2-6), 글 없는 블로그 `🌱 아직 글이 없어요.`·[첫 글 쓰기]는 주인만(US2-7), 375px 가로 스크롤 0·주인 버튼 글자 한 줄·주인 버튼·[첫 글 쓰기]·블로그 이름 링크·카테고리 링크 44×44px 이상(US2-8, SC-004), `?category=1.5`·`?category=Infinity`·`?page=99999999999999999999` → 200 전체 글 1페이지(US2-9, SC-009)

### Implementation for User Story 2

- [x] T021 [P] [US2] `src/components/blog/blog-header.tsx`에서 주인 버튼 3개([✏️ 글쓰기] [🎨 꾸미기] [⚙️ 관리])와 블로그 이름 링크를 375px에서도 44×44px 이상·`whitespace-nowrap`으로 고친다 (FR-059, research R-24)
- [x] T022 [P] [US2] `src/app/blog/[slug]/page.tsx`의 빈 목록 [첫 글 쓰기] 버튼과 기존 카테고리 링크(`px-2 py-1`)의 누르는 영역을 44×44px 이상으로 고친다 (FR-058·059). 페이지 번호(`src/components/pagination.tsx`)는 post 소유라 post에 요청만 한다
- [ ] T023 [US2] `node e2e/blog-home.mjs`, `node e2e/params.mjs`, `node e2e/social.mjs`(US2-5 이웃 버튼), `node e2e/mobile.mjs`(US2-8)가 모두 `✅`인지 확인한다

**Checkpoint**: US1 + US2로 "가입 → 내 블로그 홈 보기" MVP 완성

---

## Phase 5: User Story 3 - 블로그 이름·소개·주소와 닉네임을 바꾼다 (Priority: P1)

**Goal**: 블로그 관리에서 이름·소개·주소를, 내 정보에서 닉네임을 바꾸고, 바뀐 값이 사이트 전체에 바로 반영된다. 아이디·예약어 겹침은 auth 이름 모듈과 같은 규칙·잠금으로 막는다.

**Independent Test**: 블로그 관리에서 이름·소개·주소를 바꿔 블로그 홈·광장·글 카드 반영을 확인하고, 잘못된 값(다른 회원 아이디와 같은 주소 포함)의 문구를 확인한다. 내 정보에서 닉네임을 바꿔 반영과 오류를 확인한다 (`node e2e/blog-address.mjs <폴더>`).

### Tests for User Story 3

- [x] T024 [US3] `e2e/blog-address.mjs`를 새로 만든다 (quickstart 3.2): 이름 `  새 이름  ` → `새 이름`·`저장했어요 ✓`·블로그 홈 제목·탭 제목·광장 내 집 아랫줄·글 카드 반영·60초 안(US3-1, SC-005·006), 공백만/41자/소개 161자 → `블로그 이름을 적어 주세요`/`블로그 이름은 40자까지예요`/`소개는 160자까지예요` 하나만·DB 그대로·칸에 보낸 값 남음(US3-2), 소개 비움 → 소개 줄 없음(US3-3), 주소 `My_Blog` → `my_blog`·`/@my_blog` 200·예전 주소 404·사이트 링크에 예전 주소 0개(US3-4, SC-005), `ab`/`town`/다른 회원 주소/다른 회원 아이디 → `주소는 영문 소문자, 숫자, _ 로 3~20자예요`/`이 주소는 쓸 수 없어요`/`이미 있는 주소예요`/`이미 있는 주소예요`(US3-5, SC-012), 아이디 주소 되돌리기 본인만(US3-9), 풀린 주소 즉시 다른 회원 사용(US3-10), 연속 5번 변경 성공(US3-11), 닉네임 정상·`가`/21자/다른 회원 닉네임/`Tester2` → `닉네임은 2자 이상이에요`/`닉네임은 20자까지예요`/`이미 있는 닉네임이에요`/`이미 있는 닉네임이에요`·`😀` 하나 → 500 없이 `닉네임은 2자 이상이에요`(US3-6), 같은 값으로 가입과 주소 변경 동시 → 한쪽만 성공(SC-012), 이름·소개 요청에 `slug`·`ownerId`를 섞어도 내 블로그 이름·소개만(US3-7, SC-008), 로그아웃 `/settings/blog` → `/`(US3-8)

### Implementation for User Story 3

- [x] T025 [US3] `src/app/settings/blog/actions.ts`의 `updateBlogInfo`를 고친다: 입력은 `title`·`description`만 읽음, 검증 순서 이름 0자 → `블로그 이름을 적어 주세요` / 이름 41자 이상 → `블로그 이름은 40자까지예요` / 소개 161자 이상 → `소개는 160자까지예요`(코드 포인트, 첫 오류만), `UPDATE blogs SET title, description WHERE owner_id = 나`, 실패 시 `{ error, values: { title, description } }`, 성공 시 `{ ok: Date.now() }`와 `revalidatePath("/", "layout")` (FR-016·017·021, contracts/blog-settings.md 1절)
- [x] T026 [US3] `src/app/settings/blog/actions.ts`에 `updateBlogSlug(prev, formData)`를 새로 만든다: `requireMember()` → `normalizeName`(앞뒤 공백 제거·소문자) → `SLUG_RE` 아니면 `주소는 영문 소문자, 숫자, _ 로 3~20자예요` → 지금 주소와 같으면 저장 없이 `{ ok }` → `isReservedName`이면 `이 주소는 쓸 수 없어요`(단 `notice`는 `users.role = 'admin'`에게 허용, research R-05) → 트랜잭션 `lockName(tx, 새 주소)` → `findNameConflict(tx, 새 주소, { exceptUserId: 나 })`의 `username` 또는 `slug`가 참이면 `이미 있는 주소예요` → `UPDATE blogs SET slug WHERE owner_id = 나` → 23505(`blogs_slug_unique`)면 `이미 있는 주소예요`. 성공 `{ ok, values: { slug } }`, 실패 `{ error, values: { slug: 보낸 값 } }`, 보호 기간·횟수 제한·자동 이동 없음 (FR-009·010·018·021, contracts/blog-settings.md 2절, research R-03·R-04·R-06)
- [x] T027 [US3] `src/app/settings/blog/settings-forms.tsx`의 기본 정보 폼이 오류 때 `state.values`로 칸 값을 남기도록 고치고(React 19 폼 초기화 대응, research R-08), [저장] 왼쪽에 빨간 오류 한 줄 / 초록 `저장했어요 ✓`를 보이고, 소개 안내 `어떤 이야기를 쓰는 블로그인가요?`를 유지한다 (FR-016·017)
- [x] T028 [US3] `src/app/settings/blog/settings-forms.tsx`에 `BlogSlugForm` 컴포넌트를 새로 만든다: 라벨 `블로그 주소` *(plan 임시)*, `/@` + 입력칸(`maxLength=20`), [주소 바꾸기] *(plan 임시)*, 결과 문구는 버튼 왼쪽 한 줄, 성공 때 칸은 정규화된 값, 버튼 44×44px (contracts/blog-home.md 3절)
- [x] T029 [US3] `src/app/settings/blog/page.tsx`에서 `(주소는 바꿀 수 없어요)` 줄(현재 55행 부근)을 지우고 `BlogSlugForm`을 기본 정보 카드에 넣고, `내 블로그로 →` 링크를 44×44px 이상으로 고친다. 탭 제목 `블로그 관리 | Blogville`, 화면 제목 `⚙️ 블로그 관리`, 비로그인 `/` 이동을 유지한다 (FR-007·018·022)
- [x] T030 [P] [US3] `src/app/settings/account/nickname-actions.ts`를 새로 만들고 `updateNickname(prev, formData)`를 구현한다: `requireMember()` → `nickname`만 읽음 → 코드 포인트 2자 미만 `닉네임은 2자 이상이에요` / 21자 이상 `닉네임은 20자까지예요` → 지금 닉네임과 같으면 `{ ok }` → 트랜잭션 `lockName(tx, 닉네임)` → `findNameConflict`의 `username`이 참이면 `이미 있는 닉네임이에요`(아이디와는 대소문자 무시, 자기 아이디는 허용) → `UPDATE profiles SET nickname WHERE user_id = 나` → 23505(`profiles_nickname_unique`)면 `이미 있는 닉네임이에요` → `revalidatePath("/", "layout")`. 예약어 검사는 하지 않는다 (FR-019·020, contracts/profile-showcase.md 1절, research R-03)
- [x] T031 [P] [US3] `src/app/settings/account/nickname-form.tsx`를 새로 만든다 (클라이언트, `useActionState`): 라벨 `닉네임`, 지금 값이 채워진 칸(`maxLength=20`), [저장], 버튼 왼쪽 오류 빨간 한 줄 / 성공 `저장했어요 ✓` *(plan 임시)*, 오류 때 보낸 값 유지, 버튼 44×44px (contracts/blog-home.md 4절)
- [x] T032 [US3] auth 소유 `src/app/settings/account/page.tsx`의 "닉네임 자리"에 `<NicknameForm />` 한 줄을 끼운다 [추가] (research R-29, auth U9 골격 필요)
- [x] T033 [US3] 바뀐 이름·닉네임·주소가 블로그 홈 제목·탭 제목, 글 상세 `{블로그 이름} · {닉네임}`, 마을 소식·이웃 새 글·태그 글 카드 `{닉네임} · {블로그 이름}`, 광장 집 아랫줄·이웃집 패널, 관리자 화면 링크에서 DB 값을 읽어 그리는지(하드코딩 주소 없음) `src/` 전체에서 `/@`·`/blog/` 링크 생성 지점을 검색해 확인하고, `/blog/{주소}` 형식 링크가 남아 있으면 `/@{주소}`로 고친다 (FR-013·014·020, SC-005, research R-06)
- [x] T034 [US3] `e2e/params.mjs`에 `updateBlogSlug` 조작 인자(남의 값 섞기, 이상한 문자열)를 더한다 [추가]
- [x] T035 [US3] `node e2e/blog-address.mjs <폴더>`와 `node e2e/params.mjs`가 모두 `✅`인지 확인한다

**Checkpoint**: 기본값으로 생긴 블로그·닉네임을 주인이 바꿀 수 있음

---

## Phase 6: User Story 4 - 블로그에서 미니룸, 주인 프로필, 전시 동물, 동물 도감을 본다 (Priority: P1)

**Goal**: 누구에게나 같은 미니룸(주인 닉네임 배지, 2초 튀기), 주인 프로필(사진 또는 캐릭터 얼굴), 다 키운 동물 도감, 전시 동물 한 마리를 보이고, 주인은 도감에서 전시를 고르거나 비운다.

**Independent Test**: 배경·캐릭터를 장착하고 다 키운 동물 3마리가 있는 회원의 블로그를 방문자로 열어 미니룸·프로필·도감을 확인하고, 주인으로 전시를 바꿔 반영을 확인한다 (`node e2e/blog-showcase.mjs <폴더>`).

**선행**: town 단계 11 (`user_animals` UNIQUE(`user_id`, `id`)) → T036~T037, auth 단계 1 (`profiles.photo_key`) → T040 사진 부분.

### Tests for User Story 4

- [ ] T036 [P] [US4] `e2e/blog-showcase.mjs`를 새로 만든다 (quickstart 3.5, 새 회원 + DB로 다 키운 동물 3마리와 알 1개): 미니룸 배경·캐릭터·**닉네임** 배지·애니메이션 주기 2초(US4-1, FR-023·025), 꾸미기에서 배경 변경 반영(US4-2), 없는 그림 키 → 오류 없이 회색 몸통·초원(US4-3), 375px 높이 224px·1280px 256px·아래쪽 테두리 2px만(US4-4), `photo_key` 연결 시 `/files/키` 사진 + 닉네임·없으면 캐릭터 얼굴(US4-5), 도감 카드 3장(알 제외)·없으면 `아직 다 키운 동물이 없어요`(US4-6), [전시하기]/다른 동물/[전시 빼기]·고르지 않으면 빈 자리(US4-7·8), `setShowcaseAnimal`에 남의 동물·알·`99999999999`·`"abc"` → `showcase_animal_id` 그대로(US4-9, SC-008), 세 컨텍스트에서 미니룸·프로필·전시·도감 동일(전시 버튼만 주인)(US4-10)

### Implementation for User Story 4

- [ ] T037 [US4] `src/db/schema.ts`의 `blogs` 블록에 `showcaseAnimalId: integer("showcase_animal_id")`(NULL)와 복합 FK `blogs_showcase_owned_fk`: (`owner_id`, `showcase_animal_id`) → `user_animals` (`user_id`, `id`) `.onDelete("set null")`를 추가하고 옆에 "SET NULL (showcase_animal_id)로 생성 SQL을 손질함" 주석을 단다 (B-M3, data-model 2.1, research R-18)
- [ ] T038 [US4] `npm run db:generate`로 `drizzle/NNNN_blog_showcase.sql`을 만들고 FK 줄을 `ON DELETE SET NULL ("showcase_animal_id")`로 고친 뒤 맨 위에 `-- BLOG-04 (2026-10-07): 전시 동물 한 마리, 복합 FK로 내 동물만 (ERD 3.11)` 주석을 달고 `npm run db:migrate`, `psql \d blogs`로 FK 동작을 확인한다
- [ ] T039 [P] [US4] `src/server/blog.ts`의 `getBlogBySlug`가 `photoKey`(`profiles.photo_key`, auth 전에는 `null`)와 `showcaseAnimalId`를 함께 돌려주게 고치고, 새 함수 `getGrownAnimals(ownerId)`(`user_animals` `status = 'grown'` + `animal_species.name`·`asset_key`, `{ id, name, assetKey, grownAt }[]`, `grown_at` 최신순, 인덱스 `user_animals_user_status_idx`)를 만든다 (contracts/profile-showcase.md 4절)
- [ ] T040 [P] [US4] `src/app/settings/blog/actions.ts`에 `setShowcaseAnimal(animalId: number | null)`를 새로 만든다: `requireMember()` → 값이 정확히 `null`이면 `UPDATE blogs SET showcase_animal_id = NULL WHERE owner_id = 나` → 그 밖의 값은 `parseId()` 실패 시 아무것도 안 함 → 고르기는 UPDATE 한 문장으로 `owner_id = 나` AND `EXISTS (user_animals WHERE id = animalId AND user_id = 나 AND status = 'grown')`일 때만 저장 → `revalidatePath("/", "layout")`. 반환·문구 없음 (FR-030·031, research R-19)
- [ ] T041 [P] [US4] `src/components/character.tsx`의 `MiniRoom`에 선택 prop `showcase?: { assetKey: string; name: string } | null`을 더해 캐릭터 오른쪽에 `animalSvg`(`src/lib/art/animals.ts`)로 그린다. 없으면 빈 자리, 그림을 못 찾으면 그리지 않음. 2초 튀기·높이 224/256px·아래 2px 선은 유지 (FR-023~026·030, research R-20). shop의 가구 층과 자리를 협의한다
- [ ] T042 [P] [US4] `src/components/blog/blog-header.tsx`에 주인 프로필(프로필 사진 48px 원형 `/files/{photoKey}`, 없으면 캐릭터 얼굴 + 닉네임)을 더하고, 미니룸 닉네임 배지가 블로그 이름이 아닌 **닉네임**인지 확인한다 (FR-023·028)
- [ ] T043 [US4] `src/components/blog/animal-collection.tsx`를 새로 만든다 (클라이언트, `useTransition`): 제목 `🏅 동물 도감` *(plan 임시)*, 카드(그림·이름·다 키운 날짜), 없으면 `아직 다 키운 동물이 없어요`, 주인에게만 [전시하기] / `전시 중` + [전시 빼기] *(plan 임시)* 버튼(44×44px)이 `setShowcaseAnimal`을 부름 (FR-029·030, contracts/profile-showcase.md 2절)
- [ ] T044 [US4] `src/app/blog/[slug]/page.tsx`에서 `getGrownAnimals`를 다른 조회와 `Promise.all`로 병렬 조회하고, `showcaseAnimalId`를 결과 목록에서 찾아(없으면 빈 자리) `MiniRoom`에 넘기고, 블로그 정보 아래에 `AnimalCollection`을 놓는다 (글 목록을 첫 화면 밖으로 밀지 않는 작은 카드 줄, research R-20). 글 화면(`/@주소/글ID`)에는 미니룸 없이 캐릭터 얼굴만 유지 (FR-032)
- [ ] T045 [US4] post에 `/files/{key}`가 `profiles.photo_key` 첨부를 누구에게나 200으로 내려주도록 요청하고 결과를 확인한다 (contracts/blog-home.md 5절, research R-21)
- [ ] T046 [US4] `e2e/params.mjs`에 `setShowcaseAnimal` 조작 인자(`undefined`, `"abc"`, `1.5`, `99999999999`)를 더하고 [추가], `node e2e/blog-showcase.mjs <폴더>`와 `node e2e/params.mjs`가 `✅`인지 확인한다

**Checkpoint**: P1 스토리 4개 모두 독립적으로 동작

---

## Phase 7: User Story 5 - 카테고리를 대분류·소분류 2단계로 관리한다 (Priority: P2)

**Goal**: 블로그 관리에서 대분류·소분류를 추가·이름 바꾸기·순서 바꾸기·삭제하고, 블로그 홈 왼쪽에 트리와 글 수, `?sub=` 거르기가 동작한다. 카테고리 변경은 블로그 행 잠금으로 순서 빈틈·겹침을 막는다.

**Independent Test**: 블로그 관리에서 대분류·소분류를 바꾸고 블로그 홈 트리·글 소속이 기대대로 바뀌는지 확인한다 (`node e2e/categories.mjs <폴더>`).

**선행**: T048~T049(B-M1)는 post 단계 3의 선행이므로 단계 2에서 먼저 merge한다. 소분류 글 수·거르기(T052 일부, T056~T058)는 post 단계 3(`posts.subcategory_id` + 트리거 `posts_clear_subcategory`) 뒤.

### Tests for User Story 5

- [ ] T047 [P] [US5] `e2e/categories.mjs`를 새로 만든다 (quickstart 3.3): 대분류 추가(버튼·Enter) 맨 아래·칸 비움, "여행" 아래 "맛집" 맨 끝, 대분류 + 소분류 누름 4번 이하(US5-1·2, SC-007), 같은 이름 → `이미 있는 카테고리예요`·추가 칸 값 남음(US5-3), "여행" 아래 "맛집" 또 → 거부·"공부" 아래 "맛집" → 성공(US5-4), 공백만/21자 → `카테고리 이름을 적어 주세요`/`카테고리 이름은 20자까지예요`·`  여행  ` → `여행`(US5-5), ▲▼ 순서가 블로그 홈·글쓰기 선택과 같음·맨 위 ▲/맨 아래 ▼ 비활성·소분류는 같은 대분류 안에서만·DB `position` 0부터 빈틈 없음·두 탭 동시 추가/이동에도 겹침 없음(US5-6, FR-039), `여행 (5)`/`└ 맛집 (3)`·"여행" 5개·"맛집" 3개(US5-7), 소분류 삭제 확인 창 `'맛집' 소분류를 지울까요? 글은 '여행'에 남아요.` → 글 `subcategory_id` NULL·`category_id` 유지(US5-8), 대분류 삭제 확인 창 `'여행' 카테고리를 지울까요? 글은 남고 '카테고리 없음'이 돼요.` 취소 → 그대로·수락 → 대분류·소분류 삭제·글 두 칸 NULL·`updated_at` 그대로(US5-9), 소분류 글이 있는 회원 `DELETE FROM users` → 오류 없이 CASCADE(FR-005), 다른 블로그 ID·`99999999999`·`abc` 이름 바꾸기 → `잘못된 요청이에요`·삭제/순서/소분류 추가 → 문구 없이 변화 없음·500 없음(US5-10, SC-008·009). post 단계 3 전이면 컬럼 유무를 보고 해당 줄을 건너뜀 표시

### Implementation for User Story 5

- [ ] T048 [US5] `src/db/schema.ts`에 `subcategories` 표를 추가한다: `id` integer identity PK, `category_id` integer NOT NULL FK → `categories.id` `ON DELETE CASCADE`, `name` text NOT NULL + CHECK `subcategories_name_check`: `char_length(name) BETWEEN 1 AND 20`, `position` integer NOT NULL 기본 0, UNIQUE `subcategories_category_name_uq`(`category_id`, `name`), UNIQUE `subcategories_category_id_uq`(`category_id`, `id`). `blog_id`는 두지 않는다 (B-M1, data-model 2.3, research R-10)
- [ ] T049 [US5] `npm run db:generate`로 `drizzle/NNNN_subcategories.sql`을 만들고 맨 위에 `-- BLOG-05 (2026-10-07): 카테고리 2단계, 소분류 표 (ERD 3.18)` 주석을 단 뒤 `npm run db:migrate`, `psql \d subcategories`로 제약 이름을 확인하고, `scripts/reset-dev.ts`의 TRUNCATE CASCADE가 새 표를 함께 비우는지 확인한다
- [ ] T050 [US5] `src/app/settings/blog/actions.ts`의 대분류 처리 4개를 고친다: `addCategory`는 트랜잭션 첫 줄 블로그 행 `FOR UPDATE` → `position = MAX + 1` → UNIQUE 위반 `이미 있는 카테고리예요`·실패 시 `values: { name }`; `renameCategory`는 `parseId` 실패 또는 `RETURNING` 0행 → `잘못된 요청이에요`(현재 48-64행은 `{ ok }`); `moveCategory`는 블로그 행 잠금 → `position, id` 순으로 읽어 `swapPosition` → 0부터 다시 매김·방향 `-1`/`1`만; `deleteCategory`는 블로그 행 잠금 → 내 대분류 확인 → (post 단계 3 이후) 그 대분류 글의 `category_id`·`subcategory_id`를 함께 NULL(`updated_at` 유지) → DELETE(소분류 CASCADE) → 남은 대분류 0부터 다시 매김 (FR-033·036·037·039·042, contracts/blog-settings.md 3절, research R-11·R-12)
- [ ] T051 [US5] `src/app/settings/blog/actions.ts`에 소분류 처리 4개를 새로 만든다: `addSubcategory(categoryId, prev, formData)`(`parseId` 실패·남의 대분류 → `{}`, 블로그 행 잠금, 그 대분류 안 `MAX + 1`, UNIQUE 위반 `이미 있는 카테고리예요`, 이름 규칙은 대분류와 같음), `renameSubcategory(subcategoryId, name)`(`category_id IN (내 블로그 대분류)` `RETURNING` 0행 → `잘못된 요청이에요`), `deleteSubcategory(subcategoryId)`(블로그 행 잠금 → DELETE `RETURNING category_id` → 같은 대분류 남은 소분류 다시 매김), `moveSubcategory(subcategoryId, direction)`(같은 대분류 안에서만 맞바꿈) (FR-034~039·042, contracts/blog-settings.md 4절)
- [ ] T052 [US5] `src/server/blog.ts`의 `getCategories(blogId, includePrivate)`가 `buildCategoryTree`로 대분류(`id`, `name`, `position`, 글 수)마다 `subcategories`(`id`, `name`, `position`, 글 수) 배열을 돌려주게 고친다. 대분류 글 수는 소분류 글 포함, 주인 관리 화면은 비공개 포함, 소분류 글 수는 post 단계 3 뒤부터 (FR-039·040·056, research R-14)
- [ ] T053 [US5] `src/app/settings/blog/settings-forms.tsx`의 `CategoryManager`를 트리로 바꾸고 `SubcategoryRow` 컴포넌트를 새로 만든다: 각 줄 ▲▼(가로 배치, 각 44×44px, `aria-label` `{이름} 위로`/`{이름} 아래로`) · 이름 · (글 수) · [이름 바꾸기] [삭제], 맨 위 ▲·맨 아래 ▼ 비활성, 대분류마다 [소분류 추가] → 입력칸(안내 `새 소분류` *(plan 임시)*, `aria-label` `새 소분류 이름` *(plan 임시)*) + [추가](Enter 가능, `addSubcategory.bind(null, 대분류ID)` + `useActionState`), 삭제 확인 창 문구(FR-037·038), 오류는 그 칸·그 줄 아래 빨간 한 줄·입력값 유지 (FR-033~039, research R-24)
- [ ] T054 [P] [US5] `src/components/blog/category-nav.tsx`를 새로 만든다: `전체 글 (N)`(공개 글 수) 아래 대분류 순서대로, 소분류는 `└ 이름 (N)`으로 들여 씀, 글 없는 카테고리도 `(0)`, 고른 줄은 노란 배경 + 굵게, 링크는 `?category=`/`?sub=`만 남기고 `q`·`page`를 버림, 누르는 영역 44×44px, 768px 미만은 글 목록 위·이상은 왼쪽 220px (FR-040·057·059, contracts/blog-home.md 1.1·1.2)
- [ ] T055 [US5] `src/app/blog/[slug]/page.tsx`에서 기존 평평한 카테고리 목록을 `CategoryNav`로 바꾸고, `?sub=`를 `parseId`로 읽어 `category`보다 먼저 적용하고, 잘못된 값은 무시(전체 또는 `category`), 없는 번호·다른 블로그 번호는 빈 목록·제목 `전체 글 0개`·선택 표시 없음, 페이지 링크는 고른 `category`/`sub` 유지 (FR-040·056·057, research R-13)
- [ ] T056 [US5] `src/server/blog.ts`의 `listBlogPosts`에 선택 인자 `subcategoryId`가 post 변경 13으로 들어왔는지 확인하고, 없으면 같은 모양으로 추가만 한다 [추가] (먼저 들어간 쪽을 그대로 씀). 대분류 거르기는 소분류 글 포함
- [ ] T057 [US5] post 단계 3 merge 뒤 관리 화면 소분류 글 수, 블로그 홈 `└ 맛집 (3)`, `?sub=` 거르기, 글쓰기 대분류·소분류 두 칸의 순서가 관리 화면과 같은지(FR-041, post `e2e/post-categories.mjs`) 확인한다
- [ ] T058 [US5] 대분류·회원 삭제 경로에서 `posts` CHECK 위반이 나지 않는지(post 트리거 `posts_clear_subcategory`) 실제 DB에서 확인하고 결과를 PR에 적는다 (research R-11, plan 남은 문제 10)
- [ ] T059 [US5] `e2e/params.mjs`에 `?sub=` 이상한 값(`abc`, `1.5`, `99999999999`)과 소분류 4개 Server Action 조작 인자를 더한다 [추가]
- [ ] T060 [US5] `node e2e/categories.mjs <폴더>`, `node e2e/params.mjs`, `node e2e/blog-home.mjs`(트리 회귀)가 모두 `✅`인지 확인한다

**Checkpoint**: 카테고리 2단계가 관리·블로그 홈에서 독립적으로 동작

---

## Phase 8: User Story 6 - 내 블로그에서 마을의 글과 블로그를 검색한다 (Priority: P2)

**Goal**: 회원이 자기 블로그 홈에서만 보이는 검색창으로 마을 전체의 공개 글(제목·본문)과 블로그(이름·닉네임)를 찾는다. 새 주소 없이 블로그 홈 `?q=` 주인 전용 모드로 만든다 (research R-15).

**Independent Test**: 공개·비공개 글이 섞인 마을에서 내 블로그 검색창으로 검색해 공개 글만 나오는지, 결과에서 글·블로그로 가는지 확인하고, 방문자·다른 회원·광장에서 검색창이 없는지 확인한다 (`node e2e/blog-search.mjs <폴더>`).

### Tests for User Story 6

- [ ] T061 [P] [US6] `e2e/blog-search.mjs`를 새로 만든다 (quickstart 3.4, 실행마다 고유 검색어로 DB에 글 준비): 주인 검색 → 마을 전체 공개 글 중 제목·본문 일치만·최신순 8개·2페이지·카드 윗줄 `{닉네임} · {블로그 이름}`(US6-1), 블로그 이름·닉네임 일치 블로그 묶음·누르면 그 블로그 홈(US6-2), 남의 비공개 글·내 비공개 글 제외(US6-3, SC-008), 없는 검색어 `검색 결과가 없어요`·공백만 `검색어를 적어 주세요`(US6-4), `%`·`_` 검색어는 그 글자 든 글만, `/town`(데스크톱·휴대폰)·`/feed`에 검색창 없음(US6-5, SC-013), 방문자 블로그 홈 검색창 없음·`/@{주소}?q=` → 보통 블로그 홈(US6-6), 회원이 남의 블로그 홈 → 없음·자기 블로그 → 있음(US6-7), 375px 결과 화면 가로 스크롤 0(SC-004)

### Implementation for User Story 6

- [ ] T062 [US6] `src/server/blog.ts`의 `listFeed`에 선택 인자 `search?: string`을 추가한다 [추가]: 있으면 `visibility = 'public'` AND (`title ILIKE $1 ESCAPE '\'` OR `content_text ILIKE $1 ESCAPE '\'`), 패턴은 `toLikePattern`·바인딩, 최신순(`created_at DESC, id DESC`) 8개씩, 없으면 지금 동작 그대로. social의 `orderFirst`와 같은 함수이므로 나중에 merge하는 쪽이 최신 `main`에 맞춘다 (FR-051, research R-16)
- [ ] T063 [US6] `src/server/blog.ts`에 `searchBlogs(q)`를 새로 만든다: `blogs.title` 또는 주인 `profiles.nickname` `ILIKE`(이스케이프) 일치 블로그 최대 8곳, 최근 공개 글 순, `{ slug, title, nickname, characterAsset, photoKey }[]` (FR-052, research R-17)
- [ ] T064 [P] [US6] `src/components/blog/blog-search.tsx`를 새로 만든다: 검색창(GET 폼, 입력칸 `maxLength=50`·`aria-label="검색어"`·안내 `마을의 글·블로그 검색` *(plan 임시)* + [검색] *(plan 임시)*, 44px)과 결과(제목 `🔍 '{검색어}' 검색 결과` *(plan 임시)*, 1페이지에만 블로그 묶음 `블로그` *(plan 임시)* — 캐릭터 얼굴 또는 사진 · `{닉네임} · {블로그 이름}` · `@{주소}` → `/@{주소}`, 글 묶음 `글 N개` *(plan 임시)* — `src/components/blog/post-card.tsx`의 `showAuthor` 카드, 빈 검색어 `검색어를 적어 주세요`, 둘 다 없으면 `검색 결과가 없어요`) (FR-050~053, contracts/blog-home.md 1.4)
- [ ] T065 [US6] `src/app/blog/[slug]/page.tsx`에 검색 모드를 넣는다: 서버가 주인(`viewer.userId === blog.ownerId`)일 때만 왼쪽 상자 위에 검색창을 그리고 `?q=`를 `parseSearchQuery`로 읽어 검색 모드로 전환(51자 이상은 무시), 주인이 아니면 `q`를 무시하고 보통 블로그 홈, 검색 모드의 페이지 링크는 `q` 유지, 카테고리 트리에 선택 표시 없음, `listFeed({ page, search })`·`searchBlogs`를 병렬 조회 (FR-050·057, SC-013, research R-15)
- [ ] T066 [US6] 광장(`src/app/town/`)과 마을 소식(`src/app/feed/`)에 검색창이 없는지 확인한다 (FR-050, US6-5)
- [ ] T067 [US6] `e2e/nonfunctional.mjs`의 대상 목록에 검색 결과 주소(`/@{주인 주소}?q=…`)를 더한다 [추가]
- [ ] T068 [US6] `node e2e/blog-search.mjs <폴더>`가 모두 `✅`인지 확인한다

**Checkpoint**: 검색이 주인 블로그 홈에서만 독립적으로 동작

---

## Phase 9: User Story 7 - 블로그 방문자 수를 본다 (Priority: P3)

**Goal**: 블로그 홈·글 상세를 연 비주인 방문자를 한국 날짜 하루 1번 세고, 블로그 홈 정보 줄과 블로그 관리 `방문자` 카드에 보인다. 이미 구현되어 있어(`0006_blog_visits`, `e2e/visits.mjs`) 이번 변경이 깨뜨리지 않는지 확인한다.

**Independent Test**: 서로 다른 두 브라우저로 블로그 홈·글을 열고 새로고침·날짜 변경을 거쳐 숫자가 규칙대로 오르고 주인 방문은 세지 않는지 확인한다 (`node e2e/visits.mjs <폴더>`).

### Implementation for User Story 7

- [ ] T069 [US7] `src/app/blog/[slug]/page.tsx`·`src/components/blog/blog-header.tsx` 변경 뒤에도 `src/components/blog/visit-count.tsx`가 정보 줄 끝에 `오늘 방문 N · 어제 방문 N · 전체 방문 N`(천 단위 쉼표)을 그리고, 주인이 아닐 때만 `recordBlogVisit`(`src/app/blog/actions.ts`, 바뀌지 않음)을 부르는지 확인한다. 검색 모드·`?sub=` 이동에서도 같은 날 다시 세지 않는다 (FR-043~047·049)
- [ ] T070 [US7] `src/app/settings/blog/page.tsx`의 카드 순서가 `방문자`(최근 7일 막대, 마지막 `오늘` 노란 막대·굵게, 날짜 `10/5`, 안내 `같은 사람은 하루 1번만 셉니다. 블로그 홈이나 글을 연 사람이 방문자이고, 내 방문은 세지 않아요.`) → `기본 정보` → `카테고리`로 유지되는지 확인한다 (FR-048)
- [ ] T071 [US7] `node e2e/visits.mjs <폴더>`가 모두 `✅`인지 확인한다 (US7-1~8, SC-010)

**Checkpoint**: 모든 사용자 스토리가 독립적으로 동작

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: 성능 측정, 문서, 전체 검증

- [ ] T072 [P] `e2e/blog-scale.mjs`를 새로 만든다: 전용 회원 블로그에 공개 글 1,000개와 대분류·소분류를 DB로 채움(있으면 건너뜀), 캐시 없는 새 브라우저 컨텍스트로 블로그 홈·대분류 거르기·소분류 거르기·주인 검색 결과 첫 페이지를 각 5번 열어 load 중앙값 출력, 1,000ms 미만 기대 (SC-003·011, NF-07, research R-01)
- [ ] T073 `npm run build && npm run start`로 프로덕션 빌드에서 `node e2e/blog-scale.mjs <폴더> http://localhost:3000`, `node e2e/nonfunctional.mjs <폴더> http://localhost:3000`을 실행하고 숫자를 PR에 적는다. 1초를 넘으면 post에 `posts(category_id)`·`posts(subcategory_id)`·`pg_trgm` 인덱스를 요청한다 (research R-14·R-16)
- [ ] T074 [P] `docs/02-erd.md`의 blog 담당 부분을 고친다 (data-model 6절): 1장 `blogs`에 `roof_color` 줄, 3.11 ⏳ 제거와 `ON DELETE SET NULL (showcase_animal_id)` 손질 한 줄, 3.18 대분류 삭제 때 앱이 글 두 칸을 먼저 비운다는 한 줄, 3.14 대분류 줄 보충, 지붕 색 새 절, 7장 할 일 표 완료 표시, 부록 글자 길이·NULL 허용
- [ ] T075 [P] `README.md` 스크립트 표에 새 e2e 6줄(`blog-home`, `blog-address`, `categories`, `blog-search`, `blog-showcase`, `blog-scale`)을 더한다 [추가]
- [ ] T076 `npx tsc --noEmit`, `npx eslint`, `npm test`와 quickstart 3절 E2E 순서(auth → `blog.mjs` → `blog-home` → `blog-address` → `categories` → `blog-search` → `blog-showcase` → `visits` → `params` → `social` → `mobile`)를 모두 실행한다 (specs/002-blog/quickstart.md)
- [ ] T077 quickstart 5절 스크린샷(블로그 홈 방문자/주인 1280·375px, 검색 결과, 블로그 관리 오류·트리·44px, 닉네임 칸, 예전 주소 404)을 눈으로 확인한다
- [ ] T078 PR 본문에 quickstart 6절 항목(정적 검사·단위·E2E 결과 줄, 성능 숫자, R-28 데이터 점검 결과, 건너뛴 시나리오와 이유)과 plan 남은 문제 1의 *(plan 임시)* 문구 목록을 적어 팀 확인을 요청한다 (NF-23, 원칙 III)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 의존성 없음 — 바로 시작
- **Foundational (Phase 2)**: Setup 뒤 — 모든 사용자 스토리를 막는다. T013(B-M2)은 town 단계 11 전에 merge
- **User Stories (Phase 3~9)**: 모두 Foundational 뒤
  - 우선순위 순서: US1 → US2 → US3 → US4 (P1) → US5 → US6 (P2) → US7 (P3)
  - 인력이 있으면 Foundational 뒤 병렬 진행 가능 (아래 스토리 의존성 참고)
- **Polish (Phase 10)**: 원하는 스토리가 모두 끝난 뒤

### 다른 spec 선행 작업

| 선행 | 막는 작업 |
|---|---|
| auth 단계 1 (가입 통합, `loginDev` 수정) | 모든 e2e 실행 (T019, T023, T035, T046, T060, T068, T071, T076) |
| auth U4 (`src/lib/names.ts`, `src/server/names.ts`) | T007(정규화·예약어 import), T026, T030 |
| auth U2·U3 (`username` NOT NULL, `nickname` CHECK 2~20, `photo_key`) | T026, T030(13~20자 저장), T039·T042(사진) |
| auth U9 (`/settings/account` 골격) | T032 |
| post 단계 3 (`posts.subcategory_id` + FK + CHECK + 트리거, `listBlogPosts` `subcategoryId`) | T050의 글 칸 비우기, T052 소분류 글 수, T056~T058 |
| post (`/files/{key}` 프로필 사진 공개) | T045 |
| town 단계 11 (`user_animals` UNIQUE(`user_id`, `id`)) | T037~T038 → T040, T044, T046 |

### User Story Dependencies

- **US1 (P1)**: Foundational 뒤. 다른 스토리 의존 없음 (가입 트랜잭션은 auth)
- **US2 (P1)**: Foundational 뒤. `e2e/blog-home.mjs`를 US1과 같이 쓰므로 T020은 T015 뒤
- **US3 (P1)**: Foundational 뒤. 다른 스토리 의존 없음. `settings-forms.tsx`·`actions.ts`를 US5와 같이 고치므로 같은 사람이 이어서 하거나 순서대로 merge
- **US4 (P1)**: Foundational 뒤 + town 단계 11. `blog-header.tsx`를 US2(T021)와 같이 고치므로 T042는 T021 뒤. `actions.ts`(T040)는 US3·US5와 같은 파일
- **US5 (P2)**: Foundational 뒤. T048~T049(B-M1)는 post 단계 3의 선행이라 단계 2에서 먼저 merge. `page.tsx`(T055)는 US4(T044)·US6(T065)과 같은 파일
- **US6 (P2)**: Foundational(T008 `toLikePattern`·`parseSearchQuery`) 뒤. `page.tsx`(T065)는 T055 뒤에 맞춘다
- **US7 (P3)**: 다른 스토리 변경 뒤 회귀 확인 (T069는 T044·T055·T065 뒤)

### Within Each User Story

- 순수 규칙 테스트(T006)는 구현(T007~T010) 전에 쓰고 실패를 확인한다
- 마이그레이션(스키마 → generate → migrate) → 서버 조회·Server Action → 화면 컴포넌트 → 페이지 연결 → e2e 통과 순
- 같은 파일(`src/app/settings/blog/actions.ts`, `src/app/settings/blog/settings-forms.tsx`, `src/app/blog/[slug]/page.tsx`, `src/server/blog.ts`, `src/db/schema.ts`, `e2e/params.mjs`)을 고치는 작업은 [P]가 아니며 순서대로 한다
- 각 스토리 체크포인트에서 해당 e2e가 `✅`여야 다음 우선순위로 넘어간다

### Parallel Opportunities

- Setup: T003, T004, T005 병렬
- Foundational: T008, T009, T010 병렬 (T007 뒤, 같은 파일이지만 서로 다른 함수 — 한 사람이 이어서 쓰는 것을 권장), T012~T013은 T010 뒤
- US3: T030(`nickname-actions.ts`)·T031(`nickname-form.tsx`)은 블로그 관리 작업(T025~T029)과 병렬
- US4: T036(e2e), T039(`src/server/blog.ts`), T040(`actions.ts`), T041(`character.tsx`), T042(`blog-header.tsx`)는 서로 다른 파일이라 T038 뒤 병렬
- US5: T047(e2e), T054(`category-nav.tsx`)는 T048~T053과 병렬
- US6: T061(e2e), T064(`blog-search.tsx`)는 T062~T063과 병렬
- Polish: T072, T074, T075 병렬
- 스토리 간: Foundational 뒤 US3(관리 화면)·US4(도감)·US6(검색)을 다른 사람이 맡을 수 있다. 단 공유 파일은 merge 순서를 맞춘다

---

## Parallel Example: User Story 4

```bash
# T038(B-M3 마이그레이션) 뒤 서로 다른 파일을 함께 진행:
Task: "e2e/blog-showcase.mjs 새로 만들기 (US4-1~10)"
Task: "src/server/blog.ts getBlogBySlug에 photoKey·showcaseAnimalId, getGrownAnimals 새 함수"
Task: "src/app/settings/blog/actions.ts setShowcaseAnimal 새 함수"
Task: "src/components/character.tsx MiniRoom에 showcase prop"
Task: "src/components/blog/blog-header.tsx 주인 프로필"
```

## Parallel Example: User Story 3

```bash
# 블로그 관리 쪽과 내 정보 닉네임 쪽을 나눠서:
Task: "src/app/settings/blog/actions.ts updateBlogInfo 보강 + updateBlogSlug"
Task: "src/app/settings/account/nickname-actions.ts updateNickname"
Task: "src/app/settings/account/nickname-form.tsx 닉네임 폼"
```

---

## Implementation Strategy

### MVP First (User Story 1 + 2)

1. Phase 1: Setup 완료
2. Phase 2: Foundational 완료 (CRITICAL - 모든 스토리를 막음)
3. Phase 3: US1 — auth 가입 통합과 함께 기본 블로그 확인
4. Phase 4: US2 — 블로그 홈 보기·404·44px
5. **STOP and VALIDATE**: `e2e/blog-home.mjs`로 "가입 → 내 블로그 홈" 독립 확인
6. Deploy/demo

### Incremental Delivery

1. Setup + Foundational → 기반 완료 (B-M2는 town 단계 11 전에 merge)
2. US1 + US2 → 확인 → MVP
3. US3 → 이름·주소·닉네임 변경 확인 (공통 맥락 단계 2)
4. US5 B-M1(T048~T049)을 단계 2에서 먼저 merge → post 단계 3 진행 → US5 나머지 (단계 4)
5. US6 → 검색 확인 (단계 2)
6. US4 → town 단계 11 뒤 도감·전시 (단계 12)
7. US7 → 회귀 확인
8. Polish → 성능·문서·PR 정리

### Parallel Team Strategy

1. 팀이 Setup + Foundational을 함께 끝낸다
2. Foundational 뒤:
   - 개발자 A: US3 (블로그 관리 주소·기본 정보, 닉네임)
   - 개발자 B: US5 (B-M1 먼저, 카테고리 2단계)
   - 개발자 C: US6 (검색) → 선행이 풀리면 US4 (도감·전시)
3. 공유 파일(`actions.ts`, `settings-forms.tsx`, `[slug]/page.tsx`, `src/server/blog.ts`)은 작은 PR로 자주 merge해 충돌을 줄인다

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- 순수 규칙 테스트는 구현 전에 실패를 확인한다
- 마이그레이션 번호는 고정하지 않는다(`NNNN`). 앞선 브랜치가 merge되어 있으면 최신 `main`에서 `npm run db:generate`를 다시 돌린다
- 코드·마이그레이션 주석에 요구사항 ID(BLOG-0x, FR-0xx)를 단다 (원칙 II)
- 비밀값은 `.env.local`에만 두고 문서·PR에 적지 않는다
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
