# Research: 광장 (TOWN)

**Feature**: `007-town` | **Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Date**: 2026-10-07

Technical Context의 모르는 것과 이 기능의 기술 선택마다 결정·근거·대안을 적는다.
근거 표시: **코드 확인**(코드 저장소 `main` `feb4c05`에서 읽음), **라이브러리 동작**(패키지 동작, 이 체크아웃에 `node_modules`가 없어 **구현 전 설치된 문서로 확인**), **추측**(근거가 약함).
plan이 spec에 없는 문구를 임시로 정한 곳은 *(plan 임시)*로 표시하고 [plan.md](plan.md) 남은 문제에 모았다.

## 목록

| # | 주제 | 결정 요약 |
|---|---|---|
| R-01 | 휴대폰에서 광장 대신 간단 메뉴 | 간단 메뉴를 없애고 모든 화면에서 광장(터치는 조이스틱) |
| R-02 | 환영 문구 딱 한 번 | 일회용 쿠키 `bv_welcome`을 가입이 심고, 광장 화면이 처음 그릴 때 지운다 |
| R-03 | 즐겨찾기 저장·10명 상한 | `follows.is_favorite`(social) + `lockUser` 안에서 세기 |
| R-04 | 내 이웃 목록 | 블로그 홈 주인에게만 town 컴포넌트 `MyNeighbors` |
| R-05 | 회원 광장의 집 | 즐겨찾기 블로그만, 최근 공개 글 순, 최대 10 |
| R-06 | 🏘 이웃집 패널 | 회원은 모든 이웃(즐겨찾기 먼저), 방문자는 광장의 집 |
| R-07 | 방문자 인기 블로그 | SQL로 순위 100 → 서버에서 `randomInt`로 10 |
| R-08 | 집 단계 | 저장하지 않고 주인 원장 합계 → 레벨 → 단계 |
| R-09 | 집 그림·자리 | `HOUSE_STAGES` 3단계, 자리 10곳, 겹침 단위 테스트 |
| R-10 | 지붕 색 | `blogs.roof_color`(blog) + town의 색표·저장 Action·꾸미기 칸 |
| R-11 | 헤더 상태창·375px | 640px 미만 회원 헤더는 두 줄 |
| R-12 | 프로필 그림 | `profiles.photo_key`(auth) 있으면 사진, 없으면 캐릭터 얼굴 |
| R-13 | 출석 도장 표시 | game의 `viewer.attendance` 사용, 문구는 TOWN FR-023 (충돌은 남은 문제) |
| R-14 | 성장 아이템 사용 | `applyGrowthItem`: 동물 확인 → shop `consumeGrowthItem` → `addGrowth` |
| R-15 | 농장 문구·원장 | 꽉 참 문구 spec대로, `farm_grown`은 game `addLedgerEntry` |
| R-16 | 아이소메트릭 | 논리 좌표 유지 + 2:1 투영, 타일 64×32 |
| R-17 | 성능(SC-005) | 바닥을 텍스처로 굳힘, `actualFps` 측정 |
| R-18 | 이름표 자르기 | 닉네임만 코드 포인트 12자 + `…` 뒤에 `의 집` |
| R-19 | 캔버스 검증 수단 | 숨은 집 목록(data 속성) + 개발 모드 전용 훅 |
| R-20 | `user_animals` UNIQUE | Drizzle `unique()`, 데이터 이전 없음 |
| R-21 | 상점 부제 | `아바타·가구·배경·성장 아이템` *(plan 임시)* |
| R-22 | 광장 최소 높이와 가로 휴대폰 | 420px 유지 (spec 충돌은 남은 문제) |
| R-23 | 테스트 전략 | `test:town` + `e2e/town.mjs`·`e2e/favorites.mjs` + 기존 e2e 고치기 |
| R-24 | 구현 전 확인할 라이브러리 동작 | Next.js 16 쿠키·캐시, Phaser 4 API |

---

## R-01 휴대폰에서 광장 대신 간단 메뉴

- **Decision**: 휴대폰용 간단 메뉴(`src/components/town/town-menu.tsx`)를 지우고, 모든 화면 크기에서 Phaser 광장(`TownGame`)을 띄운다. `TownGame`의 `PHONE_MEDIA` 검사(휴대폰이면 게임을 끄고 켜는 부분)를 없앤다. 터치 화면 조작은 지금 코드의 가상 조이스틱(`(pointer: coarse)`일 때 `createJoystick()`)과 탭 이동을 그대로 쓴다. `src/app/town/page.tsx`의 `phone:hidden`·`phone:h-auto` 같은 휴대폰 분기를 뺀다. `@custom-variant phone`(`src/app/globals.css`)과 `PHONE_MEDIA`(`src/lib/device.ts`)는 지우지 않고 주석만 고친다 — blog·game plan이 `phone:` 변형을 쓴다고 적었다.
- **Rationale**:
  - spec FR-012(손가락이 주 입력인 화면에 조이스틱), SC-003(375px 휴대폰(터치)에서 모든 건물에 걸어 들어가기), US2 Independent Test(휴대폰 375px에서 조이스틱·탭), FR-007과 Out of Scope(광장 위 바로가기 카드·모바일 메뉴 제외), constitution VI("광장은 마우스와 터치로 조작할 수 있어야 한다").
  - 간단 메뉴는 요구사항 원본 v1.18 어디에도 없다. 코드 주석("10/6 회의 결정", `src/app/town/page.tsx`, `src/components/town/town-menu.tsx`)과 커밋 `39a6e5e`("모바일 간단 버전", PR #41)에만 있다 (**코드 확인**). 원본 NF-17은 "광장을 쓰기 어려운 환경을 위한 일반 링크"를 ❌로 뺐다.
  - 조이스틱·탭 이동은 이미 있다. 터치 태블릿(820×1180)에서 동작을 `e2e/decisions.mjs`가 확인한다 (**코드 확인**). 휴대폰에서 꺼 두었을 뿐이다.
- **Alternatives considered**: 간단 메뉴 유지 — SC-003·FR-007과 충돌. 광장과 메뉴를 함께 — "바로가기 카드 없음"(FR-007)과 충돌. 휴대폰 전용 작은 광장 — 같은 광장(Key Entities "모든 사용자가 같은 배치")과 충돌.
- **영향**: `e2e/mobile.mjs`를 "휴대폰도 광장(캔버스·조이스틱·건물 탭)"으로 다시 쓴다. `e2e/helpers.mjs`의 `loginDev`가 기다리는 `[data-town-menu]:visible`은 사라지지만 같은 `or`의 `canvas`가 먼저 맞으므로 깨지지 않는다 (**코드 확인**). shop plan이 요청한 `town-menu.tsx`의 `outfit` 추가는 필요 없어진다.
- **위험**: 팀의 10/6 회의 결정을 바꾸는 일이라 팀 확인이 필요하다 (남은 문제). 팀이 메뉴를 남기기로 하면 이 변경 단위(plan T7)를 빼고, 대신 `src/components/town/town-menu.tsx`도 다른 단위에 맞춰 고쳐야 한다: 환영 문구를 `welcome` prop(`?welcome=1`) 대신 `bv_welcome` 쿠키로(T6), 상점 부제(T13), shop의 `CharacterBadge` `outfit` 줄, 그리고 `src/app/town/page.tsx`의 `phone:hidden`이 붙은 패널·환영 문구 자리(T5). 이 경우 SC-003(375px 휴대폰에서 걸어 들어가기)은 지킬 수 없으므로 spec도 함께 고쳐야 한다.

## R-02 환영 문구 딱 한 번

- **Decision**: 일회용 표시 쿠키를 쓴다.
  1. auth의 `signUp`(`src/app/(auth)/actions.ts`)이 가입 트랜잭션이 성공한 뒤 쿠키 `bv_welcome=1`(Path `/town`, Max-Age 600초, SameSite=Lax, 배포 환경 Secure, **HttpOnly 아님**)을 심고 `/town`으로 보낸다. 주소의 `?welcome=1`은 없앤다.
  2. `/town` 서버 컴포넌트는 `cookies()`로 `bv_welcome`을 읽어 회원이면 `WelcomeBanner`(클라이언트, 새 파일)에 `initial`로 넘긴다.
  3. `WelcomeBanner`는 서버 렌더와 첫 그리기에서는 아무것도 그리지 않는다. 마운트된 뒤 `initial`이 참이고 **브라우저에 아직 `bv_welcome` 쿠키가 있을 때만** 문구를 열고, 그 자리에서 쿠키를 지운다(`Max-Age=0`, 같은 Path). [✕]는 문구를 닫기만 한다.
- **Rationale**:
  - 새로고침: 서버가 쿠키를 보지 못해 `initial`이 거짓. 뒤로 가기·다른 화면 갔다 오기: Next.js 클라이언트 라우터가 예전 화면 데이터를 다시 써도(`initial` 참) 쿠키가 이미 없어 열리지 않는다. 그래서 FR-004, SC-002(새로고침·뒤로 가기 10회에서 0번)를 지킨다.
  - Server Component는 쿠키를 지울 수 없다 (`src/app/blog/actions.ts`의 `recordBlogVisit` 주석 "쿠키를 새로 만들 수 있어야 해서 서버 컴포넌트가 아니라 Server Action에서", **코드 확인**).
  - Server Action으로 쿠키를 지우면 Next.js가 지금 화면을 서버에서 다시 그린다 (**라이브러리 동작**, 구현 전 확인). 그러면 `TownGame`이 새 `data` 객체를 받아 게임을 다시 만든다(`useEffect` 의존성 `[data, router]`, **코드 확인**) — 캐릭터가 처음 자리로 돌아간다. 브라우저에서 지우면 이 왕복이 없다.
  - DB를 바꾸지 않는다 (공통 맥락 3.3 "환영 문구 한 번은 DB 없이 하는 것을 우선", 표시 여부를 `profiles`에 두면 auth 표 변경 요청이 필요).
  - HttpOnly가 아닌 이유: 값이 `1`뿐인 표시용이라 로그인과 무관하다. constitution 보안 기준의 HttpOnly는 로그인 쿠키에 대한 것이다. 브라우저가 지워야 하므로 자바스크립트로 읽을 수 있어야 한다.
  - React StrictMode(개발 모드)에서 effect가 두 번 돌아도 두 번째에는 쿠키가 없어 상태만 유지된다.
- **Alternatives considered**:
  - `?welcome=1` + `router.replace("/town")`(원본 TOWN-01 열린 질문의 제안) — 새로고침·뒤로 가기는 막지만, 같은 주소(`/town?welcome=1`)를 다시 열면 또 보인다. "가입한 회원마다 정확히 한 번"의 서버 근거가 없다.
  - `profiles.welcomed_at` — auth 표 변경 요청, 가입 직후 한 번 보이는 문구에 DB 쓰기가 필요.
  - `localStorage`만 — 가입했다는 신호를 서버가 줄 수 없다.
- **확인할 것**: 가입 Action의 `redirect` 뒤 첫 `/town` 렌더가 방금 심은 쿠키를 읽는지는 구현 전에 확인한다 (R-24). 읽지 못하면 2번의 `initial`을 빼고 3번(브라우저 쿠키 확인)만으로 연다 — 그래도 새로고침·뒤로 가기에서 다시 보이지 않는다.
- **한계**: 가입한 브라우저 기준이다. 가입 뒤 10분 안에 광장이 열리지 않으면 보이지 않는다(가입은 곧바로 `/town`으로 보내므로 실제로는 생기지 않는다).
- **의존성**: 쿠키를 심는 일은 auth 소유 파일이다. auth research는 "town이 다른 신호(예: 일회용 쿠키)를 고르면 `signUp`이 그 신호를 심는다"고 적었다 (`specs/001-auth/research.md`).
- 문구 끝 문장(농장 안내)은 spec이 "팀이 확정"으로 남겼다. 지금 코드의 `왼쪽 **동물 농장**에서 첫 알을 받아 동물을 키워 보세요.`를 임시로 쓴다 (**코드 확인**, 남은 문제).

## R-03 즐겨찾기 저장과 10명 상한

- **Decision**: social이 추가하는 `follows.is_favorite boolean NOT NULL DEFAULT false`(`specs/004-social/data-model.md` U4)를 쓴다. town의 Server Action `toggleFavorite(followeeId)`(`src/app/town/actions.ts`, 새 파일)가 `db.transaction` 안에서 `lockUser(tx, 나)` → 이웃 행(`follower_id = 나`, `followee_id = 대상`) 확인 → 켜는 경우 `COUNT(*) WHERE follower_id = 나 AND is_favorite` < 10 확인 → `UPDATE follows SET is_favorite`. 끄는 경우는 세지 않는다.
- **Rationale**:
  - 원본 TOWN-08 구현 방식 제안("`lockUser` 트랜잭션 안에서 개수를 세고 10개 미만일 때만 바꾼다"), social 계약 `contracts/follows-feed.md` 4절("최대 10명은 town이 `lockUser(나)` 안에서 세어 지킨다").
  - 같은 회원의 즐겨찾기 요청은 모두 같은 advisory lock(`pg_advisory_xact_lock(hashtext(userId))`, `src/server/points.ts`, **코드 확인**)을 거쳐 차례로 처리된다 → 9명일 때 두 화면에서 동시에 눌러도 10명을 넘지 않는다 (FR-033, US4-5, SC-006).
  - 이웃 취소는 행 삭제라 즐겨찾기도 함께 사라진다 (FR-035, social `toggleFollow`). 회원 탈퇴는 `follows` CASCADE (**코드 확인**).
  - social의 `toggleFollow`는 잠금 없이 행을 지운다. 지우는 쪽은 개수를 줄이기만 하므로 상한을 넘기지 않는다. 확인과 갱신 사이에 행이 사라지면 `UPDATE`가 0행 → "이웃 아님"으로 답한다.
- **DB 제약으로 막지 않는 이유**: "회원마다 행 10개까지"는 CHECK·UNIQUE로 표현할 수 없다. ERD 3.11의 "한 번에 5마리"와 같은 방식이다. constitution V와의 차이는 [plan.md](plan.md) Complexity Tracking에 적었다.
- **Alternatives considered**: 별도 `favorites` 표 — 이웃 취소 때 함께 지울 연결이 하나 더 생긴다(즐겨찾기는 이웃 관계의 속성, ERD 3.17). 트리거로 개수 막기 — 이 저장소에 트리거 선례가 없고 규칙이 DB 안에 숨는다.
- **거부 문구**: 상한 `즐겨찾기할 이웃은 최대 10명이에요`(spec). 이웃이 아닌 대상(조작 요청, FR-034)은 spec에 문구가 없어 `이웃으로 추가한 블로그만 즐겨찾기할 수 있어요` *(plan 임시)*.

## R-04 내 이웃 목록 (블로그 홈)

- **Decision**: blog 소유 `src/app/blog/[slug]/page.tsx`의 왼쪽 `<aside>`(카테고리 카드 아래)에 주인일 때만 town의 서버 컴포넌트 `MyNeighbors`(`src/components/town/my-neighbors.tsx`, 새 파일)를 한 줄로 끼운다 (공통 모듈 추가). `MyNeighbors`는 `listMyNeighbors(userId)`(`src/server/town.ts`)를 부르고, 줄마다 캐릭터 얼굴·블로그 이름·닉네임(그 블로그 링크)과 ☆/⭐ 버튼(`FavoriteButton`, 클라이언트, 44×44px 이상)을 그린다. 제목 줄에 `⭐ N / 10` *(plan 임시)*. 이웃 0명이면 FR-034 문구.
- 버튼은 `useTransition`으로 `toggleFavorite`를 부르고 **서버 결과**로 ☆/⭐를 바꾼다(거부될 수 있으므로 미리 바꾸지 않는다 — social의 이웃 버튼과 같은 방식). 거부되면 목록 위에 문구 한 줄(`role="status"`).
- 정렬: 즐겨찾기 먼저 → 최근 공개 글 최신순(없으면 뒤) → 이웃 추가 최신순.
- **Rationale**: FR-032(주인에게만), US4-2(다른 사람에게는 서버가 아예 그리지 않음 — 화면 숨김이 아니라 데이터를 보내지 않는다), 공통 맥락 5.2("town: 주인에게만 보이는 '내 이웃 목록' 컴포넌트 끼우기"). 블로그 홈은 이미 주인 여부(`isOwner`)를 계산한다 (**코드 확인**).
- **Alternatives considered**: 광장 패널에서 ☆ — spec은 블로그 홈. 꾸미기 화면 — spec과 다름.

## R-05 회원 광장의 집

- **Decision**: `getTownHouses`를 바꿔 회원이면 즐겨찾기한 블로그만 돌려준다: `follows JOIN blogs ON blogs.owner_id = follows.followee_id … WHERE follows.follower_id = 나 AND follows.is_favorite`, 정렬 `최근 공개 글 DESC NULLS LAST, blogs.created_at DESC, blogs.id DESC`, `LIMIT 10`. 공개 글이 없는 즐겨찾기 블로그도 포함한다. 내 집은 지금처럼 `getMyHouse`.
- **Rationale**: FR-025(최근 공개 글 순, 글 없어도 기본 집, 최대 10), FR-026(비공개 글 제외 — 지금 하위 쿼리가 이미 `visibility = 'public'`, **코드 확인**), FR-027(내 블로그는 내 집 자리). 즐겨찾기는 자기 자신일 수 없다(`follows_not_self_check`). `LIMIT 10`은 자리 수에 맞춘 것이고 상한은 R-03이 지킨다.
- **Alternatives considered**: 이웃 전부를 최근 글 순으로 10개 — TOWN-08 결정과 다름.

## R-06 🏘 이웃집 패널

- **Decision**: 패널은 페이지에서 떼어 `NeighborPanel`(`src/components/town/neighbor-panel.tsx`)로 만든다.
  - 회원: 내 모든 이웃 링크(`listMyNeighbors`, ⭐ 표시, 정렬은 R-04와 같음). 즐겨찾기가 0명이면 목록 위에 `마음에 드는 블로그를 즐겨찾기하면 광장에 집이 생겨요`(FR-028). 이웃이 0명이면 그 문구만.
  - 방문자: 광장에 놓인 집 10곳(R-07)의 링크.
  - `<details open>`(처음 펼침), 목록 `max-h-[40dvh]` 스크롤 (지금과 같음, **코드 확인**).
- **Rationale**: FR-030("광장에 집이 없는 블로그도 목록의 링크로", 유일한 일반 링크 목록), US4-7(즐겨찾기하지 않은 이웃은 패널에서). 지금 패널은 광장의 집과 같은 목록이라, 즐겨찾기 방식에서는 즐겨찾기하지 않은 이웃에게 갈 길이 없어진다.
- **Alternatives considered**: 패널 = 광장의 집만 — US4-7 불가. 마을 전체 블로그 — 범위 밖, 목록이 끝없이 길어짐.

## R-07 방문자 인기 블로그 100곳 중 무작위 10곳

- **Decision**: 세 단계로 나눈다 (`getVisitorHouses()`, `src/server/town.ts`).
  1. 후보 쿼리: 공개 글이 있는 블로그를 `이웃 수(그 블로그 주인을 followee로 가진 follows 행 수) DESC, 최근 공개 글 DESC, blogs.id ASC`로 정렬해 `id` 100개.
  2. 서버 자바스크립트에서 `node:crypto`의 `randomInt`로 겹치지 않게 10개를 고른다. 고르는 함수 `sampleDistinct(list, n, randInt)`는 `src/lib/town.ts`의 순수 함수로 두고 난수 함수를 인자로 받는다(단위 테스트).
  3. 고른 `id`로 집 정보(캐릭터·지붕·집 단계)를 읽는다(R-08과 같은 쿼리).
  `/town`은 쿠키를 읽어 요청마다 렌더링되므로 열 때마다 새로 뽑는다 (**라이브러리 동작**, 구현 전 확인).
- **Rationale**: FR-029·D14. 마지막 정렬 기준 `id`로 100위 경계가 흔들리지 않는다(SC-013 시험이 재현됨). "100곳보다 적으면 전부에서", "10곳보다 적으면 있는 만큼, 남는 자리는 빈칸"(US5-4)은 `sampleDistinct`가 `min(n, list.length)`개를 돌려주는 것으로 끝난다. 농장 부화도 `randomInt`를 쓴다 (`src/app/farm/actions.ts`, **코드 확인**).
- **비용**: 이웃 수는 `follows_followee_idx`, 최근 공개 글은 `posts_blog_created_idx`로 블로그마다 찾는다 (**코드 확인**). 블로그 전체를 한 번 훑는다. 이 프로젝트 규모(블로그 수백 이하, **추측**)에서는 충분하다. 커지면 이웃 수를 캐시해야 하지만 지금 범위 밖이다.
- 관리자 블로그(`/@notice`)도 공개 글이 있으면 후보다 (spec에 예외 없음).
- **Alternatives considered**: 한 쿼리 안에서 `ORDER BY random() LIMIT 10` — 가능하지만 고르기 규칙을 DB 없이 시험할 수 없다. 최근 30일 공감 수·조회수(원본 후보) — D14로 정해짐.

## R-08 집 단계는 저장하지 않고 계산한다

- **Decision**: 집을 읽는 쿼리에 주인의 누적 경험치 `(SELECT COALESCE(SUM(exp_delta), 0) FROM point_ledger WHERE user_id = blogs.owner_id)`를 더하고, 서버에서 `levelFromExp`(`src/lib/game.ts`) → `houseStage(level)`(`src/lib/town.ts`: Lv.1~9 → 1, Lv.10~29 → 2, Lv.30 이상 → 3)로 바꿔 `TownHouse.stage`로 넘긴다. 내 집·즐겨찾기 집·방문자 집이 모두 같은 함수다.
- **Rationale**: D16, FR-059, SC-014. constitution V·ERD 3.5 "레벨은 저장하지 않고 원장 합계" — 단계는 레벨의 함수라 저장하면 같은 정보를 두 번 둔다. 보상을 회수하지 않으므로(D6) 레벨·단계는 내려가지 않는다. 광장을 열 때 한 번 계산한다(Edge Cases).
- **비용**: 한 화면에 집이 최대 11채 → 회원 11명의 원장 합계. `point_ledger_user_reason_created_idx`의 앞 열이 `user_id`라 회원별로 찾는다 (**코드 확인**).
- **Alternatives considered**: `blogs.house_stage` 컬럼 — 경험치가 생기는 모든 경로에서 갱신해야 해 어긋날 위험. 공개 글 수·코인 증축 — D16에서 버림.

## R-09 집 그림 3단계와 자리 10곳

- **Decision**:
  - `HOUSE_STAGES`(`src/lib/art/town.ts`)에 2·3단계를 더한다. 1단계: 세모 지붕 + 문 하나(지금 그림에서 창문·꽃 상자·굴뚝을 뺀다). 2단계: 1단계 + 창문, 굴뚝·꽃 상자, 1단계보다 큼. 3단계: 2단계 + 지붕 다락방 창, 2단계보다 큼. 크기 제안 150×140 / 180×170 / 210×200 (구현 때 스크린샷으로 조정).
  - 충돌 상자·입구·이름표 위치는 단계 크기에서 계산한다 (지금 `addHouse`가 이미 `HOUSE_STAGES[stage]`에서 크기를 읽고, 텍스처 키 `house:{stage}:{roof}`도 단계를 받는다 — **코드 확인**).
  - 이웃집 자리 `NEIGHBOR_SLOTS` 8 → 10(위 줄 5, 아래 줄 5). 간격은 3단계 집 폭 기준.
  - 나무를 심지 않는 반경도 키운다. 지금 `plantTrees()`(`scene.ts`)는 내 집·이웃집 자리마다 반경 150 안을 피하는데(**코드 확인**), 3단계 집(약 210 × 200)과 문 옆 주인 캐릭터를 덮지 못해 나무(부딪히는 물체)가 집이나 문 앞을 막을 수 있다. 나무 배치는 `layout()` 밖이라 아래 단위 테스트가 잡지 못하므로, 반경을 3단계 크기에서 계산하고 스크린샷으로 확인한다. 고정 시드라 바뀐 뒤에도 방문마다 같다 (FR-018).
  - `layout()`을 내보내고, 단위 테스트가 "3단계 집 10채 + 3단계 내 집"을 놓았을 때 모든 그림 사각형이 서로 겹치지 않는지 확인한다 (US11-6, FR-060).
- **Rationale**: FR-058·060, spec 기본값(굴뚝·꽃 상자는 2단계부터, 3단계 다음은 없음), 원본 TOWN-11 구현 제안("충돌 상자, 문 위치, 이름표 위치도 단계 크기에 맞춘다. 자리 간격도 3단계 크기에서 겹치지 않게").
- 아이소메트릭(R-16) 때 세 단계를 다시 그린다 (spec 기본값 TOWN-05).

## R-10 지붕 색

- **Decision**:
  - 저장: blog가 추가하는 `blogs.roof_color text NULL` + CHECK(8개 코드값 `red orange yellow green sky blue purple brown`, NULL = 배경 색 따라가기) — `specs/002-blog/research.md` R-22. 코드값 목록 `ROOF_COLORS`는 blog의 `src/lib/blog.ts`.
  - 색: town의 `src/lib/town.ts`에 코드값 → 색·이름 표 `ROOF_PALETTE`와 `houseRoof(roofColor, backgroundAsset)`(고른 색이 있으면 그 색, 없으면 `backgroundAccent(backgroundAsset)`). 색 제안 (구현 때 조정): 빨강 `#e5484d`, 주황 `#f76b15`, 노랑 `#f5c400`, 초록 `#2f9e44`, 하늘 `#4cb4e7`, 파랑 `#2563eb`, 보라 `#8e4ec6`, 갈색 `#8d5b3a`.
  - 서버가 색을 정해 `TownHouse.roof`(색 문자열)로 넘긴다 → 내 광장·남의 광장·방문자 광장이 같은 색 (FR-040, US10-3).
  - 저장 Action `setRoofColor(color)`(`src/app/town/actions.ts`): `requireMember()` → zod로 8개 코드값 또는 `null`만 통과 → `UPDATE blogs SET roof_color WHERE owner_id = 나`. 목록 밖 값은 앱이 먼저, DB CHECK가 다시 막는다 (FR-039, US10-5, constitution IV·V).
  - 고르는 화면: shop 소유 `src/app/closet/page.tsx`에 town의 서버 컴포넌트 `RoofColorSection`(`src/components/town/roof-color-section.tsx`)을 한 줄 끼운다 (공통 맥락 5.2 "town: 꾸미기 화면에 `🏠 지붕 색` 칸 추가", shop `contracts/closet.md`도 자리를 비워 둠).
  - 무료, 원장 기록 없음 (spec 기본값).
- **Rationale**: FR-039·040, US10. 배경을 바꿔도 `roof_color`는 그대로라 US10-2가 성립한다.
- **Alternatives considered**: 색(hex) 저장 — 목록 밖 거부를 CHECK로 지키기 어렵다(blog R-22). town 소유 새 표 — 집은 블로그마다 하나라 `blogs` 1:1, 별도 표가 필요 없다.
- **거부 문구**: spec에 없음 → `고를 수 없는 색이에요` *(plan 임시)*.

## R-11 헤더 유저 상태창과 375px

- **Decision**: `src/components/site-header.tsx`를 다시 짠다.

  | 화면 | 구성 (왼쪽 … 오른쪽) | 높이 |
  |---|---|---|
  | 640px 이상, 회원 | [로고] [← 광장으로 나가기] … [`Lv.N`] [`🪙 N`] [🔔(game)] [`👑 관리자`] [상태창] [로그아웃] | 58px (지금과 같음) |
  | 640px 미만, 회원 | 1줄: [로고] [← 나가기] … [👑] [로그아웃] / 2줄: [상태창(프로필+닉네임)] … [`Lv.N`] [`🪙 N`] [🔔] | 58 + 48 = 106px |
  | 방문자 | [로고] [← 나가기] … [시작하기] | 58px |

  - 로고는 늘 보인다. 지금 `HomeLogo`가 좁은 화면에서 나가기 버튼이 보이면 로고를 숨기는 처리(이슈 #5, **코드 확인**)를 없앤다 (FR-053 "모든 화면의 헤더 왼쪽에 로고").
  - 상태창은 새 컴포넌트 `UserStatus`(`src/components/user-status.tsx`): 내 블로그 홈 링크, 왼쪽 원형 36px 프로필(R-12), 오른쪽 위 닉네임(굵게), 아래 블로그 제목(작은 회색, `truncate`), 640px 미만에서는 제목을 숨김. 누르는 영역 44px 이상.
  - 광장 높이: 방문자는 지금처럼 `100dvh − var(--header-h)`, 회원은 `100dvh − var(--header-h-member)`. `--header-h-member`는 640px 이상 58px, 미만 106px (`src/app/globals.css`).
  - 640px 미만의 `👑 관리자`는 아이콘만 보이고 접근 이름 `관리자` *(plan 임시)*.
- **Rationale**: 375px(좌우 여백을 빼면 약 351px)에 넣어야 하는 것은 로고(FR-053), 나가기(FR-008), `Lv.N`(FR-055, game FR-010), `🪙 N`, 🔔(game FR-042), 👑(관리자), 상태창(프로필+닉네임, FR-057), 로그아웃이고, 버튼은 44×44px 이상이어야 한다(SC-004). 대략 400px가 넘어 한 줄이면 가로 스크롤이나 44px 미만이 생긴다 (**추측**: 글자 폭 어림, 구현 때 375px 스크린샷으로 확인). 두 줄로 나누면 두 조건을 함께 지킨다.
- **640px 바로 위 폭도 확인**: 한 줄 배치(640px 이상)도 로고·`← 광장으로 나가기`·`Lv.N`·`🪙 N`·🔔·`👑 관리자`·상태창·로그아웃을 모두 담아야 해 640~767px 근처에서 넘칠 수 있다 (**추측**). 상태창에 폭 상한을 두고 닉네임·블로그 제목을 `truncate`로 줄이며, 640px·768px 스크린샷에서도 가로 스크롤이 생기면 두 줄로 바뀌는 기준을 `md`(768px)로 올린다 (그때 `--header-h-member` 기준도 같이). constitution VI("375px와 PC에서 가로 스크롤 없이")는 중간 폭에도 적용된다.
- **Lv·🪙 위치**: 상태창 밖에 따로 둔다 (TOWN 기본값). game FR-010의 "헤더(유저 상태창, TOWN-10)에 `Lv.N`"과 표현이 다르지만 헤더 소유는 town이고(공통 맥락 5.3), game plan도 "상태창 개편 때 `Lv.N`·`🪙 N` 유지"로 받았다 (`specs/005-game/plan.md`).
- **Alternatives considered**: 로그아웃·`Lv`를 아이콘만 — 그래도 넘치고 로그아웃 버튼은 auth 소유. 햄버거 메뉴 — Out of Scope(2026-10-02 삭제 결정).

## R-12 상태창 프로필 그림

- **Decision**: auth가 1단계에서 더하는 `profiles.photo_key`(`specs/001-auth/research.md` R16)를 `getViewer()`의 프로필에 `photoKey`로 더한다 (auth 소유 `src/server/dal.ts`에 필드 하나 — 공통 모듈 추가). 값이 있으면 `<img src="/files/{photoKey}">` 원형, 없으면 `CharacterBadge`(장착 캐릭터, shop이 더하는 `outfit` 포함).
- **Rationale**: FR-054. post 계약은 "지금 프로필 사진인 첨부는 누구나 내려받기"로 정했다 (`specs/003-post/contracts/attachments-http.md`). 사진을 올리는 화면은 어느 spec도 맡지 않았다 (공통 맥락 5.3) → 지금은 늘 캐릭터 얼굴이 보인다 (남은 문제, spec 사이 공통).
- **FR-056·SC-010 (저장 직후 반영)**: 닉네임(blog `updateNickname`), 블로그 제목(blog `updateBlogInfo` — 지금도 `revalidatePath("/", "layout")`, **코드 확인**), 캐릭터(shop `equipItem` — 지금도, **코드 확인**), 사진(auth)이 모두 레이아웃을 다시 그리게 하므로 헤더가 새 값을 그린다. 헤더는 `getViewer()` 한 쿼리에서 닉네임·블로그 이름·주소를 이미 읽는다(`blogTitle`, `blogSlug`, **코드 확인**) → 상태창 때문에 쿼리가 늘지 않는다.

## R-13 출석 도장 표시

- **Decision**: 문구는 TOWN FR-023 그대로 둔다 (게시판 부제 `마을 소식 · 출석 체크 (오늘 완료 ✅)` / `(보상 받기 🎁)`, 출석 도장 안내 `출석 체크 (오늘 완료)`). 오늘 출석 여부는 game이 `getViewer()`에 더하는 `viewer.attendance`(오늘 날짜·일차)에서 읽고 `hasAttendedToday()` 쿼리를 없앤다 (game 요청, `specs/005-game/contracts/attendance.md` 3장). game 작업 전에는 지금 쿼리를 그대로 쓴다. `TownData`에 `attendanceDay: number | null`도 함께 넣어 문구를 GAME FR-028(`오늘 N일차 ✅`) 쪽으로 바꾸기로 하면 `scene.ts` 문자열만 고치면 되게 한다.
- **충돌**: GAME FR-028 "입구에 `오늘 N일차 ✅`" vs TOWN FR-023. 팀 결정이 필요하다 (남은 문제). game plan도 같은 항목을 남은 문제로 올렸다.
- **참고**: 자동 출석에서는 회원이 광장을 그리는 순간 이미 출석되어 있어 `(보상 받기 🎁)`는 실제로 거의 보이지 않는다 (`specs/README.md` "원본 문서에서 고칠 곳").

## R-14 성장 아이템 사용

- **Decision**: Server Action `applyGrowthItem(animalId, itemId)`(`src/app/farm/actions.ts`):
  1. `requireMember()` → 두 인자 모두 `parseId()` (실패하면 `잘못된 요청이에요`, 지금 농장 Action과 같음).
  2. `db.transaction` → `lockUser(tx, 나)`.
  3. 동물 확인: `user_animals`에서 `id = animalId AND user_id = 나 AND status = 'growing'`. 없으면 `돌볼 수 있는 동물이 아니에요` (알·남의 동물·다 자란 동물 모두, FR-046 문구를 함께 씀).
  4. 차감: shop의 `consumeGrowthItem(tx, 나, itemId)`(`src/server/inventory.ts`, `specs/006-shop/contracts/shared-modules.md`) — 성장 아이템이고 수량 > 0이면 1 줄이고 `{ name, growthValue }`, 아니면 `null` → `가지고 있는 성장 아이템이 없어요` *(plan 임시)*.
  5. 성장: `addGrowth(tx, 나, growthValue, [animalId])`. 다 자라면 지금 돌보기와 같은 `🎉 {이름}가 다 자랐어요! 경험치 +X, 🪙 +Y`, 아니면 `🌱 {아이템 이름} 사용! 성장 +{N}` *(plan 임시)*.
  6. `revalidatePath("/", "layout")` (다 자라면 헤더 코인·레벨, 레벨업 팝업).
- **Rationale**:
  - FR-048(아이템마다 정해진 성장치, 하나 줄고, 하루 여러 번, 가진 만큼만), US6-8·9, Edge Cases(알에는 못 씀, 넘친 성장치는 버림).
  - 확인 → 차감 → 성장 순서라 알이나 남의 동물에게는 아이템이 줄지 않는다. 모두 한 트랜잭션이라 중간에 실패하면 전부 취소된다 (FR-050, constitution V).
  - 넘친 성장치: `addGrowth`가 다 자라면 `growth = grow_exp`로 맞춰 나머지를 버린다. 대상이 `[animalId]` 하나라 다른 동물로 넘어가지 않는다 (**코드 확인**).
  - 경험치 없음(spec 기본값 "성장 아이템 사용에는 경험치를 주지 않는다"), 사용 기록 표 없음(ERD 3.11), 원장 기록 없음(구매가 shop의 `purchase`로 남음).
  - 수량이 1개일 때 두 탭에서 동시에 쓰면 회원 잠금으로 차례로 처리되고 두 번째는 수량 0이라 거부된다.
- **이름**: `use…`로 시작하면 eslint React 훅 규칙(`eslint-config-next`의 `react-hooks`)이 함수 호출을 훅으로 볼 수 있어(**라이브러리 동작**) `applyGrowthItem`으로 한다.
- **화면**: 키우는 동물(`growing`) 카드마다 `🌱` 줄에 내가 가진 성장 아이템(수량 1 이상) 버튼 `{이름} ×{N}`(44px 이상). 하나도 없으면 `상점에서 성장 아이템을 살 수 있어요` *(plan 임시)*. 알 카드에는 없다. 데이터는 `getFarm`이 `items`(`type = 'growth'`)와 `user_items`(`quantity > 0`)를 함께 읽는다 (town 영역 파일의 새 쿼리, 두 표는 참조).
- **Alternatives considered**: shop이 사용 Action까지 — shop plan이 "사용 동작은 town, shop은 수량 줄이기 도우미만"으로 나눴다. 동물 확인 없이 차감부터 — 알에 쓰면 아이템만 사라진다.

## R-15 농장 문구와 원장

- **Decision**:
  - `src/app/farm/actions.ts`의 `FULL`을 `자리가 꽉 찼어요. 다 키운 뒤에 받을 수 있어요`로 바꾼다 (FR-043). 화면(`farm-view.tsx`)은 이미 이 문구다 (**코드 확인**) → 화면과 서버 문구가 같아진다.
  - 농장 안내 문장(`src/app/farm/page.tsx`)에 "상점의 성장 아이템으로도 자라요"를 더한다 *(plan 임시)*.
  - `addGrowth`의 `farm_grown` 원장 INSERT를 game의 `addLedgerEntry(tx, entry)`로 바꾼다 (game 요청, `specs/005-game/research.md` — 다 키움으로 레벨이 오르면 레벨업 알림). game이 먼저 끝나면 game이 한 줄을 바꿔도 된다고 town이 동의한다.
  - "다 키운 동물" 칸의 주석 "도감은 2차에서 블로그로"를 지우고, blog 단계 12 뒤에 `내 블로그 도감에서도 볼 수 있어요` 링크 *(plan 임시)*.
- **Rationale**: FR-043·050, SC-008(원장 합계와 100% 일치 — 경로가 바뀌어도 원장 한 줄은 같다).

## R-16 2.5D(아이소메트릭) 광장

- **Decision**: **논리 좌표는 그대로 두고 그리기만 투영한다.**
  - 논리 세계: 지금의 1800 × 1400 탑다운 좌표 (FR-014 크기 그대로). `layout()`, 물리(발밑 zone, 벽 zone, `setCollideWorldBounds`), 입구 목록은 논리 좌표.
  - 투영: P(x, y) = (x − y, (x + y) / 2) + 원점 이동 (2:1). 논리 칸 32 → 화면 마름모 **64 × 32 (타일 크기 결정)**. 투영 세계는 약 3200 × 1600 → 카메라 bounds.
  - 그림: 건물·집·분수·가로등·나무 image, 이름표, 플레이어 그림을 P(논리 위치)에 놓는다. 카메라는 플레이어 그림을 따라간다.
  - 깊이: 투영 y(= (x + y) / 2) — 물체는 바닥면 앞쪽 가장자리, 캐릭터는 발밑. 지금 y-sorting(깊이 = 아랫변 y)을 투영 y로 바꾼 것이다.
  - 입력: 클릭·탭은 카메라 world 좌표 → 역투영 → 논리 목표. 키보드·조이스틱은 **화면 방향**(↑ = 화면 위)의 벡터를 화면에서 길이 230으로 맞춘 뒤 역투영해 논리 속도로 쓴다 → 보이는 속도가 초당 230px이고 대각선도 같다(FR-010). 입구 90px은 투영 좌표 사이 거리로 잰다(보이는 거리, FR-020).
  - 바닥: 마름모 타일(잔디 체크), 길, 돌광장을 같은 고정 시드(`"blogville"`, `"blogville-trees"`)로 그린다 → 방문할 때마다 같다(FR-018).
  - 그림 파일: `src/lib/art/town.ts`에 아이소메트릭 SVG를 코드로 다시 그린다(집 3단계 포함). 외부 에셋 없음(spec 기본값, NF-25). 2D 그리기 코드는 지운다(FR-037, US9-3 — 설정으로 고르는 기능 없음).
- **Rationale**: 원본 TOWN-05 구현 제안("이동·충돌·입구 판정 로직(`update`, `layout`)은 그대로 두고, 바닥·건물 그리기와 좌표 변환만 바꾼다"). 지금 코드가 이미 **물리 zone과 보이는 image를 따로** 둔다 — 벽은 `placeStructure`가 image와 별도 zone을, 플레이어는 발밑 zone과 그림 image를 따로 두고 매 프레임 그림을 옮긴다 (**코드 확인**). 그래서 image 위치 계산만 바꾸면 FR-038(이동·충돌·가림·들어가기가 같게)의 회귀 위험이 가장 작다. Arcade 물리는 축 정렬 사각형만 다루므로(**라이브러리 동작**) 물리까지 투영 좌표로 옮기면 마름모 충돌이 필요해진다.
- **Alternatives considered**: Phaser 아이소메트릭 타일맵 — 타일 데이터·타일셋 텍스처가 필요하고 Phaser 4의 해당 API를 이 체크아웃에서 확인할 수 없다. 바닥은 장식뿐이라 Graphics로 충분하다. 타일 128 × 64 — 길·광장 모양이 거칠어진다.
- **위험**: 큰 건물 옆에 선 캐릭터의 가림 판정이 어긋날 수 있다(바닥면이 넓은 물체의 깊이를 한 점으로 정하는 한계). 건물 바닥면을 정사각형에 가깝게 그리고 US2-7을 스크린샷으로 확인한다.
- **순서**: 공통 맥락 5.1 단계 13 (단계 11 뒤). 집 3단계 2D 그림(R-09)을 먼저 만들고, 이 단계에서 아이소메트릭으로 다시 그린다.

## R-17 성능 (SC-005, NF-20)

- **Decision**:
  1. 바닥(잔디 칸, 풀 포기 420개, 들꽃 160개, 길 돌 수백 개)을 Graphics 객체로 남겨 두지 않고 처음 한 번 그린 뒤 텍스처로 굳힌다. 지금 `drawGround()`는 Graphics 객체를 그대로 둔다 (**코드 확인**). WebGL에서 Graphics는 매 프레임 도형을 다시 그린다 (**라이브러리 동작**: Phaser 3 기준, Phaser 4에서 구현 전 확인). 투영 세계가 크므로 휴대폰 최대 텍스처 크기(흔히 4096, **추측**) 안의 조각으로 나눈다.
  2. 정적 물체는 image 하나씩, 충돌은 정적 몸체(지금과 같음).
  3. 측정: 개발 모드 훅(R-19)으로 Phaser `game.loop.actualFps`를 읽어 `e2e/town.mjs`가 5초 동안 이동하며 표본을 모은다 → 평균 55 이상, 최저 50 이상(데스크톱 Chromium 참고값). 실제 휴대폰(iOS Safari, Android Chrome)은 원격 개발자 도구 성능 탭으로 손으로 잰다 (원본 NF-20 검증 방법).
- **Rationale**: SC-005(약 60fps, 최소 50fps), 아이소메트릭으로 바닥 도형이 늘어난다.

## R-18 집 이름표 자르기

- **Decision**: `houseLabel(nickname)`(`src/lib/town.ts`): 닉네임이 코드 포인트 12자를 넘으면 앞 12자 + `…`, 그 뒤에 `의 집`. 내 집은 `내 집`. 부제(블로그 이름)는 지금처럼 두되, 3단계 집 폭을 넘는지 스크린샷으로 확인한다.
- **Rationale**: FR-024·Edge Cases("닉네임이 12자를 넘으면 `{닉네임}의 집`은 `…`로"). 지금 코드는 `{닉네임}의 집` **전체**를 12자로 자른다 — 닉네임 10자만 돼도 `의 집`이 잘린다 (`placeStructure`, **코드 확인**). 닉네임이 2~20자로 늘어나므로(auth) 꼭 고쳐야 한다. 코드 포인트로 세는 것은 blog의 닉네임 길이 규칙과 같다(이모지).

## R-19 캔버스 안을 검증하는 수단

- **Decision**:
  - **숨은 집 목록**: `/town`에 `<ul hidden data-town-houses>`를 두고 집마다 `<li data-slug data-stage data-roof data-mine>`. 서버가 게임에 넘긴 `TownData`와 같은 값이다. e2e가 집 단계(SC-014)·지붕 색(US10)·즐겨찾기 집 10채(SC-007)·방문자 후보(SC-013)를 DOM으로 확인한다.
  - **개발 모드 전용 훅**: `process.env.NODE_ENV !== "production"`일 때만 `window.__blogvilleTown = { entrances(), player(), prompt(), teleport(label), fps() }`를 연다. 배포 빌드에는 들어가지 않는다 (Next.js가 빌드 때 `NODE_ENV`를 바꿔 넣는다, **라이브러리 동작**). e2e가 입구 안내 문구(FR-020)·Space로 들어가기·클릭으로 문 앞까지 걷기·가장자리에서 멈춤·FPS를 확인한다.
  - shop이 요청한 `data-player-look`(`specs/006-shop/contracts/shared-modules.md` 5절)도 `TownGame` 바깥 요소에 둔다.
- **Rationale**: 지금 e2e는 캔버스 안을 스크린샷으로만 본다 (`e2e/decisions.mjs`, `e2e/flow.mjs`, **코드 확인**). US2·US3·US9·SC-012를 자동으로 확인할 방법이 없다. 훅은 화면에 그린 값만 보여 주고 서버 데이터를 더 내보내지 않는다.
- **Alternatives considered**: 픽셀 비교 — 글꼴·안티앨리어싱으로 흔들린다. 운영에도 훅 — 불필요한 노출.

## R-20 `user_animals` UNIQUE(`user_id`, `id`)

- **Decision**: `src/db/schema.ts`의 `userAnimals` 블록에 `unique("user_animals_user_id_id_uq").on(t.userId, t.id)` → `npm run db:generate` → 생성 SQL 맨 위에 `-- TOWN-09 (요청: blog, BLOG-04): 전시 동물 복합 FK가 가리킬 UNIQUE (ERD 3.11, 7장 6-2)` 주석 → `npm run db:migrate`. 데이터 이전은 없다 (`id`가 PK라 이미 고유하다).
- **Rationale**: PostgreSQL은 복합 FK가 가리키는 열 조합에 UNIQUE 제약(또는 부분이 아닌 고유 인덱스)이 있어야 한다 (**라이브러리 동작**). 기존 부분 고유 인덱스(`user_animals_starter_uq`, `user_animals_level_uq`)는 조건이 있어 대상이 될 수 없다. 이 저장소는 이미 `unique("…").on(…)`을 쓴다 (`categories_blog_name_uq`, **코드 확인**). blog의 `blogs.showcase_animal_id` 복합 FK(`specs/002-blog/data-model.md` B-M3)보다 먼저 있어야 한다.

## R-21 상점 이름표 부제

- **Decision**: 상점 부제 `캐릭터·배경`(`scene.ts`)을 `아바타·가구·배경·성장 아이템`으로 바꾼다 *(plan 임시)*.
- **Rationale**: spec 기본값 "상점이 실제로 파는 분류(아바타 꾸미기·가구·배경·성장 아이템)", shop의 상점 구역 4개(`SHOP_SECTIONS`). 건물 아래 12px 글씨라 `아바타 꾸미기`를 `아바타`로 줄였다 — 정확한 문구는 팀 확인 (남은 문제).

## R-22 광장 최소 높이와 가로 휴대폰

- **Decision**: 광장 최소 높이 420px(FR-005)을 유지한다.
- **충돌**: 화면 높이가 375px쯤인 가로 휴대폰에서는 헤더 + 420px이 화면보다 커 페이지 세로 스크롤이 생긴다. FR-005의 "최소 420px"과 "페이지 스크롤 없음"을 함께 지킬 수 없다. 숫자로 적힌 쪽을 따르고 남은 문제로 올린다. 지금 코드도 같다(그동안은 가로 휴대폰에 간단 메뉴를 보여서 드러나지 않았다).

## R-23 테스트 전략

- **단위** (`scripts/test-town.ts`, `package.json`에 `test:town` 추가하고 `test` 체인 끝에 붙임, 관례: `expect(name, got, want)`·`✅/❌`·실패 시 `exit(1)`):
  - `houseStage`: Lv.1·9 → 1, 10·29 → 2, 30·99 → 3 (SC-014 경계).
  - `houseRoof`: 코드값 → 색, `null` → 배경 강조색, 배경을 바꿔도 고른 색 유지.
  - `houseLabel`: 12자 이하 그대로, 13자·20자·이모지 닉네임 자르기.
  - `sampleDistinct`: 100개 중 10개(겹침 없음, 모두 목록 안), 7개면 7개 전부, 0개면 빈 배열, 고정 난수로 재현.
  - `canFavorite(count)`: 9 → 가능, 10 → 불가 (`FAVORITE_LIMIT = 10`).
  - 배치: `layout()`에 3단계 집 10채 + 내 집을 넣었을 때 그림 사각형이 서로·다른 건물과 겹치지 않음, 입구가 모두 세계 안 (US11-6). 아이소메트릭 뒤에는 투영 사각형으로도.
  - 투영: `toLogical(toScreen(p)) ≈ p`, 화면 속도 정규화(대각선 길이 = 직선 길이).
  - 부화 비중 1,000번: 고정 난수열로 `pickWeighted`를 1,000번 → 종류별 비율이 40/30/20/10%에서 ±5%p 안 (SC-009).
- **E2E** (관례: 맨 위 요구사항 ID 주석, `.env.local` + `pg`로 DB 준비·확인, 실행마다 새 아이디, `check()`·실패 시 `exit(1)`, 스크린샷):
  - 새 `e2e/town.mjs`: 로그인 → 광장, 환영 문구 한 번(새로고침·뒤로 가기 10회), [✕], 방문자 구경, 헤더(이동 메뉴 없음·나가기·상태창·375px), 입구 안내·Space·클릭·방문자 로그인 안내, 키 이동·가장자리, 집 단계(Lv.9·10·29·30), 지붕 색, 아이소메트릭 뒤 같은 시나리오, FPS.
  - 새 `e2e/favorites.mjs`: 내 이웃 목록(주인만), ☆/⭐, 11번째 거부, 9명에서 동시 요청, 이웃 취소 → 집 사라짐, 즐겨찾기 10채(글 없는 블로그 포함), 빈 안내, 패널, 이웃 아닌 대상 조작 요청 거부, 방문자 후보(100위 밖·공개 글 없는 블로그 0건, 20번 열기).
  - 고칠 `e2e/farm.mjs`: 성장 아이템 사용(성장치·수량·하루 여러 번·0개 거부·알 거부·넘침 버림·동시·조작 인자), 꽉 참 문구, 원장 합계. 지금 파일에 없는 시나리오도 더한다(**코드 확인**: 지금은 첫 알·부화·돌보기·다 키움·알 사기·5마리·글쓰기 연동만): 레벨 보상 알(US6-2), 같은 무료 알·같은 돌보기 동시 요청(SC-008), 아직 안 된 레벨 알·코인 부족·남의 알/동물 조작 요청(Edge Cases).
  - 고칠 `e2e/mobile.mjs`에 375px `/farm` 버튼 44×44px(SC-004)도 넣는다.
  - 고칠 `e2e/mobile.mjs`: 휴대폰도 광장(캔버스·조이스틱·탭으로 건물), 두 줄 헤더, 가로 스크롤 없음.
  - 고칠 `e2e/decisions.mjs`: "글 없는 블로그는 이웃집에 없음 / 공개 글을 쓰면 나타남"(TOWN-04 옛 규칙)을 즐겨찾기 규칙으로.
  - 조작 요청은 `e2e/social.mjs`·`e2e/params.mjs`처럼 실제 Server Action 요청을 잡아 인자만 바꿔 다시 보낸다 (**코드 확인**).
- **손으로**: 실제 휴대폰 터치(SC-003 1분 안에 다섯 건물, SC-005 FPS), 아이소메트릭 가림(US2-7) 스크린샷.

## R-24 구현 전에 설치된 문서로 확인할 라이브러리 동작

| 패키지 | 확인할 것 | 쓰는 곳 |
|---|---|---|
| `next` 16.3.8 | 서버 컴포넌트의 `cookies()`는 읽기만 (코드 주석으로 일부 확인) | R-02 |
| `next` 16.3.8 | Server Action에서 쿠키를 바꾸면 지금 화면을 다시 그리는지 (그래서 브라우저에서 지움) | R-02 |
| `next` 16.3.8 | `signUp`(Server Action)이 `cookies().set("bv_welcome", …)` 뒤 `redirect("/town")`할 때, 같은 응답에서 그려지는 `/town`의 `cookies()`에 그 쿠키가 보이는지 (Path `/town` 포함). 안 보이면 서버의 `initial` 없이 `WelcomeBanner`가 브라우저 쿠키만으로 연다 | R-02 |
| `next` 16.3.8 | 뒤로 가기 때 클라이언트 라우터 캐시를 다시 쓰는지 | R-02, SC-002 |
| `next` 16.3.8 | 쿠키를 읽는 페이지가 요청마다 렌더링되는지 (방문자 무작위) | R-07 |
| `next` 16.3.8 | 빌드 때 클라이언트 코드의 `process.env.NODE_ENV`가 바뀌어 개발 훅이 빠지는지 | R-19 |
| `phaser` ^4.2.1 | Graphics → 텍스처(`generateTexture` 또는 RenderTexture·DynamicTexture), 텍스처 최대 크기 | R-16, R-17 |
| `phaser` ^4.2.1 | `cameras.main.startFollow`·`setBounds`, 포인터 `worldX/worldY`, `game.loop.actualFps` | R-16, R-17, R-19 |
| `eslint-config-next` 16.3.8 | effect 안 `setState` 규칙(React 19 린트) — `WelcomeBanner` 구현 방식 | R-02 |
| `drizzle-kit` ^0.31.11 | `unique()`가 `ADD CONSTRAINT … UNIQUE`로 생성되는지 | R-20 |

Phaser 4 문서 위치(코드 저장소 `CLAUDE.md`): `node_modules/phaser/skills/`, `node_modules/phaser/changelog/v4/4.0/MIGRATION-GUIDE.md`.

---

## NEEDS CLARIFICATION 정리

Technical Context에 남긴 NEEDS CLARIFICATION은 없다. 이 문서에서 결정으로 바꾸었다:

| 처음 모르던 것 | 결정 |
|---|---|
| 휴대폰에서 광장을 띄울지 | R-01 띄운다 (팀 확인은 남은 문제) |
| 환영 문구 한 번의 방법 | R-02 일회용 쿠키 |
| 인기 블로그를 고르는 방법·비용 | R-07 |
| 집 단계 저장 여부 | R-08 저장하지 않음 |
| 아이소메트릭 타일 크기·방식 | R-16 64 × 32, 논리 좌표 유지 |
| 375px 헤더 공간 | R-11 두 줄 |
| 캔버스 검증 방법 | R-19 |

spec이 문구나 규칙을 정하지 않아 plan이 임시로 정한 것과 spec끼리 충돌하는 것은 [plan.md](plan.md) "남은 문제"에 모았다.
