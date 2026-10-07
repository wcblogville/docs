# Research: 회원 / 인증 (AUTH)

**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Date**: 2026-10-07

근거 범위:

- 코드 저장소 `main` 커밋 `feb4c05`에서 읽은 파일. 경로는 코드 저장소 기준이다 (예: `src/lib/auth.ts`).
- 이 체크아웃에는 `node_modules`가 없어 Better Auth 1.7·Next.js 16의 설치된 문서를 열지 못했다. 라이브러리 동작에 기대는 부분은 **(추측)** 으로 표시했고, 맨 끝 [구현 전 확인 목록](#구현-전-확인-목록)에 모았다. 구현 전에 설치된 패키지 문서로 확인한다.
- 이 문서에 `NEEDS CLARIFICATION`은 남지 않았다. spec에 문구가 없어 설계로 정할 수 없는 것은 [plan.md의 남은 문제](plan.md#남은-문제-open-items)에 적었다.

---

## R1. 가입을 한 트랜잭션으로 묶는 방법 (FR-006, FR-007, FR-011, SC-002, SC-003)

**지금 코드**: `signUp`(`src/app/(auth)/actions.ts`)이 `auth.api.signUpEmail`로 `users`·`accounts`를 만들고 `/onboarding`으로 보낸다. 프로필·블로그·아이템·카테고리·코인은 `completeOnboarding`(`src/app/onboarding/actions.ts`)의 **다른** 트랜잭션에서 만든다. 그래서 "가입했지만 프로필이 없는 회원"이 정상 상태로 존재한다.

- **Decision**: Better Auth의 가입 API를 쓰지 않고, 우리 Drizzle 트랜잭션 하나에서 `users` → `accounts`(credential) → `user_items`(고른 캐릭터 + `bg_meadow`) → `profiles` → `blogs` → `categories`("일상") → `point_ledger`(`grantReward(tx, userId, "signup")`)를 차례로 넣는다. 새 서버 모듈 `src/server/signup.ts`의 `createMember()`가 맡고, `signUp` Server Action은 입력 검사와 결과 처리만 한다.
  - 비밀번호 해시: `better-auth/crypto`의 `hashPassword` (트랜잭션 밖에서 먼저 계산해 트랜잭션을 짧게 둔다).
  - 회원·로그인 수단 ID: Better Auth가 만드는 것과 같은 32자 (`generateRandomString(32, "a-z", "A-Z", "0-9")`). 새 `src/lib/auth-id.ts`의 `newAuthId()`로 두고 `scripts/create-admin.ts`도 같이 쓴다 (FR-049 "같은 형식"). 지금 이 식은 `scripts/create-admin.ts` 안의 `newId`에만 있다. 스크립트가 `tsx`로 바로 가져오므로 `auth-id.ts`는 `server-only`를 import하지 않는다.
  - credential 행: `provider_id = "credential"`, `account_id = users.id`, `password = 해시`.
  - `users.name` = 아이디, `users.email` = `{아이디}@users.blogville.invalid`, `display_username` = 아이디(소문자), `email_verified` = false. `role`은 넣지 않아 DB 기본값 `user`가 된다 (FR-011).
  - 고른 캐릭터는 트랜잭션 안에서 `items.type = 'character' AND is_starter = true`로 다시 확인한다 (지금 온보딩과 같은 방식).
  - 같은 트랜잭션에서 `lockUser(tx, userId)`를 건 뒤 `grantReward`를 부른다 (CLAUDE.md 보상 규칙).
- **Rationale**:
  1. 한 트랜잭션이면 어느 단계가 실패해도 아무것도 남지 않는다 (FR-006, SC-003). 반쪽 회원이 구조적으로 생기지 않는다 (FR-007, SC-002).
  2. `scripts/create-admin.ts`가 이미 같은 방식(직접 insert + `hashPassword` + `accountId = user.id`)으로 관리자를 만들고, 관리자는 첫 화면의 `signIn`(`signInUsername`)으로 로그인한다. `e2e/auth.mjs`의 "관리자 로그인 → 광장" 시나리오가 이것을 확인하고, 원본 AUTH-08은 이 스크립트를 "구현됨"으로 적었다. 직접 만든 행이 라이브러리 로그인과 맞는다는 근거다 (이번 작업에서 e2e를 실행하지는 않았다).
  3. 이메일은 라이브러리 스키마에서 NOT NULL·UNIQUE라 비울 수 없다. 지금 가입과 같은 `.invalid` 주소를 넣는다. 메일을 보내지 않고 화면에 나오지 않는다. spec Assumption의 "대체 이메일이 필요 없다"는 소셜 계정용 대체 이메일(`{서비스}_{ID}@…`) 이야기이고 R9에서 다룬다.
- **Alternatives considered**:
  - `signUpEmail` + `databaseHooks.user.create.after`에서 나머지 생성: 훅이 사용자 insert와 같은 트랜잭션에서 도는지는 어댑터 설정에 달렸다 **(추측)**. 훅이 실패하면 사용자 행이 남을 수 있다. 거절.
  - `signUpEmail` 뒤 우리 트랜잭션, 실패하면 사용자 삭제: 두 단계 사이에 서버가 죽으면 반쪽 회원이 남는다. 거절.
- 라이브러리 가입 경로는 `emailAndPassword.disableSignUp: true`와 HTTP 허용 목록(R5)으로 닫는다.

## R2. 가입 직후 로그인 상태 만들기 (FR-008)

- **Decision**: 커밋 뒤 `auth.api.signInUsername({ body: { username, password, rememberMe: false }, headers })`를 부른다. `nextCookies()` 플러그인이 Server Action 안에서 세션 쿠키를 심는다. 이어서 `revalidatePath("/", "layout")` → `redirect("/town?welcome=1")`.
- **Rationale**: 세션 행과 쿠키 서명 형식을 라이브러리에 맡긴다. 지금 `signIn`이 같은 호출을 쓴다. 가입 화면에는 [로그인 상태 유지]가 없으므로 기본값(유지 안 함, R6)이다.
- **Alternatives considered**: 세션 행을 직접 만들고 쿠키를 서명 — 라이브러리 내부 형식에 기댄다. 거절.
- 커밋 뒤 로그인만 실패하는 경우(예상 밖 오류)에는 회원 데이터가 온전하므로 첫 화면에서 다시 로그인하면 된다. 반쪽 회원은 아니다.
- 환영 문구 연결: 지금 코드처럼 `?welcome=1`로 넘긴다. "한 번만 보이기"(TOWN-01 FR-004)는 town이 정한다. town이 다른 신호(예: 일회용 쿠키)를 고르면 `signUp`이 그 신호를 심는다 (의존성).

## R3. 예약어·이름 겹침 검사 모듈과 동시성 (FR-003, FR-009, FR-010, D1, Edge Cases)

**지금 코드**: 예약어는 온보딩의 `RESERVED_SLUGS` 11개뿐이고 주소에만 쓴다. 가입 아이디와 다른 회원의 주소·닉네임 비교가 없다. `profiles.nickname` UNIQUE는 대소문자를 구분한다.

- **Decision**:
  - `src/lib/names.ts` (DB 없음, 화면·서버·테스트 공용, 모듈 소유 auth / 값 소유 blog):
    - `RESERVED_NAMES` — blog FR-009의 16개: `admin`, `api`, `town`, `feed`, `shop`, `closet`, `write`, `settings`, `blog`, `onboarding`, `farm`, `attendance`, `tags`, `wallet`, `files`, `notice`.
    - `normalizeName(raw)` — 앞뒤 공백 제거 + 소문자.
    - `isReservedName(name)` — 정규화한 값이 목록에 있는지.
    - `USERNAME_RE = /^[a-z0-9_]{4,20}$/`.
  - `src/server/names.ts` (`server-only`, 모듈 소유 auth):
    - `lockName(tx, name)` — `pg_advisory_xact_lock(<이름 잠금 번호>, hashtext(lower(name)))`.
    - `findNameConflict(tx, name, { exceptUserId? })` — 다른 회원의 `users.username`, `blogs.slug`, `lower(profiles.nickname)` 가운데 `lower(name)`과 같은 것이 있는지 `{ username, slug, nickname }` 세 값으로 돌려준다.
  - 가입 순서: 정규화 → 형식 → 예약어(`이 아이디는 쓸 수 없어요`) → 트랜잭션 시작 → `lockName` → `findNameConflict`가 하나라도 참이면 `이미 있는 아이디예요` → insert. UNIQUE 위반(`users_username_unique`, `users_email_unique`, `blogs_slug_unique`, `profiles_nickname_unique`)도 `이미 있는 아이디예요`로 바꾼다 (`uniqueViolation()`, `src/server/db-errors.ts`).
  - blog의 주소·닉네임 변경도 같은 두 함수를 같은 순서(잠금 → 검사 → update)로 쓴다. 자기 아이디와 같은 값은 `exceptUserId`로 허용한다 (FR-009 "그 회원만").
- **Rationale**:
  - 아이디·주소·닉네임은 서로 다른 표의 칸이라 UNIQUE 하나로 "서로 겹치지 않음"을 막을 수 없다 (공통 맥락 5.3). 같은 값 X에 대한 가입과 변경이 같은 이름 잠금을 잡으면 줄을 서고, 뒤에 온 쪽은 앞 쪽이 커밋한 행을 본다. READ COMMITTED에서는 잠금을 얻은 뒤 시작하는 새 문장이 그 사이 커밋된 행을 보기 때문이다. 그래서 Edge Case "한쪽만 성공"을 지킨다.
  - 한 트랜잭션이 이름 잠금을 하나만 잡으므로(가입은 아이디 = 주소 = 닉네임이 같은 값 하나) 교착이 없다.
  - 두 정수 키 형식 `pg_advisory_xact_lock(int, int)`은 `lockUser`(`src/server/points.ts`)의 한 정수 키 형식과 키 공간이 겹치지 않는다 (PostgreSQL 문서: 두 형식의 키 공간은 겹치지 않는다).
- **Alternatives considered**:
  - 이름 등록 표(`name` PK + 종류 + 회원)를 두고 세 값을 모두 등록 — UNIQUE 하나로 막히지만 표가 하나 늘고 세 표와 늘 맞춰야 한다. 3명 팀 규모에 과하다 (VII). 거절.
  - SERIALIZABLE 격리 — 직렬화 실패 재시도 루프가 필요하다. 거절.
  - 잠금 없이 검사만 — 동시 요청에서 둘 다 통과할 수 있다. 거절.

## R4. 아이디 대소문자와 DB 제약 (FR-002, FR-003, FR-016)

- **Decision**: `users.username`을 NOT NULL로 바꾸고 CHECK `users_username_check` (`username ~ '^[a-z0-9_]{4,20}$'`)를 더한다. 기존 UNIQUE와 합쳐 "대소문자를 무시해도 유일"을 DB가 보장한다. 로그인은 입력을 정규화한 뒤 비교한다 (지금 `signIn`과 같다).
- **Rationale**: 앱은 이미 소문자로 정규화하지만 constitution V는 규칙을 DB 제약으로도 막으라고 한다. 소문자만 저장되므로 일반 UNIQUE가 곧 대소문자 무시 UNIQUE다.
- **Alternatives considered**: `lower(username)` UNIQUE 인덱스 — 대문자 저장을 허용하게 되어 비교할 때마다 `lower()`가 필요하다. 거절.
- 닉네임끼리의 대소문자(`Tester`와 `tester`를 다른 닉네임으로 볼지)는 BLOG-03(닉네임 변경) 규칙이라 blog가 정한다. auth는 지금 `profiles_nickname_unique`(대소문자 구분)를 유지한다. blog가 "대소문자 무시"로 정하면 auth가 `lower(nickname)` UNIQUE 인덱스로 바꾼다 (요청을 받는 쪽, 의존성).

## R5. Better Auth HTTP 경로 줄이기 (FR-007, FR-011, FR-028, FR-039, FR-041, constitution IV)

**지금 코드**: `src/app/api/auth/[...all]/route.ts`가 `toNextJsHandler(auth)`로 라이브러리의 모든 경로를 연다. 화면을 거치지 않고 다음을 부를 수 있다.

| 경로 | 문제 |
|---|---|
| `POST /api/auth/sign-up/email` | 프로필·블로그 없는 회원이 생긴다 (FR-007). 예약어·이름 겹침 검사를 건너뛴다 (FR-010) |
| `POST /api/auth/sign-in/username`, `/sign-in/email` | 우리 로그인 시도 제한을 건너뛴다 (FR-028) |
| `POST /api/auth/update-user` | 이름·사진, username 플러그인이면 아이디까지 바꿀 수 있다 **(추측)** |
| `POST /api/auth/link-social` | "서비스마다 1개" 검사를 건너뛴다 (FR-039) |
| `POST /api/auth/unlink-account` | 다른 수단이 남아 있으면 `credential`도 지울 수 있다 **(추측)** (FR-041) |

- **Decision**: route.ts를 **허용 목록**으로 감싼다. 아래만 라이브러리로 넘기고 나머지는 빈 본문 404를 돌려준다.

  | 메서드 | 경로 | 이유 |
  |---|---|---|
  | GET | `/api/auth/get-session` | `SessionKeeper`의 로그인 유지 연장 (R6) |
  | GET, POST | `/api/auth/callback/{google,kakao,naver}` | 소셜 로그인·연동에서 돌아오는 곳 |
  | GET | `/api/auth/error` | 라이브러리 오류 화면 (우리는 늘 `errorCallbackURL`을 주지만 안전망) |
  | POST | `/api/auth/sign-in/social` | **1단계 동안만 임시**. 첫 화면 소셜 버튼이 아직 `authClient.signIn.social`을 쓴다. 8단계에서 Server Action(`startSocialSignIn`)으로 바꾸며 뺀다 |

  가입·로그인·로그아웃·연동·해제·탈퇴는 모두 우리 Server Action이 서버에서 `auth.api.*`를 부르거나 DB를 직접 다룬다. 서버의 `auth.api.*` 호출은 HTTP 라우트를 거치지 않으므로 허용 목록의 영향을 받지 않는다. 로그아웃은 1단계에서 Server Action `signOut`으로 바꾼다.
  - 함께: `emailAndPassword.disableSignUp: true`, 소셜 공급자마다 `disableSignUp: true` (R9).
- **Rationale**: 권한과 검증을 우리 Server Action에 모은다 (constitution IV). 라이브러리를 올렸을 때 새로 생기는 경로도 기본으로 닫힌다. 몇 줄이고 e2e가 경로를 직접 POST해 404를 확인할 수 있다.
- **Alternatives considered**:
  - Better Auth `disabledPaths` 옵션 — 1.7에 있는지, 서버 `auth.api` 호출까지 막는지 확인이 필요하고 **(추측: 라우터에만 적용)**, 금지 목록이라 새 경로를 놓칠 수 있다. 거절 (허용 목록의 보조로는 써도 된다).
  - `hooks.before`로 경로마다 거부 — 같은 이유로 거절.

## R6. 로그인 유지: 기본 2시간·브라우저 종료, 선택 7일 (FR-019~FR-023, SC-005, AUTH-09)

**지금 코드**: `src/lib/auth.ts`에 `session` 설정이 없어 라이브러리 기본값이다 (원본 AUTH-09: "지금은 세션 기본값(7일)"). `signIn`은 `rememberMe`를 넘기지 않는다. `getSession`(`src/server/dal.ts`)은 Server Component에서 `auth.api.getSession`을 부른다.

**라이브러리 동작 (추측, 구현 전 확인)**:

- `rememberMe: false`로 로그인하면 세션 쿠키를 Max-Age 없이(브라우저를 닫으면 사라짐) 심고, 서명한 `dont_remember` 쿠키를 함께 심는다. 이런 세션은 만료를 1일로 두고 갱신하지 않는다.
- 보통 세션은 `updateAge`가 지날 때마다 `expiresAt`을 지금 + `expiresIn`으로 늘리고 쿠키도 다시 심는다.
- Server Component에서는 쿠키를 쓸 수 없어 `nextCookies()`가 쿠키 쓰기를 건너뛴다. 그러면 DB 만료만 늘고 쿠키 Max-Age는 로그인 때 값으로 남는다.

라이브러리 설정에는 만료 기간이 하나뿐이라 "유지 안 함 2시간"과 "유지 7일" 가운데 하나는 우리 코드가 맡아야 한다. 원본 AUTH-09의 구현 제안(`expiresIn = 2시간`, `updateAge` 짧게)은 유지 세션도 서버에서 2시간이 되어 7일을 만들지 못한다.

- **Decision**:
  1. `sessions.remember_me boolean NOT NULL DEFAULT false`를 더한다 (Better Auth `session.additionalFields.rememberMe`, `input: false`). 서버는 쿠키가 아니라 이 칸으로 정책을 정한다. spec Key Entity "로그인 세션: … [로그인 상태 유지] 여부를 가진다"와 같다.
  2. 라이브러리 설정: `session: { expiresIn: 7일, updateAge: 1시간 }`. 유지 세션은 라이브러리의 연장 그대로 쓴다.
  3. 세션을 만들 때 `databaseHooks.session.create.before`에서 `rememberMe`와 `expiresAt`을 정한다. 아이디 로그인은 요청 본문의 `rememberMe`, 소셜 로그인은 `bv_remember` 쿠키(R7)를 본다. 유지 안 함이면 `expiresAt = 지금 + 2시간`.
  4. 유지 안 함 세션의 연장은 우리 `getSession()`(`src/server/dal.ts`, `cache`라 요청당 한 번)이 맡는다. `remember_me = false`인 세션이면 다음 두 가지를 이 순서로 한다.
     1. **마지막 사용 확인**: `updated_at < now() - 2시간`이면 `expires_at`과 상관없이 만료로 본다. 그 세션 행을 지우고 "세션 없음"(`null`)을 돌려준다. ERD 3.3의 "마지막 사용(`updated_at`) 후 2시간"을 서버가 직접 지키는 줄이다. 아래 6번의 라이브러리 연장(`GET /api/auth/get-session`)은 `dont_remember` 쿠키가 없으면 유지 안 함 세션의 `expires_at`도 7일로 늘린다 **(추측)**. 그 뒤 2시간 넘게 요청이 없어도 `expires_at`이 아직 남아 있어 다음 요청에서 세션이 살아 있고, 아래 2의 UPDATE는 그때 다시 2시간으로 늘릴 뿐이다. 그래서 `expires_at`만으로는 FR-020을 지킬 수 없다. 라이브러리의 세션 갱신도 `updated_at`을 바꾸므로(`src/db/schema.ts`의 `updatedAt()`에 `$onUpdate`) 이 확인은 "마지막 요청"을 기준으로 한다.
     2. **연장**: 조건부 UPDATE 하나를 보낸다: `expires_at = now() + 2시간`, `updated_at = now()`, 조건 `remember_me = false AND (expires_at < now() + 115분 OR expires_at > now() + 2시간)`. 그래서 세션마다 5분에 한 번만 쓴다. 뒤 조건(`> 2시간`)은 라이브러리가 만료를 1일·7일로 잡은 경우를 2시간으로 되돌리는 안전망이다 (3번 훅이 안 될 때도 지켜진다).
  5. Server Component의 `auth.api.getSession` 호출에는 `query: { disableRefresh: true }`를 준다. 쿠키를 쓸 수 없는 곳에서 DB 만료만 늘어나 쿠키와 어긋나는 일을 막는다.
  6. 유지 세션의 연장(DB + 쿠키 Max-Age)은 새 컴포넌트 `SessionKeeper`(`src/components/session-keeper.tsx`)가 `GET /api/auth/get-session`을 부를 때 라이브러리가 한다. Route Handler라 쿠키를 다시 심을 수 있다. 처음 그릴 때와 창이 다시 보일 때(`visibilitychange`) 부르되 10분에 한 번까지만 부른다. `SessionKeeper`는 서버 컴포넌트 껍데기가 `getSession()`의 `rememberMe`를 보고 **유지 세션일 때만** 클라이언트 부분을 그린다 (유지 안 함 세션에서 라이브러리 연장이 일어나 쿠키에 Max-Age 7일이 다시 심기지 않게). 헤더의 회원 영역에 한 줄 추가하고, 헤더는 town 소유다.
  7. 만료된 세션은 라이브러리가 "세션 없음"으로 돌려주고 `requireMember()`가 `/`로 보낸다 (FR-022).
- **결과**:

  | | 쿠키 | 서버 만료 |
  |---|---|---|
  | 유지 안 함 (기본) | Max-Age 없음 → 브라우저를 닫으면 사라짐 | 서버가 본 마지막 요청 + 2시간 (5분 단위로 늘려 1시간 55분~2시간). `updated_at`이 2시간보다 오래되면 `expires_at`과 상관없이 끝 |
  | 유지 | Max-Age 7일 | 마지막 연장 + 7일. 쓰는 동안 1시간 단위로 다시 7일 |

- **Rationale**: 모든 회원 화면과 Server Action이 이미 지나는 `getSession()` 한 곳에서 짧은 쪽을 맡는 것이 가장 단순하다. 서버 기준(칸)이 있어 클라이언트가 쿠키를 조작해도 2시간 규칙이 유지된다.
- **Alternatives considered**:
  - `dont_remember` 쿠키만으로 구분 — 클라이언트가 그 쿠키만 지우면 라이브러리가 7일로 늘린다. 서버 기준이 없다. 거절 (쿠키는 보조로만. 그래서 4-1의 `updated_at` 확인을 서버 기준으로 둔다).
  - `proxy.ts`(Next 16에서 `middleware`의 새 이름)에서 모든 요청마다 연장 — 정적 파일 요청까지 DB를 건드리고 새 계층이 생긴다. 지금 코드에 `proxy.ts`가 없다. 거절.
  - 매 요청 `updated_at` 갱신 — 쓰기가 너무 많다. 5분 단위로 줄였다.
- **영향**: 자동 출석(game, GAME-04)은 "그날 처음 만든 세션"을 쓴다. 세션이 더 자주 새로 생기지만 규칙은 그대로다 (spec Edge Case). game이 쓸 수 있게 `getViewer()`가 `sessionId`를 돌려준다.
- **기존 세션**: 칸 기본값 false라 마이그레이션 뒤 다음 요청부터 2시간 규칙을 따른다 (기존 7일 쿠키가 남아도 서버가 2시간 미사용이면 끊는다).
- **"사용"의 범위**: 서버가 세션을 확인하는 요청(회원 화면, 헤더를 그리는 화면, Server Action)이 "사용"이다. 한 화면에 머물며 요청이 없으면 서버는 사용을 알 수 없다 (spec Edge Case "서버는 창이 닫힌 것을 알 수 없다"와 같은 한계).

## R7. 소셜 로그인의 [로그인 상태 유지] (FR-019~FR-021, User Story 3)

spec은 체크박스가 "로그인 화면"에 있다고만 적었고, 같은 카드의 소셜 버튼에 적용할지는 적지 않았다.

- **Decision**: 같은 카드의 체크박스를 소셜 버튼에도 적용한다.
  - 소셜 버튼 → Server Action `startSocialSignIn(provider, remember)`가 httpOnly 쿠키 `bv_remember`(값 `0`/`1`, Max-Age 600초, SameSite=Lax, 배포 때 Secure)를 심고 `auth.api.signInSocial`이 돌려준 소셜 서비스 주소를 돌려준다. 화면이 그 주소로 이동한다.
  - 콜백에서 R6-3의 세션 훅이 `bv_remember`를 읽어 `remember_me`·만료를 정한다.
  - `hooks.after`(경로 `/callback/*`)가 유지 안 함이면 `setSessionCookie(ctx, newSession, true)`로 세션 쿠키를 브라우저 종료형으로 다시 심고, `bv_remember`를 지운다.
- **Rationale**: 사용자는 한 카드에서 체크박스를 보고 버튼을 누른다. 쿠키 재설정이 안 되더라도 서버 2시간 규칙(R6-4)은 지켜진다.
- **Alternatives considered**: 소셜은 늘 7일(라이브러리 기본) — constitution 보안 기준 "기본은 2시간"과 다르다. 소셜은 늘 2시간 — 체크박스가 소셜에서만 무시된다. 팀 확인이 필요해 남은 문제에 적었다.
- 구현 전 확인 실패 시: 소셜 세션은 서버 2시간 규칙만 적용하고 쿠키는 라이브러리 기본(7일)으로 남긴다. 이 차이를 남은 문제에 적는다.

## R8. 로그인 시도 제한 (FR-025~FR-028, SC-004, SC-006, NF-10)

**지금 코드**: 시도 제한이 없다. 원본 AUTH-09는 "로그인 라이브러리의 시도 횟수 제한은 HTTP 요청에만 적용되는데, 지금 로그인은 Server Action에서 라이브러리를 직접 부르기 때문에 적용되지 않을 수 있다"고 적었다. 라이브러리 `rateLimit`은 경로·IP 기준이다 **(추측)**.

- **Decision**: 새 표 `login_attempts`(auth 담당, [data-model.md 2.4](data-model.md#24-로그인-실패-기록--login_attempts-새-테이블))에 아이디별 연속 실패 수와 잠금 해제 시각을 둔다. `signIn` Server Action 순서:
  1. 아이디 정규화(`normalizeName`). 아이디나 비밀번호가 비면 `아이디와 비밀번호를 적어 주세요` (기록하지 않음).
  2. 아이디가 64자를 넘으면 기록 없이 `아이디 또는 비밀번호가 맞지 않아요` (그런 아이디는 있을 수 없다. 행 크기 상한).
  3. **시도 예약** (짧은 트랜잭션): `pg_advisory_xact_lock(<로그인 잠금 번호>, hashtext(아이디))`로 같은 아이디의 예약을 한 줄로 세운 뒤 그 아이디의 행을 읽는다.
     - `locked_until > now()`이면 행을 바꾸지 않고 커밋한 뒤, 비밀번호를 확인하지 않고 `로그인을 너무 많이 시도했어요. 5분 뒤에 다시 시도해 주세요`.
     - 아니면 이번 시도를 **실패로 미리 센다**: 연속 실패 +1 (잠금이 풀린 뒤라면 1부터). 5가 되면 `locked_until = now() + 5분`, 연속 실패 0. 커밋.
  4. 트랜잭션을 끝낸 **뒤에** `auth.api.signInUsername`(R6의 `rememberMe` 포함)을 부른다.
     - 성공 → 그 아이디의 행 삭제 (미리 센 실패와 방금 건 잠금도 함께 없어진다, FR-026).
     - 실패 → 이미 셌으므로 더 쓰지 않는다. 문구는 `아이디 또는 비밀번호가 맞지 않아요` (5번째 실패 자체는 보통 실패 문구, 6번째부터 잠금 문구).
  - 한 줄씩 차례로 시도하면 결과는 "실패한 뒤에 센다"와 같다: 1~4번 실패 뒤 5번째가 맞으면 예약 때 걸린 잠금이 성공과 함께 지워져 잠기지 않고, 틀리면 보통 실패 문구 뒤 6번째부터 잠금 문구다.
  - 잠긴 동안의 시도는 잠금을 늘리지 않는다 (US6 #2 "5분이 지난 뒤").
  - **라이브러리 호출을 트랜잭션 밖에 두는 이유**: `auth.api.signInUsername`은 drizzle 어댑터로 같은 연결 풀(`src/db/index.ts`의 `pg` Pool, 기본 최대 10개, 연결 대기 시간 제한 없음)에서 **다른 연결**을 꺼낸다. 트랜잭션 연결을 잡은 채 부르면, 동시 로그인 요청 10개(같은 아이디든 다른 아이디든)가 연결 10개를 모두 잡고 두 번째 연결을 서로 기다려 서버의 DB 접근 전체가 멈춘다. 예약 방식은 잠금을 쥔 동안 다른 연결이 필요 없다.
  - 없는 아이디도 같은 행을 만든다 (FR-027). `users`와 FK를 두지 않는다 (FK가 있으면 없는 아이디를 넣을 수 없고 존재 여부가 드러난다).
  - 규칙 숫자(5번, 5분)와 예약 계산(지금 상태 + 시각 → 거부 여부와 다음 상태)은 순수 함수 `src/lib/login-limit.ts`에 두고 `scripts/test-auth.ts`가 시험한다.
  - 가입 트랜잭션은 그 아이디의 행을 지운다 (새 계정이 예전 실패를 물려받지 않게). 탈퇴 트랜잭션도 지운다 (VII).
  - 잠금은 아이디·비밀번호 로그인에만 적용한다. 소셜 로그인은 이 경로를 지나지 않는다 (spec Edge Case).
- **Rationale**: 규칙이 아이디 기준이라 아이디마다 행 하나면 된다. DB에 두면 서버를 다시 켜거나 여러 대 띄워도 같은 기록을 본다. 아이디마다 예약을 한 줄로 세우지 않으면 동시에 20번 보내 20번 추측할 수 있다. 예약이 한 줄로 서므로 잠금 창마다 비밀번호 확인은 최대 5번이다 (SC-006).
- **Alternatives considered**:
  - 메모리 Map — 재시작하면 초기화, 서버가 여러 대면 따로 센다. 거절.
  - Better Auth `rateLimit`(`storage: "database"`) — IP·경로 기준이라 "같은 아이디 5번"을 만들 수 없고, 서버 호출에는 적용되지 않는다 **(추측)**. 거절.
  - 잠금을 쥔 트랜잭션 안에서 라이브러리 로그인을 부르고 결과에 따라 센다 — 위 "라이브러리 호출을 트랜잭션 밖에 두는 이유"의 연결 풀 교착이 생긴다. 거절.
  - 트랜잭션 안에서 우리 코드가 `verifyPassword`로 비밀번호를 먼저 확인하고, 커밋 뒤 라이브러리로 다시 로그인 — 교착은 없지만 해시 확인을 두 번 하고 라이브러리의 사용자 조회를 따라 써야 한다. 거절 (예약 방식이 더 단순).
- **화면을 거치지 않은 요청 (FR-028)**: 라이브러리의 `/api/auth/sign-in/username`·`/sign-in/email`은 R5로 닫혀 있다. Server Action을 직접 부르는 요청도 같은 함수를 지난다.
- **알려진 한계** (남은 문제에 적음): 같은 아이디로 동시에 많은 요청이 오면 예약 잠금을 기다리는 동안 요청마다 DB 연결을 하나씩 잡는다 (예약 트랜잭션은 짧고 그 안에서 다른 연결을 쓰지 않으므로 교착은 없다. IP 기준 제한은 spec 범위 밖). 5번째 시도를 예약한 뒤 그 시도가 성공하기 전에 들어온 같은 아이디의 다른 시도는 잠금 문구를 받을 수 있다 (막는 쪽으로 어긋남). "연속"에 시간 창이 없어 오래된 실패도 센다 (spec 문구 그대로).

## R9. 소셜: 가입 막기, 자동 연결 막기, 이메일 받지 않기 (FR-013, FR-033~FR-035, SC-007)

**지금 코드**: `src/lib/auth.ts`가 키가 있는 서비스를 켜고, 이메일이 없으면 `fallbackEmail`로 `{서비스}_{ID}@{서비스}.blogville.invalid`를 만든다. 가입을 막는 설정이 없다. Google은 라이브러리 기본 범위(이메일 포함 **(추측)**)를 쓴다.

- **Decision** (`src/lib/auth.ts`):
  - 공급자마다 `disableSignUp: true`. 연동되지 않은 소셜 계정은 회원을 만들지 않고 `errorCallbackURL`(`/`)로 돌아간다. 오류 코드 **(추측: `signup_disabled`)** 를 첫 화면이 `연동된 계정이 없어요. 아이디로 로그인한 뒤 내 정보에서 연동해 주세요`로 보여준다 (FR-033).
  - `account.accountLinking`: `enabled: true`(연동에 필요), `allowDifferentEmails: true`(회원 이메일이 `.invalid`라 소셜 이메일과 늘 다르다), 자동 연결 끄기(`disableImplicitLinking` **(1.7에 있는지 확인)**), `trustedProviders` 없음.
  - 이메일을 요청하지 않는다 (FR-035): Google은 `disableDefaultScope: true` + `scope: ["openid", "profile"]`. 카카오는 지금처럼 `profile_nickname`·`profile_image`. 네이버는 개발자 센터에서 제공 정보로 이메일을 고르지 않는다 (운영 설정, quickstart에 적음).
  - 라이브러리는 사용자 정보에 이메일이 없으면 로그인을 거부한다 (원본 AUTH-01 "대체 이메일" 근거, 거부 코드 **(추측: `email_not_found`)**). 그래서 `mapProfileToUser`의 대체 이메일은 **값만 채우고 저장되지 않는다**. 소셜로 회원을 만들지 않으므로 `users`에 쓰일 곳이 없다.
    - 지금 대체 이메일은 카카오·네이버에만 있다 (`fallbackEmail`). Google은 기본 범위가 이메일을 받아 와서 필요 없었지만, 위처럼 `email` 범위를 빼면 Google도 이메일이 없어진다. 그래서 Google에도 `mapProfileToUser: (profile) => ({ email: fallbackEmail("google", profile.sub) })`를 더한다 (Google 프로필의 계정 식별자 칸 이름 `sub`는 **(추측)**, 구현 전 확인 목록 7). 이것을 빠뜨리면 Google 로그인·연동이 모두 실패한다.
    - 카카오·네이버의 `mapProfileToUser`도 서비스가 이메일을 주더라도 쓰지 않도록 늘 대체 이메일을 넣게 바꾼다 (지금은 `profile.kakao_account?.email ?? …`로 받은 이메일을 우선한다). 저장은 안 되지만 받은 개인정보를 쓰지 않는 쪽으로 맞춘다 (FR-035, VII).
  - 소셜 토큰을 저장하지 않는다: `databaseHooks.account.create.before`에서 소셜 행의 `access_token`·`refresh_token`·`id_token`·두 만료 칸을 비우고, `account.updateAccountOnSignIn: false`. 우리 서비스는 소셜 API를 부르지 않는다 (VII 최소 정보). 남는 것은 `provider_id`, `account_id`(서비스 쪽 계정 식별자), 연동한 날짜(`created_at`)다.
- **Rationale**: 자동 연결은 같은 이메일을 가진 회원이 있어야 일어나는데, 회원 이메일이 모두 `{아이디}@users.blogville.invalid`라 일어날 수 없다. 옵션으로 한 번 더 막는다 (FR-034).
- **Alternatives considered**:
  - `disableImplicitSignUp` — 요청에 `requestSignUp`을 주면 가입이 열린다 **(추측)**. 가입 문이 남는다. 거절.
  - `databaseHooks.user.create.before`에서 거부 — 오류 코드가 달라진다. `disableSignUp`이 기대대로 동작하지 않을 때의 안전망으로 남긴다.

## R10. 소셜 연동·해제 (FR-036~FR-042, User Story 4)

- **Decision**:
  - **연동**: Server Action `startLinkSocial(provider)` — `requireMember()` → 켜진 서비스인지(`enabledProviders`) → 이미 그 서비스 행이 있으면 아무것도 하지 않음 (FR-039, 화면에서는 버튼이 `연동됨`) → `auth.api.linkSocialAccount({ body: { provider, callbackURL: "/settings/account?linked={서비스}", errorCallbackURL: "/settings/account?provider={서비스}" }, headers })` **(서버 호출 이름 확인)** 가 돌려준 주소를 화면에 돌려준다. 라이브러리 HTTP `/link-social`은 R5로 닫혀 있다.
  - **다른 회원에 연동된 소셜 계정**: 기존 UNIQUE(`provider_id`, `account_id`)가 막고, 라이브러리가 `errorCallbackURL`에 오류 코드 **(추측: `account_already_linked_to_different_user`)** 를 붙여 보낸다 → `이미 다른 Blogville 계정에 연동된 {서비스} 계정이에요` (FR-038).
  - **서비스마다 1개**: 새 UNIQUE(`user_id`, `provider_id`) (ERD 7장 5). 두 탭에서 동시에 연동해도 DB가 막는다.
  - **해제**: Server Action `unlinkSocial(provider)` — `requireMember()` → `provider`가 `kakao`/`naver`/`google`일 때만 `DELETE FROM accounts WHERE user_id = 나 AND provider_id = ?`. `credential`이나 그 밖의 값은 아무것도 하지 않고 거부 (FR-041). 라이브러리 `unlinkAccount`는 쓰지 않는다.
  - **결과 문구**: `?linked=kakao`이고 실제로 그 연동 행이 있을 때만 `카카오 계정을 연동했어요`. 취소(소셜 화면에서 동의하지 않음)는 문구 없이 그대로 (spec Edge Case).
- **Rationale**: 권한 확인을 우리 코드에 모은다. 해제는 행 하나를 지우는 일이라 라이브러리가 필요 없다.
- **Alternatives considered**: 클라이언트 `authClient.linkSocial`/`unlinkAccount`(원본 AUTH-05 제안) — HTTP 경로를 열어야 하고 해제가 `credential`을 막지 못한다. 거절 (연동은 구현 전 확인 9번이 안 될 때의 대안).

## R11. 회원 탈퇴 (FR-050~FR-052, User Story 7, SC-014, D2)

- **Decision**: Server Action `deleteAccount(prev, formData)`.
  1. `requireMember()`. 입력한 비밀번호를 credential 행의 해시와 비교한다 (`better-auth/crypto`의 `verifyPassword` **(내보내기 확인)**). 틀리면 아무것도 지우지 않는다 (FR-050).
  2. 트랜잭션 하나:
     1. `lockUser(tx, userId)`.
     2. social이 제공하는 탈퇴용 댓글 정리 함수(가칭 `removeAuthorComments(tx, userId)`, `src/server/social.ts`)를 부른다. 결과 조건(D2, FR-052): 이 회원의 답글은 모두 사라지고, 다른 회원의 답글이 없는 이 회원의 댓글은 자리 없이 사라지고, 다른 회원의 답글이 달린 이 회원의 댓글은 내용·작성자 없이 `삭제된 댓글이에요` 자리만 남는다. 구조(삭제 동작, 내용 비우기)는 social이 정한다.
     3. `DELETE FROM login_attempts WHERE username = 내 아이디`.
     4. `DELETE FROM users WHERE id = 나`. 나머지는 FK `ON DELETE CASCADE`로 함께 지워진다: 세션, 로그인 수단(연동한 소셜 포함), 프로필, 블로그 → 카테고리·글 → 글의 댓글·답글·공감·태그 연결·방문 기록, 보유 아이템, 이웃, 출석, 원장, 동물·돌보기, 첨부 행, 알림(game).
  3. 커밋 뒤 `auth.api.signOut({ headers })`로 쿠키를 지운다 (세션 행은 이미 없다 **(추측: 행이 없어도 쿠키는 지운다, 구현 전 확인 목록 16)**). `revalidatePath("/", "layout")` → `redirect("/")`.
- **Rationale**: 한 트랜잭션이라 중간에 실패하면 아무것도 지워지지 않는다 (FR-051, spec Edge Case). 비밀번호 다시 입력이 실수 방지 단계다 (spec Assumption: 유예 없이 즉시).
- **Alternatives considered**: Better Auth `deleteUser`(`user.deleteUser.enabled`) — 라이브러리가 트랜잭션 밖에서 지우고 D2 정리를 끼워 넣을 수 없다. 거절.
- **저장소 파일**: `attachments` 행은 CASCADE로 지워지지만 파일은 디스크(`UPLOAD_DIR`)에 남는다. ERD 3.14 "저장소의 파일은 정리 작업이 지운다"에 따라 post의 정리 작업(POST FR-059)이 "DB에 행이 없는 파일"도 지우도록 요청한다 (의존성). 탈퇴 처리에서 파일을 직접 지우지 않는다 (커밋 전에 지우면 롤백할 수 없고, 커밋 뒤에 지우다 실패하면 같은 정리 작업이 필요하다).
- **관리자 탈퇴**: spec에 예외가 없어 같은 규칙을 따른다. 관리자가 탈퇴하면 `/@notice`도 지워지고 `npm run admin:create`로 다시 만든다 (남은 문제에 팀 확인으로 적음).

## R12. `src/server/dal.ts` 정리 (FR-007, FR-022, FR-043)

- **Decision**:
  - `requireUser()`를 지운다. 쓰는 곳은 온보딩 두 파일뿐이다 (`grep`으로 확인).
  - `getViewer()`: 로그인하지 않았으면 `null`. 로그인했으면 `{ userId, user, sessionId, profile }`이고 `profile`(닉네임·캐릭터·블로그 ID·주소·이름)은 늘 있다. 세션은 있는데 프로필이 없으면 데이터 오류로 던진다 (마이그레이션이 그런 회원을 정리하고 가입이 한 트랜잭션이라 생기지 않는다).
  - `requireMember()`: 이름과 반환 모양을 그대로 둔다 (다른 spec의 plan이 이 이름을 쓴다). 비로그인만 `/`로 보낸다. `/onboarding`으로 보내는 줄을 지운다.
  - `requireAdmin()`: `requireMember()`를 거치지 않고, `getViewer()`가 없거나 관리자가 아니면 `notFound()`. 지금은 비로그인을 `/`로 보내 "관리자 화면이 있다"는 것이 드러난다 (FR-043, SC-008 "로그인하지 않은 사람 포함 404").
  - `getSession()`: R6-4·5 (`disableRefresh`, 유지 안 함 세션의 마지막 사용 2시간 확인과 2시간 연장).
- **Rationale**: 온보딩이 없어지면 두 함수의 차이가 사라진다 (공통 맥락 2장). 이름을 바꾸면 모든 회원 화면이 바뀌므로 `requireMember`만 남긴다.

## R13. 관리자 화면과 관리자 계정 스크립트 (FR-043~FR-049)

- **통계 카드 5개** (FR-044): `주민 (온보딩 완료)` 카드를 지운다. 댓글 수는 social이 `replies`를 만든 뒤(social 5단계) `comments`와 `replies`의 삭제 안 된 행을 더한다. 그 전에는 지금 식 그대로다.
- **최근 가입 20명** (FR-045): `온보딩 전` 표시를 지운다 (프로필이 늘 있다, inner join). 로그인 방식은 `credential` → `아이디`, `kakao` → `카카오`, `naver` → `네이버`, `google` → `Google`로 보여준다.
- **375px** (FR-054): 지금 표는 카드 안에서 가로로 스크롤된다 (`overflow-x-auto`). 휴대폰 폭에서는 행을 쌓은 목록으로 보여준다 (`max-sm:`에서 표 행을 블록으로).
- **`adminDeletePost`**: 지금 그대로 (`requireAdmin()` → `parseId()` → 삭제 → `revalidatePath("/", "layout")`). R12의 `requireAdmin` 변경으로 비로그인도 `notFound()`. Server Action 안의 `notFound()`가 HTTP 404로 보이는지는 구현 전 확인이고, 검증 기준은 "데이터 변경 0건"이다 (SC-008).
- **`scripts/create-admin.ts`** (FR-048, FR-049):
  - 비밀번호 12~64자, 아이디와 같은 비밀번호 거부. spec은 "배포 전 12자 이상"이라 했지만 스크립트가 실행 환경을 믿을 만하게 구분할 신호가 없어 개발에서도 같은 기준을 쓴다. 팀원은 `.env.local`의 `ADMIN_PASSWORD`를 12자 이상으로 바꿔야 한다 (quickstart에 적음).
  - 아이디 형식 `^[a-z0-9_]{4,20}$` 확인 (`users_username_check`와 로그인 플러그인 규칙에 맞춤).
  - 32자 ID는 `src/lib/auth-id.ts`를 같이 쓴다. 예전 형식 ID 안내는 이미 구현되어 있다.
  - 관리자는 회원가입을 거치지 않으므로 FR-010 검사를 받지 않는다 (spec Assumption). 블로그 주소는 `notice`(예약어라 일반 회원이 먼저 차지할 수 없다).
- **Alternatives considered** (12자): 배포 환경에서만 확인 — `NODE_ENV`는 스크립트 실행 때 믿을 만하지 않다. 거절.

## R14. 다른 사이트 요청(CSRF)과 쿠키 속성 (FR-023, FR-024, SC-009, NF-11, NF-12)

- **사실**: Next.js는 Server Action 요청의 Origin과 Host(프록시 뒤면 `x-forwarded-host`)를 비교해 다르면 거부한다. `src/app/api/uploads/route.ts`의 주석이 "Server Action과 같은 기준"으로 직접 구현했다고 적었다. `next.config.ts`에 `serverActions.allowedOrigins`가 없다.
- **Decision**:
  - 상태를 바꾸는 인증 동작(가입, 로그인, 로그아웃, 소셜 로그인 시작, 연동 시작, 해제, 탈퇴, 관리자 글 삭제)은 모두 Server Action이다. 라이브러리 HTTP는 R5 허용 목록(읽기와 OAuth 콜백)만 남는다.
  - 쿠키 속성은 라이브러리 기본값 **(확인)** `HttpOnly`, `SameSite=Lax`, `BETTER_AUTH_URL`이 `https`면 `Secure`(쿠키 이름에 `__Secure-` 접두)를 쓰고 e2e가 속성을 확인한다. `bv_remember`도 같은 속성으로 심는다.
- **spec 문구 해석**: FR-023 "다른 사이트에서 시작된 요청에는 실리지 않으며"는 constitution 보안 기준(`SameSite=Lax`)대로 해석한다. 다른 사이트의 POST·iframe·fetch에는 쿠키가 실리지 않고, 다른 사이트의 링크로 들어오는 최상위 GET 이동에는 실린다. OAuth 콜백이 여기에 기대고, GET은 상태를 바꾸지 않으므로 FR-024를 지킨다 (해석 확인을 남은 문제에 적음).
- Origin이 다른 Server Action 요청의 HTTP 상태 코드는 구현 전 확인이다. 검증 기준은 "성공이 아님 + 데이터 변경 0건"이다.

## R15. 기존 데이터 옮기기 (ERD 7장 5)

- **사실**: 지금 DB에는 가입만 하고 온보딩을 마치지 않은 회원이 있을 수 있다 (`e2e/auth.mjs`의 `newbie01`, `e2e/social.mjs`의 `preonboard01`). 소셜 키가 발급되지 않아 `username`이 NULL인 회원은 없다고 본다 (ERD 7장 5). 배포 전이라 운영 데이터가 없다 (NF-08 미정).
- **Decision**: 직접 쓴 SQL 마이그레이션 "가입 미완료 회원 정리"가 프로필이 없는 회원을 지운다 (세션·로그인 수단은 CASCADE). 이런 회원은 글·댓글·원장·첨부를 가질 수 없다 (모든 회원 기능이 `requireMember()`를 거치고, 업로드도 `viewer.profile`을 확인한다). 새 규칙에서 "가입 처리 중 실패"와 같은 상태라 남기지 않는다 (FR-006, FR-007).
  - `username`이 NULL인 회원이 남아 있으면 다음 마이그레이션(NOT NULL)이 실패한다. 로컬은 `npm run db:reset`. 아이디를 자동으로 지어 주지 않는다 (비밀번호가 없어 어차피 아이디 로그인을 할 수 없다).
- **Alternatives considered**: 프로필 없는 회원에게 기본 프로필·블로그를 채워 주기 — 그 아이디가 다른 회원의 주소·닉네임과 겹치면 마이그레이션이 실패하고, 아이템이 시드되지 않은 DB에서도 실패한다. 거절.

## R16. 프로필 사진 칸 (ERD 3.9, 7장 3; blog FR-028, town FR-054, post FR-059)

- **Decision**: `profiles.photo_key text NULL` + FK → `attachments(key)` `ON DELETE SET NULL`을 1단계에 별도 마이그레이션으로 더한다. 올리는 화면은 담당 spec이 없어 만들지 않는다 (공통 맥락 5.3). 값은 비어 있고, 표시하는 spec은 "사진이 없으면 캐릭터 얼굴"로 완성한다.
- **Rationale**: post의 첨부 정리 작업(3단계)이 이 칸을 보고 프로필 사진을 제외하므로 3단계 전에 있어야 한다. 첨부 행이 지워지면 사진만 비우고 프로필은 남는다.
- **Alternatives considered**: 복합 FK (`user_id`, `photo_key`) → `attachments(user_id, key)`로 "내 첨부만" — attachments(post 담당)에 UNIQUE(`user_id`, `key`)를 요청해야 한다. 올리는 화면이 없는 지금은 과하다. 그 화면을 맡는 spec이 생길 때 다시 본다.

## R17. 화면 접근성·반응형 (FR-054, SC-011, constitution VI)

- **Decision**:
  - 첫 화면·내 정보·관리자 화면은 375px에서 가로 스크롤이 없어야 한다. `e2e/nonfunctional.mjs`의 페이지 목록에 `/settings/account`를 더하고(첫 화면은 비로그인 컨텍스트로 따로 잰다), 관리자 화면은 `e2e/auth.mjs`에서 375px로 확인한다.
  - 누르는 영역 44×44px: 탭 버튼, 소셜 버튼, [로그인 상태 유지] 줄, 로그아웃 버튼(지금은 글자만이라 작다), 내 정보의 연동 버튼, 관리자 화면의 [삭제] 버튼(`src/app/admin/delete-button.tsx`, 지금은 `text-sm` 글자만이라 작다)과 제목·블로그 링크에 `min-h-11`(+ 필요하면 `min-w-11`). 버튼 글자는 `whitespace-nowrap`.
  - 키보드: 탭은 `button`이라 Tab·Enter로 바뀐다. 캐릭터 고르기는 숨긴 radio + `peer-focus-visible:ring` (지금 온보딩 폼의 방식을 옮긴다). 버튼·체크박스에 `focus-visible:ring`.

## R18. 테스트 전략

- **단위** (`npm test`에 `test:auth` 추가, `scripts/test-auth.ts`, 관례: `expect(name, got, want)`·`✅/❌`·실패 시 `exit(1)`):
  - 아이디 정규화·형식, 예약어 16개(대소문자를 섞어도), 닉네임 길이 2~20.
  - 로그인 제한 예약 계산: 4번째 시도까지 잠금 아님, 5번째 시도 예약 → 잠금, 잠금 중 시도 → 거부·변화 없음, 잠금이 풀린 뒤 시도 → 1, 성공 → 초기화.
- **E2E** (관례: 맨 위 요구사항 ID 주석, `.env.local` + `pg`로 DB 준비·확인, 실행마다 새 아이디, `check()`·`exit(1)`, 스크린샷):
  - 새 파일: `e2e/signup.mjs`, `e2e/session.mjs`, `e2e/login-limit.mjs`, `e2e/account.mjs`.
  - 고칠 파일: `e2e/helpers.mjs`(`loginDev`: 가입 폼에서 캐릭터를 고르고 바로 광장), `e2e/auth.mjs`(온보딩 확인 → 광장 도착, 비로그인 `/admin` 404).
- **시간이 걸리는 조건은 DB로 당긴다**: 2시간 미사용 = `sessions.expires_at`을 과거로, 또는 `updated_at`을 2시간 넘게 전으로 (R6-4-1), 5분 잠금 = `login_attempts.locked_until`을 과거로, 7일 연장 = `expires_at`을 6일 뒤로 놓고 `SessionKeeper`가 부르게 한다. 브라우저 종료 = Playwright `storageState`에서 만료 없는 쿠키를 빼고 새 컨텍스트로 연다.
- **가입 중간 실패(SC-003)**: 시험 동안만 `point_ledger`에 시험용 아이디의 `signup` 기록을 거부하는 트리거를 만들고(가입의 마지막 단계에서 실패), 끝나면 `finally`로 지운다.
- **소셜**: 실제 로그인·연동은 키가 있어야 해서 수동 확인 ([quickstart.md](quickstart.md#6-소셜-수동-확인-키가-있을-때)). 키 없이 자동으로 확인하는 것: 버튼 비활성·안내 문구, DB에 연동 행을 직접 넣어 `연동됨 (날짜)`·해제·조작 요청 거부.

---

## 구현 전 확인 목록

`node_modules`가 없어 이 저장소에서 확인하지 못한 라이브러리 동작이다. 구현 전에 설치된 패키지 문서(`node_modules/better-auth`, `node_modules/next/dist/docs/`)로 확인한다.

| # | 확인할 것 | 쓰는 곳 | 안 될 때 |
|---|---|---|---|
| 1 | `better-auth/crypto`의 `hashPassword`·`verifyPassword` 내보내기 (`hashPassword`는 `scripts/create-admin.ts`가 이미 씀) | R1, R11 | `(await auth.$context).password.verify` |
| 2 | `auth.api.signInUsername`이 `rememberMe`를 받고 `token`을 돌려주는지 | R2, R6, R8 | 로그인 뒤 돌려받은 `token`으로 `sessions`를 직접 고친다 |
| 3 | `session.additionalFields`, `databaseHooks.session.create.before(session, ctx)`에서 요청 본문·쿠키를 읽을 수 있는지 | R6, R7 | 아이디 로그인은 Server Action에서 `token`으로 UPDATE, 소셜은 `hooks.after`에서 UPDATE |
| 4 | `auth.api.getSession({ query: { disableRefresh: true } })` | R6 | `session.disableSessionRefresh` + 유지 세션 연장 전용 Route Handler |
| 5 | `rememberMe: false` 세션의 기본 만료(1일로 추측)와 `dont_remember` 쿠키 | R6 | 대안 불필요 (2시간 연장 UPDATE가 되돌린다) |
| 6 | Server Component에서 `nextCookies()`가 쿠키 쓰기를 조용히 건너뛰는지 | R6 | `SessionKeeper`가 있으면 결과가 같다 |
| 7 | 공급자 `disableSignUp`과 그때의 오류 코드, 소셜 화면에서 취소했을 때의 오류 코드. 이메일이 없을 때 로그인을 거부하는지와 그 코드, Google `mapProfileToUser`의 프로필 식별자 칸(`sub`) | R9 | `databaseHooks.user.create.before`에서 거부하고 그 코드를 매핑. 식별자 칸이 다르면 대체 이메일 식만 고친다 |
| 8 | `account.accountLinking`의 `allowDifferentEmails`·`disableImplicitLinking`, 다른 회원에 연동된 계정의 오류 코드 | R9, R10 | 회원 이메일이 `.invalid`라 자동 연결은 일어나지 않는다 (옵션 없이도 FR-034 유지) |
| 9 | 서버 호출 `auth.api.linkSocialAccount`·`auth.api.signInSocial`의 이름과 반환 `{ url }`, OAuth state 쿠키를 Server Action에서 심는지 | R7, R10 | 클라이언트 `authClient.linkSocial` + 허용 목록에 `/link-social` 추가 + `hooks.before`로 중복 거부 |
| 10 | `hooks.after`에서 `ctx.context.newSession`, `setSessionCookie(ctx, session, dontRememberMe)` | R7 | 소셜 세션은 서버 2시간 규칙만 (남은 문제) |
| 11 | `databaseHooks.account.create.before`, `account.updateAccountOnSignIn` | R9 | 마이그레이션 데이터 SQL + 주기적으로 토큰 칸 비우기 |
| 12 | Server Action 안의 `notFound()`, Origin 불일치 요청의 HTTP 상태 | R13, R14 | 검증 기준을 "데이터 변경 0건"으로 둔다 |
| 13 | 라이브러리 경로 이름 (`/get-session`, `/callback/:id`, `/error`, `/sign-in/social`, `/sign-out`) | R5 | 허용 목록 값만 고친다 |
| 14 | 라이브러리 쿠키 이름·기본 속성 (`HttpOnly`, `SameSite=Lax`, `Secure`) | R14 | `advanced.defaultCookieAttributes`로 명시 |
| 15 | `npx drizzle-kit generate --custom --name=<이름>`으로 빈 SQL 마이그레이션 만들기 (`drizzle/0002_give_all_starters.sql` 같은 데이터 마이그레이션) | data-model | SQL 파일과 `drizzle/meta/_journal.json` 항목을 직접 추가 |
| 16 | 세션 행이 이미 지워진 뒤(탈퇴 CASCADE) `auth.api.signOut`이 오류 없이 세션·`dont_remember` 쿠키를 지우는지 | R11 | `next/headers`의 `cookies()`로 확인 목록 14의 쿠키 이름을 직접 지운다 |
