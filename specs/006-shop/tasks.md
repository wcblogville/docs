---

description: "상점 / 꾸미기 (SHOP) 구현 태스크 목록"
---

# Tasks: 상점 / 꾸미기 (SHOP)

**Input**: Design documents from `/specs/006-shop/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: plan.md Technical Context와 quickstart.md가 단위 테스트(`npm run test:shop`, `scripts/test-shop.ts`)와 e2e(`e2e/shop.mjs`, `e2e/closet.mjs`, 기존 e2e 수정)를 검증 방법으로 정했으므로 각 User Story에 테스트 태스크를 넣는다. 이 저장소에는 CI가 없으므로, 작성자가 직접 돌리고 결과를 PR 설명에 적는다.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- 모든 코드 경로는 **코드 저장소 `ehgo508/blogville`** 루트 기준이다 (Next.js 풀스택 단일 프로젝트: `src/app` 화면·Server Action, `src/server` DB 처리, `src/lib` 순수 규칙·그림, `scripts` 시드·단위 테스트, `e2e` 시나리오, `drizzle` 마이그레이션).
- `NNNN`은 구현할 때의 다음 마이그레이션 번호다 (지금 마지막 `0006_blog_visits`). 다른 spec 마이그레이션이 먼저 merge되면 최신 `main`에서 `npm run db:generate`를 다시 돌린다.
- 다른 spec 소유 파일에는 "공통 모듈 추가"(필드·prop·호출 한두 줄)만 한다. `요청`으로 표시한 태스크는 소유 spec 담당의 리뷰·동의가 필요하다 (plan.md 의존성).

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 구현 전 확인과 테스트 스크립트 자리 만들기

- [ ] T001 `node_modules`에서 구현 전 확인 항목을 확인하고 결과를 PR 설명에 적는다: drizzle node-postgres 마이그레이터의 한 트랜잭션 실행(R2), `drizzle-kit generate --custom --name` 옵션, Drizzle `real()`·`smallint()`·복합 target `onConflictDoUpdate`(R4, R10), node-postgres `text[]` 변환(R8), `node_modules/next/dist/docs/`의 Server Action 인자 직렬화·`revalidatePath`(R17)
- [x] T002 [P] `scripts/test-shop.ts` 파일을 만들고(`check(이름, 참거짓)`로 `✅`/`❌` 출력, `❌`가 있으면 exit 1, 맨 위에 SHOP ID 주석) `package.json`에 `"test:shop"` 스크립트를 더하고 `test` 체인 끝(`test:game && test:ids && test:sanitize && test:shop`)에 붙인다
- [x] T003 [P] `e2e/shop.mjs`, `e2e/closet.mjs` 뼈대를 만든다: 맨 위 요구사항 ID·사용법 주석, 첫 인자 스크린샷 폴더, `pg` Pool, 실행마다 새 회원(`e2e/helpers.mjs`의 `loginDev`), `check()`·`collectErrors`, 원장 기록으로 코인·경험치를 맞추는 도우미(그날 첫 화면을 연 뒤 원장 합계를 다시 읽어 차이만큼 기록), Server Action 요청을 잡아 인자만 바꿔 다시 보내는 도우미(`e2e/params.mjs` 방식)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 모든 User Story가 기대는 스키마·데이터 이전·시드·그림·순수 규칙의 뼈대

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [x] T004 `src/db/schema.ts` 변경: `itemType`에 `avatar`, `growth`를 뒤에 더하고, 새 enum `avatarSlot`(`hat`, `outfit`, `accessory`)을 만들고, `items`에 `avatar_slot`(`avatar_slot` NULL), `growth_value`(integer NULL), `is_on_sale`(boolean NOT NULL 기본 true)과 CHECK 3개 — `items_avatar_slot_check CHECK (("type"::text = 'avatar') = ("avatar_slot" IS NOT NULL))`, `items_growth_type_check CHECK (("type"::text = 'growth') = ("growth_value" IS NOT NULL))`, `items_growth_value_check CHECK ("growth_value" IS NULL OR "growth_value" > 0)` — 를 더하고, `userItems`에 `quantity`(integer NOT NULL 기본 1) + `user_items_quantity_check CHECK (quantity >= 0)`를 더한다 (data-model 2.1·2.2·2.5, R1~R4)
- [x] T005 `npm run db:generate`로 `drizzle/NNNN_shop_items.sql`을 만들고 맨 위에 `-- SHOP-01·SHOP-06, TOWN-09 (2026-10-07): 상점 아이템 종류(아바타·성장), 판매 여부, 성장치, 보유 수량` 주석을 단다. 같은 실행에서 새 enum 값을 리터럴로 쓰지 않는지 확인한다 (R2, T004 뒤)
- [x] T006 직접 쓴 데이터 마이그레이션 `drizzle/NNNN_shop_stop_character_sales.sql`을 만든다(`npx drizzle-kit generate --custom --name=shop_stop_character_sales`): `UPDATE "items" SET "is_on_sale" = false WHERE "is_starter" OR "type" = 'character';` 하나만, SHOP-01·D12 주석. `user_items`·`profiles`·`blogs`·`point_ledger`는 건드리지 않는다 (FR-004, FR-011, SC-010, T005 뒤)
- [x] T007 `scripts/seed.ts`의 `ITEMS` 변경: 모든 행에 `isOnSale`을 두고(기본 아이템 `char_boy`·`char_girl`·`bg_meadow`와 과거 캐릭터 9종 `char_human`~`char_unicorn`은 `false`, 행 유지), 첫 출시 17종을 data-model 8장 표대로 넣는다 — `hat_straw` 밀짚모자 50·Lv.1, `hat_ribbon` 리본 60·Lv.1, `hat_beanie` 털모자 80·Lv.2, `outfit_overalls` 멜빵바지 100·Lv.1, `outfit_hoodie` 후드티 150·Lv.2, `outfit_dress` 원피스 200·Lv.3, `acc_glasses` 안경 70·Lv.1, `acc_scarf` 목도리 120·Lv.2, `acc_bag` 가방 180·Lv.3 (`avatarSlot` 각각 hat/outfit/accessory), `furn_pot` 화분 60·Lv.1, `furn_chair` 의자 80·Lv.1, `furn_lamp` 램프 120·Lv.2, `furn_desk` 책상 150·Lv.2, `furn_bed` 침대 250·Lv.3, `growth_feed` 동물 먹이 20·Lv.1·성장 20, `growth_premium` 고급 먹이 60·Lv.1·성장 50, `growth_booster` 성장 촉진제 150·Lv.3·성장 100 (`asset_key`는 표의 `hat.straw` 등). `onConflictDoUpdate`의 `set`에 `avatarSlot`, `growthValue`, `isOnSale`을 더한다. 새 17종 `description`은 카드 2줄 안으로 쓴다 (FR-013, SC-011, T004 뒤)
- [x] T008 [P] `src/lib/art/avatar.ts` 새로: 모자 3·옷 3·소품 3(`hat.straw` … `acc.bag`) 겹침 SVG를 담은 `AVATAR_PARTS`(`asset_key → { slot, svg }`)와 겹치는 순서(옷 → 소품 → 모자)로 정렬하는 `orderOutfit(outfit)` (FR-040, R8, R15)
- [x] T009 [P] `src/lib/art/furniture.ts` 새로: 화분·의자·램프·책상·침대(`furn.pot` … `furn.bed`) `furnitureSvg(assetKey)`와 가구마다 미니룸 높이 대비 크기 (R12, R15)
- [x] T010 [P] `src/lib/art/growth.ts` 새로: `growth.feed`, `growth.premium`, `growth.booster`의 `growthSvg(assetKey)` (R15)
- [x] T011 `src/lib/assets.ts`에서 T008~T010의 그림 함수를 다시 내보낸다(기존 내보내기 유지)
- [x] T012 `src/components/item-art.tsx` 변경: `type`이 `avatar`(회색 몸 위에 겹침), `furniture`, `growth`일 때 그림을 그린다 (T011 뒤)
- [x] T013 `src/lib/shop.ts` 새로(DB 없음, 화면·서버 공용): `SHOP_SECTIONS` = `[{ type: "avatar", title: "👕 아바타 꾸미기" }, { type: "furniture", title: "🪑 가구" }, { type: "background", title: "🖼 배경" }, { type: "growth", title: "🌱 성장 아이템" }]`과 공용 타입만 먼저 둔다 (FR-003, contracts/shared-modules.md 2장)
- [x] T014 [P] `scripts/test-shop.ts`에 구역 묶음 테스트: 종류 → 구역 제목, `character`는 구역 없음 (FR-003, T013 뒤)
- [ ] T015 quickstart 1장(S-00) 확인: 기존 데이터가 있는 로컬 DB에서 `npm run db:migrate && npm run db:seed` 뒤 캐릭터 11행 모두 `is_on_sale = false`, 판매 중 22행(avatar 9·furniture 5·background 5·growth 3), `quantity <> 1` 0행, `profiles.character_item_id`·`blogs.background_item_id`·회원별 원장 합계 변화 없음, `db:migrate` 재실행 시 "적용할 것 없음"

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - 가진 캐릭터와 배경 장착하기 (Priority: P1) 🎯 MVP

**Goal**: 꾸미기 화면에서 가진 캐릭터·배경을 눌러 장착하고, 낙관적 갱신·실패 되돌리기·가진 것만 장착을 지킨다. 새 구역 구성(`👕`·`🖼`·`🪑`, 캐릭터 2개 이상이면 `🐾`)과 맨 아래 안내 문구를 반영한다 (SHOP-04, FR-021~030, FR-011).

**Independent Test**: 상점 없이, 기본 캐릭터·초원과 DB로 지급한 바닷가를 가진 회원으로 `/closet`에서 바닷가를 장착하고 블로그 홈 미니룸이 바뀌는지 확인한다 (quickstart CL-01~08).

### Tests for User Story 1

- [ ] T016 [P] [US1] `e2e/closet.mjs`에 CL-01~CL-08 시나리오: 블로그 홈 `🎨 꾸미기`로 진입·바닷가 장착·`바닷가 장착을 저장했어요 ✓` 2초 뒤 사라짐·블로그 미니룸 반영(CL-01), `✓`·강조·`aria-pressed="true"`와 구역 표시 조건(CL-02), 배경 하나만(CL-03), 미보유·범위 밖 ID `가지고 있지 않은 아이템이에요`(CL-04), 동물 먹이 `아직 장착할 수 없는 종류예요`(CL-05), `page.route`로 요청을 끊어 되돌리기와 `장착하지 못했어요. 잠시 뒤 다시 시도해 주세요`(CL-06), 광장 캐릭터·지붕 색(CL-07), 판매 중단 고양이 보유·장착 유지와 환불 기록 없음(CL-08), 로그인하지 않으면 `/`로 이동(CL-23)
- [x] T017 [P] [US1] `e2e/decisions.mjs` 변경: `GAME-01` 확인을 꾸미기 `🐾` 구역 `✓` 대신 미니룸 `data-look` 또는 `profiles.character_item_id`로 바꾸고, `SHOP-04` 고양이 지급·장착 확인은 그대로 둔다 (quickstart 5.3)

### Implementation for User Story 1

- [x] T018 [US1] `src/server/inventory.ts`에 `listOwnedItems(userId)`를 더한다: `quantity > 0`, 성장 아이템 제외, 판매 중단 아이템 포함(D12). `OwnedItem` 타입을 내보낸다 (contracts/closet.md 1.1)
- [x] T019 [US1] `src/server/inventory.ts`의 `getEquipped(userId)`를 `{ characterItemId; backgroundItemId; avatar: {}; furniture: [] }` 모양으로 넓힌다(착용·배치는 US4·US7에서 채움) (T018과 같은 파일, T018 뒤)
- [x] T020 [US1] `src/app/closet/actions.ts`의 `equipItem` 변경: `parseId()`를 거치고(null이면 `가지고 있지 않은 아이템이에요`), 보유는 `user_items.quantity > 0`으로 확인하며, 종류가 character → `profiles.character_item_id`, background → `blogs.background_item_id`, 그 밖(furniture, growth) → `아직 장착할 수 없는 종류예요`. `ClosetResult` = `{ ok: true; name } | { ok: false; error }`, 성공 뒤 `revalidatePath("/", "layout")` (FR-026~028, contracts/closet.md 2.1)
- [x] T021 [US1] `src/app/closet/page.tsx` 변경: `requireMember()`, `listOwnedItems`·`getEquipped`·지갑 읽기, 가진 캐릭터가 2개 이상일 때만 `🐾 내 캐릭터` 구역(가입 때 고른 캐릭터 포함), 맨 아래 안내 문구 `더 많은 꾸미기 아이템과 배경은 상점에서 만날 수 있어요.` (FR-022, FR-030, T018·T019 뒤)
- [x] T022 [US1] `src/app/closet/closet-view.tsx` 변경: 제목 `🎨 꾸미기`·레벨 진행 막대·미니룸, 구역 `👕 아바타 꾸미기`·`🖼 내 배경`·`🪑 가구`(가진 것이 없어도 제목은 보임)·(조건부) `🐾 내 캐릭터`, 카드 칸 수 모바일 3·640px 4·1024px 6, 장착 카드 `✓` + 노란 테두리·배경 + `aria-pressed`, 낙관적 갱신 → 성공 시 `{아이템 이름} 장착을 저장했어요 ✓` 2초 / 실패 시 원래대로 되돌리고 `장착하지 못했어요. 잠시 뒤 다시 시도해 주세요`(오류 화면 아님), 마지막으로 누른 아이템 기준으로 문구 갱신 (FR-022~025, Edge)
- [x] T023 [US1] FR-021·FR-029 회귀 확인: `src/components/blog/blog-header.tsx`의 주인 전용 `🎨 꾸미기` 링크와 광장 캐릭터·지붕 색(배경 색) 반영이 그대로인지 T016 실행으로 확인한다

**Checkpoint**: User Story 1 동작 — 기본·지급 아이템만으로 장착 흐름이 완성된다 (MVP)

---

## Phase 4: User Story 2 - 코인으로 꾸미기 아이템 사기 (Priority: P2)

**Goal**: 상점 4구역에서 아바타·가구·배경을 코인으로 산다. 판매 여부·보유·코인을 그 순간의 값으로 다시 확인하고, 지급과 원장 차감을 한 트랜잭션으로 처리한다 (SHOP-01, SHOP-03, FR-001~013, FR-019~020, FR-049).

**Independent Test**: Lv.1·🪙 120 회원으로 바닷가를 사고 잔액 🪙 0, `user_items` 1행, `/wallet` `−120`을 확인한다 (quickstart SH-01~11, SH-25).

### Tests for User Story 2

- [x] T024 [P] [US2] `scripts/test-shop.ts`에 버튼 상태(`owned` → `locked` → `short` → `buy`, 코인 = 가격이면 `buy`), 부족 코인(가격 120·잔액 30 → 90), 꾸미기 구매 문구가 `🎉 {이름}을(를) 샀어요! 꾸미기에서 장착해 보세요.`와 글자까지 같은지 테스트 (FR-006, FR-008, FR-019)
- [ ] T025 [P] [US2] `e2e/shop.mjs`에 SH-01~SH-11, SH-25: 화면 제목·안내·`Lv.N 🪙 N`·4구역·기본/과거 캐릭터 카드 없음(SH-01), FR-013 목록 100% 일치(SH-02), 잔액 = 가격 구매·헤더 `🪙 0`·`/wallet` 기록(SH-03), `보유 중`·흐림(SH-04), `코인 부족`·`코인이 N개 부족해요`(SH-05), `이미 가지고 있는 아이템이에요`(SH-06), 초원·고양이·없는 ID·범위 밖·`"abc"` → `살 수 없는 아이템이에요`·500 없음(SH-07), 같은 아이템 동시 10개 → 성공 1·거부 9(SH-08), 🪙 150으로 120짜리 두 개 동시 → 하나 성공·`코인이 90개 부족해요`·잔액 30(SH-09), 반쪽 반영 없음(SH-10), 되팔기·환불 없음(SH-11), 처음 사기까지 시간(SH-25), 로그인하지 않으면 `/`로(SH-23)

### Implementation for User Story 2

- [x] T026 [P] [US2] `src/lib/shop.ts`에 `buttonState(item, { level, coins })`(`"owned" | "locked" | "short" | "buy"`, 위 상태가 우선, 성장 아이템은 `owned` 없음), `shortage(price, coins)` = `price − coins`, `purchaseMessage(kind: "decor" | "growth", name)`를 더한다 (contracts/shared-modules.md 2장)
- [x] T027 [US2] `src/server/inventory.ts`에 `listShopItems(userId)`를 더한다: `is_on_sale = true`만, 내 `quantity`(없으면 0), `ownerCount`(보유 수량 ≥ 1인 회원 수, `GROUP BY` 한 번). `ShopItem` 타입을 내보낸다 (FR-004, contracts/shop.md 1.1)
- [x] T028 [US2] `src/app/shop/actions.ts`의 `buyItem` 재작성: `BuyResult` = `{ ok: true; name; kind: "decor" | "growth"; quantity } | { ok: false; error }`, `requireMember()`, `parseId(itemId)`(null → `살 수 없는 아이템이에요`), `db.transaction` 안 `lockUser(tx, userId)` → 아이템 존재·`is_on_sale = true`·`is_starter = false`(아니면 `살 수 없는 아이템이에요`) → 꾸미기 아이템(avatar·furniture·background)이면 `quantity > 0` 보유 시 `이미 가지고 있는 아이템이에요` → `getWallet(userId, tx).level >= required_level`(레벨 확인 본체는 US3) → `coins >= price`(아니면 `코인이 {가격 − 잔액}개 부족해요`) → `INSERT user_items`(PK 위반 23505 → `이미 가지고 있는 아이템이에요`) → `point_ledger`(`reason 'purchase'`, `exp_delta 0`, `coin_delta −price`, `ref_id = String(itemId)`) → `revalidatePath("/", "layout")` (FR-007, FR-009~010, FR-019~020, FR-049, contracts/shop.md 2장)
- [x] T029 [US2] `src/app/shop/page.tsx` 변경: `requireMember()`, `listShopItems`·지갑 읽기, 제목 `🏪 마을 상점`·안내 문구 `글을 쓰고 출석해서 모은 코인으로 내 캐릭터와 미니룸을 꾸며 보세요.`·오른쪽 `Lv.N 🪙 N`, 헤더 `[← 광장으로 나가기]`, `ShopView`에 넘김 (FR-001~002, T027 뒤)
- [x] T030 [US2] `src/app/shop/shop-view.tsx` 새로: `SHOP_SECTIONS` 순서로 4구역을 그리고 구역별 카드를 `ShopGrid`에 넘긴다(정렬은 US6에서) (FR-003, T029 뒤)
- [x] T031 [US2] `src/app/shop/shop-grid.tsx` 변경: `buttonState`로 버튼 글자(`보유 중`·`🔒 Lv.N`·`코인 부족`·`사기`, `보유 중`이면 카드 흐림, 누를 수 없는 상태는 `disabled`), 카드에 그림·이름·2줄 설명·`🪙 가격`, 카드 칸 수 모바일 2·640px 3·1024px 4, [사기] 버튼 최소 44×44px·글자 한 줄, 성공 시 그 구역 위에 `purchaseMessage("decor", name)` 문구와 헤더 코인 즉시 갱신, 실패 시 서버 문구 표시 (FR-005~006, FR-008, FR-046)
- [x] T032 [US2] FR-012 회귀 확인: `/shop`·`/closet`·`/wallet`에 되팔기·환불 버튼이나 Server Action이 없음을 확인한다 (T025 SH-11)

**Checkpoint**: User Story 1·2가 각각 독립적으로 동작

---

## Phase 5: User Story 3 - 레벨이 되어야 살 수 있는 아이템 (Priority: P2)

**Goal**: 필요 레벨 이상이어야 사고, 잠긴 아이템도 목록에서 보인다. 구매 순간의 레벨로 다시 판단한다 (SHOP-02, FR-016~018).

**Independent Test**: Lv.1 회원으로 눈 마을(🪙 200, Lv.2)이 `🔒 Lv.2`로 보이고 요청이 `레벨 2부터 살 수 있어요`로 거부되는지, 경험치로 Lv.2가 된 뒤 풀리는지 확인한다 (quickstart SH-12~15).

### Tests for User Story 3

- [x] T033 [P] [US3] `scripts/test-shop.ts`에 레벨 테스트: 가졌고 레벨도 모자라면 `owned`, 레벨·코인 모두 모자라면 `locked`, 레벨 99에서도 같은 규칙 (FR-006, Edge)
- [ ] T034 [P] [US3] `e2e/shop.mjs`에 SH-12(`Lv.2+`·`🔒 Lv.2`, 필요 레벨 1이면 `Lv.N+` 없음), SH-13(`레벨 2부터 살 수 있어요`, 코인 그대로), SH-14(Lv.1 화면에서 잡은 요청을 Lv.2가 된 뒤 재생하면 성공, 새로고침하면 `사기`), SH-15(Lv.99 잠금 없음)
- [x] T035 [P] [US3] `e2e/game.mjs` 변경: 상점 잠김 확인을 `토끼` 카드 → `눈 마을` 버튼이 `🔒 Lv.2`인지 출력으로 바꾼다 (상점 부분만, quickstart 5.3)

### Implementation for User Story 3

- [x] T036 [US3] `src/app/shop/actions.ts`의 `buyItem`에서 레벨 확인을 트랜잭션 안 `getWallet(userId, tx).level`로 하고, 모자라면 `레벨 {필요 레벨}부터 살 수 있어요`로 거부하는지 확인·보완한다(확인 순서: 판매 여부 → 보유 → 레벨 → 코인) (FR-018, AC 3-4)
- [x] T037 [US3] `src/app/shop/shop-grid.tsx`에 필요 레벨이 2 이상일 때만 가격 옆 `Lv.N+`, 잠긴 버튼 `🔒 Lv.N`(누를 수 없음)을 표시한다 (FR-005, FR-017, AC 3-5)
- [x] T038 [P] [US3] `src/server/inventory.ts`에 `listItemsUnlockedBetween(fromLevel, toLevel)`를 더한다: `is_on_sale = true AND required_level BETWEEN fromLevel AND toLevel`, 필요 레벨 → 가격 → ID 순 (game GAME-06 레벨업 팝업용, contracts/shared-modules.md 1장)

**Checkpoint**: User Story 1·2·3 독립 동작

---

## Phase 6: User Story 5 - 동물 농장용 성장 아이템 사기 (Priority: P2)

**Goal**: 성장 아이템을 여러 번 사서 수량을 늘리고, 농장에서 쓸 때 줄이는 서버 도우미를 town에 제공한다 (SHOP-01 2026-10-07 결정, FR-042~045). plan.md 구현 순서상 town(TOWN-09)의 선행이므로 아바타·가구(US4·US7)보다 먼저 merge한다.

**Independent Test**: 🪙 100 회원으로 동물 먹이(🪙 20)를 세 번 사고 수량 3·잔액 🪙 40을 확인한다 (quickstart SH-16~20, SH-22).

### Tests for User Story 5

- [x] T039 [P] [US5] `scripts/test-shop.ts`에 성장 테스트: 성장 아이템은 가져도 `owned`가 아님, 성장 구매 문구가 `🎉 {이름}을(를) 샀어요! 동물 농장에서 써 보세요.`와 글자까지 같음
- [ ] T040 [P] [US5] `e2e/shop.mjs`에 SH-16(3번 사서 수량 3·잔액 40·`보유 3개`·성장 구역 위 문구), SH-17(DB로 수량 0 → `보유 0개`·다시 사면 1), SH-18(`코인이 N개 부족해요`), SH-19(동시 2개: 🪙 100이면 둘 다 성공 −40, 🪙 30이면 하나 성공·`코인이 10개 부족해요`, 늘어난 수량 × 20 = 빠진 코인), SH-20(Lv.2·🪙 200에서 성장 촉진제 `Lv.3+`·`🔒 Lv.3`), SH-22(구매 1,000번과 같은 advisory lock 보상 삽입을 섞어 원장 누적합 최솟값 ≥ 0, 헤더 코인 = 원장 합계, `quantity × 가격` = 차감 합)

### Implementation for User Story 5

- [x] T041 [US5] `src/app/shop/actions.ts`의 `buyItem`에 성장 아이템 지급을 더한다: 보유 확인을 건너뛰고 `INSERT … ON CONFLICT (user_id, item_id) DO UPDATE SET quantity = quantity + 1`, 결과에 `kind: "growth"`와 산 뒤 `quantity`. `acquired_at`은 바꾸지 않는다 (FR-042, FR-045, data-model 2.2)
- [x] T042 [US5] `src/app/shop/shop-grid.tsx`에 성장 아이템 카드의 `보유 N개`(0개도 표시)와 성장 구역 위 `purchaseMessage("growth", name)` 문구, 성공 뒤 카드 수량 갱신을 더한다 (FR-043, AC 5-2, 5-4)
- [x] T043 [P] [US5] `src/server/inventory.ts`에 `consumeGrowthItem(tx, userId, itemId)`를 더한다: 부르는 쪽이 `lockUser`를 건 트랜잭션 안에서, `type = 'growth'`이고 `quantity > 0`일 때만 1 줄이고 `{ name, growthValue }`를 돌려주며 아니면 `null`. 원장은 쓰지 않는다 (FR-044, SC-009, contracts/shared-modules.md 1장)

**Checkpoint**: 성장 아이템 구매와 수량 도우미 완성 — town(TOWN-09)이 쓸 수 있다

---

## Phase 7: User Story 6 - 상점 정렬 바꾸기 (Priority: P2)

**Goal**: 정렬 6가지(기본 레벨순, 기억하지 않음)로 모든 구역을 다시 정렬한다 (SHOP-01 정렬 결정, FR-014~015).

**Independent Test**: 가격·레벨·보유·가진 회원 수가 다른 상태에서 정렬 6가지를 차례로 고르고 구역별 순서를 확인한다 (quickstart SH-21).

### Tests for User Story 6

- [x] T044 [P] [US6] `scripts/test-shop.ts`에 정렬 테스트: 같은 고정 목록에서 `레벨순`(같으면 싼 순), `인기순`(같으면 싼 순), `비싼 순`·`싼 순`(같으면 레벨 낮은 순), `최신순`(id 큰 순), `보유순`(가진 것 먼저, 같으면 싼 순, 수량 0은 안 가진 것), 마지막 동률은 `id` 작은 순 (SC-008, R13)
- [ ] T045 [P] [US6] `e2e/shop.mjs`에 SH-21: 다른 회원 2명이 일부를 보유하게 DB 준비, 정렬 6가지마다 기대 순서를 그 순간 DB 값으로 계산해 비교, 새로고침하면 `레벨순`

### Implementation for User Story 6

- [x] T046 [US6] `src/lib/shop.ts`에 `SHOP_SORTS`(`"level" | "popular" | "priceDesc" | "priceAsc" | "newest" | "owned"`와 화면 이름 `레벨순`·`인기순`·`비싼 순`·`싼 순`·`최신순`·`보유순`, 기본 `level`)와 `sortShopItems(items, sort)`(새 배열 반환)를 더한다 (FR-014~015)
- [x] T047 [US6] `src/app/shop/shop-view.tsx`에 정렬 고르기(클라이언트 상태, 기본 `level`, 저장하지 않음)를 더하고 고르면 4구역 모두 `sortShopItems`로 다시 정렬한다. 키보드로 고를 수 있고 44px 이상 (FR-015, FR-046~047, T046 뒤)

**Checkpoint**: 상점(US2·3·5·6) 전체 완성

---

## Phase 8: User Story 4 - 산 꾸미기 아이템을 내 캐릭터에 입히기 (Priority: P2)

**Goal**: 모자·옷·소품을 부위마다 하나씩 입고 벗으며, 차림이 미니룸·헤더·광장에 같은 모습으로 보인다 (SHOP-06, FR-037~041, FR-029, SC-006).

**Independent Test**: 모자 2개·옷 1개를 가진 회원으로 입기·바꾸기·벗기를 하고 새로고침·헤더·광장·블로그 미니룸이 같은지 확인한다 (quickstart CL-09~13).

### Tests for User Story 4

- [x] T048 [P] [US4] `scripts/test-shop.ts`에 차림 테스트: 어떤 순서로 넣어도 옷 → 소품 → 모자, `lookKey`는 순서와 상관없이 같고 차림이 없으면 `assetKey` 그대로 (FR-040)
- [x] T049 [P] [US4] `e2e/closet.mjs`에 CL-09(밀짚모자 입기·새로고침 유지, `data-look`에 `hat.straw`), CL-10(털모자로 바뀌고 hat 1행), CL-11(다시 누르면 벗기·hat 행 없음), CL-12(헤더·`/@{아이디}`·`/closet`·`/town` `data-look`/`data-player-look` 같음), CL-13(미보유 안경 `equipItem`·`unequipAvatar` 재생 → `가지고 있지 않은 아이템이에요`), CL-06의 모자 실패 되돌리기

### Implementation for User Story 4

- [x] T050 [US4] `src/db/schema.ts`에 `avatarEquips` 표를 더한다: `user_id` text NOT NULL(PK, FK → `users.id` ON DELETE CASCADE), `slot` `avatar_slot` NOT NULL(PK), `item_id` integer NOT NULL, 복합 FK (`user_id`, `item_id`) → `user_items` (`user_id`, `item_id`) ON DELETE CASCADE (`avatar_equips_owned_fk`), `equipped_at` timestamptz NOT NULL 기본 now() (data-model 2.3, R6)
- [x] T051 [US4] `npm run db:generate`로 `drizzle/NNNN_avatar_equips.sql`을 만들고 맨 위에 `-- SHOP-06: 아바타 착용 상태 (부위마다 하나, 가진 것만)` 주석을 단다 (T050 뒤)
- [x] T052 [US4] `src/lib/art/characters.ts` 변경: `characterSvg(assetKey, size = 64, outfit: string[] = [])`·`characterDataUri(assetKey, size = 64, outfit = [])`가 몸 위에 `orderOutfit`대로 옷 → 소품 → 모자를 겹치고(비면 지금과 같은 SVG, 모르는 키는 무시), `lookKey(assetKey, outfit = [])`(정렬해서 이은 문자열)를 내보낸다 (FR-040, contracts/shared-modules.md 3장)
- [x] T053 [US4] `src/server/inventory.ts`에 `outfitOf(userIdColumn)` SQL 조각(`ARRAY(SELECT i.asset_key FROM avatar_equips ae JOIN items i ON i.id = ae.item_id WHERE ae.user_id = <col>)`)을 더하고, `getEquipped`의 `avatar`(`Partial<Record<"hat" | "outfit" | "accessory", number>>`)를 채우며, `getMiniRoomDecor(userId)`를 `{ outfit, furniture: [] }`로 더한다 (T051 뒤)
- [x] T054 [US4] `src/app/closet/actions.ts` 변경: `equipItem`에 avatar 분기(`INSERT avatar_equips (user_id, slot = items.avatar_slot, item_id) ON CONFLICT (user_id, slot) DO UPDATE SET item_id, equipped_at`)를 더하고, 새 `unequipAvatar(itemId)`(parseId·보유 확인 → avatar가 아니면 `아직 장착할 수 없는 종류예요` → `DELETE FROM avatar_equips WHERE user_id = 나 AND item_id = ?`, 입지 않았어도 성공)를 더한다. 서버는 토글하지 않는다 (FR-038~039, FR-041, R7, contracts/closet.md 2.1·2.2)
- [x] T055 [US4] `src/components/character.tsx` 변경: `CharacterArt`·`CharacterBadge`에 선택 prop `outfit?: string[]`와 `<img>`의 `data-look={lookKey(asset, outfit)}`, `MiniRoom`(blog 소유, 추가만)에 선택 prop `outfit?: string[]` (T052 뒤)
- [x] T056 [US4] `src/app/closet/closet-view.tsx`에 `👕 아바타 꾸미기` 카드 입기·바꾸기·벗기(입은 카드를 다시 누르면 `unequipAvatar`)와 낙관적 갱신·실패 되돌리기, 미니룸 `outfit` 전달을 더한다. 성공 문구는 FR-024 문구 그대로 (FR-038~041, plan 남은 문제 1, T054·T055 뒤)
- [x] T057 [P] [US4] 공통 모듈 추가(auth): `src/server/dal.ts`의 `getViewer` select에 `outfit: outfitOf(profiles.userId)`를 더해 `viewer.profile.outfit`을 만든다 (T053 뒤)
- [x] T058 [P] [US4] 공통 모듈 추가(town): `src/server/town.ts`의 `getTownHouses`·`getMyHouse` select에 `outfit: outfitOf(profiles.userId)`, `src/components/town/types.ts`에 `TownHouse.outfit: string[]`·`TownData.player.outfit: string[]`, `src/app/town/page.tsx`에서 `player.outfit = member.profile.outfit` (T053·T057 뒤)
- [x] T059 [P] [US4] 공통 모듈 추가(town): `src/components/site-header.tsx`, `src/components/town/town-menu.tsx`의 `<CharacterBadge … outfit={…} />` (프로필 사진이 없을 때만 캐릭터 얼굴에 차림, plan 남은 문제 7) (T055·T057 뒤)
- [x] T060 [US4] 요청(town): `src/components/town/scene.ts`에서 `charKey(asset)` → `charKey(lookKey(asset, outfit))`, `characterDataUri(asset, size, outfit)`로 플레이어와 집 문 옆 캐릭터를 그리고, 검증용으로 `TownGame` 바깥 요소에 `data-player-look` 속성 하나를 둔다. 같은 PR에서 town 담당 리뷰 (T052·T058 뒤)
- [x] T061 [US4] 공통 모듈 추가(blog): `src/app/blog/[slug]/page.tsx`에서 `getMiniRoomDecor(blog.ownerId)`를 부르고 `src/components/blog/blog-header.tsx`가 `outfit`을 `MiniRoom`에 넘긴다 (T053·T055 뒤)

**Checkpoint**: 차림이 꾸미기·헤더·광장·블로그 미니룸 네 곳에서 같다 (SC-006)

---

## Phase 9: User Story 7 - 미니룸에 가구 놓기 (Priority: P3)

**Goal**: 산 가구를 미니룸 안에 끌어다 놓고 옮기고 빼며, 최대 5개·비율 위치로 저장해 누구나 블로그 홈에서 본다. 광장 집 겉모습에는 보이지 않는다 (SHOP-05, FR-031~036).

**Independent Test**: 가구 5종 + 테스트 가구 1개를 가진 회원으로 5개를 놓고 6번째가 막히는지, 새로고침·375px·다른 회원 화면에서 같은 위치인지 확인한다 (quickstart CL-14~20).

### Tests for User Story 7

- [ ] T062 [P] [US7] `scripts/test-shop.ts`에 가구 자리 테스트: 사용 중 [1,2,4] → 3, [1..5] → null, `FURNITURE_LIMIT` = 5 (FR-033) *(2026-10-08 개편으로 바뀜: 가구는 미니룸 5자리가 아니라 블로그 '우리 집' 칸에 놓고 칸 수는 집 단계로 정함, 시험은 test-game의 `furnitureSlots`)*
- [ ] T063 [P] [US7] `e2e/closet.mjs`에 CL-14(끌어 놓기·새로고침 뒤 1%p 안), CL-15(옮기기 → 밖으로 빼기·행 삭제), CL-16(DB로 `is_on_sale = false` 테스트 가구를 만들어 지급, 6번째 → `가구는 5개까지 놓을 수 있어요`·행 5개, 끝나면 정리), CL-17(하나 빼고 다시 놓기), CL-18(미보유 침대 → `가지고 있지 않은 아이템이에요`, 좌표 `-1`·`101`·`"x"` → `잘못된 요청이에요`), CL-19(다른 회원이 블로그 홈에서 같은 배치, 광장 집 겉모습에 가구 없음), CL-20(375×812에서 같은 % 위치) *(2026-10-08 개편으로 바뀜: 끌어 놓기 대신 '우리 집' 칸마다 [가구 놓기]로 고르고, 확인은 `e2e/house.mjs`)*

### Implementation for User Story 7

- [ ] T064 [US7] `src/db/schema.ts`에 `roomFurniture` 표를 더한다: `user_id` text NOT NULL(PK, FK → `users.id` ON DELETE CASCADE), `item_id` integer NOT NULL(PK, 복합 FK (`user_id`, `item_id`) → `user_items` ON DELETE CASCADE `room_furniture_owned_fk`), `slot` smallint NOT NULL + `room_furniture_slot_check CHECK (slot BETWEEN 1 AND 5)` + `room_furniture_slot_uq UNIQUE (user_id, slot)`, `x`·`y` real NOT NULL + `room_furniture_position_check CHECK (x BETWEEN 0 AND 100 AND y BETWEEN 0 AND 100)`, `placed_at`·`updated_at` timestamptz NOT NULL 기본 now() (data-model 2.4, R10) *(2026-10-08 개편으로 바뀜: `room_furniture` 대신 칸 번호로 놓는 `house_furniture` 표)*
- [ ] T065 [US7] `npm run db:generate`로 `drizzle/NNNN_room_furniture.sql`을 만들고 맨 위에 `-- SHOP-05: 미니룸 가구 배치 (최대 5개, 비율 위치, 가진 것만)` 주석을 단다 (T064 뒤) *(2026-10-08 개편으로 바뀜: 마이그레이션은 `0026_house_furniture`)*
- [ ] T066 [P] [US7] `src/lib/shop.ts`에 `FURNITURE_LIMIT = 5`, `nextFreeSlot(usedSlots)`(1~5 중 비어 있는 가장 작은 번호, 없으면 null), `DEFAULT_SPOTS`(키보드·클릭으로 놓을 기본 자리 5곳)를 더한다 *(2026-10-08 개편으로 바뀜: 칸 수는 `src/lib/house.ts` `furnitureSlots`(집 단계), 비율 위치·기본 자리 없음)*
- [ ] T067 [US7] `src/server/inventory.ts`의 `getEquipped`에 `furniture: { itemId; x; y }[]`, `getMiniRoomDecor`에 `furniture: { assetKey; name; x; y }[]`(`y` 오름차순, 누구나 볼 수 있는 정보만)를 채운다 (T065 뒤) *(2026-10-08 개편으로 바뀜: 놓은 가구는 `src/server/house.ts` `getPlacedFurniture`가 칸 번호로 읽고 미니룸에는 가구 없음)*
- [ ] T068 [US7] `src/app/closet/actions.ts`에 `placeFurniture(itemId, x, y)`(zod로 `x`·`y` 유한한 숫자 0~100, 아니면 `잘못된 요청이에요`, 소수 둘째 자리 반올림 → 트랜잭션·`lockUser` → parseId·보유(`가지고 있지 않은 아이템이에요`) → furniture가 아니면 `아직 장착할 수 없는 종류예요` → 이미 놓였으면 `UPDATE x, y, updated_at` → 아니면 `nextFreeSlot`, 없으면 `가구는 5개까지 놓을 수 있어요` → `INSERT`, UNIQUE 위반 23505도 같은 문구)와 `removeFurniture(itemId)`(parseId·보유 → `DELETE`, 놓여 있지 않아도 성공)를 더한다 (FR-032~034, contracts/closet.md 2.3·2.4, T066 뒤) *(2026-10-08 개편으로 바뀜: `src/app/house/actions.ts` `placeFurniture(slot, itemId)`로 칸에 놓음)*
- [ ] T069 [US7] `src/components/character.tsx`의 `MiniRoom`(blog 소유, 추가만)에 선택 prop `furniture?: { assetKey; name; x; y }[]`(가구는 맨 뒤 층, 아래쪽 `y`가 앞, 캐릭터는 늘 가구 앞, 아랫변 가운데를 % 위치에)와 `children?`(끌기 층)을 더한다. blog의 `showcase`와 층 순서는 같은 PR에서 맞춘다 (FR-035~036, R12) *(2026-10-08 개편으로 바뀜: 가구는 미니룸이 아니라 '우리 집'(`src/components/blog/house-room.tsx`)에 그림)*
- [ ] T070 [US7] `src/app/closet/furniture-room.tsx` 새로: Pointer Events로 가구 카드 → 미니룸 끌어 놓기, 놓인 가구 옮기기, 미니룸 밖으로 끌어 빼기(라이브러리 없음, 터치 동작), 키보드 Enter로 `DEFAULT_SPOTS`에 놓기·빼기, 낙관적 갱신·실패 되돌리기, 6번째면 `가구는 5개까지 놓을 수 있어요` (FR-032~033, FR-047, R11, T068·T069 뒤) *(2026-10-08 개편으로 바뀜: 끌어 놓기 화면 없이 '우리 집' 칸의 [가구 놓기]로 고름)*
- [ ] T071 [US7] `src/app/closet/closet-view.tsx`의 `🪑 가구` 구역과 미니룸에 `FurnitureRoom`을 연결한다 (T070 뒤) *(2026-10-08 개편으로 바뀜: 꾸미기의 🪑 가구 구역은 가진 가구를 보여 주고 '우리 집'으로 안내)*
- [ ] T072 [US7] 공통 모듈 추가(blog): `src/components/blog/blog-header.tsx`가 `getMiniRoomDecor`의 `furniture`를 `MiniRoom`에 넘긴다. 광장 집 겉모습(`src/components/town/scene.ts`)에는 가구를 그리지 않음을 확인한다 (FR-036, T061·T067·T069 뒤) *(2026-10-08 개편으로 바뀜: 가구는 블로그 '우리 집' 구역에 보이고 미니룸에는 넘기지 않음)*

**Checkpoint**: 모든 User Story가 독립적으로 동작

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: 여러 Story에 걸친 검증·문서·정리

- [ ] T073 [P] `e2e/params.mjs`에 `buyItem`, `equipItem`, `unequipAvatar`, `placeFurniture`, `removeFurniture`의 범위 밖(99999999999)·글자(`"abc"`) 인자 재생을 더한다(`check` 이름에 SHOP ID): 모두 HTTP 200·서버 오류 없음, 구매 `살 수 없는 아이템이에요` / 장착·배치 `가지고 있지 않은 아이템이에요` (NF-02, SC-004)
- [ ] T074 [P] `e2e/closet.mjs`·`e2e/shop.mjs`에 CL-21(375px `/shop`·`/closet` `scrollWidth <= 375`, 버튼 글자 줄바꿈 없음, 44×44px 이상), CL-22·SH-24(Tab·Enter만으로 사기·장착·입기·벗기·가구 놓기/빼기, 포커스 보임)와 quickstart 6장 스크린샷 저장을 더한다 (FR-046~047, SC-007)
- [x] T075 [P] `e2e/nonfunctional.mjs`·`e2e/mobile.mjs` 대상에 `/shop`, `/closet`이 포함되는지 확인하고 필요하면 더한다
- [x] T076 `src/server/inventory.ts`에서 상점·꾸미기만 쓰던 `listItemsWithOwnership`을 지우고 남은 참조가 없는지 확인한다
- [ ] T077 [P] `docs/02-erd.md` 갱신: data-model 9장 표대로 1장·2장·3.4·3.7·3.11·3.14·3.15·3.17·5장·7장·부록 (`items` 컬럼, `user_items.quantity`, `avatar_equips`, `room_furniture`, `avatar_slot`)
- [ ] T078 [P] `docs/erdcloud-import.sql`에 새 표 2개와 `items`·`user_items` 컬럼을 더한다 (R20, 팀 확인 뒤)
- [x] T079 [P] `README.md` 스크립트 표에 `test:shop`, `e2e/shop.mjs`, `e2e/closet.mjs`를 한 줄씩 더한다
- [x] T080 `npx tsc --noEmit`, `npx eslint`, `npm test`를 돌려 모두 통과시킨다
- [ ] T081 quickstart.md 4장 명령(`e2e/shop.mjs`, `e2e/closet.mjs`, `e2e/game.mjs`, `e2e/decisions.mjs`, `e2e/blog.mjs`·`e2e/params.mjs`, `e2e/mobile.mjs`, `e2e/nonfunctional.mjs`)을 모두 돌려 `❌`·콘솔 오류가 없음을 확인하고, 6장 스크린샷을 눈으로 본 뒤 결과와 SC-001 직접 측정 시간을 PR 설명에 적는다

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 의존 없음. 바로 시작
- **Foundational (Phase 2)**: Setup 뒤. 모든 User Story를 막는다 (스키마·마이그레이션·시드·그림·`src/lib/shop.ts` 뼈대)
- **User Stories (Phase 3~9)**: 모두 Foundational 뒤
  - US1(P1)은 상점 없이 동작하므로 가장 먼저 (MVP)
  - P2 안에서는 plan.md 구현 순서를 따른다: US2 → US3 → US5 → US6 → US4. US5(성장 아이템)는 town(TOWN-09)의 선행이라 US4·US7보다 먼저 merge
  - US7(P3)은 마지막
- **Polish (Phase 10)**: 원하는 Story가 끝난 뒤

### User Story Dependencies

- **US1 (P1)**: Foundational 뒤 바로. 다른 Story에 기대지 않음
- **US2 (P2)**: Foundational 뒤. US1과 독립 (구매 결과를 꾸미기에서 쓰는 것은 통합일 뿐)
- **US3 (P2)**: `buyItem`·`shop-grid.tsx`를 US2와 같이 고치므로 US2 뒤
- **US5 (P2)**: `buyItem`·`shop-grid.tsx`를 고치므로 US2 뒤. `consumeGrowthItem`(T043)은 독립
- **US6 (P2)**: `shop-view.tsx`(T030)와 `listShopItems`의 `ownerCount`(T027)가 있어야 하므로 US2 뒤
- **US4 (P2)**: `equipItem`·`closet-view.tsx`를 고치므로 US1 뒤. 상점 구매 없이 DB 지급으로 시험 가능
- **US7 (P3)**: `closet/actions.ts`·`closet-view.tsx`·`MiniRoom`을 고치므로 US1 뒤, `getMiniRoomDecor`·`blog-header.tsx` 연결(T061)이 있으므로 US4 뒤

### 다른 spec과의 의존

- **auth 단계 1**(가입 통합, `e2e/helpers.mjs`의 `loginDev`)이 `main`에 있어야 새 e2e(T016, T025 등)가 돈다. 코드 작업은 병행 가능
- **town**: T060(`scene.ts`)은 town 담당 동의 필요. town TOWN-09가 T041·T043을 쓴다
- **blog**: T055·T061·T069·T072의 `MiniRoom`·블로그 홈 변경은 blog 담당과 같은 PR에서 층 순서를 맞춘다
- **game**: GAME-06 레벨업 팝업이 T038(`listItemsUnlockedBetween`) 또는 `items.is_on_sale`을 쓴다

### Within Each User Story

- 테스트(단위·e2e)를 먼저 쓰고 실패를 확인한 뒤 구현
- 스키마 → 마이그레이션 → 서버(`src/server/inventory.ts`) → Server Action → 화면 → 다른 spec 파일 연결
- 같은 파일을 고치는 태스크(`src/server/inventory.ts`, `src/app/shop/actions.ts`, `src/app/closet/actions.ts`, `src/app/closet/closet-view.tsx`, `scripts/test-shop.ts`, `e2e/*.mjs`)는 차례로 한다

### Parallel Opportunities

- Setup: T002, T003
- Foundational: T008, T009, T010 (그림 3파일), T014는 T013 뒤
- US1: T016, T017
- US2: T024, T025, T026 (서로 다른 파일)
- US3: T033, T034, T035, T038
- US5: T039, T040, T043
- US6: T044, T045
- US4: T048, T049 / T057, T058, T059 (서로 다른 다른-spec 파일)
- US7: T062, T063, T066
- Polish: T073, T074, T075, T077, T078, T079
- 팀 인원이 있으면 Foundational 뒤 "상점 줄"(US2 → US3 → US5 → US6)과 "꾸미기 줄"(US1 → US4 → US7)을 나눠 병행할 수 있다

---

## Parallel Example: User Story 2

```bash
# US2 테스트를 함께 시작:
Task: "scripts/test-shop.ts에 버튼 상태·부족 코인·꾸미기 구매 문구 테스트"
Task: "e2e/shop.mjs에 SH-01~SH-11, SH-23, SH-25"

# 테스트와 동시에 순수 규칙:
Task: "src/lib/shop.ts에 buttonState, shortage, purchaseMessage"
```

## Parallel Example: User Story 4

```bash
# 차림 필드를 다른 spec 파일에 끼우는 일을 함께:
Task: "src/server/dal.ts getViewer에 outfit: outfitOf(profiles.userId)"
Task: "src/server/town.ts·src/components/town/types.ts·src/app/town/page.tsx에 outfit"
Task: "src/components/site-header.tsx·src/components/town/town-menu.tsx CharacterBadge outfit"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: Setup
2. Phase 2: Foundational (스키마·판매 중단 이전·시드 — 모든 Story를 막음)
3. Phase 3: User Story 1 (가진 캐릭터·배경 장착)
4. **STOP and VALIDATE**: `e2e/closet.mjs` CL-01~08, `e2e/decisions.mjs`
5. 준비되면 PR·데모

### Incremental Delivery

1. Setup + Foundational → 기반 준비 (판매 중단 캐릭터 보유·장착 유지 확인, S-00)
2. US1 → 꾸미기 장착 (MVP)
3. US2 → US3 → 꾸미기 아이템 구매와 레벨 잠금
4. US5 → 성장 아이템 구매·수량 도우미 (town TOWN-09가 이어서 시작 가능)
5. US6 → 정렬
6. US4 → 아바타 착용과 네 곳 같은 차림
7. US7 → 가구 배치
8. Polish → 조작 요청·모바일·키보드·문서·quickstart 전체 실행

### Parallel Team Strategy

1. 팀이 Setup + Foundational을 함께 끝낸다
2. 그 뒤:
   - 개발자 A: 상점 줄 US2 → US3 → US5 → US6 (`src/app/shop/*`, `buyItem`)
   - 개발자 B: 꾸미기 줄 US1 → US4 → US7 (`src/app/closet/*`, 그림, 다른 spec 연결)
3. `src/server/inventory.ts`와 `scripts/test-shop.ts`는 두 줄이 함께 고치므로 작은 PR로 자주 합친다

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Verify tests fail before implementing
- 화면 문구는 spec 문구를 글자 그대로 쓰고, spec에 없는 문구(상점 안내, `보유 N개`, 벗기·가구 성공 문구, `잘못된 요청이에요`)는 research R19·plan 남은 문제의 기본값을 쓴다
- 마이그레이션·코드·e2e `check` 이름에 SHOP ID를 단다 (constitution II)
- 잔액 컬럼을 만들지 않고, 지급·차감은 언제나 `lockUser` 트랜잭션 안에서 한다 (constitution V)
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
