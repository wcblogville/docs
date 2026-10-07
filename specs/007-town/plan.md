# Implementation Plan: 광장 (TOWN)

**Branch**: `007-town` | **Date**: 2026-10-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/007-town/spec.md`

**Note**: 코드 위치는 코드 저장소(`blogville`, `main` `feb4c05`) 기준 상대 경로, 문서는 이 저장소 기준 상대 경로다. 공통 약속(테이블 담당, 공통 모듈 소유, 구현 순서)은 7개 spec이 함께 쓰는 plan 공통 맥락을 따른다. 근거는 [research.md](research.md)(R-번호), 데이터는 [data-model.md](data-model.md), 바깥 인터페이스는 [contracts/](contracts/), 검증은 [quickstart.md](quickstart.md).

## Summary

광장(TOWN-01~11)을 지금의 "공개 글이 있는 블로그 8채를 모두에게 같은 순서로 보여 주는 2D 광장"에서 spec 목표로 옮긴다.

- **광장의 집**: 회원은 즐겨찾기한 이웃(최대 10명)의 집만, 방문자는 인기 블로그(이웃 수 → 최근 공개 글) 100곳 중 무작위 10곳. 자리 8 → 10. 즐겨찾기는 social이 더하는 `follows.is_favorite`를 town의 Server Action이 회원 잠금 안에서 세며 바꾼다. 내 블로그 홈에 주인만 보는 "내 이웃 목록"(☆/⭐), 🏘 패널은 모든 이웃의 링크.
- **집**: 주인 레벨로 1~3단계(저장하지 않고 원장 합계에서 계산), 지붕 색은 blog가 더하는 `blogs.roof_color`(8색, NULL = 배경 색)를 꾸미기 화면에서 고른다.
- **광장 화면**: 환영 문구를 일회용 쿠키로 딱 한 번, 휴대폰에서도 간단 메뉴 대신 조이스틱 광장, 헤더에 유저 상태창(640px 미만은 두 줄 헤더).
- **농장**: 상점에서 산 성장 아이템을 쓰는 `applyGrowthItem`(shop의 수량 도우미 + 기존 `addGrowth`), 꽉 참 문구를 spec대로. 도감·전시(US7)는 blog가 만들고 town은 `user_animals` UNIQUE(`user_id`, `id`)를 준다.
- **2.5D**: 마지막 단계에서 논리 좌표·물리·입구 판정은 그대로 두고 그리기만 2:1 아이소메트릭(타일 64 × 32)으로 바꾼다.

town이 담당하는 표의 구조 변경은 `user_animals` UNIQUE 하나뿐이다. 나머지 데이터는 다른 spec의 표를 참조한다 (요청은 각 담당 spec plan에 이미 들어가 있다).

## Technical Context

**Language/Version**: TypeScript ^5 (`strict: true`, 별칭 `@/*` → `src/*`), Node.js 20.9 이상

**Primary Dependencies**: Next.js 16.3.8 (App Router, Server Component, Server Action — 이 기능에 Route Handler 없음), React 19.2.8 (`useTransition`), Phaser ^4.2.1 (광장, 브라우저에서만 동적 import, Arcade 물리, `Scale.RESIZE`), drizzle-orm ^0.45.3 / drizzle-kit ^0.31.11, better-auth ^1.7.7 (`getViewer()`로만 씀), zod ^4.6.5 (Action 입력), Tailwind CSS ^4 (`pointer-coarse:` 변형, `sm` 640px), `node:crypto` `randomInt`(방문자 무작위·부화). **새 의존성 없음.** `node_modules`가 없는 체크아웃이라 Next.js 16·Phaser 4 세부 API는 구현 전 설치된 문서로 확인한다 (research R-24)

**Storage**: PostgreSQL (README 안내 17). 담당 표 `animal_species`·`user_animals`·`animal_cares`(구조 변경: `user_animals` UNIQUE 하나). 참조 표 `follows`(`is_favorite`, social), `blogs`(`roof_color`, blog), `items`·`user_items`(성장 아이템·수량, shop), `point_ledger`(game), `profiles`(`photo_key`, auth), `posts`, `attendances`, `users`, `attachments`. 브라우저 쿠키 `bv_welcome`(환영 문구 한 번, 로그인과 무관). 집 단계·방문자 후보는 저장하지 않고 계산

**Testing**: `npm test`(`tsx scripts/test-*.ts`, 테스트 프레임워크 없이 `expect`·`✅/❌`)에 `test:town`(`scripts/test-town.ts`) 추가. E2E는 `@playwright/test`의 `chromium`을 Node 스크립트로(`e2e/*.mjs`, 개발 서버 대상, `pg`로 DB 준비): 새 `e2e/town.mjs`·`e2e/favorites.mjs`, 고칠 `e2e/farm.mjs`·`e2e/mobile.mjs`·`e2e/decisions.mjs`. 캔버스 안은 숨은 집 목록(`data-*`)과 개발 모드 전용 훅으로 확인 (R-19). 실제 휴대폰 터치·FPS는 손으로

**Target Platform**: 최신 Chrome·Edge·Safari, 모바일 Chrome·Safari (NF-21). 375px 휴대폰(터치)부터 PC(키보드·마우스)까지 같은 광장

**Project Type**: 웹 애플리케이션 — 한 Next.js 프로젝트(화면·Server Action·DB 접근이 한 저장소, 프런트/백엔드 분리 없음)

**Performance Goals**: 광장 약 60fps, 최저 50fps (SC-005, NF-20) — 바닥을 텍스처로 굳힘(R-17). `/town` 서버 데이터는 쿼리 4~5개를 `Promise.all`로 (집 ≤ 11채의 경험치 합계 포함). 헤더는 상태창 때문에 쿼리가 늘지 않는다(`getViewer()` 한 쿼리). 글 목록 1초(NF-07)에 영향 없음

**Constraints**: 원장 무결성(잔액·레벨·집 단계 저장 안 함), 지급·차감은 `lockUser` 트랜잭션 안, 권한·검증은 서버(constitution IV·V). 그림은 `src/lib/art/`의 코드 SVG만(외부 그림 파일 없음). 실시간 멀티플레이 없음(FR-061). 광장 데이터는 열 때 한 번 읽음. 375px 가로 스크롤 없음·누르는 영역 44×44px(SC-004). 테이블 담당·공통 모듈 소유 규칙(남의 파일은 한두 줄 추가만)

**Scale/Scope**: 수업 프로젝트(3명 팀), 회원·블로그 수백 이하(**추측**). 한 화면의 집 ≤ 11, 즐겨찾기 ≤ 10, 방문자 후보 100, 이웃 수 제한 없음. 화면: `/town`(재구성), `/farm`(추가), 블로그 홈 내 이웃 목록·꾸미기 지붕 색 칸·헤더(모든 화면). spec: User Story 11개, FR 61개, SC 14개

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` v1.0.0 기준.

| 원칙 | 설계 전 | 근거 | 설계 후 재평가 |
|---|---|---|---|
| I. 글쓰기가 먼저 | 통과 | 광장은 블로그·마을 소식·글쓰기로 가는 허브이고, 즐겨찾기는 읽고 교류할 블로그를 가까이 두는 기능이다. 집 단계는 글쓰기 보상(레벨)의 결과를 보여 줄 뿐 글쓰기를 막지 않는다. 실시간 멀티플레이 없음(FR-061), 유료 결제 없음(성장 아이템은 코인만) | 통과. 증축 구매·새 코인 사용처를 만들지 않았다. 성장 아이템 사용에 경험치를 주지 않는다(돌보기만) |
| II. 요구사항 ID | 통과 | TOWN-01~11, NF-06·17·20을 그대로 가리킨다. 마이그레이션 SQL·코드 주석에 ID를 단다 | 통과. 구현 뒤 `docs/01-requirements.md` TOWN 상태·수용 기준을 갱신하고 변경 이력에 한 줄 더한다(T16). TOWN-06은 ❌ 그대로 |
| III. 확인할 수 있는 수용 기준 | 통과 (주의) | spec의 문구를 그대로 쓴다. 모든 수용 시나리오·SC를 [quickstart.md](quickstart.md)의 검증 줄에 연결했다 | **주의**: spec에 없는 사용자 문구 몇 개(이웃 아닌 즐겨찾기 거부, 성장 아이템 없음·사용 성공, 지붕 색 거부, 상점 부제, 접근 이름)를 *(plan 임시)*로 정했다. 원칙은 "보여줄 문구는 spec에" → spec 보강을 남은 문제로 올렸다. 설계를 막지는 않는다 |
| IV. 권한·검증은 서버 (NON-NEGOTIABLE) | 통과 | 새 Action(`toggleFavorite`, `setRoofColor`, `applyGrowthItem`)은 첫 줄 `requireMember()`, 입력은 zod·`parseId`, 대상은 요청 값이 아니라 로그인한 회원으로 정한다(내 `follows` 행, `blogs.owner_id = 나`, 내 동물). 내 이웃 목록은 서버가 주인에게만 그린다. 쿼리는 Drizzle 값 바인딩. 비밀값 없음 | 통과. 조작 요청(남의 ID, 목록 밖 색, 이웃 아닌 대상, 수량 0, 알)을 e2e로 확인한다. 개발 모드 훅은 배포 빌드에서 빠진다(R-19). `bv_welcome`은 HttpOnly가 아니지만 로그인과 무관한 표시 쿠키(값 `1`)다(R-02) |
| V. 원장 무결성 (NON-NEGOTIABLE) | 통과 (정당화 1건) | 농장 보상·알 구매는 지금처럼 원장, `lockUser` 트랜잭션. 성장 아이템 사용은 동물 확인 → 수량 −1(shop 도우미) → 성장 → 다 자라면 원장을 한 트랜잭션으로. 레벨·집 단계는 저장하지 않는다. 구조 변경은 마이그레이션 파일로 | 통과. DB 제약: 돌보기 PK, 무료 알 부분 UNIQUE, `quantity ≥ 0`(shop), `roof_color` CHECK(blog), `user_animals` UNIQUE(전시 복합 FK). **즐겨찾기 최대 10명은 DB 제약이 아니라 회원 잠금 안에서 세기** → Complexity Tracking |
| VI. 모바일에서도 | **위반 고침** | 지금 코드는 휴대폰에서 광장 대신 간단 메뉴를 보여 "광장은 마우스와 터치로 조작"을 휴대폰에서 지키지 않는다. 이 plan이 고친다(R-01) | 통과. 휴대폰도 조이스틱 광장, 640px 미만 회원 헤더 두 줄로 44px·가로 스크롤 없음(R-11), `e2e/mobile.mjs`·`e2e/nonfunctional.mjs`로 확인 |
| VII. 단순하게, 최소 정보 | 통과 | 새 개인정보 없음(방문자 정보·IP 저장 없음, 쿠키 값 `1`). 범위 밖 기능 없음(가방·랭킹·증축·실시간). 그림은 코드 SVG. 새 의존성·새 표 없음 | 통과. 집 단계·방문자 후보·지붕 색(hex)을 저장하지 않고 계산한다. 개발 훅은 검증용으로만 |

**판정**: 진행 가능. 정당화가 필요한 항목 1건(V)은 아래 Complexity Tracking에 적었다.

## Project Structure

### Documentation (this feature)

```text
specs/007-town/
├── spec.md                  # 기능 명세 (고치지 않음)
├── checklists/
│   └── requirements.md      # spec 품질 점검
├── plan.md                  # 이 파일
├── research.md              # Phase 0: 결정·근거·대안 (R-01~R-24)
├── data-model.md            # Phase 1: 표 현재·목표, 상태 전이, 마이그레이션 순서
├── quickstart.md            # Phase 1: 실행·검증 시나리오 (US·SC 연결)
└── contracts/               # Phase 1: 바깥 인터페이스
    ├── town-screen.md       # /town 화면, TownData, 환영 쿠키, 입구, 검증용 DOM·훅
    ├── favorites.md         # 내 이웃 목록, toggleFavorite, 🏘 패널, 집 고르기 함수
    ├── farm.md              # /farm, 농장 Server Action (applyGrowthItem 새로), 도감·전시 경계
    └── header-roof.md       # 헤더 유저 상태창(375px 두 줄), 지붕 색 칸과 setRoofColor
```

`tasks.md`는 `/speckit-tasks`가 만든다.

### Source Code (repository root)

코드 저장소의 실제 구조에서 이 기능이 만지는 파일만 적었다.
표시: `★` 새 파일 · `✎` 바꿈(town 소유) · `✂` 지움 · `＋` 공통 모듈 추가(다른 spec 소유 파일에 한두 줄) · `⇠` 다른 spec이 town 파일에 끼우는 줄(요청 수락)

```text
src/
├── app/
│   ├── town/
│   │   ├── page.tsx                 ✎ TownData 새 모양, 환영 쿠키 읽기, NeighborPanel·WelcomeBanner, 숨은 집 목록,
│   │   │                              휴대폰 분기(phone:) 제거, 회원 높이 --header-h-member, /onboarding 리다이렉트 제거
│   │   └── actions.ts               ★ toggleFavorite, setRoofColor
│   ├── farm/
│   │   ├── actions.ts               ✎ FULL 문구, applyGrowthItem 추가
│   │   ├── page.tsx                 ✎ 안내 문장, 성장 아이템 데이터 넘기기
│   │   └── farm-view.tsx            ✎ 동물 카드의 🌱 성장 아이템 줄, 돌보기 버튼 44px, 도감 링크(블로그 단계 12 뒤)
│   ├── blog/[slug]/page.tsx         ＋ (blog 소유) 주인일 때 <MyNeighbors /> 한 줄
│   ├── closet/page.tsx              ＋ (shop 소유) <RoofColorSection /> 한 줄
│   ├── layout.tsx                   (town 소유, 바꿀 것 없음 — 헤더·푸터 그대로)
│   └── globals.css                  ✎ --header-h-member 변수, phone 변형 주석 (변형 자체는 남김)
│                                      (공통 맥락 소유 표에 없는 파일 — 헤더 높이 변수가 헤더 소유 town의 일이라 town이 고친다)
├── components/
│   ├── site-header.tsx              ✎ 상태창, 640px 미만 두 줄, 로고 늘 보임
│   │                                ⇠ game: 🔔·LevelUpPopup·AttendanceDayWatcher / auth: SessionKeeper / shop: CharacterBadge outfit
│   ├── user-status.tsx              ★ 유저 상태창 (프로필 사진 또는 캐릭터 얼굴, 닉네임, 블로그 제목)
│   ├── exit-button.tsx              ✎ HIDDEN_ON에서 /onboarding 제거, HomeLogo 숨김 처리 제거
│   ├── character.tsx                (shop·blog 소유, town은 CharacterBadge를 쓰기만)
│   └── town/
│       ├── scene.ts                 ✎ 집 단계·지붕(서버 값), NEIGHBOR_SLOTS 10, houseLabel, layout() 내보내기,
│       │                              상점 부제, 개발 모드 훅, (단계 13) 아이소메트릭 투영·바닥 텍스처
│       │                            ⇠ shop: charKey(lookKey(asset, outfit)), characterDataUri(…, outfit)
│       ├── town-game.tsx            ✎ PHONE_MEDIA 분기 제거, data-player-look(shop 요청), 훅 연결
│       ├── types.ts                 ✎ TownHouse(stage, roof, outfit), TownLink, TownData(panel, attendanceDay)
│       ├── town-menu.tsx            ✂ 휴대폰 간단 메뉴 (R-01)
│       ├── welcome-banner.tsx       ★ 환영 문구 (클라이언트, 쿠키 확인·지우기)
│       ├── neighbor-panel.tsx       ★ 🏘 이웃집 패널 (빈 안내 포함)
│       ├── my-neighbors.tsx         ★ 내 이웃 목록 (서버, 블로그 홈 주인용)
│       ├── favorite-button.tsx      ★ ☆/⭐ 버튼 (클라이언트)
│       ├── roof-color-section.tsx   ★ 꾸미기 🏠 지붕 색 (서버)
│       └── roof-color-picker.tsx    ★ 색 8개 + 배경 색 따라가기 (클라이언트)
├── db/
│   └── schema.ts                    ✎ userAnimals 블록에 unique("user_animals_user_id_id_uq")
├── lib/
│   ├── town.ts                      ★ houseStage, ROOF_PALETTE, houseRoof, houseLabel, FAVORITE_LIMIT, sampleDistinct,
│   │                                  (단계 13) toScreen·toLogical
│   ├── farm.ts                      (바꿀 것 없음 — 숫자·단계 계산 그대로)
│   ├── device.ts                    ✎ 주석만 (PHONE_MEDIA는 남김)
│   ├── blog.ts                      (blog 소유, 새 파일) ROOF_COLORS를 읽어 씀
│   └── art/
│       ├── town.ts                  ✎ HOUSE_STAGES 1~3단계 그림, (단계 13) 아이소메트릭 건물·바닥 그림
│       └── animals.ts               (바꿀 것 없음 — blog 도감도 이 그림을 씀)
└── server/
    ├── town.ts                      ✎ getTownHouses(즐겨찾기), getMyHouse, listMyNeighbors, getVisitorHouses, 집 단계·지붕
    │                                ⇠ shop: outfitOf / game: hasAttendedToday → viewer.attendance
    ├── farm.ts                      ✎ getFarm에 성장 아이템, ⇠ game: farm_grown INSERT → addLedgerEntry
    ├── dal.ts                       ＋ (auth 소유) getViewer 프로필에 photoKey
    └── inventory.ts                 (shop 소유) consumeGrowthItem을 부름

drizzle/
└── NNNN_<이름>.sql                  ★ user_animals UNIQUE(user_id, id) (번호는 generate가 정함)

scripts/
└── test-town.ts                     ★ 집 단계·지붕·이름표·무작위 고르기·배치 겹침·투영·부화 1,000번

e2e/
├── town.mjs                         ★ 광장·환영·이동·입구·헤더·집 단계·지붕 색·FPS
├── favorites.mjs                    ★ 즐겨찾기·광장 집·패널·방문자 인기 블로그
├── farm.mjs                         ✎ 성장 아이템, 꽉 참 문구
├── mobile.mjs                       ✎ 휴대폰도 광장, 두 줄 헤더
├── decisions.mjs                    ✎ TOWN-04 옛 규칙(글 없는 블로그 제외) → 즐겨찾기 규칙
└── helpers.mjs                      ＋ (auth 소유) 광장 훅을 기다리는 도우미 추가만

package.json                         ✎ test:town 추가, test 체인 끝에
docs/02-erd.md                       ✎ user_animals 부분(3.11, 5장 집 성장, 7장 6-2)
docs/01-requirements.md              ✎ (구현 뒤) TOWN 상태·수용 기준 + 변경 이력 한 줄
CLAUDE.md, README.md                 ✎ 광장 규칙(휴대폰도 광장, 아이소메트릭 뒤 2D 문구), e2e 표 두 줄
```

**Structure Decision**: 한 Next.js 프로젝트의 기존 폴더 역할을 그대로 쓴다 — 화면·Action은 `src/app/`, 화면 조각은 `src/components/town/`(광장 전용)과 `src/components/`(헤더), DB 쿼리는 `src/server/town.ts`·`farm.ts`, DB 없는 규칙은 `src/lib/town.ts`, 그림은 `src/lib/art/town.ts`. town의 새 Server Action은 영역 파일 `src/app/town/actions.ts`에 둔다. 다른 spec 소유 파일(`dal.ts`, 블로그 홈, 꾸미기, `helpers.mjs`)에는 한두 줄만 끼운다.

## 변경 단위 (현재 코드 → spec 목표)

"단계"는 공통 맥락 5.1의 구현 순서다 (town은 11, 아이소메트릭은 13).

| # | 단위 | 종류 | 현재 코드 | 목표 | spec | 선행 | 단계 |
|---|---|---|---|---|---|---|---|
| T1 | `user_animals` UNIQUE(`user_id`, `id`) | 마이그레이션 | 부분 고유 인덱스 2개뿐 | `user_animals_user_id_id_uq` 추가, 데이터 이전 없음 ([data-model.md](data-model.md) 5) | FR-052 (blog 전시 FK) | 없음 | 11 (blog B-M3 전, 가장 먼저 merge 가능) |
| T2 | 광장 규칙 순수 함수 | 순수 함수 | 없음 (단계 상수 `DEFAULT_HOUSE_STAGE = 1`, 지붕 = 배경 색) | `src/lib/town.ts`: `houseStage`, `ROOF_PALETTE`, `houseRoof`, `houseLabel`, `FAVORITE_LIMIT`, `sampleDistinct` | FR-024·029·033·039·040·059 | blog `ROOF_COLORS` | 11 |
| T3 | 집 데이터 서버 함수 | 서버 함수 | `getTownHouses(나, 8)`: 공개 글 있는 모든 블로그 최근 글 순, 회원·방문자 같은 목록 | `getTownHouses`(즐겨찾기 ≤ 10), `listMyNeighbors`, `getVisitorHouses`(100 → 무작위 10), 집마다 `stage`·`roof`(+ shop `outfit`) | FR-025~030, FR-059, SC-007·013·014 | social U4, blog B2, T2 | 11 |
| T4 | 즐겨찾기 Action과 내 이웃 목록 | Server Action + 화면 | 없음 | `toggleFavorite`(잠금 안에서 10명), 블로그 홈 주인용 `MyNeighbors`·`FavoriteButton` | FR-031~035, US4, SC-006 | social U4, T3 | 11 |
| T5 | 광장 화면 재구성 | 화면 | 패널 = 광장 집, 빈 안내 없음, `TownData`에 단계·지붕 없음 | `NeighborPanel`(모든 이웃·빈 안내, 줄 높이 44px), 숨은 집 목록, 새 `TownData`, 회원 높이 변수 | FR-005·006·028·030, SC-004 | T3 | 11 |
| T6 | 환영 문구 딱 한 번 | 화면 + 쿠키 | `?welcome=1`이면 보임(새로고침해도), [✕] = `/town` 링크 | `bv_welcome` 쿠키 + `WelcomeBanner`(브라우저에서 확인·지우기) | FR-003·004, SC-002 | auth: `signUp`이 쿠키를 심음 | 11 |
| T7 | 휴대폰도 광장 | 화면 | 640px 미만·가로 휴대폰은 `TownMenu`(게임 꺼짐) | `TownMenu` 삭제, 모든 크기에서 게임, 터치는 조이스틱 | FR-012, SC-003·004, VI | 팀 확인 (남은 문제 1) | 11 |
| T8 | 집 3단계·자리 10·이름표 | 그림 + 광장 코드 | `HOUSE_STAGES` 1단계, 자리 8, 이름표가 `의 집`까지 잘림 | 3단계 그림, 자리 10(위 5·아래 5), `houseLabel`, `layout()` 내보내 겹침 시험, `plantTrees()`의 집 자리 피하는 반경(지금 150)을 3단계 크기로 | FR-024, FR-058~060, US11, SC-014 | T2·T3 | 11 |
| T9 | 지붕 색 | Server Action + 화면 | 장착 배경 강조색만 | `setRoofColor`, 꾸미기 `RoofColorSection`, 서버가 정한 `roof` | FR-039·040, US10 | blog B2 | 11 |
| T10 | 헤더 유저 상태창 | 화면 | 로고(좁으면 숨김)·나가기·Lv·🪙·👑·캐릭터 얼굴·로그아웃 한 줄 | `UserStatus`, 640px 미만 두 줄, 로고 늘 보임, `getViewer`에 `photoKey` | FR-053~057, US8, SC-004·010 | auth `photo_key`(없으면 캐릭터 얼굴로 먼저), game 🔔 자리 | 11 |
| T11 | 농장 성장 아이템·문구 | Server Action + 화면 | 성장 아이템 없음, 서버 꽉 참 문구가 spec과 다름, 돌보기 버튼이 `py-1.5 text-xs`라 44px보다 낮음 | `applyGrowthItem`, `getFarm` 성장 아이템, 카드 🌱 줄, `FULL` 문구, `farm_grown` → `addLedgerEntry`, 농장 버튼 44×44px 이상 | FR-041·043·048·050, US6, SC-004·008 | shop 성장 아이템·`consumeGrowthItem`, game `addLedgerEntry` | 11 |
| T12 | 출석 도장 값 | 서버 함수 | `hasAttendedToday()` 쿼리 | `viewer.attendance`(game)로, `attendanceDay` 함께 | FR-023 | game 단계 6 | 11 |
| T13 | 상점 부제·검증 수단 | 광장 코드 | 부제 `캐릭터·배경`, 캔버스 검증 수단 없음 | 부제 *(plan 임시)*, 개발 모드 훅, `data-player-look` | FR-019, R-19·21 | 없음 | 11 |
| T14 | 2.5D 아이소메트릭 | 그림 + 광장 코드 | 2D 탑다운 | 논리 좌표 유지 + 2:1 투영, 타일 64 × 32, 바닥 텍스처, 건물·집 3단계 다시 그림, 2D 코드 삭제 | FR-037·038, US9, SC-005·012 | T8, T13 | 13 |
| T15 | 검증 | 테스트 | `test-game`의 농장 계산, `e2e/farm.mjs`·`mobile.mjs`·`decisions.mjs`(옛 규칙) | `test:town`, `e2e/town.mjs`, `e2e/favorites.mjs`, 기존 3개 고침(`e2e/farm.mjs`에는 지금 없는 레벨 알·동시 요청·조작 요청도 추가) ([quickstart.md](quickstart.md)) | 전체 | 각 단위와 함께 | 11·13 |
| T16 | 문서 | 문서 | ERD ⏳, `CLAUDE.md` "아이소메트릭 전까지 2D" | ERD(3.11·5장·7장), `CLAUDE.md`·`README.md`, 요구사항 문서 상태 + 변경 이력 | II, 테이블 담당 규칙 5 | 각 단위 뒤 | 11·13 |

묶음 순서 제안: **T1 → T2·T3·T5·T8·T13(광장 데이터·그림) → T4(즐겨찾기) → T9(지붕) → T10(헤더) → T6(환영, auth 쿠키와 같은 PR) → T11(농장, shop 뒤) → T12(game 뒤) → T7(팀 확인 뒤) → T14(단계 13)**. T15·T16은 각 묶음에 함께 넣는다.

## 의존성 (다른 spec)

### town이 기다리는 것

| spec (단계) | 먼저 되어야 하는 일 | 쓰는 town 단위 | 없으면 |
|---|---|---|---|
| auth (1) | 가입 통합(온보딩 없음, `getViewer()`의 프로필이 늘 있음), `e2e/helpers.mjs` `loginDev` 고침 | 전체(특히 e2e) | 광장 페이지의 `/onboarding` 분기를 지울 수 없고 e2e 로그인이 깨진다 |
| auth (1) | `signUp`이 성공 뒤 쿠키 `bv_welcome`을 심고 `/town`으로 보냄 (auth research가 "town이 신호를 고르면 심는다"고 받음) | T6 | 환영 문구가 보이지 않는다 (임시로 `?welcome=1`을 함께 읽을 수는 있다) |
| auth (1) | `profiles.photo_key` (R16) | T10 | 프로필 자리는 늘 캐릭터 얼굴 (FR-054의 대체 동작이라 기능은 완성) |
| social (5) | `follows.is_favorite` (U4) | T3·T4 | 즐겨찾기·광장 집·내 이웃 목록 불가 |
| social (5) | 이웃 새 글 7일 우선 정렬 (`listFeed`, FR-036) | (town FR-036은 social이 구현) | 즐겨찾기가 이웃 새 글 순서에 반영되지 않음 |
| shop (10) | `items.type = 'growth'`·`growth_value`, `user_items.quantity`, 성장 아이템 3종 시드·구매, `consumeGrowthItem` | T11 | 성장 아이템 사용 불가 |
| shop (10) | `outfitOf`, `lookKey`, `characterDataUri(…, outfit)`, `CharacterBadge` `outfit` | T3·T10 (선택) | 차림 없이 기본 캐릭터로 그린다 |
| blog (B2, town 11 전) | `blogs.roof_color` + CHECK, `src/lib/blog.ts` `ROOF_COLORS` | T2·T9 | 지붕 색 고르기 불가 (배경 색만) |
| blog (블로그 홈) | `src/app/blog/[slug]/page.tsx`에 `MyNeighbors` 자리 | T4 | 내 이웃 목록을 둘 곳이 없다 |
| game (6) | `getViewer()`의 `attendance`, `addLedgerEntry` | T12, T11 | 지금 쿼리·직접 INSERT로 동작은 같다 |

### town이 다른 spec에 주는 것 (요청 수락)

| spec | town이 하는 일 / 허락하는 줄 | town 단위 |
|---|---|---|
| blog (12) | `user_animals` UNIQUE(`user_id`, `id`) — blog의 `showcase_animal_id` 복합 FK(B-M3) 선행. 동물 그림 `src/lib/art/animals.ts` 공유. `grown`은 되돌아가지 않는다 | T1 |
| blog | 바뀐 닉네임·블로그 이름·주소가 다음 광장 방문에 반영됨(광장은 그릴 때 `slug`·이름을 읽음) | T3 |
| shop | `scene.ts`의 캐릭터 텍스처 키·그림에 `outfit` (`lookKey`, `characterDataUri` 셋째 인자), `data-player-look`, `site-header.tsx`·`types.ts`·`src/server/town.ts`·`page.tsx`의 `outfit` 줄. `town-menu.tsx` 줄은 T7로 필요 없어짐 | T3·T10·T13 |
| game | `site-header.tsx`에 🔔·`LevelUpPopup`·`AttendanceDayWatcher`, `src/server/farm.ts`의 `farm_grown` → `addLedgerEntry`, 성장 아이템으로 다 키우는 Action도 `revalidatePath("/", "layout")` | T10·T11 |
| auth | 헤더 회원 영역에 `SessionKeeper` 한 줄, 로그아웃 버튼 44px(auth 소유 컴포넌트) | T10 |
| post | `savePost`가 `growForPost(tx, 나)`를 계속 부른다 (모양 그대로) | T11 |

### 이 기능이 다른 spec 화면에 기대는 spec 요구

- US7(도감·전시, FR-051·052)은 blog가 구현한다 (`specs/002-blog/contracts/profile-showcase.md`). town은 T1과 그림을 준다.
- FR-036(💛 이웃 새 글 순서)은 social이 구현한다.
- FR-009(글쓰기·꾸미기는 블로그 홈 버튼, 내역은 헤더 코인)는 blog·game 화면이 이미 만든다.

## 남은 문제

spec을 고치지 않았다. 아래는 팀 결정이나 spec 보강이 필요하다.

1. **휴대폰 광장** (R-01): spec(FR-012, SC-003)을 따라 간단 메뉴를 없앤다. 10/6 회의 결정(코드 주석·PR #41에만 있음)과 반대라 팀 확인이 필요하다. 유지로 정하면 T7을 빼고, `town-menu.tsx`에 환영 쿠키(T6)·상점 부제(T13)·shop `outfit` 줄을 함께 고친다(research R-01). 그때는 SC-003을 지킬 수 없어 spec 수정도 필요하다.
2. **출석 도장 문구** (R-13): GAME FR-028(`오늘 N일차 ✅`)과 TOWN FR-023(`(오늘 완료 ✅)`, `출석 체크 (오늘 완료)`)이 다르다. plan은 TOWN FR-023을 따르고 일차 값도 넘겨 둔다.
3. **환영 문구 끝 문장**: spec이 "팀이 확정"으로 남겼다. 지금 코드 문장 `왼쪽 **동물 농장**에서 첫 알을 받아 동물을 키워 보세요.`를 임시로 쓴다.
4. **spec에 없는 사용자 문구** (constitution III): 이웃 아닌 대상 즐겨찾기 거부 `이웃으로 추가한 블로그만 즐겨찾기할 수 있어요`, 성장 아이템 없음 `가지고 있는 성장 아이템이 없어요`, 사용 성공 `🌱 {아이템 이름} 사용! 성장 +{N}`, 성장 아이템 안내 `상점에서 성장 아이템을 살 수 있어요`, 지붕 색 거부 `고를 수 없는 색이에요`, 상점 부제 `아바타·가구·배경·성장 아이템`, 내 이웃 목록 제목 `🏘 내 이웃 ⭐ N / 10`, 농장 안내 문장 `상점의 성장 아이템으로도 자라요`·도감 링크 `내 블로그 도감에서도 볼 수 있어요`, 버튼·상태창 접근 이름 — 모두 *(plan 임시)*. spec에 넣어 확정해야 한다.
5. **광장 최소 높이와 가로 휴대폰** (R-22): FR-005의 "최소 420px"과 "페이지 스크롤 없음"을 높이 375px쯤의 화면에서 함께 지킬 수 없다. plan은 420px을 유지한다.
6. **프로필 사진 올리기 화면 담당 없음** (spec 사이 공통): 헤더는 사진이 있으면 보여 주지만 올릴 곳이 없어 지금은 늘 캐릭터 얼굴이다.
7. **`Lv.N` 위치 표현**: game FR-010은 "헤더(유저 상태창)에", town 기본값은 "상태창 밖에 따로". plan은 town 기본값(헤더 소유 spec)을 따른다. game spec 문구 정리가 필요하다.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| 원칙 V "규칙은 DB 제약으로도 막는다" — 즐겨찾기 최대 10명을 DB 제약 없이 `toggleFavorite`가 `lockUser(나)` 트랜잭션 안에서 세어 지킨다 | "회원마다 `is_favorite = true` 행 10개까지"는 행 개수 규칙이라 CHECK·UNIQUE로 표현할 수 없다. 같은 회원의 모든 즐겨찾기 변경은 이 Action 하나를 지나고 같은 advisory lock을 잡으므로 동시 요청에도 10을 넘지 않는다(SC-006). ERD 3.11의 "한 번에 5마리"와 같은 방식이고, 원본 TOWN-08과 social 계약도 이 방식으로 정했다 | 트리거로 개수 검사: 이 저장소에 트리거 선례가 없고 규칙이 DB 안에 숨어 앱 문구(`즐겨찾기할 이웃은 최대 10명이에요`)와 따로 관리된다. 별도 `favorites` 표에 자리 번호 1~10 + UNIQUE: 이웃 취소 때 함께 지울 연결이 하나 더 생기고(즐겨찾기는 이웃 관계의 속성, ERD 3.17) social의 `follows` 구조와 겹친다 |
