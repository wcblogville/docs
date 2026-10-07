# Implementation Plan: 상점 / 꾸미기 (SHOP)

**Branch**: `006-shop` | **Date**: 2026-10-07 | **Spec**: [specs/006-shop/spec.md](spec.md)

**Input**: Feature specification from `/specs/006-shop/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

> 기준 코드: 코드 저장소 `main` `feb4c05` (2026-10-07). 코드 위치는 코드 저장소 기준 상대 경로, 문서 위치는 문서 저장소 기준 상대 경로로 적는다.
> 공통 약속(테이블 담당, 공통 모듈 소유, 구현 순서)은 7개 spec이 함께 쓰는 plan 공통 맥락을 따른다. 이 spec이 담당하는 테이블은 `items`, `user_items`와 새 표 `avatar_equips`, `room_furniture`, 열거형은 `item_type`과 새 `avatar_slot`, 시드는 `scripts/seed.ts`의 `ITEMS`다.

## Summary

상점은 캐릭터 판매를 멈추고 **아바타 꾸미기(모자·옷·소품)·가구·배경·성장 아이템**을 판다. 꾸미기 화면은 지금의 캐릭터·배경 장착 위에 **부위별 아바타 착용**과 **미니룸 가구 배치(최대 5개)**를 더한다 (SHOP-01~06, clarify D12·D13).

접근 방식:

- **데이터**: `item_type`에 `avatar`·`growth`를 더하고 `items`에 `avatar_slot`(모자·옷·소품), `growth_value`(성장치), `is_on_sale`(판매 중)을 둔다. `user_items.quantity`로 성장 아이템을 여러 개 갖게 한다. 착용은 `avatar_equips`(회원·부위마다 한 줄), 배치는 `room_furniture`(가구마다 한 줄, 자리 번호 1~5)로 나누고, 둘 다 `user_items`를 가리키는 **복합 외래 키**로 "가진 것만"을 DB가 막는다 (ERD 3.4와 같은 방식). 판매 중단 캐릭터는 행을 지우지 않고 `is_on_sale = false`로만 바꾼다 (D12).
- **구매**: 지금 `buyItem`의 "회원 잠금 → 원장 합계 → 지급 + 원장 기록" 트랜잭션을 그대로 쓰고, 판매 여부·보유(꾸미기만)·레벨·코인 순서로 다시 확인한다. 성장 아이템은 `ON CONFLICT … DO UPDATE quantity + 1`로 한 개씩 늘린다. 잔액 컬럼은 만들지 않는다 (constitution V).
- **화면**: 상점은 4구역(`👕 아바타 꾸미기`·`🪑 가구`·`🖼 배경`·`🌱 성장 아이템`)과 정렬 6가지를 클라이언트 상태로 보여 준다. 정렬·버튼 상태는 순수 함수(`src/lib/shop.ts`)로 두고 단위 테스트한다. 꾸미기는 낙관적 갱신 + 실패 시 되돌리기(지금 방식)를 착용·배치에도 쓰고, 가구는 라이브러리 없이 Pointer Events로 끌어다 놓는다.
- **그림**: 외부 그림 없이 `src/lib/art/`에 코드 SVG를 더한다. 캐릭터 SVG 위에 몸 → 옷 → 소품 → 모자 순서로 겹쳐 그리고(FR-040), 같은 함수로 헤더·광장·블로그 미니룸을 그려 세 곳이 같은 모습이 되게 한다 (SC-006).
- **경계**: 성장 아이템을 쓰는 동작(TOWN-09)은 town이 만들고, shop은 수량을 줄이는 서버 도우미만 제공한다. 차림·가구를 다른 spec의 화면(헤더·광장·블로그 홈)에 보이게 하는 일은 "공통 모듈 추가"(필드·prop 추가)로 한다.

## Technical Context

**Language/Version**: TypeScript ^5 (`strict: true`, 별칭 `@/*` → `src/*`), Node.js 20.9 이상 (README)

**Primary Dependencies**: `next` 16.3.8 (App Router, Server Component, Server Action), `react`/`react-dom` 19.2.8 (`useTransition`, `useState`), `drizzle-orm` ^0.45.3 / `drizzle-kit` ^0.31.11, `zod` ^4.6.5 (Server Action 입력 검증), `better-auth` ^1.7.7 (`requireMember()`로만 씀), `phaser` ^4.2.1 (광장 캐릭터 텍스처), `tailwindcss` ^4. **새 의존성 없음** (끌어다 놓기 라이브러리를 쓰지 않는다, research R11)

**Storage**: PostgreSQL (README 설치 안내 17, `pg` ^8.23.1). 담당 표 `items`·`user_items` 변경, 새 표 `avatar_equips`·`room_furniture`. 코인은 `point_ledger` 합계(`getWallet`, `src/server/points.ts`). 그림은 DB에 `asset_key`만, 실제 모양은 `src/lib/art/`의 SVG

**Testing**: `npm test`(= `test:game && test:ids && test:sanitize`)에 새 `test:shop`(`scripts/test-shop.ts`, 정렬·버튼 상태·문구·차림 겹침 순서·가구 자리 번호)을 더한다. E2E는 Playwright `chromium`을 쓰는 Node 스크립트: 새 `e2e/shop.mjs`, `e2e/closet.mjs`, 고칠 `e2e/game.mjs`·`e2e/decisions.mjs`, 시험을 더할 `e2e/params.mjs`(상점·꾸미기 Server Action 범위 밖 인자), 다시 돌릴 `e2e/nonfunctional.mjs`·`e2e/mobile.mjs`

**Target Platform**: 웹 브라우저(PC 마우스·키보드, 375px 휴대폰 터치) + Node 서버(`next dev` / `next start`). 배포 환경은 미정(NF-08)

**Project Type**: web-service (Next.js 풀스택 단일 프로젝트, `src/app` 화면·Server Action + `src/server` DB 처리)

**Performance Goals**: 꾸미기에서 누르면 1초 안에 미니룸이 바뀌어 보인다(SC-005, 낙관적 갱신이라 서버 왕복을 기다리지 않음). 저장 문구 2초. 상점·꾸미기 화면은 글 목록 기준(1초, 배포 환경·캐시 없는 첫 방문)을 넘지 않게 쿼리를 화면당 3~4개(아이템 목록 1 + 가진 회원 수 집계 1 + 지갑 1 + 꾸미기 상태 1)로 둔다

**Constraints**: 잔액 컬럼 없음, 지급·차감은 `lockUser` 트랜잭션 안(constitution V). "가진 것만 장착·착용·배치", "같은 꾸미기 아이템 하나", "부위마다 하나", "가구 5개", "수량 0 이상"은 DB 제약으로도 막는다. 외부 그림 파일 금지(코드 SVG). 아이소메트릭(TOWN-05) 전까지 2D. `ALTER TYPE … ADD VALUE`로 더한 열거형 값은 같은 트랜잭션에서 쓸 수 없으므로 마이그레이션의 CHECK는 `type::text` 비교로 쓴다 (research R2). `node_modules`가 없는 체크아웃이라 Next.js 16·Drizzle 세부 API는 구현 전에 설치된 패키지 문서로 다시 확인한다

**Scale/Scope**: 아이템 카탈로그 34행(지금 17 + 첫 출시 17: 아바타 9·가구 5·성장 3). 회원 규모는 팀 프로젝트 수준(수백~천 명, 추측)이라 "가진 회원 수"는 화면을 열 때 `GROUP BY` 한 번으로 계산한다. 화면 2개(`/shop`, `/closet`), Server Action 5개, 새 표 2개, User Story 7개, FR 49개

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 원칙 | 설계 전 (Phase 0 전) | 근거 | 설계 후 재평가 (Phase 1 뒤) |
|---|---|---|---|
| I. 글쓰기가 먼저 | 통과 | 상점은 활동(글·출석·댓글·공감·농장)으로 모은 코인을 쓰는 곳이다. 코인을 파는 기능·유료 결제가 없다 (spec Assumptions). 꾸미기는 글쓰기·읽기를 막지 않는다 | 통과. 새 원장 사유를 만들지 않고 `purchase`만 쓴다. 성장 아이템 사용에 경험치를 주지 않는다(TOWN 기본값) |
| II. 요구사항 ID 유지 | 통과 | SHOP-01~06과 NF-02·04·06·15·16·18·19를 그대로 가리킨다 | 통과. 마이그레이션 주석·코드 주석·e2e `check` 이름에 SHOP ID를 단다 (contracts, quickstart) |
| III. 확인할 수 있는 수용 기준 | 통과 (빈칸 있음) | 화면 문구는 spec 문구를 그대로 쓴다. spec에 없는 문구(벗기·가구 놓기 성공 문구, 상점 안내, 성장 아이템 개수 표시)는 지어내지 않고 기본값으로 표시한 뒤 열린 항목으로 넘긴다 | 통과. research R19에 기본값과 근거를 모았고, quickstart가 모든 수용 시나리오·SC를 검증 위치에 연결한다 |
| IV. 권한과 검증은 서버에서 (NON-NEGOTIABLE) | 통과 | 모든 페이지·Server Action이 `requireMember()`를 부른다. 아이템 ID는 `parseId()`, 좌표는 zod로 다시 검증한다. 판매 여부·레벨·잔액·보유·종류를 서버가 그 순간의 값으로 판단한다 | 통과. 지금 `buyItem`·`equipItem`이 `parseId`를 거치지 않는 빈틈(범위 밖 ID → DB 오류)을 함께 막는다 (contracts) |
| V. 원장 무결성 (NON-NEGOTIABLE) | 통과 | 구매 = `lockUser` → `getWallet(tx)` → 지급 → `point_ledger` `−가격`을 한 트랜잭션으로. 잔액 컬럼 없음. 구조 변경은 마이그레이션 파일로만 | 통과. DB 제약: `user_items` PK(같은 꾸미기 아이템 하나), `quantity >= 0` CHECK, `avatar_equips` PK(부위마다 하나) + 복합 FK(가진 것만), `room_furniture` PK·UNIQUE(`user_id`, `slot`)·`slot` 1~5 CHECK(가구 5개) + 복합 FK. 판매 중단은 데이터 마이그레이션으로. "그 아이템이 그 부위의 아바타인지·가구인지" 같은 종류 일치는 ERD 3.4 선례(캐릭터 칸의 종류 검사)대로 서버 코드가 확인한다 (R6) |
| VI. 모바일 | 통과 | 카드 칸 수(상점 2/3/4, 꾸미기 3/4/6)는 지금 코드와 같다. 375px 가로 스크롤 없음, 버튼 44×44px | 통과. 지금 상점 [사기] 버튼(`py-1.5 text-sm`, 약 32px)을 44px로 키운다. 가구 끌기는 Pointer Events라 터치에서도 된다(HTML5 drag and drop은 터치에서 안 됨, R11) |
| VII. 단순하게 | 통과 | 새 의존성 없음, 개인정보 추가 없음, 그림은 코드 SVG(상업 이용 걱정 없음). 필터·정렬 기억·수량 골라 사기는 넣지 않는다 (spec 기본값) | 통과. 새 표 2개는 spec Key Entities(아바타 착용 상태, 가구 배치)가 요구한다. ERD 5장의 "profiles 컬럼 3개 vs 별도 표"는 별도 표로 정해 auth 표를 건드리지 않는다 |

**결과**: 위반 없음. Complexity Tracking은 비워 둔다.

## Project Structure

### Documentation (this feature)

```text
specs/006-shop/
├── spec.md                    # 기능 명세 (clarify 2026-10-07 반영, 고치지 않음)
├── checklists/
│   └── requirements.md        # spec 품질 점검
├── plan.md                    # 이 파일 (/speckit-plan)
├── research.md                # Phase 0: 기술 결정 R1~R20
├── data-model.md              # Phase 1: 엔터티·현재/목표·마이그레이션·백필·시드
├── quickstart.md              # Phase 1: 실행·검증 시나리오 (수용 시나리오·SC 연결)
├── contracts/
│   ├── shop.md                # /shop 화면, buyItem
│   ├── closet.md              # /closet 화면, equipItem·unequipAvatar·placeFurniture·removeFurniture
│   └── shared-modules.md      # 다른 spec이 쓰는 서버 도우미·그림 함수·컴포넌트 prop
└── tasks.md                   # Phase 2 (/speckit-tasks, 아직 없음)
```

### Source Code (repository root)

코드 저장소의 실제 구조에서 이 기능이 만지는 파일만 적었다. `새`는 새 파일, `변경`은 이 spec 소유 파일의 수정, `추가(소유 spec)`는 다른 spec 소유 파일에 필드·prop·호출 한두 줄을 끼워 넣는 "공통 모듈 추가", `요청(소유 spec)`은 소유 spec의 동의가 필요한 변경이다.

```text
src/
├── db/
│   └── schema.ts                       변경: itemType(+avatar, growth), avatarSlot(새 enum), items(+avatar_slot, growth_value, is_on_sale, CHECK),
│                                             userItems(+quantity, CHECK), avatarEquips(새), roomFurniture(새)
├── app/
│   ├── shop/
│   │   ├── page.tsx                    변경: 판매 중 아이템·지갑 읽기, 안내 문구, ShopView에 넘김
│   │   ├── shop-view.tsx               새: 정렬 고르기(클라이언트 상태) + 4구역
│   │   ├── shop-grid.tsx               변경: 버튼 상태 함수, 성장 아이템 개수, 종류별 구매 문구, 44px 버튼
│   │   └── actions.ts                  변경: buyItem (parseId, 판매 여부, 보유→레벨→코인, 성장 수량 +1)
│   ├── closet/
│   │   ├── page.tsx                    변경: 보유 아이템·착용·배치 읽기, 🐾 구역 조건, 맨 아래 안내 문구
│   │   ├── closet-view.tsx             변경: 👕·🖼·🪑(·🐾) 구역, 입기·벗기, 낙관적 갱신
│   │   ├── furniture-room.tsx          새: 미니룸 위 가구 끌어다 놓기·옮기기·빼기 (Pointer Events), Enter로 기본 자리에 놓기·빼기
│   │   └── actions.ts                  변경: equipItem(+avatar), 새 unequipAvatar·placeFurniture·removeFurniture
│   ├── town/page.tsx                   추가(town): player에 outfit 넘기기
│   └── blog/[slug]/page.tsx            추가(blog): getMiniRoomDecor(ownerId) 호출 → BlogHeader
├── components/
│   ├── item-art.tsx                    변경: avatar(회색 몸 위에 겹침)·furniture·growth 그림
│   ├── character.tsx                   변경: CharacterBadge·CharacterArt에 outfit prop (그리기는 shop 소유)
│   │                                   추가(blog): MiniRoom에 outfit·furniture·children prop (MiniRoom은 blog 소유)
│   ├── site-header.tsx                 추가(town): CharacterBadge에 outfit 넘기기
│   ├── blog/blog-header.tsx            추가(blog): outfit·furniture를 MiniRoom에 넘기기
│   └── town/
│       ├── types.ts                    추가(town): TownHouse·player에 outfit 필드
│       ├── town-menu.tsx               추가(town): CharacterBadge에 outfit 넘기기
│       └── scene.ts                    요청(town): 캐릭터 텍스처 키·그림에 outfit 포함 (lookKey, characterDataUri 셋째 인자)
├── lib/
│   ├── shop.ts                         새: 구역, 정렬 6가지, 버튼 상태, 부족 코인, 구매 문구, 가구 상한·빈 자리 (순수 함수)
│   ├── assets.ts                       추가: 새 그림 함수 다시 내보내기 (기존 내보내기 유지. 소유 표에 없는 파일이고 shop 그림만 다시 내보낸다)
│   └── art/
│       ├── characters.ts               변경: characterSvg/characterDataUri(asset, size, outfit), lookKey
│       ├── avatar.ts                   새: 모자 3·옷 3·소품 3 겹침 SVG, 겹침 순서
│       ├── furniture.ts                새: 화분·의자·램프·책상·침대 SVG
│       └── growth.ts                   새: 동물 먹이·고급 먹이·성장 촉진제 SVG
└── server/
    ├── inventory.ts                    변경: listShopItems, listOwnedItems, getEquipped(+착용·배치), getMiniRoomDecor,
    │                                         outfitOf(SQL 조각), consumeGrowthItem, listItemsUnlockedBetween
    │                                         (listItemsWithOwnership은 상점·꾸미기만 쓰므로 새 함수로 바꾼 뒤 지운다)
    ├── dal.ts                          추가(auth): getViewer.profile에 outfit 필드
    └── town.ts                         추가(town): getTownHouses·getMyHouse에 outfit 필드
scripts/
├── seed.ts                             변경: ITEMS에 첫 출시 17종, isOnSale·avatarSlot·growthValue, 판매 중단 캐릭터 9종 유지
└── test-shop.ts                        새: npm run test:shop
e2e/
├── shop.mjs                            새: 구매·레벨·성장 아이템·정렬·동시 요청·조작 요청·첫 출시 목록
├── closet.mjs                          새: 장착·착용·가구 배치·실패 되돌리기·세 곳 같은 모습·375px
├── game.mjs                            변경: 상점 잠김 확인을 토끼 → 눈 마을로 (상점·꾸미기 부분만, 출석 부분은 game)
├── decisions.mjs                       변경: 꾸미기 🐾 구역이 캐릭터 2개 이상일 때만 보이는 것에 맞춰 GAME-01 확인 방법 변경
└── params.mjs                          추가: buyItem·equipItem·unequipAvatar·placeFurniture·removeFurniture에 범위 밖·글자 인자 재생 (500 없음)
drizzle/
├── NNNN_shop_items.sql                 새(생성): item_type 값, avatar_slot, items·user_items 컬럼·CHECK
├── NNNN_shop_stop_character_sales.sql  새(직접 쓴 SQL): 캐릭터·기본 아이템 is_on_sale = false
├── NNNN_avatar_equips.sql              새(생성): 아바타 착용 표
├── NNNN_room_furniture.sql             새(생성): 가구 배치 표
└── meta/                               생성된 스냅숏
docs/
├── 02-erd.md                           변경: items·user_items·새 표 2개 (1장, 2장, 3.4, 3.7, 3.11, 3.14, 3.15, 3.17, 5장, 7장, 부록)
└── erdcloud-import.sql                 변경: 새 표 2개·items 컬럼 (팀 확인 뒤, research R20)
package.json                            변경: test:shop 추가, test 체인 끝에 붙이기
README.md                               변경: 스크립트 표에 test:shop, e2e/shop.mjs, e2e/closet.mjs 한 줄씩
```

`NNNN`은 구현할 때의 다음 번호다(지금 마지막은 `0006_blog_visits`). 다른 spec의 마이그레이션이 먼저 merge되면 최신 `main`에서 `npm run db:generate`를 다시 돌려 번호와 `drizzle/meta/` 스냅숏 충돌을 피한다.

**Structure Decision**: 코드 저장소의 기존 단일 Next.js 구조(`src/app` 화면·Server Action, `src/server` DB 처리, `src/lib` 순수 규칙·그림, `scripts` 시드·단위 테스트, `e2e` 시나리오, `drizzle` 마이그레이션)를 그대로 쓴다. 새 Route Handler는 없다(요청이 작아 Server Action으로 충분). 새 서버 쿼리는 shop 소유 `src/server/inventory.ts`에, 새 순수 규칙은 `src/lib/shop.ts`에 둔다.

## 변경 단위 (현재 코드 → spec 목표)

| # | 종류 | 지금 코드 | 목표 | 근거 | Story |
|---|---|---|---|---|---|
| 1 | 마이그레이션 | `item_type` = character/background/furniture. `items`에 판매 여부·성장치·부위 없음. `user_items`에 수량 없음 | `item_type` += `avatar`, `growth`. 새 enum `avatar_slot`(hat/outfit/accessory). `items.avatar_slot`·`growth_value`·`is_on_sale` + CHECK. `user_items.quantity`(기본 1, ≥0) | FR-003, FR-013, FR-042, ERD 3.11·7장 6-2 | US2, US5 |
| 2 | 데이터 마이그레이션 | 캐릭터 9종이 판매 중 (`is_starter = false`) | 캐릭터와 기본 아이템은 `is_on_sale = false`. 행·보유·장착은 그대로 | FR-004, FR-011, D12, SC-010 | US1, US2 |
| 3 | 시드 | 캐릭터 11·배경 6 | + 모자 3·옷 3·소품 3·가구 5·성장 3 (FR-013 값), 모든 행에 `isOnSale` | FR-013, SC-011, D13 | US2, US5 |
| 4 | 서버 | `buyItem`: `parseId` 없음, `isStarter`만 거부, 보유는 PK 위반으로만 앎 | `parseId` → 판매 여부 → (꾸미기) 보유 → 레벨 → 코인 → 지급(성장은 수량 +1) + 원장 `purchase`(`ref_id` = 아이템 ID) | FR-007~010, FR-018~020, FR-042·045, FR-049 | US2, US3, US5 |
| 5 | 서버 | `listItemsWithOwnership`: 전체 아이템 + 보유 여부, 고정 정렬 | `listShopItems`: 판매 중만, 내 수량, 가진 회원 수 / `listOwnedItems`: 수량 1 이상 | FR-004, FR-015, FR-043 | US2, US6 |
| 6 | 화면 | 상점 `🐾 캐릭터`·`🖼 배경` 2구역, 문구 고정, 버튼 약 32px | 4구역 + 정렬 6가지(기본 레벨순, 기억 안 함), 버튼 상태 함수(`🔒 Lv.N`, `Lv.N+`), 성장 개수, 종류별 구매 문구, 44px | FR-002~006, FR-008, FR-014~017, FR-043, FR-046~048 | US2, US3, US5, US6 |
| 7 | 서버·화면 | 꾸미기 `🐾 내 캐릭터`·`🖼 내 배경`, 안내 `더 많은 캐릭터와 배경은…` | `👕`·`🖼`·`🪑` 구역(캐릭터 2개 이상이면 `🐾`도), 안내 `더 많은 꾸미기 아이템과 배경은 상점에서 만날 수 있어요.` | FR-021~030 | US1 |
| 8 | 마이그레이션 | 착용 저장 곳 없음 | 새 표 `avatar_equips` (PK `user_id`·`slot`, 복합 FK → `user_items`) | FR-026, FR-039, ERD 5장 | US4 |
| 9 | 서버 | `equipItem`: 캐릭터·배경만 | + 아바타 입기(부위 upsert), 새 `unequipAvatar`(벗기). `outfitOf` SQL 조각 | FR-038~041 | US4 |
| 10 | 그림 | 캐릭터 SVG에 겹칠 것 없음 | `src/lib/art/avatar.ts` + `characterSvg(asset, size, outfit)` 몸 → 옷 → 소품 → 모자 | FR-040 | US4 |
| 11 | 반영 (공통 모듈 추가) | 헤더·광장·블로그 미니룸이 `characterAsset` 하나만 받음 | `outfit` 필드·prop을 더해 세 곳 모두 같은 함수로 그림 | FR-029, FR-040, SC-006 | US1, US4 |
| 12 | 마이그레이션 | 가구 배치 저장 곳 없음 | 새 표 `room_furniture` (PK `user_id`·`item_id`, UNIQUE `user_id`·`slot`, `slot` 1~5, `x`·`y` 0~100, 복합 FK) | FR-032~035 | US7 |
| 13 | 서버 | 가구는 `아직 장착할 수 없는 종류예요` | `placeFurniture`(놓기·옮기기), `removeFurniture`(빼기), `getMiniRoomDecor` | FR-031~036 | US7 |
| 14 | 화면 | 미니룸에 가구 층 없음 | 꾸미기: 끌어다 놓기·옮기기·밖으로 빼기, 키보드는 Enter로 기본 자리에 놓기·빼기 / 블로그 홈: 같은 비율 위치로 표시 | FR-032, FR-035~036, FR-047 | US7 |
| 15 | 서버 도우미 | 수량 개념 없음 | `consumeGrowthItem(tx, userId, itemId)`: 수량 1 줄이고 성장치 돌려줌 (town이 농장에서 부름) | FR-044, SC-009 | US5 |
| 16 | 검증 | `e2e/game.mjs`가 토끼 잠김, `e2e/decisions.mjs`가 🐾 구역을 읽음, `e2e/params.mjs`에 상점·꾸미기 인자 없음 | `test:shop`, `e2e/shop.mjs`, `e2e/closet.mjs` 새로, `e2e/game.mjs`·`e2e/decisions.mjs` 고치기, `e2e/params.mjs`에 상점·꾸미기 인자 시험 추가 | SC-001~011 | 전체 |
| 17 | 문서 | ERD에 `growth`·`quantity`만 ⏳, 아바타·가구 미정 | ERD·ERDCloud 가져오기 SQL·README 스크립트 표 갱신 | 테이블 담당 규칙 5 | 전체 |

지금 코드가 이미 지키고 있어 동작을 바꾸지 않는 FR (회귀만 확인): FR-012(되팔기·환불 기능 없음), FR-016(`items_required_level_check` ≥ 1), FR-021(블로그 홈 주인에게만 보이는 `🎨 꾸미기` 링크, `src/components/blog/blog-header.tsx`), FR-028(캐릭터·배경은 NOT NULL 컬럼 하나씩), FR-049의 내역 표시(`listLedger`가 `purchase` 줄에 아이템 이름을 붙임, `src/server/points.ts`). 확인 위치는 quickstart SH-11(FR-012)·SH-12(FR-016)·CL-01(FR-021)·CL-03(FR-028)·SH-03(FR-049).

구현 순서(태스크 묶음): **1·2·3 → 4·5·6(US2·3·5·6) → 7(US1) → 8·9·10·11(US4) → 15(town 전에) → 12·13·14(US7) → 16·17은 각 묶음과 함께**. 성장 아이템 사기(US5)는 town이 쓰는 기능의 선행이므로 아바타·가구보다 먼저 merge한다.

## 의존성

**이 spec이 먼저 필요로 하는 것**

| 선행 | 무엇 | 왜 | 막히는 범위 |
|---|---|---|---|
| auth (공통 맥락 5.1 단계 1) | 가입 통합(온보딩 없애기), `e2e/helpers.mjs`의 `loginDev` 고치기, `requireMember()` 의미 정리 | 새 e2e가 실행마다 새 회원을 만든다. 가입 트랜잭션이 기본 캐릭터 1종 + 초원을 `user_items`에 넣는다(수량 기본값 1이라 auth 코드 변경 필요 없음) | e2e 실행. 코드 작업은 병행 가능 |
| 없음 (DB) | - | 이 spec의 표 변경은 다른 spec 표에 기대지 않는다 | - |

**다른 spec이 이 spec에 기대는 것 (이 spec이 먼저 해야 하는 일)**

| 받는 spec | 무엇 | 인터페이스 |
|---|---|---|
| town (TOWN-09, 단계 11) | 성장 아이템 종류·성장치·수량, 수량 줄이기 | `items.type = 'growth'`, `items.growth_value`, `user_items.quantity`, `consumeGrowthItem()` (contracts/shared-modules.md) |
| town (TOWN-07, TOWN-10) | 꾸미기 화면에 `🏠 지붕 색` 칸 끼우기, 헤더 상태창의 캐릭터 얼굴에 차림 | `ClosetView` 아래 구역 자리, `CharacterBadge`의 `outfit` prop, `getViewer().profile.outfit` |
| game (GAME-06 레벨업 팝업) | "안 본 레벨 범위 L1~L2에서 살 수 있게 된 아이템" (캐릭터 제외, game research R12) | `items.is_on_sale` 또는 `listItemsUnlockedBetween(fromLevel, toLevel)` |
| blog (BLOG-04, 단계 12) | 블로그 홈 미니룸에 차림·가구, 옆에 전시 동물 | `getMiniRoomDecor(ownerId)`, `MiniRoom`의 새 prop (MiniRoom은 blog 소유라 둘이 같은 컴포넌트를 고친다) |

**다른 spec 소유 파일에 하는 일** (공통 모듈 소유 규칙)

- 공통 모듈 추가: `src/server/dal.ts`(auth) — `getViewer`가 돌려주는 `profile`에 `outfit: string[]` 필드.
- 공통 모듈 추가: `src/server/town.ts`, `src/components/town/types.ts`, `src/app/town/page.tsx`, `src/components/town/town-menu.tsx`, `src/components/site-header.tsx`(town) — `outfit` 필드를 고르고 `CharacterBadge`에 넘기는 한두 줄.
- 요청: town이 `src/components/town/scene.ts`의 `charKey()`와 텍스처 만들기에 `outfit`을 넣는 것에 동의 (함수 모양이 바뀌므로). shop이 `lookKey(asset, outfit)`와 `characterDataUri(asset, size, outfit)`을 제공하고, 같은 PR에서 town 담당이 리뷰한다. 검증용으로 `TownGame` 바깥 요소에 `data-player-look` 속성을 하나 두는 것도 함께 요청한다 (quickstart CL-07·CL-12).
- 공통 모듈 추가: `src/components/character.tsx`의 `MiniRoom`(blog) — `outfit`, `furniture`, `children` 선택 prop. 넘기지 않으면 지금과 같게 그린다.
- 공통 모듈 추가: `src/app/blog/[slug]/page.tsx`, `src/components/blog/blog-header.tsx`(blog) — `getMiniRoomDecor(blog.ownerId)`를 부르고 `MiniRoom`에 넘긴다.
- 문구 담당이 다른 곳(이 spec이 바꾸지 않음): 광장 상점 이름표 부제 `캐릭터·배경`(`src/components/town/scene.ts`, `town-menu.tsx`)은 town spec 기본값대로 town이 바꾼다. 레벨업 팝업 문구는 game.

## 남은 문제 (팀 확인 필요)

spec이 정하지 않아 기본값으로 설계한 것과 확인하지 못한 것이다. 기본값과 근거는 [research.md](research.md) R19에 있다. 정해지면 spec(문구)과 이 plan을 함께 고친다.

| # | 문제 | 지금 기본값 | 관련 |
|---|---|---|---|
| 1 | 벗기·가구 놓기·옮기기·빼기의 성공 문구가 spec에 없다 | FR-024 문구 `{아이템 이름} 장착을 저장했어요 ✓`를 그대로 | FR-024, SC-005 |
| 2 | 상점 안내 문구(FR-002)의 글자가 spec에 없다. 지금 코드 문구는 캐릭터 판매 중단과 맞지 않는다 | `글을 쓰고 출석해서 모은 코인으로 내 캐릭터와 미니룸을 꾸며 보세요.` | FR-002 |
| 3 | 성장 아이템 카드의 가진 개수 표시 형식 | `보유 N개`, 0개도 표시 | FR-043 |
| 4 | 상점 구매 중 네트워크·서버 오류 문구가 없다 (꾸미기는 FR-025가 있음) | 지금 동작 그대로(오류 화면). 코인·보유는 트랜잭션이라 바뀌지 않음 | AC 2-10 |
| 5 | 꾸미기에서 가진 것이 없는 `👕`·`🪑` 구역 | FR-022 글자대로 제목은 늘 보이고 카드 자리는 비움. 빈 구역 안내 문구는 없음 | FR-022 |
| 6 | 판매가 끝난 캐릭터(고양이 등)에도 모자·옷·소품을 입힐지. spec 기본값은 남자·여자 주민만 적었다 | 몸 좌표가 같아 모든 캐릭터에 허용. 뿔·귀와 모자가 겹칠 수 있다 | FR-040, SHOP-06 열린 질문 |
| 7 | 헤더에 프로필 사진을 쓰는 회원은 차림이 헤더에 보이지 않는다 (FR-040·SC-006 vs town FR-054). 프로필 사진 올리기 화면을 맡은 spec도 없다 | 사진이 없을 때만 캐릭터 얼굴에 차림 | town, auth |
| 8 | 댓글·글 카드·글 상세의 작은 캐릭터 얼굴에 차림을 그릴지 | 그리지 않음 (SC-006의 세 곳만) | R9 |
| 9 | 놓인 가구를 키보드 방향키로 옮기기 | 범위에 넣지 않음 (FR-047은 Tab·Enter로 사기·장착만 요구). Enter로 기본 자리에 놓기·빼기는 함 | R11, FR-047 |
| 10 | 새 아이템 17종의 설명(`description`) 글자 | 구현 때 카드 2줄 안으로 쓰고 PR에서 확인 | data-model 8장 |
| 11 | 좌표 조작 요청의 `잘못된 요청이에요`는 spec에 없는 기존 코드 관례 문구 | 그대로 씀 | R19 |
| 12 | `docs/erdcloud-import.sql`에 새 표 2개·컬럼을 더할지 | 더한다 | R20 |
| 13 | SC-009의 "농장에서 M번 쓰면 M 줄어든다"는 town FR-048(사용 화면)이 생겨야 검증된다 | 그 전까지 e2e가 DB로 수량을 바꿔 대신 확인 | quickstart SH-17 |
| 14 | `node_modules`가 없어 확인하지 못한 라이브러리 동작: drizzle 마이그레이터의 한 트랜잭션 실행, `drizzle-kit generate --custom --name`, Drizzle `real()`·`smallint()`·복합 target `onConflictDoUpdate`, node-postgres의 `text[]` 변환, Next.js 16 `revalidatePath`·Server Action 인자 직렬화 | 구현 전에 설치된 패키지 문서로 확인 | R2, R4, R8, R10, R17 |

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

해당 없음. Constitution Check에 위반이 없다.
