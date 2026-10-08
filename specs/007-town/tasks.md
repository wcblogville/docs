---

description: "광장 (TOWN) 구현 작업 목록"
---

# Tasks: 광장 (TOWN)

**Input**: Design documents from `/specs/007-town/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: plan.md(Testing)·research.md R-23·quickstart.md가 단위 테스트(`scripts/test-town.ts`, `npm run test:town`)와 E2E(`e2e/town.mjs`, `e2e/favorites.mjs` 새로, `e2e/farm.mjs`·`e2e/mobile.mjs`·`e2e/decisions.mjs` 고침)를 요구하므로 테스트 작업을 넣는다. 각 이야기의 테스트는 구현 전에 쓰고 실패하는지 먼저 확인한다.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- 코드 경로는 모두 코드 저장소 `ehgo508/blogville`(`main` `feb4c05`) 루트 기준이다 — 한 Next.js 프로젝트(`src/app/`, `src/components/`, `src/server/`, `src/lib/`, `src/db/`, `drizzle/`, `scripts/`, `e2e/`).
- 문서 경로(`specs/007-town/…`)는 이 문서 저장소 기준이다.
- 다른 spec 소유 파일(`src/server/dal.ts`, `src/app/blog/[slug]/page.tsx`, `src/app/closet/page.tsx`, `e2e/helpers.mjs`)에는 한두 줄만 끼운다 (테이블 담당·공통 모듈 소유 규칙).
- *(plan 임시)* 표시 문구는 spec에 없는 문구로, plan.md 남은 문제 4에 따라 팀 확정 전 임시로 쓴다.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 라이브러리 동작 확인과 테스트 뼈대

- [ ] T001 `npm install` 뒤 research.md R-24 표의 항목을 설치된 문서로 확인하고 결과를 PR 설명에 적는다: Next.js 16.3.8 `cookies()` 읽기 전용·`signUp` 뒤 `redirect("/town")` 응답에서 `bv_welcome` 보이는지·뒤로 가기 라우터 캐시·쿠키 읽는 페이지의 요청마다 렌더링·`process.env.NODE_ENV` 빌드 치환, Phaser ^4.2.1 Graphics→텍스처(`generateTexture`/RenderTexture/DynamicTexture)·최대 텍스처 크기·`cameras.main.startFollow`/`setBounds`·포인터 `worldX/worldY`·`game.loop.actualFps` (`node_modules/phaser/skills/`, `node_modules/phaser/changelog/v4/4.0/MIGRATION-GUIDE.md`), drizzle-kit `unique()` → `ADD CONSTRAINT … UNIQUE`
- [ ] T002 [P] `scripts/test-town.ts` 뼈대를 만든다 (관례: `expect(name, got, want)`·`✅/❌`·실패 시 `process.exit(1)`, 맨 위 요구사항 ID 주석 `TOWN-04·07·09·11`)
- [ ] T003 `package.json`에 `"test:town": "tsx scripts/test-town.ts"`를 더하고 `test` 체인 끝에 `npm run test:town`을 붙인다 (T002 뒤)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 모든 이야기가 쓰는 마이그레이션·순수 함수·광장 데이터 모양·집 공통 계산·검증 수단

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T004 `src/db/schema.ts`의 `userAnimals` 블록에 `unique("user_animals_user_id_id_uq").on(t.userId, t.id)`를 더한다 (data-model.md T-M1, FR-052 blog 전시 복합 FK 대상, 데이터 이전 없음)
- [ ] T005 `npm run db:generate`로 `drizzle/NNNN_<이름>.sql`을 만들고 맨 위에 `-- TOWN-09 (요청: blog, BLOG-04): 전시 동물 복합 FK가 가리킬 UNIQUE (ERD 3.11, 7장 6-2)` 주석을 단 뒤 `npm run db:migrate`, `\d user_animals`에 `user_animals_user_id_id_uq`가 보이는지 확인한다 (T004 뒤. 다른 spec 마이그레이션이 먼저 merge되면 최신 `main`에서 generate를 다시 돌린다)
- [ ] T006 [P] `src/lib/town.ts`(새)에 순수 함수·상수를 만든다: `houseStage(level)` (1~9 → 1, 10~29 → 2, 30 이상 → 3), `ROOF_PALETTE` (`red`·`orange`·`yellow`·`green`·`sky`·`blue`·`purple`·`brown` → `{ hex, label }` 빨강·주황·노랑·초록·하늘·파랑·보라·갈색), `houseRoof(roofColor, backgroundAsset)` (`roofColor`가 있으면 `ROOF_PALETTE[roofColor].hex`, 없으면 `backgroundAccent(backgroundAsset)`), `houseLabel(nickname)` (코드 포인트 12자 초과면 앞 12자 + `…`, 뒤에 `의 집`), `FAVORITE_LIMIT = 10`·`canFavorite(count)` (= count < 10), `VISITOR_POOL = 100`·`TOWN_HOUSE_SLOTS = 10`·`sampleDistinct(list, n, randInt)` (겹치지 않게 `min(n, list.length)`개). 코드값 목록은 blog의 `src/lib/blog.ts` `ROOF_COLORS`를 읽어 쓴다 (없으면 같은 8개 값으로 임시 선언하고 blog B2 뒤 바꾼다)
- [ ] T007 [P] `scripts/test-town.ts`에 순수 함수 시험을 쓴다: `houseStage` Lv.1·9·10·29·30·99 → 1·1·2·2·3·3, `houseRoof` 코드값 → 정한 색 / `null` → 배경 강조색 / 배경이 바뀌어도 고른 색 그대로, `houseLabel` 12자 이하 그대로 + `의 집` / 13자·20자·이모지 닉네임은 12자 + `…의 집`, `sampleDistinct` 100개 → 10개(겹침 없음·모두 후보 안)·7개 → 7개·0개 → 빈 배열(고정 난수로 재현), `canFavorite` 9 → 가능 / 10 → 불가, 부화 비중 `pickWeighted` 1,000번(고정 난수열) 40·30·20·10%에서 ±5%p (SC-009)
- [ ] T008 [P] `src/components/town/types.ts`를 data-model.md 3.1 모양으로 바꾼다: `TownHouse { slug, title, nickname, characterAsset, outfit: string[], stage: 1 | 2 | 3, roof: string }`, `TownLink { slug, title, nickname, favorite }`, `MyNeighbor = TownLink & { userId, characterAsset, outfit }`, `TownData { player | null, myHouse | null, neighbors: TownHouse[], panel: TownLink[], attendedToday, attendanceDay: number | null }` (`backgroundAsset` 제거, 환영 여부는 넣지 않음)
- [ ] T009 `src/server/town.ts`에 집 공통 조립 도우미를 만든다: 블로그 행 목록(≤ 11) → 주인들의 `point_ledger` `SUM(exp_delta)`를 한 쿼리로 읽어 `levelFromExp` → `houseStage`로 `stage`, `blogs.roof_color`와 장착 배경으로 `houseRoof`로 `roof`, shop `outfitOf`가 있으면 `outfit`(없으면 `[]`)을 채운 `TownHouse[]`. 집 단계는 저장하지 않는다 (FR-059, R-08) (T006·T008 뒤)
- [ ] T010 `src/server/town.ts`에 `getMyHouse(userId): Promise<TownHouse | null>`를 T009 도우미로 만든다 (FR-027)
- [ ] T011 `src/components/town/scene.ts`에서 `NEIGHBOR_SLOTS`를 8 → 10(위 5·아래 5)으로 바꾸고 `layout()`을 내보내며, 집 그리기가 서버 값 `stage`·`roof`를 받도록 바꾼다 (`DEFAULT_HOUSE_STAGE`·배경 강조색 계산 제거) (FR-024, FR-040, FR-059) (T008 뒤)
- [ ] T012 `src/components/town/scene.ts`·`src/components/town/town-game.tsx`에 개발 모드 전용 훅 `window.__blogvilleTown`을 연다 — `process.env.NODE_ENV !== "production"`일 때만, `entrances()` → `[{ label, emoji, target, screen: { x, y } }]`, `player()` → `{ name, logical: { x, y }, screen: { x, y } }`, `prompt()` → 안내 문구 또는 `null`, `teleport(label)` → 입구 앞 90px 안, `fps()` → `game.loop.actualFps`. `TownGame` 바깥 요소에 `data-player-look`(shop `lookKey(asset, outfit)`)를 둔다 (R-19, contracts/town-screen.md 6절) (T011 뒤)
- [ ] T013 [P] `e2e/helpers.mjs`(auth 소유)에 광장 훅 `window.__blogvilleTown`이 생길 때까지 기다리는 도우미를 한두 줄 더한다

**Checkpoint**: 마이그레이션 적용, `npm run test:town` 순수 함수 줄 모두 ✅, 광장 데이터 모양·집 조립·검증 훅 준비 완료

---

## Phase 3: User Story 1 - 광장에서 시작하고 구경하기 (Priority: P1) 🎯 MVP

**Goal**: 로그인한 회원·방문자가 광장에 도착하고, 가입 직후 환영 문구가 딱 한 번 보이며, 헤더에 이동 메뉴가 없고 광장 밖 화면에만 나가기 버튼이 있다 (TOWN-01, FR-001~009)

**Independent Test**: 아이디로 로그인 → `/town` 도착, 방문자로 `/town`에서 `구경하는 중` 캐릭터 이동, 가입 직후 환영 문구 1번(새로고침·뒤로 가기 10회에 0번), [✕]로 닫힘, 375px에서 `← 나가기`

### Tests for User Story 1 ⚠️

- [ ] T014 [P] [US1] `e2e/town.mjs`(새, 맨 위 `TOWN-01·02·03·05·07·10·11` 주석, `.env.local` + `pg`, 실행마다 새 아이디, `check()`·실패 시 `exit(1)`, 스크린샷 폴더 첫 인자)에 US1 시나리오를 쓴다: 로그인 → `/town`, 가입 직후 환영 문구가 가운데 위에 보임, 새로고침·뒤로 가기·다른 화면 → [← 광장으로 나가기] 10회 모두 안 보임(SC-002), [✕](접근 이름 `환영 문구 닫기`) 닫힘·주소 그대로, 방문자 `/town`에서 `player().name === "구경하는 중"`·이동, `/town` 헤더에 이동 메뉴·나가기 버튼 없음, `/feed`·`/attendance`·`/shop`·`/farm` 헤더 [← 광장으로 나가기] → `/town`, 375px에서 `← 나가기`, 콘솔 오류 0개

### Implementation for User Story 1

- [ ] T015 [P] [US1] `src/components/town/welcome-banner.tsx`(새, 클라이언트)를 만든다: 첫 그리기에서는 아무것도 없고, 마운트 뒤 `initial`이 참이고 `document.cookie`에 `bv_welcome`이 있으면 문구를 열고 즉시 `bv_welcome=; Max-Age=0; Path=/town`로 지운다. 문구 `🎉` + "**{닉네임}**님, Blogville에 오신 걸 환영해요! 가입 선물로 🪙 100 코인을 드렸어요. 광장 아래쪽 **내 집**에 들어가서 첫 글을 써 보세요. 위쪽 **게시판**에서 출석 도장도 받을 수 있어요." + `왼쪽 **동물 농장**에서 첫 알을 받아 동물을 키워 보세요.`(지금 코드 문장, 팀 확정 전), 오른쪽 [✕] 44×44px·접근 이름 `환영 문구 닫기`·주소 이동 없음 (FR-003·004, R-02. React 19 린트 effect 안 `setState` 규칙은 T001 결과대로)
- [ ] T016 [US1] `src/app/town/page.tsx`를 고친다: `?welcome=1` 처리와 `/onboarding` 리다이렉트를 지우고(auth 단계 1 뒤), `cookies().get("bv_welcome")?.value === "1"`이고 회원이면 `WelcomeBanner`에 `initial = true`, 새 `TownData`(`player`, `myHouse` = `getMyHouse`, `neighbors`, `panel`, `attendedToday`, `attendanceDay`)를 `Promise.all`로 읽어 넘기고, `<h1 class="sr-only">중앙 광장</h1>`·탭 제목 `중앙 광장 | Blogville`, 높이 방문자 `100dvh − var(--header-h)` / 회원 `100dvh − var(--header-h-member)`·최소 420px·페이지 스크롤·푸터 없음, 640px 미만에서 [환영 문구] → [🏘 패널] 차례로 쌓기 (FR-005·006, contracts/town-screen.md 1~3절) (T015 뒤. `neighbors`·`panel`은 US4·US5 전까지 빈 배열)
- [ ] T017 [P] [US1] `src/app/town/page.tsx`에 숨은 집 목록 `<ul hidden data-town-houses>`를 둔다 — 집마다 `<li data-slug data-stage="1|2|3" data-roof="#rrggbb" data-mine="true|false">{이름표}</li>`, `myHouse` + `neighbors`와 같은 값 (R-19) (T016과 같은 파일이면 T016 뒤에 순서대로)
- [ ] T018 [P] [US1] `src/components/exit-button.tsx`의 `HIDDEN_ON`에서 `/onboarding`을 빼고 `HomeLogo` 숨김 처리를 지운다 — `/`·`/town`에서만 숨고 그 밖 모든 화면에서 `← 광장으로 나가기`(640px 미만 `← 나가기`) → `/town` (FR-008, contracts/town-screen.md 7절)
- [ ] T019 [US1] auth 담당과 `signUp`이 가입 성공 뒤 `bv_welcome`(값 `1`, Path `/town`, Max-Age 600, SameSite=Lax, 배포 Secure, HttpOnly 아님)을 심고 `/town`으로 보내는지 확인하고, 아직이면 `src/app/town/page.tsx`에서 `?welcome=1`을 임시로 함께 읽는다 (plan 의존성 auth (1), T6과 같은 PR)

**Checkpoint**: US1 단독으로 로그인·방문자 진입·환영 문구 1회·나가기 버튼이 동작

---

## Phase 4: User Story 2 - 캐릭터를 움직여 광장 돌아다니기 (Priority: P1)

**Goal**: 키보드·클릭·터치(조이스틱·탭)로 모든 화면 크기에서 광장을 걷고, 벽·가장자리에서 멈춘다. 휴대폰 간단 메뉴를 없앤다 (TOWN-02, FR-010~018, constitution VI)

**Independent Test**: PC에서 방향키·WASD·클릭으로, 375px 터치에서 조이스틱·탭으로 이동, 가장자리·벽에서 멈춤, 방향키·Space가 페이지를 스크롤하지 않음

### Tests for User Story 2 ⚠️

- [ ] T020 [P] [US2] `e2e/town.mjs`에 이동 시나리오를 더한다: 방향키·WASD로 `player().logical` 변화·대각선 속도 = 직선 속도(초당 230px), 땅 클릭 → 그 지점까지 걷기, 걷는 중 키 우선, 가장자리에서 1800 × 1400 밖으로 안 나감, 방향키·Space에 `window.scrollY` 그대로, 키보드 화면 조작 안내 `방향키·WASD 또는 클릭으로 이동 · 건물 앞에서 Space로 들어가기`·조이스틱 없음
- [ ] T021 [P] [US2] `e2e/mobile.mjs`를 고친다: 375px(터치)에서 간단 메뉴 대신 캔버스가 보이고 왼쪽 아래 조이스틱, 조작 안내 `조이스틱이나 탭으로 이동 · 건물을 탭해서 들어가기`, 조이스틱 끌기로 이동, 가로 스크롤 없음, `❌`가 있으면 `exit(1)` 추가
- [ ] T022 [P] [US2] `e2e/decisions.mjs`의 조이스틱 확인을 새 광장(휴대폰도 게임) 기준으로 고치고 `exit(1)`을 더한다

### Implementation for User Story 2

- [ ] T023 [US2] 팀에 휴대폰 광장(간단 메뉴 삭제, 10/6 회의 결정과 반대) 확인을 받는다 — 유지로 정해지면 T024~T026을 빼고 `town-menu.tsx`에 환영 쿠키·상점 부제·shop `outfit` 줄을 고친다 (plan 남은 문제 1, R-01)
- [ ] T024 [US2] `src/components/town/town-game.tsx`에서 `PHONE_MEDIA` 분기를 지워 모든 화면 크기에서 게임을 띄운다 (`Scale.RESIZE`로 크기 변화에 맞춤, 터치는 `JOYSTICK` 그대로: 여백 28px, 바깥 원 56px, 손잡이 26px, 가운데 8px 멈춤) (FR-012, SC-003) (T023 뒤)
- [ ] T025 [US2] `src/components/town/town-menu.tsx`를 지우고 `src/app/town/page.tsx`의 휴대폰 분기(`phone:` 변형)를 없앤다 (T024 뒤)
- [ ] T026 [P] [US2] `src/lib/device.ts`(`PHONE_MEDIA`는 남김)와 `src/app/globals.css`(phone 변형은 남김)에 "광장은 휴대폰에서도 게임" 주석을 단다
- [ ] T027 [US2] `src/components/town/scene.ts`의 이동·충돌·가림(FR-010~018: 속도 230, 키보드 → 조이스틱 → 탭 우선순위, 건물·분수·나무·가로등 밑동 충돌, 아랫변 y 깊이, `addCapture`, 이름표·좌우 반전·통통, 고정 시드)이 휴대폰 크기에서도 그대로인지 확인하고 어긋나는 곳만 고친다

**Checkpoint**: US1 + US2 — 모든 화면 크기에서 광장 진입·이동

---

## Phase 5: User Story 3 - 건물에 들어가 다른 장소로 가기 (Priority: P1)

**Goal**: 입구 90px 안 안내, Space·Enter·클릭·탭으로 들어가기, 방문자 로그인 안내, 출석 표시, 이름표 규칙 (TOWN-03, FR-019~024)

**Independent Test**: 각 입구 앞에서 Space로 올바른 화면, 90px 밖 Space 무시, 방문자는 회원 전용 입구에서 `Space 로그인하고 이용하기` → `/`, 출석한 회원의 게시판 부제

### Tests for User Story 3 ⚠️

- [ ] T028 [P] [US3] `e2e/town.mjs`에 입구 시나리오를 더한다: `teleport(label)` 뒤 `prompt()`가 `{아이콘} {이름} · Space 들어가기`(예: `📮 출석 체크 · Space 들어가기`), Space·Enter로 `/feed`·`/attendance`·`/shop`·`/farm`·`/@{내 주소}`, 90px 밖 `prompt() === null`·Space 무시, 먼 건물 클릭 → 문 앞까지 걸어감·가까우면 바로 들어감, 게시판 왼쪽 절반 → `/feed`·오른쪽 → `/attendance`, 출석한 회원 부제 `마을 소식 · 출석 체크 (오늘 완료 ✅)`·안내 `출석 체크 (오늘 완료)`, 방문자 상점·출석·농장 `Space 로그인하고 이용하기` → `/`, 방문자 마을 소식·이웃집은 로그인 없이 들어감, 방문자가 `/attendance`·`/shop`·`/farm`·`/closet` 직접 주소 → `/`

### Implementation for User Story 3

- [ ] T029 [US3] `src/components/town/scene.ts`의 상점 부제 `캐릭터·배경`을 `아바타·가구·배경·성장 아이템` *(plan 임시)*으로, 농장 부제를 `알 부화 · 동물 키우기`로 바꾸고 "🚧 준비 중 🚧" 표시를 지운다 (FR-019, R-21, spec 기본값)
- [ ] T030 [US3] `src/components/town/scene.ts`의 집 이름표를 `houseLabel(nickname)`로 바꾼다 — 이웃집 `{닉네임}의 집`(닉네임 12자 넘으면 앞 12자 + `…`) / 블로그 이름, 내 집 `내 집` / 블로그 이름, 문 옆 주인 캐릭터 (FR-024, R-18) (T029 뒤, 같은 파일)
- [ ] T031 [US3] `src/components/town/scene.ts`의 게시판 부제를 `attendedToday`로 `마을 소식 · 출석 체크 (오늘 완료 ✅)` / `마을 소식 · 출석 체크 (보상 받기 🎁)`, 출석 입구 이름을 `출석 체크 (오늘 완료)`로 맞춘다 (FR-023, plan 남은 문제 2: TOWN FR-023 문구 우선, `attendanceDay`는 넘겨만 둠) (T030 뒤)
- [ ] T032 [US3] `src/server/town.ts`의 `hasAttendedToday()` 쿼리를 game의 `getViewer()` `attendance` 값으로 바꾸고 `attendanceDay`도 함께 넘긴다 — game 단계 6 전이면 지금 쿼리(`attendances` 오늘 `todayKST()` 행) 그대로 둔다 (T12, FR-023)

**Checkpoint**: P1 이야기 1~3 완료 — 광장이 모든 장소로 가는 허브로 동작 (MVP)

---

## Phase 6: User Story 4 - 즐겨찾는 이웃의 집이 광장에 생기기 (Priority: P2)

**Goal**: 이웃 중 최대 10명을 즐겨찾기하고 그 집만 광장에 놓는다. 내 블로그 홈에 주인만 보는 내 이웃 목록(☆/⭐), 🏘 패널은 모든 이웃 링크 (TOWN-04, TOWN-08, FR-025~028·030~035)

**Independent Test**: 이웃 3명 중 2명 즐겨찾기 → 광장에 2채, 11번째 거부, 9명에서 동시 요청 → 10 초과 0건, 이웃 취소 → 다음 방문에 집 사라짐

**선행**: social `follows.is_favorite` (U4)

### Tests for User Story 4 ⚠️

- [ ] T033 [P] [US4] `e2e/favorites.mjs`(새, 맨 위 `TOWN-04·08` 주석)에 회원 시나리오를 쓴다: 내 블로그 홈에 내 이웃 목록·☆ (주인만, 다른 회원·방문자에게는 DOM에 없음), ☆ → ⭐ → ☆, 10명에서 11번째 → `즐겨찾기할 이웃은 최대 10명이에요`, 9명에서 두 탭 동시 요청 → DB `is_favorite = true` 10개 이하(SC-006), 즐겨찾기 10곳(글 없는 블로그 포함) → `[data-town-houses] li` 10개·즐겨찾기 안 한 블로그 0개(SC-007), 즐겨찾기 0명 → 빈 안내 `마음에 드는 블로그를 즐겨찾기하면 광장에 집이 생겨요`, 이웃 0명 → `아직 이웃이 없어요. 마을 소식에서 마음에 드는 블로그를 이웃으로 추가해 보세요.`, 이웃 취소 → 다음 방문에 집 사라짐(US4-8), 즐겨찾기 안 한 이웃은 🏘 패널 링크로 들어감, 이웃집 입구 → 그 블로그 홈, 이웃 아닌 대상 ID로 조작한 `toggleFavorite` 요청 → 거부·DB 그대로
- [ ] T034 [P] [US4] `e2e/decisions.mjs`의 TOWN-04 옛 규칙("글 없는 블로그는 이웃집에 없음 / 공개 글을 쓰면 나타남")을 즐겨찾기 규칙(즐겨찾기한 블로그만, 글 없어도 기본 집)으로 고친다

### Implementation for User Story 4

- [ ] T035 [US4] `src/server/town.ts`의 `getTownHouses(userId)`를 바꾼다: 내가 `is_favorite = true`로 즐겨찾기한 이웃의 블로그만 최대 10, 정렬 최근 공개 글(`visibility = 'public'`의 `MAX(created_at)`) DESC(없으면 뒤) → `blogs.created_at` DESC → `blogs.id` DESC, 공개 글 없는 블로그도 기본 집, T009 도우미로 `stage`·`roof` (FR-025·026, contracts/favorites.md 4절)
- [ ] T036 [US4] `src/server/town.ts`에 `listMyNeighbors(userId): Promise<MyNeighbor[]>`를 만든다 — 내 모든 이웃, 정렬 즐겨찾기 먼저 → 최근 공개 글 최신순(없으면 뒤) → 이웃 추가 최신순 (contracts/favorites.md 1절) (T035 뒤, 같은 파일)
- [ ] T037 [US4] `src/app/town/actions.ts`(새, `"use server"`)에 `toggleFavorite(followeeId: unknown): Promise<{ ok: true; favorite: boolean } | { ok: false; error: string }>`를 만든다: 첫 줄 `requireMember()`, `followeeId`가 1~64자 문자열이 아니거나 나 자신이면 `이웃으로 추가한 블로그만 즐겨찾기할 수 있어요` *(plan 임시)*, 트랜잭션 + `lockUser(tx, 나)`, (나, 대상) `follows` 행이 없으면 같은 거부, 켜져 있으면 `is_favorite = false`, 꺼져 있고 내 즐겨찾기 수 ≥ 10이면 `즐겨찾기할 이웃은 최대 10명이에요`, 아니면 `is_favorite = true`, 성공 뒤 `revalidatePath("/", "layout")`. 원장 기록 없음, Drizzle 값 바인딩만 (FR-031~035, SC-006, Complexity Tracking)
- [ ] T038 [P] [US4] `src/components/town/favorite-button.tsx`(새, 클라이언트)를 만든다: ☆/⭐ 버튼 44×44px 이상, 접근 이름 `{닉네임} 즐겨찾기` / `{닉네임} 즐겨찾기 취소` *(plan 임시)*, `toggleFavorite(userId)` 결과의 `favorite`로만 바꿈(미리 바꾸지 않음), 실패 문구는 `role="status"` 빨간 한 줄
- [ ] T039 [US4] `src/components/town/my-neighbors.tsx`(새, 서버)를 만든다: 제목 `🏘 내 이웃` + `⭐ {즐겨찾기 수} / 10` *(plan 임시)*, 줄마다 캐릭터 얼굴·블로그 이름(굵게)·닉네임 → `/@{주소}` 링크 + `FavoriteButton`, 빈 목록 `아직 이웃이 없어요. 마을 소식에서 마음에 드는 블로그를 이웃으로 추가해 보세요.`, 375px `truncate`·가로 스크롤 없음 (T036·T038 뒤)
- [ ] T040 [US4] `src/app/blog/[slug]/page.tsx`(blog 소유)의 왼쪽 `<aside>` 카테고리 카드 아래에 `{isOwner && <MyNeighbors userId={…} />}` 한 줄을 끼운다 — 주인이 아니면 서버가 그리지 않는다 (FR-032, US4-2) (T039 뒤)
- [ ] T041 [P] [US4] `src/components/town/neighbor-panel.tsx`(새)를 만든다: `<details open>`, 제목 `🏘 이웃집 {이웃 수}`(이웃 0명이면 `🏘 이웃집`), 즐겨찾기는 이름 앞 ⭐, 줄마다 `/@{주소}` 링크 `{블로그 이름} · {닉네임}`, 즐겨찾기 0명 또는 이웃 0명이면 `마음에 드는 블로그를 즐겨찾기하면 광장에 집이 생겨요`, 패널 안 스크롤 `max-h-[40dvh]`, 접기 제목·링크 줄 높이 44px 이상 (FR-006·028·030, contracts/favorites.md 3절)
- [ ] T042 [US4] `src/app/town/page.tsx`에서 회원이면 `neighbors = getTownHouses(나)`, `panel = listMyNeighbors(나)`에서 `TownLink` 필드(`slug`, `title`, `nickname`, `favorite`)만 골라 넘기고(회원 ID를 게임 데이터에 싣지 않음), 기존 패널을 `NeighborPanel`로 바꾼다 (T035·T036·T041 뒤)

**Checkpoint**: 회원 광장이 즐겨찾기 기준으로 동작, 내 이웃 목록·패널 완료

---

## Phase 7: User Story 5 - 방문자에게 인기 블로그 마을 보여주기 (Priority: P2)

**Goal**: 방문자 광장에 인기 블로그(이웃 수 → 최근 공개 글) 상위 100곳 중 무작위 10곳 (TOWN-04, FR-029)

**Independent Test**: 이웃 수가 다른 블로그(공개 글 없는 블로그 포함)로 방문자 광장을 20번 열어 10채·조합 변화·100위 밖/공개 글 없는 블로그 0건 확인

### Tests for User Story 5 ⚠️

- [ ] T043 [P] [US5] `e2e/favorites.mjs`에 방문자 시나리오를 더한다: 공개 글 블로그 10곳 이상 → `[data-town-houses] li` 10개, 새로고침으로 조합이 바뀔 수 있음, 공개 글 10곳 미만 → 그 수만큼만, 공개 글 없는 블로그(비공개 글만 포함) 20번 열기 0건, 100곳 초과 시 100위 밖 블로그 20번 열기 0건(SC-013), 방문자가 집에 들어가면 로그인 없이 블로그 홈, 패널 = 광장 집과 같은 블로그

### Implementation for User Story 5

- [ ] T044 [US5] `src/server/town.ts`에 `getVisitorHouses(): Promise<TownHouse[]>`를 만든다: 후보 = 공개 글이 있는 블로그를 이웃 수(`follows_followee_idx`로 센 팔로워 수) DESC → 최근 공개 글 DESC → `blogs.id` ASC로 상위 `VISITOR_POOL`(100), `sampleDistinct(후보, TOWN_HOUSE_SLOTS, randomInt)`(`node:crypto`)로 겹치지 않게 고르고 뽑힌 순서대로, T009 도우미로 `stage`·`roof` (FR-029, SC-013)
- [ ] T045 [US5] `src/app/town/page.tsx`에서 방문자면 `neighbors = getVisitorHouses()`, `panel` = 같은 블로그(`favorite = false`), `myHouse = null`, `attendedToday = false`, `attendanceDay = null`로 넘기고 요청마다 렌더링되는지(T001 결과) 확인한다 (T044 뒤)

**Checkpoint**: 회원·방문자 광장 집 규칙 모두 완료

---

## Phase 8: User Story 6 - 동물 농장에서 동물 키우기 (Priority: P2)

**Goal**: 알 받기·사기·부화·돌보기·공개 글 성장에 더해 상점 성장 아이템 사용, 꽉 참 문구를 spec대로, 버튼 44px (TOWN-09, FR-041~050)

**Independent Test**: 첫 알 → 부화 → 돌보기 3가지 → 성장 아이템 → 다 자람까지 성장치·보상·원장 합계 확인

**선행**: shop 성장 아이템(`items.type = 'growth'`, `growth_value`, `user_items.quantity`, 시드 3종, `consumeGrowthItem`), game `addLedgerEntry`(없으면 지금 직접 INSERT)

### Tests for User Story 6 ⚠️

- [ ] T046 [P] [US6] `e2e/farm.mjs`를 고친다: 성장 아이템 사용 → 성장 +성장치·수량 −1·같은 날 여러 번, 수량 0 → `가지고 있는 성장 아이템이 없어요` *(plan 임시)*, 알·다 자란 동물·남의 동물 → `돌볼 수 있는 동물이 아니에요`, 넘친 성장치 버림(`growth = grow_exp`), 수량 1로 두 탭 동시 → 하나만 성공, 조작 인자 → `잘못된 요청이에요`, 5마리 → `자리가 꽉 찼어요. 다 키운 뒤에 받을 수 있어요`, 레벨 보상 알 `[🎁 레벨 N 보상 알 (무료)]`(US6-2), 같은 무료 알 동시 → 한 번·`이미 받은 알이에요`, 같은 돌보기 동시 → 한 번(SC-008), 아직 안 된 레벨 알 → `아직 받을 수 없는 알이에요`, 코인 부족 → `코인이 N개 부족해요`, 남의 알 → `부화시킬 수 있는 알이 아니에요`, 오늘 이미 한 돌보기 → `오늘은 이미 밥 주기를 했어요`, 원장 `farm_care`·`farm_grown`·`egg_purchase` 합계 = 화면 잔액·레벨, [← 광장으로 나가기] → `/town`
- [ ] T047 [P] [US6] `e2e/mobile.mjs`에 375px `/farm`의 돌보기·🌱·알 버튼 44×44px 이상과 가로 스크롤 없음을 더한다 (SC-004)

### Implementation for User Story 6

- [ ] T048 [US6] `src/app/farm/actions.ts`의 꽉 참 문구 `FULL`을 `자리가 꽉 찼어요. 다 키운 뒤에 받을 수 있어요`로 바꾼다 (`claimEgg`·`buyEgg`, FR-043)
- [ ] T049 [US6] `src/app/farm/actions.ts`에 `applyGrowthItem(animalId, itemId)`를 더한다: 첫 줄 `requireMember()`, 두 인자 `parseId()`(이상하면 `잘못된 요청이에요`), `lockUser` 트랜잭션 안에서 내 `growing` 동물 확인(아니면 `돌볼 수 있는 동물이 아니에요`) → shop `consumeGrowthItem(tx, 나, itemId)`로 수량 −1(성장 아이템이 아니거나 수량 0·없음 → `가지고 있는 성장 아이템이 없어요` *(plan 임시)*) → 기존 `addGrowth`(성장 + `growth_value`, 넘치면 버림, 다 자라면 `farm_grown`)를 한 트랜잭션으로, 성공 `🌱 {아이템 이름} 사용! 성장 +{성장치}` *(plan 임시)* 또는 `🎉 {이름}가 다 자랐어요! 경험치 +X, 🪙 +Y`, 경험치·원장 기록 없음(다 자랄 때 `farm_grown`만), `revalidatePath("/", "layout")` (FR-048·050, contracts/farm.md 2절) (T048 뒤)
- [ ] T050 [P] [US6] `src/server/farm.ts`의 `getFarm`에 내 성장 아이템(`items.type = 'growth' AND user_items.quantity > 0`: id·이름·`growth_value`·수량)을 더하고, `farm_grown` 직접 INSERT를 game `addLedgerEntry`로 바꾼다 (game 단계 6 뒤, 기록 한 줄·값 같음)
- [ ] T051 [US6] `src/app/farm/page.tsx`의 안내 문장에 `상점의 성장 아이템으로도 자라요` *(plan 임시)*를 더하고 성장 아이템 데이터를 `farm-view.tsx`에 넘긴다 (T050 뒤)
- [ ] T052 [US6] `src/app/farm/farm-view.tsx`를 고친다: 동물 카드에 🌱 성장 아이템 줄(가진 아이템마다 [{이름} ×{수량}] → `applyGrowthItem`, 없으면 `상점에서 성장 아이템을 살 수 있어요` *(plan 임시)*), 돌보기 버튼(`px-2.5 py-1.5 text-xs`)과 새 🌱 버튼을 44×44px 이상으로, 결과는 `role="status"` 한 줄 (FR-048, SC-004) (T049·T051 뒤)

**Checkpoint**: 농장이 성장 아이템까지 동작, 원장 합계 일치

---

## Phase 9: User Story 7 - 다 키운 동물을 블로그 도감에 모으고 한 마리 전시하기 (Priority: P2)

**Goal**: blog가 도감·전시(FR-051·052)를 구현하고, town은 `user_animals` UNIQUE(T004·T005)·동물 그림·"`grown`은 되돌아가지 않는다" 규칙을 준다

**Independent Test**: 동물 하나를 다 키운 뒤 내 블로그·다른 회원 시점에서 도감 카드, 전시 고르기·바꾸기·비우기, 남의/덜 자란 동물 전시 조작 거부 (blog 검증 문서 `specs/002-blog/contracts/profile-showcase.md`)

**선행**: T005, blog 단계 12 (B-M3 `showcase_animal_id` 복합 FK)

### Implementation for User Story 7

- [ ] T053 [US7] `src/server/farm.ts`·`src/app/farm/actions.ts`에 `grown`에서 다른 상태로 되돌리는 코드가 없는지 확인하고 `src/lib/art/animals.ts`(4종 × 아기·청소년·어른 + 알, 어른 리본)를 blog 도감이 그대로 쓰도록 그림 함수 모양을 바꾸지 않는다 (data-model.md 4.1)
- [ ] T054 [US7] `src/app/farm/farm-view.tsx`의 🏅 다 키운 동물 구역에 `내 블로그 도감에서도 볼 수 있어요` *(plan 임시)* 링크(`/@{내 주소}`)를 단다 (blog 단계 12 뒤) (T052 뒤)

**Checkpoint**: 다 키운 동물이 블로그 도감·전시로 이어짐 (blog 쪽 검증으로 확인)

---

## Phase 10: User Story 8 - 헤더의 로고와 유저 상태창 (Priority: P2)

**Goal**: 모든 화면 헤더에 늘 보이는 로고와 유저 상태창(프로필·닉네임·블로그 제목), 640px 미만 두 줄 헤더 (TOWN-10, FR-053~057)

**Independent Test**: 회원 헤더 구성, 블로그 제목 저장 직후 상태창 반영, 375px에서 제목 숨김·가로 스크롤 없음, 방문자 [시작하기]

### Tests for User Story 8 ⚠️

- [ ] T055 [P] [US8] `e2e/town.mjs`에 헤더 시나리오를 더한다: 회원 헤더 왼쪽 로고·오른쪽 상태창(프로필·닉네임·블로그 제목), 로고 → `/town`, 상태창 → `/@{내 주소}`, 블로그 제목 저장 직후 상태창 갱신(SC-010), 사진 없는 회원은 캐릭터 얼굴, 방문자 [시작하기] → `/`, `Lv.N`·`🪙 N`(→ `/wallet`)·`👑 관리자`(관리자만)·로그아웃 그대로, 누르는 것 모두 44×44px 이상
- [ ] T056 [P] [US8] `e2e/mobile.mjs`에 375px 회원 두 줄 헤더(1줄 로고·`← 나가기`·👑·로그아웃 / 2줄 상태창(프로필 + 닉네임)·`Lv.N`·`🪙 N`·🔔), 블로그 제목 숨김, 로고 보임, 가로 스크롤 없음, 640px 바로 위 폭에서도 가로 스크롤 없음을 더한다

### Implementation for User Story 8

- [ ] T057 [P] [US8] `src/server/dal.ts`(auth 소유)의 `getViewer()` 프로필에 `photoKey`(`profiles.photo_key`, auth R16 뒤) 한 줄을 더한다 — 쿼리 수는 늘리지 않는다
- [ ] T058 [P] [US8] `src/components/user-status.tsx`(새)를 만든다: 둥근 카드, 왼쪽 원형 36px 프로필(`photoKey`가 있으면 `<img src="/files/{photoKey}" alt="">`, 없으면 `CharacterBadge` 장착 캐릭터 얼굴 + shop `outfit`), 오른쪽 위 닉네임(굵게)·아래 블로그 제목(작은 회색, `truncate`), 640px 미만 제목 숨김, 폭 상한, `/@{blogSlug}` 링크·접근 이름 `내 블로그: {블로그 제목}` *(plan 임시)*, 44px 이상 (FR-054·057)
- [ ] T059 [US8] `src/components/site-header.tsx`를 고친다: 로고 늘 보임(회원 → `/town`, 방문자 → `/`), 640px 이상 한 줄 [로고][← 광장으로 나가기] … [`Lv.N`][`🪙 N`][🔔 자리][`👑 관리자`][상태창][로그아웃] 58px, 640px 미만 회원 두 줄(58 + 48 = 106px, 👑는 접근 이름 `관리자`), 방문자 [시작하기], 이동 메뉴 없음, game(🔔·`LevelUpPopup`·`AttendanceDayWatcher`)·auth(`SessionKeeper`)·shop(`CharacterBadge` `outfit`) 줄 자리 유지. 넘치면 두 줄 기준을 768px로 (FR-053~057, R-11) (T057·T058 뒤)
- [ ] T060 [US8] `src/app/globals.css`에 `--header-h`(58px)와 `--header-h-member`(640px 미만 106px, 이상 58px)를 정의하고, `src/app/town/page.tsx` 회원 광장 높이가 이 변수를 쓰는지 확인한다 (T059 뒤)

**Checkpoint**: 모든 화면 헤더에 로고·상태창, 375px 가로 스크롤 없음

---

## Phase 11: User Story 10 - 내 집 지붕 색 고르기 (Priority: P3)

**Goal**: 꾸미기 화면 `🏠 지붕 색`에서 8색 중 하나 또는 [배경 색 따라가기], 모든 광장에서 같은 색 (TOWN-07, FR-039·040)

**Independent Test**: 지붕 색을 고른 뒤 배경을 바꿔도 유지, 다른 회원·방문자 광장에서도 같은 `data-roof`, 목록 밖 색 조작 거부

**선행**: blog `blogs.roof_color` + CHECK `blogs_roof_color_check` (B-M2), `src/lib/blog.ts` `ROOF_COLORS`

### Tests for User Story 10 ⚠️

- [ ] T061 [P] [US10] `e2e/town.mjs`에 지붕 색 시나리오를 더한다: `/closet`에서 색을 고르면 내 광장 `data-mine="true"`의 `data-roof`가 그 색, 배경을 바꿔도 그대로, 나를 즐겨찾기한 회원의 광장에서도 같은 색, [배경 색 따라가기] → 배경 강조색, 목록 밖 색으로 조작한 `setRoofColor` → `고를 수 없는 색이에요` *(plan 임시)*·DB 그대로, 상점·꾸미기에 증축 메뉴 없음

### Implementation for User Story 10

- [ ] T062 [US10] `src/app/town/actions.ts`에 `setRoofColor(color: unknown): Promise<{ ok: true } | { ok: false; error: string }>`를 더한다: 첫 줄 `requireMember()`, zod `z.enum(ROOF_COLORS).nullable()`(NULL 또는 `red`·`orange`·`yellow`·`green`·`sky`·`blue`·`purple`·`brown`) 실패 → `고를 수 없는 색이에요` *(plan 임시)*, `UPDATE blogs SET roof_color = $color WHERE owner_id = 나`(대상은 요청 값이 아님), DB CHECK 위반도 같은 문구, 성공 뒤 `revalidatePath("/closet")`·`revalidatePath("/town")`, 무료·원장 없음 (FR-039·040, contracts/header-roof.md 2.2절) (T037 뒤, 같은 파일)
- [ ] T063 [P] [US10] `src/components/town/roof-color-picker.tsx`(새, 클라이언트, `useTransition`)를 만든다: 작은 1단계 집 미리보기(지금 고른 색), 색 8개(색 동그라미 + 이름, 44×44px 이상, 고른 색 테두리·`aria-pressed`), [배경 색 따라가기](고른 색이 없으면 눌린 상태), 결과 문구 한 줄
- [ ] T064 [US10] `src/components/town/roof-color-section.tsx`(새, 서버)를 만든다: 제목 `🏠 지붕 색`, 내 블로그의 `roof_color`·장착 배경을 읽어 `RoofColorPicker`에 넘김 (T063 뒤)
- [ ] T065 [US10] `src/app/closet/page.tsx`(shop 소유) 꾸미기 내용 아래에 `<RoofColorSection userId={viewer.userId} />` 한 줄을 끼운다 (T064 뒤)

**Checkpoint**: 지붕 색이 모든 광장에서 같게 보임

---

## Phase 12: User Story 11 - 집이 단계별로 자라기 (Priority: P3)

**Goal**: 주인 레벨로 1~3단계 집 그림, 가장 큰 집끼리도 겹치지 않음, 증축 구매 없음 (TOWN-11, FR-058~060)

**Independent Test**: Lv.9·10·29·30 회원의 집이 1·2·2·3단계로 내 광장·남의 광장·방문자 광장에서 같게 보이고 문 앞에서 들어감

### Tests for User Story 11 ⚠️

- [ ] T066 [P] [US11] `scripts/test-town.ts`에 배치 시험을 더한다: `layout()`에 3단계 집 10채 + 3단계 내 집을 넣었을 때 그림 사각형이 서로·다른 건물과 겹치지 않고 입구가 모두 1800 × 1400 안 (FR-060, US11-6)
- [ ] T067 [P] [US11] `e2e/town.mjs`에 집 단계 시나리오를 더한다: `pg`로 Lv.9·10·29·30이 되게 원장을 준비한 회원들의 집 `data-stage`가 1·2·2·3(SC-014), 내 광장·즐겨찾기한 회원 광장·방문자 광장에서 같은 값, Lv.9 → Lv.10 뒤 다시 열면 2단계, 각 단계 집 문 앞 `teleport` + Space → 그 블로그

### Implementation for User Story 11

- [ ] T068 [P] [US11] `src/lib/art/town.ts`의 `HOUSE_STAGES`에 3단계 코드 SVG 그림을 만든다: 1단계 세모 지붕 + 문 하나, 2단계 + 창문·굴뚝·꽃 상자(더 큼), 3단계 + 다락방 창(더 큼), 지붕 색은 인자로 (FR-058, 외부 그림 파일 없음)
- [ ] T069 [US11] `src/components/town/scene.ts`가 `stage`별 그림·충돌 크기·문 앞 입구 위치를 쓰고, 지붕 색·문 옆 주인 캐릭터·이름표를 단계와 상관없이 그리며, `plantTrees()`의 집 자리 피하는 반경(지금 150)을 3단계 크기에 맞춘다 (FR-060) (T068 뒤)

**Checkpoint**: 집 3단계가 모든 광장에서 같게 보이고 겹치지 않음

---

## Phase 13: User Story 9 - 2.5D(아이소메트릭) 광장 (Priority: P2, 공통 맥락 단계 13이라 맨 뒤)

**Goal**: 논리 좌표·물리·입구 판정은 그대로 두고 그리기만 2:1 아이소메트릭(타일 64 × 32)으로 완전히 바꾼다. 2D 코드는 지운다 (TOWN-05, FR-037·038)

**Independent Test**: 바뀐 광장에서 US2·US3 e2e를 다시 돌려 100% 통과(SC-012), 2D를 고르는 설정·화면이 없음

**선행**: US2·US3·US11 (집 3단계 2D 그림), T012 (훅)

### Tests for User Story 9 ⚠️

- [ ] T070 [P] [US9] `scripts/test-town.ts`에 투영 시험을 더한다: `toLogical(toScreen(p)) ≈ p`, 화면 방향 속도 정규화 뒤 대각선 길이 = 직선 길이(230), 투영 사각형으로도 3단계 집 10채 + 내 집 겹침 없음
- [ ] T071 [P] [US9] `e2e/town.mjs`에 아이소메트릭 확인을 더한다: 바닥이 마름모 타일(스크린샷), US2·US3 시나리오 재실행 통과, 입구 90px을 투영 좌표 거리로, 5초 이동 중 `fps()` 표본 평균 55 이상·최저 50 이상(데스크톱 Chromium 참고값, SC-005), 2D 선택 설정 없음

### Implementation for User Story 9

- [ ] T072 [P] [US9] `src/lib/town.ts`에 `toScreen(p)` = (x − y, (x + y) / 2) + 원점, `toLogical(p)` 역변환을 더한다 (논리 칸 32 → 화면 마름모 64 × 32, 투영 세계 약 3200 × 1600, R-16)
- [ ] T073 [P] [US9] `src/lib/art/town.ts`에 아이소메트릭 건물(게시판·상점·농장)·분수·가로등·나무·집 3단계 SVG를 코드로 다시 그린다 (건물 바닥면을 정사각형에 가깝게, 외부 에셋 없음)
- [ ] T074 [US9] `src/components/town/scene.ts`의 그리기를 투영으로 바꾼다: image·이름표·플레이어 그림을 `toScreen(논리 위치)`에, 깊이 = 투영 y, 카메라 bounds·`startFollow`는 투영 세계, 클릭·탭은 world 좌표 → `toLogical`, 키보드·조이스틱은 화면 방향 벡터를 길이 230으로 맞춘 뒤 역투영, 입구 90px은 투영 거리. `layout()`·물리 zone·`setCollideWorldBounds`·입구 목록은 논리 좌표 그대로 (FR-038) (T072·T073 뒤)
- [ ] T075 [US9] `src/components/town/scene.ts`의 바닥(마름모 잔디 체크·길·돌광장, 고정 시드 `"blogville"`·`"blogville-trees"`)을 처음 한 번 그린 뒤 텍스처로 굳히고 휴대폰 최대 텍스처 크기(T001 결과) 안의 조각으로 나눈다 (R-17, SC-005) (T074 뒤)
- [ ] T076 [US9] `src/components/town/scene.ts`·`src/lib/art/town.ts`에서 2D 그리기 코드를 지우고 개발 훅 `entrances()`·`player()`의 `screen` 값이 투영 좌표인지 맞춘다 (FR-037, US9-3) (T075 뒤)

**Checkpoint**: 아이소메트릭 광장에서 모든 이동·입구 시나리오 통과

---

## Phase 14: Polish & Cross-Cutting Concerns

**Purpose**: 문서·비기능 확인·전체 검증

- [ ] T077 [P] `docs/02-erd.md`를 고친다: 3.11 "블로그에 한 마리 전시" UNIQUE ⏳ 해제·성장 아이템 사용 순서(내 키우는 동물 확인 → 수량 −1 → 성장) 한 줄, 5장 집 성장 부분을 "집 단계는 저장하지 않는다: 주인 레벨(Lv.1~9 / 10~29 / 30+)에서 계산 (3.5와 같은 이유, D16)"으로, 7장 6-2 `user_animals` UNIQUE 완료 표시 (data-model.md 7절)
- [ ] T078 [P] `CLAUDE.md`·`README.md`에 광장 규칙(휴대폰도 광장, 아이소메트릭 뒤 "아이소메트릭 전까지 2D" 문구 정리)과 e2e 표에 `e2e/town.mjs`·`e2e/favorites.mjs` 두 줄을 더한다
- [ ] T079 [P] `e2e/nonfunctional.mjs`에서 375px `/town`·`/farm` 가로 스크롤 없음을 확인한다 (SC-004)
- [ ] T080 `npx tsc --noEmit`, `npx eslint`, `npm test`(끝에 `test:town`), quickstart.md 3절 e2e 전체(`auth`·`blog`·`town`·`favorites`·`farm`·`mobile`·`decisions`·`social`·`nonfunctional`)를 돌려 모든 줄 ✅·콘솔 오류 0개를 확인한다
- [ ] T081 quickstart.md 5절 손 확인: 실제 휴대폰 터치로 다섯 건물 1분 안(SC-003), iOS Safari·Android Chrome 원격 개발자 도구 FPS(SC-005), 아이소메트릭 가림(US2-7) 스크린샷, `next build` 결과에 `__blogvilleTown`이 없음
- [ ] T082 구현 뒤 `docs/01-requirements.md`의 TOWN 상태·수용 기준을 갱신하고 변경 이력에 한 줄 더한다 (TOWN-06은 ❌ 그대로), *(plan 임시)* 문구 목록을 spec 보강 안건으로 팀에 올린다 (plan 남은 문제 4)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 바로 시작
- **Foundational (Phase 2)**: Setup 뒤 — 모든 이야기를 막는다. T004 → T005 (마이그레이션, blog B-M3보다 먼저 merge), T006·T008 → T009 → T010, T008 → T011 → T012
- **User Stories (Phase 3~13)**: 모두 Foundational 뒤
  - P1 (US1 → US2 → US3)이 MVP
  - P2 (US4, US5, US6, US7, US8)는 P1과 병렬 가능하나 다른 spec 선행을 기다린다
  - P3 (US10, US11)
  - US9(아이소메트릭)은 P2지만 공통 맥락 단계 13이라 US2·US3·US11 뒤 맨 마지막
- **Polish (Phase 14)**: 원하는 이야기가 끝난 뒤

### User Story Dependencies

- **US1 (P1)**: Foundational 뒤. 환영 쿠키는 auth `signUp`(단계 1)과 같은 PR (없으면 `?welcome=1` 임시)
- **US2 (P1)**: Foundational 뒤. T024~T026은 팀 확인(T023) 뒤
- **US3 (P1)**: Foundational 뒤. T032는 game 단계 6 뒤 (없으면 지금 쿼리)
- **US4 (P2)**: Foundational 뒤 + social `follows.is_favorite`(U4). `src/app/town/page.tsx`는 US1(T016) 뒤에 고친다
- **US5 (P2)**: Foundational 뒤. US4와 독립 (같은 `src/server/town.ts`·`page.tsx`라 순서대로)
- **US6 (P2)**: Foundational 뒤 + shop 성장 아이템·`consumeGrowthItem`(단계 10), game `addLedgerEntry`(단계 6)
- **US7 (P2)**: T005 + blog 단계 12. town 쪽은 T053·T054만
- **US8 (P2)**: Foundational 뒤. `photoKey`는 auth R16 뒤 (없으면 캐릭터 얼굴), 🔔 자리는 game
- **US10 (P3)**: Foundational 뒤 + blog `roof_color`(B2). `src/app/town/actions.ts`는 US4(T037) 뒤
- **US11 (P3)**: Foundational 뒤 (T011의 `stage` 입력). US3의 `houseLabel`(T030)과 같은 `scene.ts`라 순서대로
- **US9 (P2, 단계 13)**: US2·US3·US11 완료 뒤

### Within Each User Story

- 테스트를 먼저 쓰고 실패를 확인한 뒤 구현
- 서버 함수 → Server Action → 화면 컴포넌트 → 페이지 연결
- 같은 파일(`src/components/town/scene.ts`, `src/server/town.ts`, `src/app/town/page.tsx`, `src/app/town/actions.ts`, `src/app/farm/farm-view.tsx`, `e2e/town.mjs`)을 만지는 작업은 [P]여도 다른 이야기와 동시에 고치지 않는다
- 이야기를 마친 뒤 다음 우선순위로

### Parallel Opportunities

- Setup: T002 (T001과 독립)
- Foundational: T006·T007·T008·T013 병렬
- US1: T014·T015·T018 병렬
- US2: T020·T021·T022·T026 병렬
- US4: T033·T034·T038·T041 병렬
- US6: T046·T047·T050 병렬
- US8: T055·T056·T057·T058 병렬
- US10: T061·T063 병렬
- US11: T066·T067·T068 병렬
- US9: T070·T071·T072·T073 병렬
- Polish: T077·T078·T079 병렬
- 팀이 나눠 맡으면 Foundational 뒤 US4(즐겨찾기)·US6(농장)·US8(헤더)을 사람마다 동시에 진행할 수 있다

---

## Parallel Example: User Story 4

```bash
# US4 테스트를 함께 쓴다:
Task: "e2e/favorites.mjs에 회원 즐겨찾기 시나리오"
Task: "e2e/decisions.mjs의 TOWN-04 옛 규칙을 즐겨찾기 규칙으로"

# 서로 다른 파일의 컴포넌트를 함께 만든다:
Task: "src/components/town/favorite-button.tsx (☆/⭐, 44×44px)"
Task: "src/components/town/neighbor-panel.tsx (🏘 패널, 빈 안내)"
```

## Parallel Example: User Story 8

```bash
Task: "src/server/dal.ts getViewer 프로필에 photoKey 한 줄"
Task: "src/components/user-status.tsx 유저 상태창"
Task: "e2e/town.mjs 헤더 시나리오"
Task: "e2e/mobile.mjs 375px 두 줄 헤더"
```

---

## Implementation Strategy

### MVP First (User Story 1~3)

1. Phase 1: Setup
2. Phase 2: Foundational (마이그레이션 T005를 가장 먼저 merge)
3. Phase 3~5: US1 → US2 → US3
4. **STOP and VALIDATE**: `e2e/town.mjs`·`e2e/mobile.mjs`로 광장 진입·이동·입구 확인
5. 배포/데모

### Incremental Delivery

plan.md 묶음 순서를 따른다: **T1(마이그레이션) → 광장 데이터·그림(Foundational, US3, US11) → US4 즐겨찾기 → US5 방문자 → US10 지붕 → US8 헤더 → US1 환영(auth 쿠키와 같은 PR) → US6 농장(shop 뒤) → T032 출석(game 뒤) → US2 휴대폰 광장(팀 확인 뒤) → US9 아이소메트릭(단계 13)**. 각 묶음에 해당 테스트와 문서를 함께 넣는다.

### Parallel Team Strategy

1. 함께 Setup + Foundational
2. 그 뒤:
   - 개발자 A: US1·US2·US3 (광장 화면) → US11 → US9
   - 개발자 B: US4·US5·US10 (즐겨찾기·방문자·지붕, social·blog 선행 확인)
   - 개발자 C: US6·US8 (농장·헤더, shop·game·auth 선행 확인)
3. 이야기마다 독립적으로 merge

---

## Notes

- [P] = 다른 파일, 끝나지 않은 작업에 기대지 않음
- [Story] 라벨로 spec의 User Story와 연결
- 각 이야기는 단독으로 완성·시험할 수 있어야 한다
- 구현 전에 테스트가 실패하는지 확인
- 작업이나 논리 묶음마다 커밋, 마이그레이션 SQL·코드 주석에 요구사항 ID(TOWN-xx)
- 체크포인트마다 멈추고 이야기를 단독으로 확인
- 피할 것: 모호한 작업, 같은 파일 충돌, 이야기 독립성을 깨는 교차 의존
