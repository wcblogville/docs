# Contract: 첫 화면 — 회원가입·로그인·로그아웃·소셜 로그인

**관련**: FR-001~FR-035, FR-053, FR-054 / User Story 1, 2, 3, 6 / [data-model.md](../data-model.md) / [research.md](../research.md)

공통 약속:

- Server Action은 Next.js가 요청의 Origin과 Host를 비교한다. 다른 사이트에서 보낸 요청은 처리되지 않고 데이터가 바뀌지 않는다 (FR-024, SC-009).
- 화면·오류 문구는 spec 문구 그대로다 (FR-053). spec에 문구가 없어 이 plan이 제안한 것은 **(spec에 없음 — 제안)** 으로 표시했고 plan의 남은 문제에 있다.
- "1단계"·"8단계"는 [plan.md 변경 단위 표](../plan.md#현재-코드--spec-목표-변경-단위)의 구현 단계다.

---

## 1. 화면 라우트 `/`

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/page.tsx`, `src/components/login-buttons.tsx` |
| 접근 | 누구나. 로그인한 회원이면 `redirect("/town")` (FR-014). 온보딩 분기는 없다 |
| 서버가 화면에 넘기는 값 | `providers` (`enabledProviders`: 키가 있는 서비스), `starters` (기본 캐릭터: `items.type = 'character' AND is_starter`의 `id`·`name`·`description`·`assetKey`, 기본 선택은 `char_boy`), `socialError` (아래 `error` 해석 결과) |
| searchParams `error` | 연동 없음 코드 **(라이브러리 코드 확인, research 확인 목록 7)** → `연동된 계정이 없어요. 아이디로 로그인한 뒤 내 정보에서 연동해 주세요` (FR-033). 그 밖의 코드(소셜 화면에서 취소 등) → 문구 없이 첫 화면 그대로 (spec Edge Case) |

**화면 구성**

| 영역 | 내용 | 근거 |
|---|---|---|
| 탭 | [로그인] [회원가입], 처음에는 [로그인] | FR-001 |
| 로그인 폼 | `username`(아이디), `password`(비밀번호), `rememberMe` 체크박스 [로그인 상태 유지] (처음엔 해제), 오류 한 줄, [로그인] (처리 중 `들어가는 중...`, 누를 수 없음) | FR-015, FR-018, FR-019 |
| 회원가입 폼 | `username` (최대 20자, 칸 아래 `영문 소문자, 숫자, _ 로 4~20자`), `password` (안내 `비밀번호 (8자 이상)`, 최대 64자), `passwordConfirm` (`비밀번호 확인`), `characterId` 라디오 2개 (남자 주민 / 여자 주민, 처음엔 남자 주민), [회원가입] 바로 위 빨간 굵은 글씨 오류 한 줄, [회원가입] (처리 중 `가입하는 중...`, 누를 수 없음) | FR-001, FR-002, FR-004, FR-005, SC-001 (입력 칸 4개) |
| 간편 로그인 | `간편 로그인` 구분선, [카카오] (노랑) [네이버] (초록) [Google] (흰색) 3칸. 키 없는 서비스는 흐리게 비활성 + 마우스를 올리면 `아직 연결 준비 중이에요`. 셋 다 없으면 `간편 로그인은 준비 중이에요`. 그 아래 `처음이라면 회원가입 후 내 정보에서 연동해 주세요` | FR-030, FR-031 |
| 소셜 오류 | 간편 로그인 영역 위에 빨간 글씨 한 줄 (위치는 spec에 없음 — 제안) | FR-033 |
| 접근성 | 375px 가로 스크롤 없음, 누르는 영역 44×44px, 버튼 글자 한 줄, Tab·Enter만으로 로그인·가입, 지금 선택된 곳 표시 (`focus-visible`) | FR-054, SC-011 |

---

## 2. Server Action `signUp(prev, formData)`

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/(auth)/actions.ts` (처리 본체는 `src/server/signup.ts`의 `createMember`) |
| 권한 | 비로그인. 이미 로그인한 사람이 부르면 `redirect("/town")` |
| 입력 (FormData) | `username`, `password`, `passwordConfirm`, `characterId` (문자열). 그 밖의 칸(예: `role`)은 읽지 않는다 (FR-011) |
| 결과 (실패) | `{ error: string, values: { username: 입력 원문, characterId: 입력 값 } }` — 아이디와 고른 캐릭터를 다시 채운다. 비밀번호 칸은 비운다 (FR-005) |
| 결과 (성공) | 아래 처리 뒤 `redirect("/town?welcome=1")` (환영 신호는 town 결정을 따른다, research R2) |

**검사 순서와 문구** (위에서 처음 걸린 것 하나만 보여준다)

| # | 검사 | 실패 문구 | 근거 |
|---|---|---|---|
| 1 | `username` 앞뒤 공백 제거·소문자 → `^[a-z0-9_]{4,20}$` | `아이디는 영문 소문자, 숫자, _ 로 4~20자예요` | FR-002 |
| 2 | `password` 8자 이상 | `비밀번호는 8자 이상이에요` | FR-004 |
| 3 | `password` 64자 이하 (화면은 입력 칸이 더 받지 않음) | `비밀번호는 64자까지예요` (지금 코드 문구 유지. spec은 화면 문구를 두지 않는다) | FR-004, spec Assumption |
| 4 | `passwordConfirm` = `password` | `비밀번호가 서로 달라요` | FR-004 |
| 5 | 예약어 16개가 아님 (`isReservedName`) | `이 아이디는 쓸 수 없어요` | FR-010 |
| 6 | `characterId`가 `parseId()`를 통과하고 기본 캐릭터임 (트랜잭션 안에서 다시 확인) | `고를 수 없는 캐릭터예요` (spec에 없음 — 제안, 예전 온보딩 문구) | FR-011 |
| 7 | 이름 잠금 안에서 다른 회원의 아이디·블로그 주소·닉네임(대소문자 무시)과 겹치지 않음. UNIQUE 위반도 같은 문구 | `이미 있는 아이디예요` | FR-003, FR-010 |

**처리 (성공 경로)**

1. 비밀번호 해시 계산.
2. 트랜잭션 하나: [data-model.md 2.6](../data-model.md#26-가입-때-함께-만드는-행-참조-테이블)의 순서 (회원 → 로그인 수단 → 실패 기록 삭제 → 아이템 2개 → 프로필 → 블로그 → "일상" → 🪙 100). 하나라도 실패하면 아무것도 남지 않는다 (FR-006, SC-003).
3. 커밋 뒤 `auth.api.signInUsername({ body: { username, password, rememberMe: false } })` → 세션 쿠키 (유지 안 함, 2시간).
4. `revalidatePath("/", "layout")` → `redirect("/town?welcome=1")`.

예상 밖 오류(DB 연결 등)는 트랜잭션이 롤백되고 공통 오류 화면(`src/app/error.tsx`)이 보인다. 남는 행은 없다.

---

## 3. Server Action `signIn(prev, formData)`

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/(auth)/actions.ts` (실패 기록은 `src/server/login-attempts.ts`) |
| 권한 | 누구나 |
| 입력 (FormData) | `username`, `password`, `rememberMe` (`"on"`이면 참, 8단계부터) |
| 결과 (실패) | `{ error: string, values: { username, rememberMe } }` |
| 결과 (성공) | 세션 생성(`remember_me` 반영), 그 아이디의 실패 기록 삭제, `revalidatePath("/", "layout")`, `redirect("/town")` |

| # | 상황 | 문구 | 실패 기록 | 근거 |
|---|---|---|---|---|
| 1 | 아이디나 비밀번호가 비어 있음 | `아이디와 비밀번호를 적어 주세요` | 안 함 | FR-017 |
| 2 | 정규화한 아이디가 64자 초과 | `아이디 또는 비밀번호가 맞지 않아요` | 안 함 | research R8 |
| 3 | 그 아이디가 잠겨 있음 (`locked_until > now()`) — 비밀번호가 맞아도 | `로그인을 너무 많이 시도했어요. 5분 뒤에 다시 시도해 주세요` | 바꾸지 않음 | FR-025 |
| 4 | 없는 아이디, 또는 비밀번호 틀림 | `아이디 또는 비밀번호가 맞지 않아요` (두 경우 같은 문구) | 비밀번호 확인 전에 +1로 예약해 둔 그대로 (5번째 시도면 5분 잠금) | FR-017, FR-025, FR-027, SC-004 |
| 5 | 맞음 | (광장으로 이동) | 삭제 (예약한 +1과 잠금도 함께) | FR-015, FR-026 |

- 같은 아이디의 시도는 비밀번호를 확인하기 전에 서버에서 한 줄로 "예약"한다 (동시에 보내도 잠금 창마다 5번까지만 비밀번호를 확인, SC-006). 라이브러리 로그인 호출은 예약 트랜잭션이 끝난 뒤에 한다 (research R8).
- 아이디 대소문자·앞뒤 공백은 무시한다 (FR-016).
- 잠금은 이 경로에만 적용된다. 연동한 소셜 계정 로그인은 막지 않는다 (spec Edge Case).
- 화면을 거치지 않고 이 Server Action을 직접 불러도 같은 규칙이다. 라이브러리의 로그인 HTTP 경로는 닫혀 있다 (6장) (FR-028).

---

## 4. Server Action `startSocialSignIn(provider, remember)` (8단계)

1단계에서는 지금처럼 화면이 `authClient.signIn.social`을 부르고, 8단계에서 이 Server Action으로 바꾼다.

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/(auth)/actions.ts` |
| 권한 | 비로그인. 로그인한 사람이면 `{ url: "/town" }` |
| 입력 | `provider`: `"kakao" \| "naver" \| "google"`, `remember`: boolean (같은 카드의 [로그인 상태 유지]) |
| 거부 | `provider`가 목록 밖이거나 키가 없는 서비스 → `{ error: "아직 연결 준비 중이에요" }` (화면에서는 버튼이 비활성이라 보이지 않는다) |
| 처리 | 쿠키 `bv_remember` 심기 (7장) → `auth.api.signInSocial({ body: { provider, callbackURL: "/town", errorCallbackURL: "/" } })` |
| 결과 | `{ url: string }` → 화면이 그 주소로 이동 (소셜 서비스 로그인 화면) |

**소셜 서비스에서 돌아온 뒤**

| 상황 | 이동 | 결과 | 근거 |
|---|---|---|---|
| 그 소셜 계정이 어떤 회원에 연동되어 있음 | `/town` | 그 회원으로 로그인 (`remember_me` = `bv_remember`) | FR-032, SC-012 |
| 어떤 회원에도 연동되어 있지 않음 | `/?error=<연동 없음 코드>` | 회원이 생기지 않고 연동 없음 문구 | FR-013, FR-033, SC-007 |
| 소셜 계정 이메일이 어떤 회원과 같음 (연동은 안 됨) | 위와 같음 | 자동으로 붙지 않음 | FR-034 |
| 소셜 화면에서 취소·동의하지 않음 | `/?error=<취소 코드>` | 문구 없음, 아무것도 바뀌지 않음 | spec Edge Case |

소셜 서비스에 이메일을 요청하지 않고, 받은 토큰을 저장하지 않는다 (FR-035, research R9).

---

## 5. Server Action `signOut()`

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/(auth)/actions.ts`, 버튼 `src/components/sign-out-button.tsx` (`<form action={signOut}>`, 누르는 영역 44×44px) |
| 권한 | 누구나 (비로그인이면 아무것도 지우지 않고 `/`로) |
| 입력 | 없음. 확인 창 없음 (FR-029, spec Assumption) |
| 처리 | `auth.api.signOut()` → 세션 행 삭제·쿠키 삭제 → `revalidatePath("/", "layout")` → `redirect("/")` |
| 결과 | 헤더에 [시작하기], 회원 화면에 들어가면 `/` (FR-029) |

---

## 6. Route Handler `/api/auth/[...all]` (라이브러리 HTTP)

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/api/auth/[...all]/route.ts` |
| 규칙 | 아래 허용 목록만 라이브러리 핸들러(`toNextJsHandler(auth)`)로 넘기고, 나머지는 **404 (빈 본문)** |

| 메서드 | 경로 | 1단계 | 8단계부터 | 용도 |
|---|---|---|---|---|
| GET | `/api/auth/get-session` | 허용 | 허용 | `SessionKeeper`의 로그인 유지 연장 |
| GET, POST | `/api/auth/callback/{google,kakao,naver}` | 허용 | 허용 | 소셜 로그인·연동에서 돌아오기 |
| GET | `/api/auth/error` | 허용 | 허용 | 라이브러리 오류 화면 (안전망) |
| POST | `/api/auth/sign-in/social` | 허용 (임시) | **404** | 1단계의 소셜 버튼 |
| 그 밖 모두 | 예: `/sign-up/email`, `/sign-in/username`, `/sign-in/email`, `/sign-out`, `/update-user`, `/link-social`, `/unlink-account`, `/change-password`, `/delete-user`, `/is-username-available` | **404** | **404** | 우리 Server Action만 쓴다 |

경로 이름은 구현 전 확인 대상이다 (research 확인 목록 13).

---

## 7. 쿠키

| 이름 | 심는 곳 | 속성 | 값·수명 |
|---|---|---|---|
| 세션 쿠키 (라이브러리 기본 이름 **(확인)**, `https`면 `__Secure-` 접두) | 라이브러리 | `HttpOnly`, `SameSite=Lax`, `Path=/`, 배포(`https`) 때 `Secure` | 서명한 세션 토큰. 유지: Max-Age 7일 / 유지 안 함: Max-Age 없음 (브라우저를 닫으면 삭제) |
| 유지 안 함 표시 쿠키 (`dont_remember`, 라이브러리) | 라이브러리 | 위와 같음, Max-Age 없음 | 라이브러리가 유지 안 함 세션의 7일 연장을 건너뛰게 함 (보조. 서버 기준은 `sessions.remember_me`와 `updated_at`: 이 쿠키가 지워져 라이브러리가 만료를 늘려도 마지막 사용 2시간 뒤에는 `getSession()`이 세션을 끝낸다, research R6-4) |
| `bv_remember` | `startSocialSignIn` | `HttpOnly`, `SameSite=Lax`, `Path=/`, Max-Age 600, 배포 때 `Secure` | `0`/`1`. 소셜 콜백이 읽은 뒤 지운다 |

로그인 상태 정보는 페이지 스크립트로 읽을 수 없고, 다른 사이트의 POST·iframe·fetch 요청에는 실리지 않는다 (FR-023, constitution 보안 기준 `SameSite=Lax`. 해석은 research R14).

---

## 8. 다른 spec과의 화면 약속

| 상대 | 약속 |
|---|---|
| town (TOWN-01) | 가입 성공은 `/town?welcome=1`로 보낸다. town이 "한 번만" 신호를 다른 방식(예: 일회용 쿠키)으로 정하면 `signUp`이 그 신호를 심는다 |
| town (헤더 소유) | 헤더 회원 영역에 `SignOutButton`(auth)과 `SessionKeeper`(auth)를 둔다. `SessionKeeper`는 [로그인 상태 유지] 세션일 때만 클라이언트 부분을 그리므로 헤더는 조건 없이 한 줄만 넣는다. 지금 캐릭터 배지를 `/settings/account` 링크로 감싼다 (FR-036 "헤더 프로필에서"). TOWN-10 상태창이 생기면 town FR-054는 "상태창을 누르면 내 블로그 홈"이라 같은 자리를 내 정보 링크로 쓸 수 없다. 상태창 안 어디에 내 정보 입구를 둘지 town과 정한다 (plan 남은 문제 15) |
| game (GAME-04) | `getViewer()`가 `sessionId`를 돌려준다. 자동 출석이 쓴다 |
