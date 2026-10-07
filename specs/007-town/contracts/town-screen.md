# Contract: 광장 화면 `/town` (TOWN-01, 02, 03, 04, 05, 11)

**Feature**: `007-town` | 관련: [../data-model.md](../data-model.md) 3.1, [../research.md](../research.md) R-01·02·05~09·16~19

이 기능에 HTTP Route Handler는 없다. 바깥 인터페이스는 화면 라우트, Server Action, 쿠키 하나, 검증용 DOM·개발 훅이다.

## 1. 화면 라우트

| 주소 | 파일 | 접근 | 리다이렉트 | 탭 제목 |
|---|---|---|---|---|
| `/town` | `src/app/town/page.tsx` (서버) | 누구나 (방문자·회원·관리자) | 없음 (지금의 "프로필 없으면 `/onboarding`"은 auth 단계 1 뒤 지운다) | `중앙 광장 \| Blogville` |
| `/` (첫 화면) | auth 소유 | 방문자 | 로그인한 사람 → `/town` (FR-001, auth FR-014) | - |
| 아이디 로그인 성공, 연동한 소셜 로그인 성공 | auth 소유 | - | → `/town` (FR-001, Edge Cases 소셜) | - |
| 회원가입 성공 | auth 소유 `signUp` | - | 쿠키 `bv_welcome` 심고 → `/town` (3절) | - |

화면 구성 (FR-005·006):

| 요소 | 위치·모양 | 규칙 |
|---|---|---|
| 광장(캔버스) | 헤더 아래 화면 전체. 높이 방문자 `100dvh − var(--header-h)`, 회원 `100dvh − var(--header-h-member)`, 최소 420px. 페이지 스크롤·푸터 없음 | 모든 화면 크기(휴대폰 포함)에서 띄운다 (R-01). 화면 크기가 바뀌면 다시 맞춘다 (Phaser `Scale.RESIZE`) |
| 화면 낭독기용 제목 | `<h1 class="sr-only">중앙 광장</h1>` | 지금과 같음 |
| 조작 안내 (오른쪽 아래) | 키보드·마우스: `방향키·WASD 또는 클릭으로 이동 · 건물 앞에서 Space로 들어가기` / 터치(`pointer: coarse`): `조이스틱이나 탭으로 이동 · 건물을 탭해서 들어가기` | FR-013. 누를 수 없음(`pointer-events-none`) |
| 가상 조이스틱 (왼쪽 아래) | 터치 화면만. 여백 28px, 바깥 원 반지름 56px, 손잡이 반지름 26px, 가운데 8px 안 멈춤 (`scene.ts`의 `JOYSTICK`) | FR-012. 지금 코드 그대로 |
| 🏘 이웃집 패널 (왼쪽 위) | [favorites.md](favorites.md) 3절 | FR-030, FR-028 |
| 환영 문구 (가운데 위) | 3절 | FR-003·004 |
| 640px 미만 겹침 | 왼쪽 위에서 아래로 [환영 문구] → [🏘 패널] 차례로 쌓는다 (겹치지 않음) | SC-004 |

## 2. 광장 데이터 (`TownData`, 서버 → `TownGame`)

모양은 [../data-model.md](../data-model.md) 3.1. `page.tsx`가 `Promise.all`로 읽는다:

| 값 | 회원 | 방문자 |
|---|---|---|
| `player` | `{ nickname, characterAsset, outfit }` (`getViewer()`) | `null` → 회색 방문자 캐릭터(`char.visitor`), 이름표 `구경하는 중` (FR-002) |
| `myHouse` | `getMyHouse(나)` | `null` (Edge Cases: 방문자에게는 내 집이 없다) |
| `neighbors` | `getTownHouses(나)`: 즐겨찾기 블로그, 최근 공개 글 순, 최대 10 ([favorites.md](favorites.md) 4절) | `getVisitorHouses()`: 인기 100곳 중 무작위 10 |
| `panel` | `listMyNeighbors(나)`에서 `TownLink` 필드(`slug`, `title`, `nickname`, `favorite`)만 | `neighbors`와 같은 블로그 (`favorite` = false) |
| `attendedToday` / `attendanceDay` | 오늘 출석 여부 / game의 일차 | `false` / `null` |

- 광장을 열 때 한 번 읽는다. 열린 동안 바뀌지 않는다 (Edge Cases).
- 집마다 `stage`(주인 레벨로 1~3), `roof`(고른 지붕 색, 없으면 배경 색)를 서버가 정한다 → 내 광장·남의 광장·방문자 광장이 같다 (FR-040, FR-059, SC-014).

## 3. 환영 문구 (FR-003·004, SC-002)

| 항목 | 계약 |
|---|---|
| 신호 | 쿠키 `bv_welcome` — 값 `1`, Path `/town`, Max-Age 600, SameSite=Lax, 배포 환경 Secure, HttpOnly 아님. **auth의 `signUp`이 가입 성공 뒤 심는다** (지금의 `/town?welcome=1` 대신) |
| 서버 | `/town`이 `cookies().get("bv_welcome")?.value === "1"`이고 회원이면 `WelcomeBanner`(`src/components/town/welcome-banner.tsx`, 클라이언트)에 `initial = true` |
| 브라우저 | 첫 그리기에서는 아무것도 없음. 마운트 뒤 `initial`이 참이고 `document.cookie`에 `bv_welcome`이 아직 있으면 문구를 열고 쿠키를 즉시 지운다(`Max-Age=0; Path=/town`) |
| 문구 | `🎉` + "**{닉네임}**님, Blogville에 오신 걸 환영해요! 가입 선물로 🪙 100 코인을 드렸어요. 광장 아래쪽 **내 집**에 들어가서 첫 글을 써 보세요. 위쪽 **게시판**에서 출석 도장도 받을 수 있어요." + 농장 안내 한 문장 `왼쪽 **동물 농장**에서 첫 알을 받아 동물을 키워 보세요.` *(지금 코드 문장, spec은 팀 확정으로 남김)* |
| 닫기 | 오른쪽 [✕] (접근 이름 `환영 문구 닫기`, 44×44px) → 화면에서 닫힘. 주소 이동 없음 |
| 다시 보이지 않음 | 새로고침, 다른 화면 → 뒤로 가기, 다른 화면 → [← 광장으로 나가기], 같은 주소 다시 열기 — 모두 쿠키가 없어 안 보인다 |
| 문구가 없는 경우 | 방문자, 쿠키 없음, 가입 뒤 10분이 지남 |

## 4. 건물과 입구 (FR-019~024, US3)

| 입구 | 아이콘·이름 (안내 문구의 `{아이콘} {이름}`) | 회원 이동 | 방문자 |
|---|---|---|---|
| 마을 게시판 왼쪽 절반 | `📋 마을 소식` | `/feed` | `/feed` (들어감) |
| 마을 게시판 오른쪽 절반 | `📮 출석 체크` (오늘 출석했으면 `📮 출석 체크 (오늘 완료)`) | `/attendance` | 로그인 안내 → `/` |
| 상점 | `🏪 상점` | `/shop` | 로그인 안내 → `/` |
| 동물 농장 (광장 왼쪽) | `🐮 동물 농장` | `/farm` | 로그인 안내 → `/` |
| 내 집 (아래) | `🏠 내 집` | `/@{내 주소}` | 없음 |
| 이웃집 (바깥 위·아래 줄, 최대 10) | `🏠 {닉네임}의 집` | `/@{주소}` | `/@{주소}` (들어감) |

| 규칙 | 계약 |
|---|---|
| 안내 | 캐릭터 발밑이 입구에서 90px 안이면 가장 가까운 입구 하나에 `{아이콘} {이름} · Space 들어가기`, 방문자 + 회원 전용 입구면 `{아이콘} {이름} · Space 로그인하고 이용하기` (FR-020, FR-022). 90px 밖이면 안내 없음, Space·Enter 무시 |
| 들어가기 | 안내가 보일 때 Space 또는 Enter → 이동. 건물 클릭·탭: 90px 안이면 바로 들어가고, 밖이면 문 앞까지 걸어간다(벽에 막히면 멈춤) (FR-021) |
| 이동 방법 | `router.push(href)`, 로그인 안내는 `router.push("/")` (지금과 같음) |
| 건물 이름표 / 부제 | 게시판 `마을 게시판` / `마을 소식 · 출석 체크 (오늘 완료 ✅)` 또는 `(보상 받기 🎁)` (FR-023) · 상점 `상점` / `아바타·가구·배경·성장 아이템` *(plan 임시)* · 농장 `동물 농장` / `알 부화 · 동물 키우기` · 내 집 `내 집` / 블로그 이름 · 이웃집 `{닉네임}의 집`(닉네임 12자 넘으면 앞 12자 + `…`) / 블로그 이름 (FR-024) |
| 집 모양 | `stage` 1: 세모 지붕 + 문 하나 / 2: + 창문·굴뚝·꽃 상자, 더 큼 / 3: + 다락방 창, 더 큼. 지붕 = `roof`, 문 옆에 주인 캐릭터(차림 포함). 단계와 상관없이 문 앞 입구로 들어간다. 가장 큰 집끼리도 겹치지 않는다 (FR-058·060) |
| 회원 전용 화면 직접 주소 | `/attendance`, `/shop`, `/farm`, `/closet` 등은 각 페이지의 `requireMember()`가 방문자를 `/`로 보낸다 (Edge Cases) |

## 5. 이동 (FR-010~018, US2)

| 입력 | 계약 |
|---|---|
| 방향키·WASD | 상하좌우·대각선, 화면에서 보이는 속도 초당 230px(대각선도 같음). 걷는 중 키를 누르면 키가 이긴다 |
| 클릭·탭 (땅) | 그 지점까지 걷는다. 벽에 막히거나 8px 안이면 멈춘다 |
| 조이스틱 | 많이 밀수록 빠르게(최대 230), 가운데 8px 안은 멈춤. 우선순위 키보드 → 조이스틱 → 탭 목표 |
| 경계 | 광장(논리 1800 × 1400) 밖으로 나가지 않는다. 건물·분수·나무·가로등 밑동은 지나갈 수 없다 |
| 가림 | 물체 앞(화면 아래쪽)의 캐릭터는 물체보다 앞, 뒤(위쪽)는 가려진다 |
| 스크롤 | 광장 안에서 방향키·Space가 페이지를 스크롤하지 않는다 (`addCapture`) |
| 캐릭터 | 이름표(흰 바탕), 이동 방향에 따라 좌우 반전, 걷는 동안 통통. 바닥·길·나무 배치는 고정 시드로 늘 같다 |
| 2.5D (단계 13) | 바닥이 마름모 타일(64 × 32)이고 위 표가 모두 같게 동작한다. 2D로 돌아가는 설정·화면은 없다 (FR-037·038) |

## 6. 검증용 인터페이스 (research R-19)

| 이름 | 어디 | 계약 |
|---|---|---|
| 숨은 집 목록 | `/town` 안 `<ul hidden data-town-houses>` | 광장에 놓인 집마다 `<li data-slug="…" data-stage="1\|2\|3" data-roof="#rrggbb" data-mine="true\|false">{이름표}</li>`. `TownData.myHouse` + `neighbors`와 같은 값 |
| 차림 표시 | `TownGame` 바깥 요소 `data-player-look` | shop 요청 (`lookKey(asset, outfit)`) |
| 개발 모드 훅 | `window.__blogvilleTown` — `process.env.NODE_ENV !== "production"`일 때만 | `entrances()` → `[{ label, emoji, target, screen: { x, y } }]`, `player()` → `{ name, logical: { x, y }, screen: { x, y } }`(`name` = 머리 위 이름표 글자), `prompt()` → 지금 안내 문구 또는 `null`, `teleport(label)` → 그 입구 앞(90px 안)으로 옮김, `fps()` → Phaser `game.loop.actualFps`. 배포 빌드(`next build`)에는 없다 |

## 7. 다른 화면 헤더의 나가기 버튼 (FR-008)

| 주소 | `← 광장으로 나가기` (640px 미만 `← 나가기`) |
|---|---|
| `/`, `/town` | 없음 |
| 그 밖의 모든 화면 (`/feed`, `/attendance`, `/shop`, `/closet`, `/farm`, `/@…`, `/write`, `/settings/…`, `/wallet`, `/notifications`, 404 등) | 있음 → `/town` |

`src/components/exit-button.tsx`의 숨김 목록에서 `/onboarding`을 뺀다(auth가 화면을 없앤 뒤). 헤더 전체 배치는 [header-roof.md](header-roof.md).
