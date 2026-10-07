# Data Model: 회원 / 인증 (AUTH)

**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Research**: [research.md](research.md) | **Date**: 2026-10-07

- 기준: 코드 저장소 `src/db/schema.ts`(지금), `docs/02-erd.md` v1.4(결정된 설계), 공통 맥락 3장 테이블 담당 규칙.
- 표기: 이 spec이 담당하는 테이블은 **변경**, 다른 spec이 담당하는 테이블은 **참조**. 구조 변경이 필요한 남의 테이블은 `참조 (요청: …)`.
- 실제 DB 타입은 지금처럼 `text` + 길이 CHECK, `integer`, `timestamptz`다 (ERD 부록 첫 문단).

## 1. 테이블 한눈에

| 테이블 | 구분 | 내용 | 단계 |
|---|---|---|---|
| `users` | 변경 | `username` NOT NULL + 형식 CHECK. 가입 미완료 회원 정리 (데이터) | 1 |
| `profiles` | 변경 | `nickname` CHECK 2~12 → 2~20 | 1 |
| `profiles` | 변경 (요청: blog FR-028, town FR-054, post FR-059 / ERD 7장 3) | `photo_key` NULL + FK → `attachments(key)` `ON DELETE SET NULL` | 1 |
| `sessions` | 변경 | `remember_me` boolean 추가 (로그인 유지 여부) | 8 |
| `login_attempts` | 변경 (새 테이블) | 아이디별 연속 실패 수·잠금 해제 시각 | 8 |
| `accounts` | 변경 | UNIQUE (`user_id`, `provider_id`) (ERD 7장 5). 소셜 토큰 칸 비우기 (데이터) | 8 |
| `verifications` | 변경 없음 (auth 담당) | 라이브러리가 OAuth state 등에 쓴다 | - |
| `blogs` | 참조 | 가입 때 1행 insert, 아이디 겹침 검사에서 `slug`를 읽는다 | 1 |
| `categories` | 참조 | 가입 때 "일상" 1행 insert | 1 |
| `items` | 참조 | 가입 때 기본 캐릭터(`type = 'character' AND is_starter`)와 `bg_meadow`를 찾는다 | 1 |
| `user_items` | 참조 | 가입 때 2행 insert (캐릭터, 초원). shop이 `quantity`를 더해도 기본값 1에 기댄다 | 1 |
| `point_ledger` | 참조 | 가입 축하 `grantReward(tx, userId, "signup")` (🪙 100, `REWARD_RULES`는 game 소유) | 1 |
| `posts` | 참조 | 관리자 글 삭제·통계, 탈퇴 CASCADE | 1, 9 |
| `attendances` | 참조 | 관리자 통계 "오늘 출석". game이 `session_id` → `sessions` `ON DELETE SET NULL`을 더한다 | 8 |
| `comments` | 참조 (요청: social이 탈퇴용 댓글 정리 구조, D2) | 탈퇴 트랜잭션에서 social 함수를 부른다. 관리자 통계 댓글 수 | 9 |
| `replies` (social 새 테이블) | 참조 (요청: social, D2) | 위와 같음. 관리자 통계 댓글 수에 더한다 | 8, 9 |
| `notifications` (game 새 테이블, 가칭) | 참조 (요청: game이 받는 회원·행동한 회원 FK를 CASCADE로, FR-051·game FR-046) | 탈퇴 때 함께 지워진다 | 9 |
| `attachments` | 참조 | `profiles.photo_key`가 가리킨다. 탈퇴 때 행은 CASCADE (요청: post 정리 작업이 행 없는 파일도 지움) | 1, 9 |
| `follows`, `post_likes`, `tags`·`post_tags`, `blog_visits`, `user_animals`, `animal_cares` | 참조 | 탈퇴 때 FK CASCADE로 함께 지워진다 (지금 구조 그대로) | 9 |
| shop 새 테이블 (아바타 착용, 가구 배치) | 참조 (요청: shop, 회원 삭제 때 CASCADE) | 탈퇴가 FK로 막히지 않아야 한다 | 9 |

## 2. 엔터티와 테이블

### 2.1 회원 — `users` (변경)

| 컬럼 | 지금 | 목표 | 메모 |
|---|---|---|---|
| `id` | text PK | 같음 | 32자 영문 대소문자·숫자. 가입·관리자 스크립트 모두 `newAuthId()` (`src/lib/auth-id.ts`) |
| `name` | text NOT NULL | 같음 | 가입 때 아이디. 화면에 쓰지 않는다 |
| `email` | text NOT NULL UK | 같음 | 가입 때 `{아이디}@users.blogville.invalid` (라이브러리 필수 칸, 메일 안 보냄, 화면에 안 나옴) |
| `email_verified` | boolean | 같음 | false |
| `image` | text NULL | 같음 | 쓰지 않는다 (소셜 사진을 받아 오지 않음) |
| `username` | text **NULL** UK | text **NOT NULL** UK + CHECK `users_username_check` (`username ~ '^[a-z0-9_]{4,20}$'`) | 소문자만 저장되므로 UNIQUE가 곧 대소문자 무시 유일 (FR-002, FR-003) |
| `display_username` | text NULL | 같음 | 가입 때 아이디(소문자) |
| `role` | user_role 기본 `user` | 같음 | 가입은 넣지 않아 늘 `user` (FR-011). `admin`은 `scripts/create-admin.ts`만 |
| `created_at`, `updated_at` | timestamptz | 같음 | 관리자 "최근 가입" 정렬 |

**검증 규칙**

- 아이디: 앞뒤 공백 제거 → 소문자 → `^[a-z0-9_]{4,20}$` (FR-002). 화면 칸 안내 `영문 소문자, 숫자, _ 로 4~20자`.
- 가입 때만: 예약어 16개(`RESERVED_NAMES`, blog FR-009)와 같으면 거부, 다른 회원의 `username`·`blogs.slug`·`lower(profiles.nickname)`과 같으면 거부 (FR-010). 이름 잠금(`lockName`) 안에서 확인한다 ([research.md R3](research.md#r3-예약어이름-겹침-검사-모듈과-동시성-fr-003-fr-009-fr-010-d1-edge-cases)).
- 관리자 계정은 이 가입 검사를 받지 않는다 (spec Assumption). 형식 CHECK는 받는다.

**상태 전이**

```text
(없음) ──회원가입: 한 트랜잭션으로 회원·로그인 수단·프로필·블로그·카테고리·아이템·코인──▶ 회원
회원 ──탈퇴: 한 트랜잭션, 비밀번호 확인──▶ (없음, 회원에 딸린 행 모두 삭제)
```

"가입했지만 프로필·블로그가 없는 회원"(지금의 "온보딩 전") 상태는 없다 (FR-007).

### 2.2 로그인 수단 — `accounts` (변경)

| 컬럼 | 지금 | 목표 | 메모 |
|---|---|---|---|
| `id` | text PK | 같음 | credential 행은 `newAuthId()`, 소셜 행은 라이브러리가 만든다 |
| `user_id` | FK → users CASCADE | 같음 | |
| `provider_id` | text | 같음 | `credential` / `kakao` / `naver` / `google` |
| `account_id` | text | 같음 | credential은 `users.id`, 소셜은 서비스 쪽 계정 식별자 |
| `password` | text NULL | 같음 | credential만, `hashPassword` 해시 (FR-012) |
| `access_token`, `refresh_token`, `id_token`, 두 만료 칸, `scope` | NULL 허용 | 같음 (늘 NULL) | 소셜 토큰을 저장하지 않는다 (`databaseHooks.account.create.before`, FR-035, VII) |
| `created_at` | timestamptz | 같음 | 내 정보의 "연동한 날짜" |
| 제약 | UK (`provider_id`, `account_id`) | + **UK `accounts_user_provider_uq` (`user_id`, `provider_id`)** | 소셜 계정 하나는 한 회원에만 + 한 회원에 서비스마다 1개 (FR-038, FR-039) |

**상태 전이 (서비스마다)**

```text
credential: 가입 때 생김 ──(지울 수 없음, FR-041)──▶ 회원 삭제 때 CASCADE
소셜:      없음 ──연동 (startLinkSocial → OAuth 콜백)──▶ 연동됨 ──해제 (unlinkSocial)──▶ 없음
```

### 2.3 로그인 세션 — `sessions` (변경)

| 컬럼 | 지금 | 목표 | 메모 |
|---|---|---|---|
| `id` | text PK | 같음 | game의 `attendances.session_id`가 가리킨다 (game 변경) |
| `user_id` | FK → users CASCADE | 같음 | |
| `token` | text UK | 같음 | 쿠키에 담기는 값 |
| `expires_at` | timestamptz | 같음 | 아래 규칙으로 정한다 |
| `ip_address`, `user_agent` | text NULL | 같음 | 라이브러리가 채운다 |
| `created_at` | timestamptz | 같음 | |
| `updated_at` | timestamptz | 같음 | 마지막 사용(근사): 유지 안 함은 5분 단위 연장 때, 유지는 1시간 단위 연장 때 바뀐다. 유지 안 함 세션은 이 값이 2시간보다 오래되면 `expires_at`과 상관없이 끝난다 (ERD 3.3) |
| `remember_me` | 없음 | **boolean NOT NULL DEFAULT false** | [로그인 상태 유지] 여부 (spec Key Entity). 라이브러리 `session.additionalFields.rememberMe` |

**규칙** ([research.md R6](research.md#r6-로그인-유지-기본-2시간브라우저-종료-선택-7일-fr-019fr-023-sc-005-auth-09))

| `remember_me` | 만들 때 `expires_at` | 쓰는 동안 | 쿠키 |
|---|---|---|---|
| false (기본, 가입 직후 포함) | 지금 + 2시간 | `getSession()`이 먼저 `updated_at < now() - 2시간`이면 행을 지우고 로그아웃으로 처리하고, 아니면 5분 단위로 `expires_at = now() + 2시간`, `updated_at = now()` | Max-Age 없음. `SessionKeeper`를 그리지 않는다 |
| true | 지금 + 7일 | `GET /api/auth/get-session`(SessionKeeper)이 1시간 단위로 `now() + 7일` | Max-Age 7일 |

**상태 전이**

```text
(없음) ──로그인(아이디·소셜)·가입 직후──▶ 사용 중 ──요청──▶ 사용 중(연장)
사용 중 ──expires_at 지남──▶ 무효 (라이브러리가 세션 없음으로 처리, 회원 화면은 `/`로)
사용 중(remember_me=false) ──updated_at 뒤 2시간 지남──▶ (getSession이 행 삭제, 회원 화면은 `/`로)
사용 중 ──로그아웃(signOut)──▶ (행 삭제)
사용 중 ──탈퇴·회원 삭제──▶ (CASCADE 삭제)
```

### 2.4 로그인 실패 기록 — `login_attempts` (새 테이블)

| 컬럼 | 타입 | 제약 | 메모 |
|---|---|---|---|
| `username` | text | PK, CHECK `char_length(username) BETWEEN 1 AND 64` | 정규화한 입력 그대로. 없는 아이디도 들어간다 (FR-027) |
| `failed_count` | integer | NOT NULL DEFAULT 0, CHECK `failed_count >= 0` | 마지막 성공·잠금 이후 연속 시도 수. 비밀번호를 확인하기 **전에** 실패로 미리 세고, 성공하면 행을 지운다 ([research.md R8](research.md#r8-로그인-시도-제한-fr-025fr-028-sc-004-sc-006-nf-10)) |
| `locked_until` | timestamptz | NULL 허용 | 이 시각 전까지 아이디·비밀번호 로그인 거부 |
| `updated_at` | timestamptz | NOT NULL DEFAULT now() | |

- `users`와 FK가 없다. 있으면 없는 아이디를 기록할 수 없고 존재 여부가 드러난다.
- 회원·글 삭제의 영향을 받지 않는다. 대신 코드가 지운다: 로그인 성공, 그 아이디로 가입, 그 회원의 탈퇴.
- `scripts/reset-dev.ts`의 `TRUNCATE users, tags … CASCADE`로는 비워지지 않으므로 이 표도 비운다 (공통 파일, 이유: FK가 없어 CASCADE로 안 비워짐). 단 표가 있을 때만 비운다 (`to_regclass('public.login_attempts')`가 NULL이 아닐 때). 4장 "적용 전 점검"이 실패하면 마이그레이션(A4) 전에 `npm run db:reset`을 돌리므로, TRUNCATE 목록에 그냥 넣으면 표가 없어 초기화가 실패한다.
- 규칙 숫자(5번, 5분)는 DB에 두지 않고 `src/lib/login-limit.ts`에 둔다 (CHECK에 넣으면 숫자를 바꿀 때 마이그레이션이 필요).

**상태 전이** (`n` = `failed_count`, 잠금 = `locked_until > now()`)

```text
예약 (짧은 트랜잭션, 비밀번호 확인 전)
(행 없음) ──시도──▶ n=1
n=k (1~3) ──시도──▶ n=k+1
n=4 ──시도(5번째)──▶ 잠금 (locked_until = now()+5분, n=0)
잠금 ──어떤 시도든──▶ 잠금 그대로, 비밀번호 확인 안 함, 잠금 문구 (FR-025)
잠금 풀림(locked_until ≤ now()) ──시도──▶ n=1, locked_until=NULL

비밀번호 확인 (트랜잭션 밖)
예약한 시도가 성공 ──▶ (행 삭제, 방금 건 잠금도 함께) (FR-026)
예약한 시도가 실패 ──▶ 그대로 (이미 셌다), 문구는 보통 실패
```

같은 아이디의 예약은 `pg_advisory_xact_lock(<로그인 잠금 번호>, hashtext(username))`로 한 줄로 처리한다. 라이브러리 로그인 호출은 그 트랜잭션이 끝난 뒤에 한다 (트랜잭션 안에서 부르면 연결 풀 교착, [research.md R8](research.md#r8-로그인-시도-제한-fr-025fr-028-sc-004-sc-006-nf-10)).

### 2.5 프로필 — `profiles` (변경)

| 컬럼 | 지금 | 목표 | 메모 |
|---|---|---|---|
| `user_id` | PK·FK → users CASCADE | 같음 | |
| `nickname` | text UK, CHECK 2~**12** | text UK, CHECK `profiles_nickname_check` 2~**20** | 가입 때 아이디 그대로 (4~20자라 늘 맞음). 닉네임 변경 규칙·문구는 blog (BLOG-03, FR-019) |
| `character_item_id` | integer + 복합 FK (`user_id`, `character_item_id`) → `user_items` | 같음 | 가입 때 고른 기본 캐릭터 |
| `photo_key` | 없음 | **text NULL**, FK `profiles_photo_key_fk` → `attachments(key)` `ON DELETE SET NULL` | 프로필 사진 (ERD 3.9). 올리는 화면은 담당 spec 없음. 표시는 blog·town |
| `created_at` | timestamptz | 같음 | |

- 스키마 주석 "profiles 행이 있다 = 온보딩을 마친 회원"을 "가입 때 함께 생긴다"로 고친다.
- 닉네임끼리의 대소문자 구분은 blog가 정한다. 지금 UNIQUE(대소문자 구분)를 유지한다. 다른 회원의 **아이디**와의 비교는 대소문자 무시(`findNameConflict`).

### 2.6 가입 때 함께 만드는 행 (참조 테이블)

한 트랜잭션, 이 순서 (`src/server/signup.ts`의 `createMember`):

| 순서 | 테이블 | 값 | 근거 |
|---|---|---|---|
| 0 | - | `lockName(tx, 아이디)` → `findNameConflict` → 고른 캐릭터·초원 확인 | FR-010, FR-011 |
| 1 | `users` | 위 2.1 | FR-006 |
| 2 | `accounts` | credential, 해시 | FR-006, FR-012 |
| 3 | `login_attempts` | 그 아이디의 행 삭제 (8단계부터) | R8 |
| 4 | `user_items` | (회원, 고른 캐릭터), (회원, `bg_meadow`) | GAME-01, FR-006 |
| 5 | `profiles` | `nickname` = 아이디, `character_item_id` = 고른 캐릭터 | FR-006, FR-009 |
| 6 | `blogs` | `slug` = 아이디, `title` = `{아이디}의 블로그`(최대 25자, CHECK 1~40), `description` = `''`, `background_item_id` = 초원 | FR-006, blog FR-001 |
| 7 | `categories` | `name` = "일상", `position` = 0 | blog FR-002 |
| 8 | `point_ledger` | `lockUser(tx, 회원)` → `grantReward(tx, 회원, "signup")` (🪙 100) | FR-006, game FR-002 |

하나라도 실패하면 전부 롤백된다 (FR-006, SC-003). UNIQUE 위반(`users_username_unique`, `users_email_unique`, `blogs_slug_unique`, `profiles_nickname_unique`)은 `이미 있는 아이디예요`로 바꾼다.

### 2.7 관리자 계정 (users.role = 'admin')

`scripts/create-admin.ts`만 만든다 (FR-048). 일반 가입과 같은 표·행(회원, credential, 기본 캐릭터 `char_boy`와 초원, 프로필, 블로그, 대분류)을 만들되 값이 다르다: `role = 'admin'`, `users.name` = `관리자`, `email_verified` = true(지금 스크립트 그대로), 닉네임 `관리자`, 블로그 `notice` / `Blogville 공지사항` / 대분류 "공지". 가입 축하 🪙 100은 주지 않는다 (지금 스크립트 그대로). 다시 실행하면 비밀번호만 갱신한다. 예전 형식(32자가 아닌) ID는 바꾸지 않고 안내만 한다 (FR-049).

## 3. 탈퇴 때 지워지는 것 (FR-051, FR-052)

`deleteAccount`의 트랜잭션 하나 ([research.md R11](research.md#r11-회원-탈퇴-fr-050fr-052-user-story-7-sc-014-d2)):

| 순서 | 처리 | 결과 |
|---|---|---|
| 1 | `lockUser(tx, 회원)` | 같은 회원의 보상·구매와 겹치지 않게 |
| 2 | social의 `removeAuthorComments(tx, 회원)` (가칭) | 이 회원의 답글 삭제, 남의 답글이 없는 이 회원의 댓글 삭제, 남의 답글이 달린 댓글은 내용·작성자 없는 `삭제된 댓글이에요` 자리 (D2). 구조는 social 결정 |
| 3 | `login_attempts`에서 그 아이디 행 삭제 | |
| 4 | `users` 행 삭제 | 아래 CASCADE |

CASCADE로 함께 지워지는 것 (지금 FK 기준 + 다른 spec이 만들 표):

| 테이블 | 경로 | 비고 |
|---|---|---|
| `sessions`, `accounts`, `profiles`, `user_items`, `follows`(양쪽), `post_likes`, `attendances`, `point_ledger`, `user_animals`, `attachments` | `users` CASCADE | 지금 그대로 |
| `blogs` → `categories`(→ `subcategories`), `posts` → `comments`·`replies`·`post_likes`·`post_tags`, `blog_visits` | `blogs.owner_id` CASCADE → 하위 CASCADE | 남이 그 글에 단 댓글·답글·공감도 함께. 남이 받은 보상은 회수하지 않는다 (spec Edge Case, D6) |
| `animal_cares` | `user_animals` CASCADE | |
| `notifications` (game) | 받는 회원·행동한 회원 FK CASCADE (요청) | game FR-046 |
| shop 새 표 | CASCADE (요청) | |
| `profiles.photo_key`, `blogs.showcase_animal_id`(blog) | 같은 문장에서 함께 지워짐 | NO ACTION 복합 FK도 문장 끝에 검사하므로 막히지 않는다 (지금 `profiles_character_owned_fk`와 같다) |

- 저장소 파일(`UPLOAD_DIR`)은 post의 정리 작업이 지운다 (요청).
- 회원을 가리키는 새 표는 모두 `ON DELETE CASCADE` 또는 `SET NULL`이어야 탈퇴가 FK로 막히지 않는다 (모든 spec에 대한 요청, plan 의존성).

## 4. 마이그레이션 순서와 데이터 이전

번호는 정하지 않는다 (공통 맥락 3.6). 구현 때 최신 `main`에서 `npm run db:generate`를 다시 돌린다. 생성된 SQL 맨 위에 요구사항 ID를 단 한국어 주석을 붙인다 (`drizzle/0006_blog_visits.sql` 관례).

| 순서 | 단계 | 마이그레이션 (내용 이름) | 종류 | 내용 |
|---|---|---|---|---|
| A1 | 1 | 가입 미완료 회원 정리 | 직접 쓴 SQL (`npx drizzle-kit generate --custom --name=<이름>`) | 프로필이 없는 회원 삭제 (세션·로그인 수단 CASCADE). 여러 번 실행해도 안전 |
| A2 | 1 | 아이디 필수·닉네임 20자 | 생성 (`npm run db:generate`) | `users.username` SET NOT NULL, `users_username_check` 추가, `profiles_nickname_check` 2~20으로 교체 |
| A3 | 1 | 프로필 사진 칸 | 생성 | `profiles.photo_key` + FK `ON DELETE SET NULL` |
| A4 | 8 | 로그인 유지·시도 제한 | 생성 | `sessions.remember_me` (기본 false), `login_attempts` 생성 |
| A5 | 8 | 소셜 토큰 비우기 | 직접 쓴 SQL | `provider_id <> 'credential'`인 행의 토큰·만료 칸을 NULL로 (여러 번 실행해도 안전) |
| A6 | 8 | 서비스마다 연동 1개 | 생성 | `accounts_user_provider_uq` (`user_id`, `provider_id`) |

9단계(탈퇴)는 auth 테이블 구조 변경이 없다. 댓글 구조 변경은 social 마이그레이션이다.

**적용 전 점검** (모두 0행이어야 한다. 결과가 있으면 로컬은 `npm run db:reset`. 배포 전이라 운영 데이터는 없다)

| 마이그레이션 | 점검 | 0이 아니면 |
|---|---|---|
| A2 | `username IS NULL`인 회원 (A1 뒤에 남은 것 = 프로필까지 만든 소셜 가입 회원) | NOT NULL이 실패한다. 아이디를 지어 주지 않는다 ([research.md R15](research.md#r15-기존-데이터-옮기기-erd-7장-5)) |
| A2 | `username !~ '^[a-z0-9_]{4,20}$'`인 회원 (예: 형식이 다른 `ADMIN_USERNAME`) | CHECK가 실패한다. `.env.local` 아이디를 고치고 관리자를 다시 만든다 |
| A6 | `accounts`에서 (`user_id`, `provider_id`)가 2행 이상인 묶음 | UNIQUE가 실패한다. 연동 기능이 없던 지금은 생길 수 없다 |
| (보고용) | 다른 회원의 아이디와 같은 블로그 주소 또는 `lower(닉네임)` | 마이그레이션은 실패하지 않는다. SC-013 위반이라 시험 DB는 `db:reset` 뒤에 쓴다 |

**데이터 이전 요약**

| 대상 | 방법 |
|---|---|
| 프로필 없는 회원 | 삭제 (A1) |
| 기존 회원의 닉네임(2~12자) | 그대로 (규칙이 넓어짐) |
| 기존 회원의 블로그 주소 ≠ 아이디 | 그대로 (주소는 나중에 바꿀 수 있는 값) |
| 기존 세션 | `remember_me = false` → 다음 요청부터 2시간 규칙 |
| 기존 소셜 토큰 | NULL로 (A5) |
| `login_attempts` | 빈 표로 시작 |

## 5. `docs/02-erd.md`에서 auth가 고칠 곳

| 위치 | 고칠 내용 |
|---|---|
| 1장 관계도 | `sessions`에 `remember_me`, `login_attempts` 엔터티 추가 (다른 표와 관계 없음, `verifications`처럼 설명), `users.username` 설명 "사이트 아이디 (모두 가짐, 소문자)" |
| 2장 테이블 그룹 | 인증 그룹에 `login_attempts` |
| 3.1 | ⏳ 표시 해제, 가입 트랜잭션 순서와 이름 잠금 한 줄 |
| 3.2 | ⏳ 해제(UK, NOT NULL). "가짜 이메일이 필요 없다" → "소셜 계정용 대체 이메일은 저장하지 않는다. 회원 이메일은 가입 때 `{아이디}@users.blogville.invalid`(라이브러리 필수 칸)". 소셜 토큰을 저장하지 않음 |
| 3.3 | ⏳ 해제, `remember_me`와 2시간·7일 연장 방식 (getSession 5분 단위, SessionKeeper) |
| 새 절 | 로그인 실패 기록: FK를 두지 않는 이유, 상태 전이 |
| 3.14 삭제 규칙 | 회원 → `login_attempts` 행은 코드가 지움, 첨부 → `profiles.photo_key` SET NULL, 탈퇴 때 댓글 처리는 3.8(social) |
| 3.17 BCNF | 후보 키에 `login_attempts.username`, `accounts (user_id, provider_id)` (이미 있음) |
| 7장 5·7 | 완료 표시 (가입 통합, 닉네임 2~20, 소셜 연동 화면) |
| 부록 | 글자 길이 `login_attempts.username VARCHAR(64)`, NULL 허용 목록에 `login_attempts.locked_until` |
