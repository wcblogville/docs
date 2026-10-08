---

description: "회원 / 인증 (AUTH) 구현 작업 목록"
---

# Tasks: 회원 / 인증 (AUTH)

**Input**: Design documents from `/specs/001-auth/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: plan.md Technical Context와 quickstart.md가 단위 테스트(`npm run test:auth`)와 e2e 스크립트(`e2e/signup.mjs`, `e2e/session.mjs`, `e2e/login-limit.mjs`, `e2e/account.mjs`, 고친 `e2e/auth.mjs`)를 검증 수단으로 정했으므로 테스트 작업을 포함한다. CI가 없으므로 PR 작성자가 직접 돌리고 결과를 PR에 적는다 (NF-23).

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- 코드 경로는 모두 코드 저장소 `ehgo508/blogville` 루트 기준이다 (단일 Next.js 프로젝트: `src/app/`, `src/components/`, `src/lib/`, `src/server/`, `src/db/`, `scripts/`, `drizzle/`, `e2e/`).
- 문서 경로(`specs/001-auth/...`)는 이 문서 저장소 기준이다.
- 마이그레이션 파일 번호는 박지 않는다. 구현 때 최신 `main`에서 `npm run db:generate`(또는 `npx drizzle-kit generate --custom --name=<이름>`)로 만들고, SQL 맨 위에 요구사항 ID를 단 한국어 주석을 붙인다 (data-model.md 4장).
- plan.md의 단계 번호(1단계 가입 통합, 8단계 로그인 유지·시도 제한·내 정보·연동·관리자, 9단계 탈퇴)는 7개 spec 공통 구현 순서다. 아래 Phase는 user story 기준이며, 각 Phase에 해당 단계를 적었다.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 구현 전 확인과 테스트 스크립트 골격

- [x] T001 의존성을 설치한 체크아웃에서 [research.md 구현 전 확인 목록](research.md#구현-전-확인-목록) 16건(특히 3 세션 훅, 7 소셜 가입 막기 오류 코드·이메일 없는 소셜 계정, 9 서버 연동 호출, 12 Server Action Origin 불일치 응답 코드)을 Better Auth 1.7·Next.js 16 소스로 확인하고, 결과가 설계와 다르면 plan.md 남은 문제에 적는다 (`node_modules/better-auth`, `node_modules/next`)
- [x] T002 [P] `scripts/test-auth.ts`를 새로 만들고(기존 `scripts/test-*.ts` 형식, `❌` 출력·종료 코드 1 관례) `package.json`의 `scripts`에 `"test:auth": "tsx scripts/test-auth.ts"`를 추가하고 `test` 체인 끝(`test:game → test:ids → test:sanitize → test:auth`)에 잇는다
- [x] T003 [P] 마이그레이션 적용 전 점검 SQL(data-model.md 4장 "적용 전 점검": `username IS NULL` 회원, `username !~ '^[a-z0-9_]{4,20}$'` 회원, (`user_id`, `provider_id`) 2행 이상 묶음, 다른 회원 아이디와 같은 주소·`lower(닉네임)`)을 로컬 DB에서 돌려 0행인지 확인하고, 아니면 `npm run db:reset` 한다 (`.env.local`의 `ADMIN_USERNAME` 형식·`ADMIN_PASSWORD` 12자 이상도 함께 맞춘다, research R13)

---

## Phase 2: Foundational (Blocking Prerequisites) — 1단계

**Purpose**: 모든 user story와 다른 spec(blog·post·game·town·social)이 기대는 표 구조, 공용 모듈, 접근 규칙

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [x] T004 마이그레이션 A1 "가입 미완료 회원 정리"를 직접 쓴 SQL로 만든다: 프로필이 없는 `users` 행 삭제(세션·로그인 수단 CASCADE), 여러 번 실행해도 안전 (FR-007) in `drizzle/` (`npx drizzle-kit generate --custom --name=auth_drop_incomplete_members`)
- [x] T005 `src/db/schema.ts`의 `users` 블록을 고친다: `username` text **NOT NULL** UK + CHECK `users_username_check` (`username ~ '^[a-z0-9_]{4,20}$'`) (FR-002, FR-003). `profiles` 블록: CHECK `profiles_nickname_check`를 2~**20**자로 교체, `photo_key` **text NULL** + FK `profiles_photo_key_fk` → `attachments(key)` `ON DELETE SET NULL` 추가, 스키마 주석 "profiles 행이 있다 = 온보딩을 마친 회원"을 "가입 때 함께 생긴다"로 수정 (FR-009, ERD 3.9) in `src/db/schema.ts`
- [x] T006 T005를 바탕으로 `npm run db:generate`로 마이그레이션 A2 "아이디 필수·닉네임 20자"와 A3 "프로필 사진 칸"을 만들고(각 SQL 맨 위에 AUTH-07·AUTH-03 주석), A1 → A2 → A3 순서로 `npm run db:migrate` 적용 뒤 프로필 없는 회원 0명을 확인한다 in `drizzle/`
- [x] T007 [P] `newAuthId()`(32자 영문 대소문자·숫자)를 새로 만든다 (가입·관리자 스크립트 공용, FR-049) in `src/lib/auth-id.ts`
- [x] T008 [P] 이름 규칙 공용 모듈을 새로 만든다: `RESERVED_NAMES` 16개(지금 온보딩 안의 11개 + `notice` 등, 값은 blog FR-009 소유), `normalizeName(raw)`(앞뒤 공백 제거·소문자), `isReservedName(name)`, `USERNAME_RE = /^[a-z0-9_]{4,20}$/`, 닉네임 길이 2~20 확인 함수 (FR-002, FR-009, FR-010) in `src/lib/names.ts`
- [x] T009 서버 이름 모듈을 새로 만든다(`import "server-only"`): `lockName(tx, name)`(대소문자 무시 이름 단위 advisory 잠금), `findNameConflict(tx, name, { exceptUserId })` → `{ username, slug, nickname }`(다른 회원의 `users.username`·`blogs.slug`·`lower(profiles.nickname)`과 대소문자 무시 비교). blog가 주소·닉네임 변경에 그대로 쓴다 (FR-003, FR-009, FR-010, research R3) in `src/server/names.ts` (depends on T008)
- [x] T010 `src/server/dal.ts`를 고친다: `requireUser` 삭제, `getViewer()`가 `null` 또는 `{ userId, user, sessionId, profile: { nickname, characterAsset, blogId, blogSlug, blogTitle } }`(프로필 필수)를 돌려주게, `requireMember()`는 비로그인 → `/`, `requireAdmin()`은 비로그인·일반 회원 모두 `notFound()` (FR-007, FR-022, FR-043, research R12) in `src/server/dal.ts`
- [x] T011 `src/lib/auth.ts`에서 이메일 가입과 소셜 가입을 끈다(`disableSignUp`, 이메일·소셜 모두), 가입 요청으로 `role`을 정할 수 없게 한다 (FR-011, FR-013, research R5·R9) in `src/lib/auth.ts`
- [x] T012 라이브러리 HTTP를 허용 목록(세션 확인 `get-session`, OAuth 콜백, 임시로 `sign-in/social`)만 통과시키고 나머지는 404로 돌려주게 고친다 (FR-007, FR-011, FR-028, FR-041, contracts/auth-entry.md 6장) in `src/app/api/auth/[...all]/route.ts`
- [x] T013 [P] 온보딩 화면을 지운다: `src/app/onboarding/page.tsx`, `src/app/onboarding/actions.ts`, `src/app/onboarding/onboarding-form.tsx` 삭제 (FR-007)
- [x] T014 [P] 없어지는 라우트 참조를 정리한다 (town 합의): `src/app/town/page.tsx`의 온보딩 redirect 한 줄 삭제, `src/components/exit-button.tsx` `HIDDEN_ON`에서 `/onboarding` 삭제
- [x] T015 [P] 내 정보 화면 골격을 새로 만든다: `requireMember()`, 제목, blog가 닉네임 칸을 올릴 자리(빈 영역) (FR-036, blog 2단계 선행) in `src/app/settings/account/page.tsx`
- [x] T016 [P] 이름 규칙 단위 테스트를 추가한다: `" Tester_1 "` → `tester_1` 통과, 한글·특수문자·3자·21자 거부, `admin`·`Admin`·`settings`·`notice`·`onboarding` 예약어, `tester1`은 아님, 닉네임 2~20자만 통과 (quickstart 2장) in `scripts/test-auth.ts`

**Checkpoint**: Foundation ready — `npx tsc --noEmit`, `npx eslint`, `npm test` 통과. user story 구현을 시작할 수 있다

---

## Phase 3: User Story 1 - 아이디로 회원가입하고 바로 마을 주민이 되기 (Priority: P1) 🎯 MVP — 1단계

**Goal**: 아이디·비밀번호·비밀번호 확인·기본 캐릭터 4칸으로 가입하면 회원·로그인 수단·프로필·블로그·카테고리·기본 아이템·🪙 100이 한 트랜잭션으로 생기고, 로그인 상태로 광장에 도착한다

**Independent Test**: 로그인하지 않은 상태에서 첫 화면 [회원가입] 탭으로 가입하고, 광장 도착·헤더 레벨·코인·캐릭터·`/@{아이디}` 블로그·코인 100 지급을 확인한다 (`node e2e/signup.mjs shots`)

### Tests for User Story 1

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [x] T017 [P] [US1] `e2e/signup.mjs`를 새로 만든다: quickstart 4.2의 1~16번(첫 화면 표시, 대문자 아이디 + 여자 주민 가입 결과, 대문자 중복, 형식 오류, 비밀번호 7자/65자, 오류 뒤 입력 유지, 예약어 `admin`·`settings`·`notice`·`Admin`, 남의 주소·닉네임과 같은 아이디, 동시 가입, `role=admin`·비기본 캐릭터 조작, 트리거로 마지막 단계 실패 시 0행, 로그인 상태 `/` → `/town`, 키보드만으로 가입, 1분·4칸, 라이브러리 HTTP 직접 POST 404, 375px). 아이디는 `` `su${Date.now() % 100_000_000}` `` 관례 in `e2e/signup.mjs`
- [x] T018 [P] [US1] `loginDev`를 고친다: 가입 폼에서 캐릭터를 고르고 온보딩을 기다리지 않고 바로 `/town` 도착을 기다린다 in `e2e/helpers.mjs`
- [x] T019 [P] [US1] `e2e/auth.mjs` 1~4번을 고친다: 온보딩 확인 삭제, 가입 뒤 `/town`과 환영 문구 `{닉네임}님, Blogville에 오신 걸 환영해요! …`, `이미 있는 아이디예요`, `비밀번호가 서로 달라요`, 틀린/맞는 비밀번호 (quickstart 4.1) in `e2e/auth.mjs`

### Implementation for User Story 1

- [x] T020 [US1] 가입 트랜잭션 `createMember`를 새로 만든다(`import "server-only"`). data-model.md 2.6 순서 그대로: 0) `lockName(tx, 아이디)` → `findNameConflict` → 예약어면 `이 아이디는 쓸 수 없어요`, 겹치면 `이미 있는 아이디예요`, 고른 아이템이 `items.type = 'character' AND is_starter`인지와 `bg_meadow` 확인(아니면 거부, 제안 문구 `고를 수 없는 캐릭터예요`), 1) `users`(`id` = `newAuthId()`, `name` = 아이디, `email` = `{아이디}@users.blogville.invalid`, `email_verified` = false, `display_username` = 아이디, `role` 넣지 않음), 2) `accounts` credential(`hashPassword` 해시, `account_id` = `users.id`), 4) `user_items` (고른 캐릭터, `bg_meadow`), 5) `profiles`(`nickname` = 아이디, `character_item_id` = 고른 캐릭터), 6) `blogs`(`slug` = 아이디, `title` = `{아이디}의 블로그`, `description` = `''`, `background_item_id` = 초원), 7) `categories`("일상", `position` = 0), 8) `lockUser(tx, 회원)` → `grantReward(tx, 회원, "signup")`. UNIQUE 위반(`users_username_unique`, `users_email_unique`, `blogs_slug_unique`, `profiles_nickname_unique`)은 `이미 있는 아이디예요`로 바꾼다. 하나라도 실패하면 전부 롤백 (FR-003, FR-006, FR-010, FR-011, FR-012) in `src/server/signup.ts`
- [x] T021 [US1] `signUp(prev, formData)` Server Action을 고친다: zod로 아이디(`normalizeName` 뒤 `USERNAME_RE`, 실패 `아이디는 영문 소문자, 숫자, _ 로 4~20자예요`), 비밀번호 8~64자(8자 미만 `비밀번호는 8자 이상이에요`, 64자 초과 거부), 확인 불일치 `비밀번호가 서로 달라요`, 캐릭터 아이템 번호 검사 → `createMember` → 커밋 뒤 `signInUsername`으로 로그인 상태 만들기 → `/town?welcome=1`로 이동. 오류 시 입력한 아이디·캐릭터를 돌려준다 (FR-002~FR-005, FR-008, FR-011, contracts/auth-entry.md 2장, research R1·R2) in `src/app/(auth)/actions.ts`
- [x] T022 [US1] 가입 폼을 고친다: [로그인] [회원가입] 탭(처음 [로그인]), 아이디 칸 아래 `영문 소문자, 숫자, _ 로 4~20자`, 비밀번호 칸 `비밀번호 (8자 이상)`·`maxLength=64`, 캐릭터 고르기 2개(남자 주민 기본 선택), 오류는 [회원가입] 바로 위 빨간 굵은 글씨 한 줄, 처리 중 `가입하는 중...`·비활성, 탭·버튼 누르는 영역 44×44px·글자 한 줄·초점 테두리 (FR-001, FR-002, FR-004, FR-005, FR-054) in `src/components/login-buttons.tsx`
- [x] T023 [US1] 첫 화면을 고친다: 로그인한 회원은 `/town`으로 이동, 기본 캐릭터 목록(`is_starter` 캐릭터)을 서버에서 읽어 폼에 전달 (FR-001, FR-014, contracts/auth-entry.md 1장) in `src/app/page.tsx`
- [x] T024 [US1] `e2e/signup.mjs`, `e2e/auth.mjs` 1~4, 회귀 e2e(`blog`·`visits`·`params`·`write-count`·`attachments`·`farm`·`mobile`)를 돌려 가입·로그인 단계 실패가 없는지 확인하고 결과를 PR에 적는다. `e2e/social.mjs`의 "온보딩 전 회원" 준비 블록은 social에 교체를 요청한다 (quickstart 3장)

**Checkpoint**: User Story 1이 단독으로 동작한다. 다른 spec의 e2e 전제(새 `loginDev`)가 준비된다

---

## Phase 4: User Story 2 - 아이디로 로그인하고, 로그인 유지를 고르고, 로그아웃하기 (Priority: P1) — 8단계 (로그아웃은 1단계)

**Goal**: 아이디·비밀번호 로그인, [로그인 상태 유지] 없으면 브라우저 종료·마지막 사용 2시간 뒤 로그아웃, 고르면 7일(쓰는 동안 연장), 헤더 [로그아웃]

**Independent Test**: 테스트 회원으로 로그인·잘못된 비밀번호·없는 아이디·유지 선택/미선택·2시간 미사용·로그아웃을 차례로 시험한다 (`node e2e/session.mjs shots`)

### Tests for User Story 2

- [x] T025 [P] [US2] `e2e/session.mjs`를 새로 만든다: quickstart 4.3의 1~12번(쿠키 `HttpOnly`·`SameSite=Lax`·만료 없음, `remember_me = false`·`expires_at` ≈ +2시간, 새 컨텍스트 `/write` → `/`, 100분 → 2시간 재연장, 만료 → `/`, 4-2 `dont_remember` 쿠키 삭제 + `get-session` 뒤 `updated_at` 2시간 10분 전 → `/`·세션 행 없음, 유지 7일·새 컨텍스트 유지, `SessionKeeper` 재연장, 유지 만료, 로그아웃, `TESTER`·공백 로그인, 빈 칸 `아이디와 비밀번호를 적어 주세요`, `들어가는 중...`, 다른 Origin 상태 변경 요청 거부·데이터 그대로). 시간 조건은 DB의 `sessions.expires_at`·`updated_at`을 당겨 만든다 in `e2e/session.mjs`

### Implementation for User Story 2

- [x] T026 [US2] `src/db/schema.ts`에 `sessions.remember_me` **boolean NOT NULL DEFAULT false**와 새 표 `loginAttempts`(`login_attempts`: `username` text PK + CHECK `char_length(username) BETWEEN 1 AND 64`, `failed_count` integer NOT NULL DEFAULT 0 + CHECK `failed_count >= 0`, `locked_until` timestamptz NULL, `updated_at` timestamptz NOT NULL DEFAULT now(), `users` FK 없음)를 추가하고 `npm run db:generate`로 마이그레이션 A4 "로그인 유지·시도 제한"을 만들어 적용한다 (FR-019~FR-021, FR-025) in `src/db/schema.ts`, `drizzle/`
- [x] T027 [US2] `src/lib/auth.ts`에 세션 설정을 넣는다: `session.additionalFields.rememberMe`(→ `remember_me`), 유지 안 함 세션은 `expires_at` = 지금 + 2시간·쿠키 Max-Age 없음, 유지 세션은 7일·라이브러리 1시간 단위 연장, 세션 생성 훅, 쿠키 `HttpOnly`·`SameSite=Lax`·배포(`https`) 때 `Secure` (FR-020, FR-021, FR-023, research R6·R14) in `src/lib/auth.ts`
- [x] T028 [US2] `getSession()`에 2시간 규칙을 넣는다: `remember_me = false`이고 `updated_at < now() - 2시간`이면 세션 행을 지우고 로그아웃으로 처리, 아니면 5분 단위로 `expires_at = now() + 2시간`, `updated_at = now()` (요청당 UPDATE 1번 이하, `disableRefresh`) (FR-020, FR-022, SC-005) in `src/server/dal.ts`
- [x] T029 [P] [US2] 유지 세션에서만 그리는 `SessionKeeper`를 새로 만든다 (`GET /api/auth/get-session`으로 7일 연장, `src/lib/auth-client.ts` 사용) (FR-021) in `src/components/session-keeper.tsx`
- [x] T030 [US2] 헤더에 `SessionKeeper`를 유지 세션일 때만 그리게 추가한다 (town 소유 파일에 추가) in `src/components/site-header.tsx` (depends on T029)
- [x] T031 [US2] `signIn(prev, formData)` Server Action을 고친다: 아이디 `normalizeName`(앞뒤 공백·대문자 무시), 빈 칸 `아이디와 비밀번호를 적어 주세요`, 없는 아이디·틀린 비밀번호 모두 `아이디 또는 비밀번호가 맞지 않아요`, `rememberMe` 전달, 성공 시 `/town` (FR-015~FR-017, contracts/auth-entry.md 3장) in `src/app/(auth)/actions.ts`
- [x] T032 [US2] `signOut()` Server Action을 추가한다: 확인 없이 세션 행 삭제·쿠키 삭제 → `/` (FR-029, contracts/auth-entry.md 5장) in `src/app/(auth)/actions.ts`
- [x] T033 [P] [US2] 로그아웃 버튼을 `signOut` Server Action 폼으로 바꾸고 누르는 영역을 44×44px로 (FR-029, FR-054) in `src/components/sign-out-button.tsx`
- [x] T034 [US2] 로그인 폼에 [로그인 상태 유지] 체크박스(기본 선택 안 됨)와 처리 중 `들어가는 중...`·비활성 버튼을 넣는다 (FR-018, FR-019) in `src/components/login-buttons.tsx`

**Checkpoint**: User Stories 1과 2가 각각 단독으로 동작한다

---

## Phase 5: User Story 3 - 연동해 둔 소셜 계정으로 로그인하기 (Priority: P1) — 8단계

**Goal**: 연동한 카카오·네이버·구글 계정으로 비밀번호 없이 로그인. 연동하지 않은 소셜 계정은 새 회원을 만들지 않고 안내한다. 키가 없는 서비스 버튼은 비활성

**Independent Test**: 키가 없는 서비스 버튼 비활성 표시, 연동하지 않은 소셜 계정의 안내 문구·회원 수 불변, 미리 연동한 테스트 회원의 소셜 로그인 성공 (`e2e/signup.mjs` 1번 + quickstart 6장 수동 확인)

### Tests for User Story 3

- [x] T035 [P] [US3] 소셜 화면 표시 확인을 `e2e/signup.mjs` 1번에 맞춘다: 키 없는 서비스 버튼 흐림·비활성, 마우스를 올리면 `아직 연결 준비 중이에요`, 셋 다 없으면 `간편 로그인은 준비 중이에요`, `처음이라면 회원가입 후 내 정보에서 연동해 주세요`, `/?error=<연동 없음 코드>` → `연동된 계정이 없어요. 아이디로 로그인한 뒤 내 정보에서 연동해 주세요` in `e2e/signup.mjs`

### Implementation for User Story 3

- [x] T036 [US3] `src/lib/auth.ts`의 소셜 설정을 고친다: 이메일을 받지 않음(Google 범위 `openid profile`), Google에도 대체 이메일 `mapProfileToUser`, 소셜 토큰을 저장하지 않음(`databaseHooks.account.create.before`에서 토큰·만료·`scope` 칸 비움), 이메일 같음으로 자동 연결 끔, 연동 없는 소셜 계정은 새 회원을 만들지 않고 첫 화면 오류 코드로 돌려보냄 (FR-013, FR-032~FR-035, research R9) in `src/lib/auth.ts`
- [x] T037 [US3] 마이그레이션 A5 "소셜 토큰 비우기"를 직접 쓴 SQL로 만든다: `provider_id <> 'credential'`인 행의 `access_token`·`refresh_token`·`id_token`·두 만료 칸·`scope`를 NULL로, 여러 번 실행해도 안전 (FR-035) in `drizzle/`
- [x] T038 [US3] `startSocialSignIn(provider, remember)` Server Action을 추가한다: 키가 있는 서비스만 허용, `bv_remember` 쿠키로 [로그인 상태 유지] 전달, 라이브러리 소셜 로그인 주소로 이동 (FR-032, contracts/auth-entry.md 4장, research R7) in `src/app/(auth)/actions.ts`
- [x] T039 [US3] 소셜 버튼을 클라이언트 직접 호출에서 `startSocialSignIn` Server Action으로 바꾼다: `간편 로그인` 구분선 아래 [카카오](노랑) [네이버](초록) [Google](흰색) 3칸, 키 없는 버튼 비활성 + `아직 연결 준비 중이에요`, 셋 다 없으면 `간편 로그인은 준비 중이에요`, 안내 `처음이라면 회원가입 후 내 정보에서 연동해 주세요`, 44×44px (FR-030, FR-031, FR-054) in `src/components/login-buttons.tsx`
- [x] T040 [US3] 첫 화면이 서비스별 키 준비 여부와 소셜 오류(연동 없음 → `연동된 계정이 없어요. 아이디로 로그인한 뒤 내 정보에서 연동해 주세요`, 취소·미동의 → 문구 없이 그대로)를 폼에 전달하게 한다 (FR-031, FR-033) in `src/app/page.tsx`
- [x] T041 [US3] 허용 목록에서 임시로 열어 둔 `sign-in/social`을 뺀다 (T038 뒤, FR-028) in `src/app/api/auth/[...all]/route.ts`
- [ ] T042 [US3] 소셜 키가 있으면 quickstart 6장 수동 확인(연동 계정 로그인, 연동 없는 계정 안내·회원 수 불변, 같은 이메일 자동 연결 없음)을 하고 결과를 PR에 적는다. 키가 없으면 미확인 항목(SC-007, SC-012, US3 #1·#2·#6)을 PR에 남긴다 (plan 남은 문제 10)

**Checkpoint**: P1 세 이야기(가입·로그인·소셜 로그인)가 모두 동작한다

---

## Phase 6: User Story 4 - 내 정보에서 소셜 계정 연동·해제하기 (Priority: P2) — 8단계

**Goal**: 내 정보에서 서비스마다 `연동됨 (연동한 날짜)` 또는 [연동하기]를 보고, 연동·해제한다. 아이디 로그인은 지울 수 없다

**Independent Test**: 테스트 회원으로 카카오 연동 → 로그아웃 → [카카오] 로그인 → 연동 해제 → [카카오] 로그인 실패·아이디 로그인 성공 (키 없이: `node e2e/account.mjs shots` 1~11번, DB에 연동 행을 직접 넣음)

### Tests for User Story 4

- [x] T043 [P] [US4] `e2e/account.mjs`를 새로 만들고 quickstart 4.5의 1~11번을 넣는다: 비로그인 `/settings/account` → `/`, `아이디 로그인 · {아이디}`(해제 버튼 없음), [연동하기] 비활성, 블로그 관리 `내 정보` 링크·헤더 캐릭터 배지 이동, DB 연동 행 → `연동됨 ({오늘 날짜})` + [연동 해제], 연동 시작 조작 거부, 같은 회원 카카오 행 추가 UNIQUE 거부, 해제 확인 창 `카카오 연동을 해제할까요? 아이디 로그인은 그대로 쓸 수 있어요`, `credential` 해제 조작 거부, `?linked=kakao`(행 없음) 성공 문구 없음, `?provider=kakao&error=<코드>` → `이미 다른 Blogville 계정에 연동된 카카오 계정이에요`, 375px in `e2e/account.mjs`

### Implementation for User Story 4

- [x] T044 [US4] `src/db/schema.ts`의 `accounts`에 UNIQUE `accounts_user_provider_uq` (`user_id`, `provider_id`)를 추가하고 `npm run db:generate`로 마이그레이션 A6 "서비스마다 연동 1개"를 만들어 적용한다 (FR-038, FR-039) in `src/db/schema.ts`, `drizzle/`
- [x] T045 [US4] 회원의 로그인 수단 목록 조회(서비스별 연동 여부·`accounts.created_at` 연동 날짜)를 새로 만든다(`import "server-only"`) (FR-036) in `src/server/account.ts`
- [x] T046 [US4] `startLinkSocial(provider)`와 `unlinkSocial(provider)` Server Action을 새로 만든다: `requireMember()` 본인만, 이미 연동한 서비스·키 없는 서비스·`credential`은 거부, 해제는 그 서비스의 소셜 행만 삭제 (FR-037~FR-042, contracts/account.md 2·3장, research R10) in `src/app/settings/account/actions.ts`
- [x] T047 [P] [US4] 클라이언트 폼을 새로 만든다: 서비스별 [연동하기] / `연동됨` / [연동 해제], 해제 확인 창 `{서비스} 연동을 해제할까요? 아이디 로그인은 그대로 쓸 수 있어요`, 44×44px in `src/app/settings/account/account-forms.tsx`
- [x] T048 [US4] 내 정보 화면을 채운다: `아이디 로그인 · {아이디}` 줄(해제 버튼 없음), 카카오·네이버·Google 줄, 결과 문구 `{서비스} 계정을 연동했어요`(연동 행이 있을 때만)·`이미 다른 Blogville 계정에 연동된 {서비스} 계정이에요`, 375px 가로 스크롤 없음 (FR-036~FR-041, FR-054, contracts/account.md 1장; 표기 세부는 plan 남은 문제 3을 spec에서 확정한 뒤) in `src/app/settings/account/page.tsx`
- [x] T049 [P] [US4] 블로그 관리 화면에 `내 정보` 링크 한 줄을 추가한다 (blog 소유 파일에 추가) in `src/app/settings/blog/page.tsx`
- [x] T050 [US4] 헤더 캐릭터 배지를 `/settings/account` 링크로 만든다 (town 소유 파일에 추가, 상태창 입구는 plan 남은 문제 15) in `src/components/site-header.tsx`

**Checkpoint**: 연동·해제가 동작하고, 키가 있으면 User Story 3의 실제 소셜 로그인도 확인할 수 있다

---

## Phase 7: User Story 5 - 관리자가 통계·회원을 보고 아무 글이나 지우기 (Priority: P2) — 1단계(스크립트·온보딩 흔적) + 8단계(화면)

**Goal**: 운영자 설정으로 만든 관리자가 통계 카드 5개, 최근 가입 20명, 최근 30일 글 30개를 보고 글을 지운다. 관리자가 아니면 404

**Independent Test**: 관리자·일반 회원·다른 회원 글을 준비해 관리자 화면 표시·글 삭제·일반 회원/비로그인 404를 확인한다 (`node e2e/auth.mjs shots` 5~9번, quickstart 4.6)

### Tests for User Story 5

- [x] T051 [P] [US5] `e2e/auth.mjs` 5~9번을 고친다: 비로그인 `/admin` 404·`최근 가입` 없음, 일반 회원 `/admin` 404·관리자 글 삭제 조작 거부·[👑 관리자] 없음, 관리자 화면 카드 5개(`주민 (온보딩 완료)` 없음)·최근 가입(`온보딩 전` 없음, 로그인 방식 `아이디`)·최근 글, 삭제 확인 창 `'{제목}' 글을 삭제할까요?`, 375px 가로 스크롤 없음·[삭제]·링크 44px (quickstart 4.1) in `e2e/auth.mjs`

### Implementation for User Story 5

- [x] T052 [US5] 관리자 계정 스크립트를 고친다: `ADMIN_PASSWORD` 12~64자(미만이면 `관리자 비밀번호는 12자 이상이어야 해요`, 종료 코드 1, DB 변화 없음), `ADMIN_USERNAME` 형식 `^[a-z0-9_]{4,20}$`, 아이디와 같은 비밀번호 거부, 새 회원 ID는 `newAuthId()`, 없으면 관리자(`role = 'admin'`, 닉네임 `관리자`, `char_boy`·초원, 블로그 `notice` / `Blogville 공지사항` / 대분류 "공지", 🪙 100 없음)를 만들고 있으면 비밀번호만 갱신, 32자가 아닌 예전 ID는 바꾸지 않고 다시 만드는 방법만 안내 (FR-048, FR-049, contracts/admin.md 3장) in `scripts/create-admin.ts`
- [x] T053 [US5] 관리자 화면을 고친다: 통계 카드 5개(가입 계정 · 전체 글 · 댓글(삭제 제외, `comments` + social의 `replies`) · 오늘 새 글(한국 시간 0시 이후) · 오늘 출석), `주민 (온보딩 완료)`·`온보딩 전` 삭제, 최근 가입 최신 20명(아이디(로그인 방식 `아이디`/`카카오`/`네이버`/`Google`) · 닉네임 · 블로그 링크 · 권한 · 가입 시각), 최근 30일·최대 30개 글 최신순, 375px에서 표 대신 쌓인 목록, 링크 44px, `requireAdmin()` 404 (FR-043~FR-046, FR-054, contracts/admin.md 1장; `replies` 합산은 social 5단계 뒤) in `src/app/admin/page.tsx`
- [x] T054 [P] [US5] [삭제] 버튼 누르는 영역을 44×44px로 넓힌다 (확인 창 `'{제목}' 글을 삭제할까요?`와 `adminDeletePost` 동작은 그대로) (FR-047, FR-054) in `src/app/admin/delete-button.tsx`
- [x] T055 [US5] `npm run admin:create`를 quickstart 4.6 1~4번대로 확인한다(두 번 실행, 12자 미만, UUID 36자 예전 ID, 새 ID 32자) 결과를 PR에 적는다

**Checkpoint**: 관리자 기능이 단독으로 동작하고 관리자가 아니면 존재가 드러나지 않는다

---

## Phase 8: User Story 6 - 같은 아이디로 반복 실패하면 잠시 로그인 막기 (Priority: P2) — 8단계

**Goal**: 같은 아이디로 5번 연속 실패하면 5분 동안 아이디·비밀번호 로그인을 막는다. 없는 아이디도 같은 동작

**Independent Test**: 5번 틀린 뒤 6번째, 5분 경과 후, 성공 뒤 초기화, 없는 아이디의 같은 동작 (`node e2e/login-limit.mjs shots`)

### Tests for User Story 6

- [x] T056 [P] [US6] 로그인 제한 순수 함수 단위 테스트를 추가한다: 4번째 시도까지 잠금 아님, 5번째 시도 예약 → 5분 잠금, 잠금 중 시도 → 거부·변화 없음, 잠금이 풀린 뒤 시도 → 1, 성공 → 초기화 (quickstart 2장) in `scripts/test-auth.ts`
- [x] T057 [P] [US6] `e2e/login-limit.mjs`를 새로 만든다: quickstart 4.4의 1~8번(5번 실패 → 6번째 `로그인을 너무 많이 시도했어요. 5분 뒤에 다시 시도해 주세요`, `locked_until` 과거 → 성공·행 없음, 3번 실패 → 성공 → 4번 실패 → 성공, 없는 아이디도 같은 문구, 동시 10개, `Next-Action` 헤더 직접 5번, 잠긴 아이디 `/api/auth/sign-in/username` 404, 서로 다른/같은 아이디 12개 동시 요청 30초 안 응답) in `e2e/login-limit.mjs`

### Implementation for User Story 6

- [x] T058 [P] [US6] 5번·5분 규칙과 다음 상태 계산 순수 함수를 새로 만든다 (data-model.md 2.4 상태 전이: 행 없음 → n=1, n=1~3 → n+1, n=4 → 잠금(`locked_until` = now()+5분, n=0), 잠금 중 → 그대로·거부, 풀린 뒤 → n=1·`locked_until` NULL) (FR-025, FR-026) in `src/lib/login-limit.ts`
- [x] T059 [US6] 아이디별 실패 기록 모듈을 새로 만든다(`import "server-only"`): 비밀번호 확인 **전에** 짧은 트랜잭션에서 `pg_advisory_xact_lock(<로그인 잠금 번호>, hashtext(username))`으로 시도를 예약(잠금이면 거부), 성공 시 행 삭제, 트랜잭션 안에서 라이브러리나 다른 DB 연결을 쓰지 않음 (FR-025~FR-028, research R8) in `src/server/login-attempts.ts` (depends on T058)
- [x] T060 [US6] `signIn`에 시도 제한을 넣는다: 예약 트랜잭션이 끝난 뒤 라이브러리 로그인 호출, 잠금이면 `로그인을 너무 많이 시도했어요. 5분 뒤에 다시 시도해 주세요`(없는 아이디도 같은 문구), 성공 시 초기화. 소셜 로그인에는 적용하지 않음 (FR-025~FR-028, SC-004, SC-006) in `src/app/(auth)/actions.ts`
- [x] T061 [US6] `createMember`에서 그 아이디의 `login_attempts` 행을 지운다 (data-model.md 2.6 순서 3) in `src/server/signup.ts`
- [x] T062 [P] [US6] 개발 초기화에서 `login_attempts`도 비운다. 단 `to_regclass('public.login_attempts')`가 NULL이 아닐 때만 (FK가 없어 CASCADE로 안 비워짐, data-model.md 2.4) in `scripts/reset-dev.ts`

**Checkpoint**: 시도 제한이 화면·직접 요청 모두에 같게 적용되고, 아이디 존재 여부가 드러나지 않는다

---

## Phase 9: User Story 7 - 회원 탈퇴하기 (Priority: P2) — 9단계

**Goal**: 비밀번호를 다시 확인하면 회원에 딸린 데이터를 한 트랜잭션으로 지우고 로그아웃한다. 남의 답글이 달린 댓글은 내용·작성자 없이 `삭제된 댓글이에요` 자리만 남는다

**Independent Test**: 글·코인 내역·연동 계정, 남의 글에 단 댓글·답글(다른 회원 답글이 달린 댓글 포함)을 가진 회원으로 탈퇴한 뒤 로그인 불가·블로그 404·관련 데이터 0행·댓글 자리 처리 확인 (`node e2e/account.mjs shots` 12~20번)

**선행 조건**: social 5단계(`removeAuthorComments(tx, userId)` 가칭과 작성자 없는 `삭제된 댓글이에요` 자리), game 6단계(`notifications` FK `ON DELETE CASCADE`), shop 새 표 CASCADE, 탈퇴 화면 문구 확정(plan 남은 문제 1)

### Tests for User Story 7

- [ ] T063 [P] [US7] `e2e/account.mjs`에 quickstart 4.5의 12~20번을 추가한다: 회원 W 준비, 틀린 비밀번호 거부·행 수 그대로, 맞는 비밀번호 → `/`, W의 회원·로그인 수단·세션·프로필·블로그·글·원장·보유 아이템·출석·이웃·공감·첨부 행·실패 기록·알림 0행, W 아이디 로그인 `아이디 또는 비밀번호가 맞지 않아요`, `/@{W 주소}` 404, 댓글 자리 처리, HTML·RSC 응답에 W 닉네임·원문 0건, 같은 아이디 재가입 in `e2e/account.mjs`

### Implementation for User Story 7

- [ ] T064 [US7] 탈퇴 트랜잭션을 추가한다 (data-model.md 3장 순서): 1) `lockUser(tx, 회원)`, 2) social의 `removeAuthorComments(tx, 회원)`, 3) `login_attempts`에서 그 아이디 행 삭제, 4) `users` 행 삭제(CASCADE로 세션·로그인 수단·프로필·블로그·글·원장·아이템 등). 하나라도 실패하면 전부 롤백 (FR-051, FR-052, research R11) in `src/server/account.ts`
- [ ] T065 [US7] `deleteAccount(prev, formData)` Server Action을 추가한다: `requireMember()` 본인, 비밀번호 재확인(빈 칸·틀림 → 제안 문구 `비밀번호를 적어 주세요`·`비밀번호가 맞지 않아요`, 아무것도 지우지 않음), 성공 시 탈퇴 트랜잭션 → 세션 쿠키 삭제 → `/` (FR-050, contracts/account.md 4장) in `src/app/settings/account/actions.ts`
- [ ] T066 [US7] 탈퇴 폼(비밀번호 입력, 제안 버튼 [회원 탈퇴], 처리 중 비활성, 44×44px)을 넣고 내 정보 화면에 붙인다 in `src/app/settings/account/account-forms.tsx`, `src/app/settings/account/page.tsx`

**Checkpoint**: 모든 user story가 각각 단독으로 동작한다

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: 여러 이야기에 걸친 문서·접근성·최종 검증

- [ ] T067 [P] 375px 회원 컨텍스트 `/settings/account`, 비로그인 컨텍스트 `/` 가로 스크롤 없음 확인을 추가한다 (FR-054, SC-011, quickstart 4.7) in `e2e/nonfunctional.mjs`
- [ ] T068 [P] auth 담당 절을 고친다: 3.1·3.2(`users.username` NOT NULL·CHECK, 닉네임 2~20, `photo_key`), 3.3(`remember_me`, 2시간/7일 규칙), `login_attempts` 새 절, `accounts_user_provider_uq`, 3.14 탈퇴 삭제 규칙, 7장 (data-model.md 5장) in `docs/02-erd.md`
- [ ] T069 [P] 로그인·온보딩 규칙을 고친다: `requireUser` 삭제·`requireMember`/`requireAdmin` 404, 라이브러리 HTTP 허용 목록, 이름 공용 모듈 위치 in `CLAUDE.md`
- [ ] T070 [P] 기능 표(가입 통합, 로그인 유지, 시도 제한, 내 정보·연동, 탈퇴)와 스크립트 표(`test:auth`, `admin:create` 12자)를 고친다 in `README.md`
- [ ] T071 quickstart 5장 DB 기준을 확인한다: 프로필·블로그 없는 회원 0, `role = 'user'`인 예약어 아이디 0, 가입 직후 주소·닉네임 = 아이디, 남의 아이디와 같은 주소·닉네임 0, `pg_dump --data-only`·서버 출력에서 테스트 비밀번호 원문 0건, 소셜 행 토큰 칸 모두 NULL (SC-002, SC-010, SC-013, FR-035)
- [ ] T072 `npx tsc --noEmit`, `npx eslint`, `npm test`, quickstart 3장의 auth e2e 5개와 회귀 e2e 전체를 돌리고, 결과와 남은 문제(plan 1~15 중 미해결, 특히 CSRF 응답 코드·소셜 미확인 항목)를 PR에 적는다. `e2e/flow.mjs`가 낡았다는 사실은 고치지 않고 팀에 알린다 (plan 남은 문제 14)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 의존 없음. T001의 확인 결과가 설계와 다르면 해당 이야기 착수 전에 대안을 정한다
- **Foundational (Phase 2)**: Setup 뒤. 모든 user story를 막는다. 마이그레이션은 A1(T004) → A2·A3(T006) 순서
- **User Stories (Phase 3+)**: 모두 Foundational 뒤
  - US1(P1, 1단계)은 다른 spec의 e2e 전제라 가장 먼저 한다
  - US2 → US3, US6은 US2의 마이그레이션 A4(T026)와 `signIn`(T031)에 기댄다
  - US7(9단계)은 social 5단계·game 6단계가 끝난 뒤
- **Polish (Final Phase)**: 원하는 user story가 모두 끝난 뒤

### User Story Dependencies

- **User Story 1 (P1)**: Foundational 뒤 바로. 다른 이야기에 의존 없음 (MVP)
- **User Story 2 (P1)**: Foundational 뒤. 시험 회원을 만들 때 US1의 가입을 쓴다. 로그아웃(T032·T033)은 1단계라 US1과 함께 해도 된다
- **User Story 3 (P1)**: US2 뒤 (`bv_remember`가 [로그인 상태 유지] 설정 T027·T034에 기댐). 실제 로그인 성공 확인은 US4 연동(또는 DB에 넣은 연동 행)과 소셜 키가 있어야 한다
- **User Story 4 (P2)**: Foundational(T015 내 정보 골격) 뒤. US3의 소셜 설정(T036)이 있어야 실제 연동 흐름이 돈다. 키 없는 e2e는 단독으로 된다
- **User Story 5 (P2)**: Foundational(T010 `requireAdmin` 404) 뒤. 스크립트(T052)는 1단계, 화면(T053)의 `replies` 합산은 social 5단계 뒤
- **User Story 6 (P2)**: US2 뒤 (T026의 `login_attempts` 표, T031의 `signIn`). T061은 US1의 `src/server/signup.ts`를 고친다
- **User Story 7 (P2)**: US4(T045 `src/server/account.ts`, T046 `actions.ts`, T047 `account-forms.tsx`)와 US6(`login_attempts`) 뒤, 그리고 social·game 선행 조건 뒤

### Same-file 순서 (같은 파일을 여러 이야기가 고침)

- `src/app/(auth)/actions.ts`: T021(US1) → T031·T032(US2) → T038(US3) → T060(US6)
- `src/components/login-buttons.tsx`: T022(US1) → T034(US2) → T039(US3)
- `src/lib/auth.ts`: T011 → T027(US2) → T036(US3)
- `src/server/dal.ts`: T010 → T028(US2)
- `src/app/page.tsx`: T023(US1) → T040(US3)
- `src/app/api/auth/[...all]/route.ts`: T012 → T041(US3)
- `src/components/site-header.tsx`: T030(US2) → T050(US4)
- `src/db/schema.ts`: T005 → T026(US2) → T044(US4)
- `scripts/test-auth.ts`: T002 → T016 → T056(US6)
- `src/server/signup.ts`: T020(US1) → T061(US6)
- `src/server/account.ts`, `src/app/settings/account/*`: T015 → T045~T048(US4) → T064~T066(US7)
- `e2e/auth.mjs`: T019(US1) → T051(US5); `e2e/account.mjs`: T043(US4) → T063(US7); `e2e/signup.mjs`: T017 → T035(US3)

### Within Each User Story

- 테스트(e2e·단위)를 먼저 쓰고 실패하는지 확인한 뒤 구현한다
- 스키마·마이그레이션 → 서버 모듈(`src/server/*`, `src/lib/*`) → Server Action → 화면
- 이야기를 마치면 해당 e2e를 돌리고 결과를 PR에 적은 뒤 다음 우선순위로 간다

### Parallel Opportunities

- Setup: T002, T003
- Foundational: T007, T008, T013, T014, T015, T016 (T009는 T008 뒤, T006은 T005 뒤)
- US1: 테스트 T017, T018, T019
- US2: T025, T029, T033
- US4: T043, T047, T049
- US5: T051, T054 (T052 스크립트는 1단계에 US1과 함께 해도 된다)
- US6: T056, T057, T058, T062
- Polish: T067, T068, T069, T070
- US4·US5는 서로 다른 파일만 만지므로 Foundational 뒤 다른 사람이 동시에 진행할 수 있다

---

## Parallel Example: User Story 1

```bash
# User Story 1 테스트를 함께 시작:
Task: "e2e/signup.mjs 새로 작성 (quickstart 4.2 1~16번)"
Task: "e2e/helpers.mjs loginDev를 가입 폼 캐릭터 + 바로 광장으로 고침"
Task: "e2e/auth.mjs 1~4번 온보딩 확인 삭제"

# Foundational의 서로 다른 파일을 함께:
Task: "src/lib/auth-id.ts newAuthId() 작성"
Task: "src/lib/names.ts RESERVED_NAMES·normalizeName·isReservedName·USERNAME_RE 작성"
Task: "src/app/onboarding/* 삭제"
Task: "src/app/settings/account/page.tsx 골격 작성"
```

## Parallel Example: User Story 6

```bash
Task: "scripts/test-auth.ts 로그인 제한 단위 테스트 추가"
Task: "e2e/login-limit.mjs 새로 작성 (quickstart 4.4 1~8번)"
Task: "src/lib/login-limit.ts 5번·5분 규칙 순수 함수 작성"
Task: "scripts/reset-dev.ts login_attempts 조건부 TRUNCATE"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: Setup 완료
2. Phase 2: Foundational 완료 (모든 이야기를 막음)
3. Phase 3: User Story 1 완료
4. **STOP and VALIDATE**: `e2e/signup.mjs`·`e2e/auth.mjs` 1~4·회귀 e2e로 가입을 단독 검증
5. PR로 올려 다른 spec이 새 `loginDev`와 이름 모듈·내 정보 골격을 쓸 수 있게 한다 (plan 1단계)

### Incremental Delivery

1. Setup + Foundational + US1 (+ US5 스크립트 T052, 로그아웃 T032·T033) → 1단계 PR
2. US2 → US6 → US3 → US4 → US5 화면 → 8단계 PR들 (각 이야기 e2e 통과 뒤)
3. social·game 선행 조건이 끝나면 US7 → 9단계 PR
4. Polish → 문서·전체 회귀

### Parallel Team Strategy

3명 팀 기준:

1. 함께 Setup + Foundational + US1 (1단계)
2. 그 뒤:
   - 개발자 A: US2 → US6 (`src/app/(auth)/actions.ts`, `src/server/dal.ts`, `src/server/login-attempts.ts`)
   - 개발자 B: US4 → US7 (`src/app/settings/account/*`, `src/server/account.ts`)
   - 개발자 C: US5 화면 → US3 (`src/app/admin/*`, 소셜 설정). US3은 A의 T027·T034 뒤에 `src/lib/auth.ts`·`login-buttons.tsx`를 만진다
3. 같은 파일 순서(위 "Same-file 순서")를 지켜 충돌을 줄인다

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- 화면 문구는 spec 그대로 쓴다 (FR-053). spec에 없는 문구(탈퇴 오류·버튼, 캐릭터 거부, 내 정보 세부)는 "제안"이며 plan 남은 문제 1~3을 spec에서 확정한 뒤 넣는다
- 남의 파일(`site-header.tsx`, `settings/blog/page.tsx`, `town/page.tsx`, `exit-button.tsx`, `reset-dev.ts`, `package.json`, `nonfunctional.mjs`)은 소유 spec 규칙대로 한두 줄만 추가·정리한다 (공통 맥락 5.2)
- 비밀값은 환경 변수로만, 이름만 문서화한다. 비밀번호 원문을 로그에 남기지 않는다 (FR-012)
- Commit after each task or logical group, stop at any checkpoint to validate story independently
