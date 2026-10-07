# Research: 상점 / 꾸미기 (SHOP)

Phase 0 결과다. plan의 Technical Context에서 정해야 했던 것과 이 기능의 기술 선택을 "Decision / Rationale / Alternatives considered"로 적었다.

- 근거 표시: **[코드]** 코드 저장소 `main` `feb4c05`에서 읽고 확인한 사실, **[ERD]** `docs/02-erd.md` v1.4, **[원본]** `docs/01-requirements.md` v1.18, **[추측]** 확인하지 못한 라이브러리 동작이나 규모 가정.
- 이 체크아웃에는 `node_modules`가 없어 Next.js 16·Drizzle·Phaser 4 문서를 열 수 없었다. 그런 결정에는 "구현 전 확인"을 붙였다.
- Technical Context에 "확인 필요"로 남긴 칸은 없다. spec이 문구나 범위를 정하지 않아 기본값을 고른 것은 R19에 모았고, 팀 확인이 필요한 것은 plan 보고의 열린 항목으로 넘긴다.

---

## R1. 아바타 꾸미기·성장 아이템을 아이템 종류로 어떻게 나타내나

**Decision**: `item_type` 열거형에 `avatar`와 `growth`를 더한다. 아바타 꾸미기의 부위는 새 열거형 `avatar_slot`(`hat` 모자, `outfit` 옷, `accessory` 소품)과 `items.avatar_slot` 컬럼으로 둔다. 성장치는 `items.growth_value`. CHECK로 "아바타면 부위가 있고, 아니면 없다", "성장 아이템이면 성장치가 있고(> 0), 아니면 없다"를 묶는다.

**Rationale**:
- spec Key Entities가 "종류(아바타 꾸미기·가구·배경·성장 아이템…)"와 "아바타 꾸미기는 부위를"을 따로 적는다. 원본 SHOP-06 구현 방식 제안도 "`item_type`에 `avatar` 추가 + 부위(`slot`: hat / outfit / accessory) 칸"이다 [원본].
- ERD 3.11·3.15가 이미 `item_type`에 `growth`, `items.growth_value`, CHECK `(type = 'growth') = (growth_value IS NOT NULL)`, `growth_value > 0`을 정했다 [ERD]. 그대로 따른다.
- 부위를 따로 두면 착용 표(R6)의 `slot` 컬럼이 같은 열거형을 쓰고, 부위가 늘면 열거형 값 하나만 더하면 된다.
- 상점 구역은 종류로 바로 나뉜다: `avatar` → `👕 아바타 꾸미기`, `furniture` → `🪑 가구`, `background` → `🖼 배경`, `growth` → `🌱 성장 아이템`. `character`는 상점에 나오지 않는다.

**Alternatives considered**:
- `item_type`에 `hat`·`outfit`·`accessory` 3개를 바로 더해 종류 = 부위로 쓰기: 컬럼이 하나 줄지만 "아바타 꾸미기"라는 묶음이 코드 상수로만 남고, 착용 표의 `slot`이 `item_type` 전체를 받게 돼 CHECK가 하나 더 필요하다. 원본 제안과도 다르다.
- `profiles`에 `hat_item_id`·`outfit_item_id`·`accessory_item_id` 컬럼(ERD 5장의 다른 안): R6에서 기각.

## R2. 새 열거형 값을 같은 마이그레이션 실행에서 써도 되나

**Decision**: 마이그레이션의 CHECK는 새 값을 열거형 리터럴로 쓰지 않고 글자로 바꿔 비교한다: `(type::text = 'growth') = (growth_value IS NOT NULL)`, `(type::text = 'avatar') = (avatar_slot IS NOT NULL)`. 새 값을 쓰는 데이터(첫 출시 아이템)는 마이그레이션이 아니라 시드(`npm run db:seed`, 마이그레이션이 끝난 뒤 별도 연결)로 넣는다. `avatar_slot`은 같은 마이그레이션에서 `CREATE TYPE`으로 새로 만들므로 그 값은 바로 써도 된다.

**Rationale**:
- PostgreSQL은 트랜잭션 안의 `ALTER TYPE … ADD VALUE`를 허용하지만, 그 값은 트랜잭션이 커밋되기 전에는 쓸 수 없다("unsafe use of new value" 오류). 같은 트랜잭션에서 새로 만든 열거형 타입의 값은 예외다 (PostgreSQL 12 이후 문서) [추측: 버전별 문서로 재확인].
- Drizzle의 node-postgres 마이그레이터는 대기 중인 마이그레이션을 **한 트랜잭션 안에서** 차례로 실행한다고 알고 있다 [추측, 구현 전 `node_modules/drizzle-orm/pg-core/dialect.js`의 `migrate`로 확인]. 그러면 마이그레이션 파일을 둘로 나눠도 함께 대기 중이면 같은 트랜잭션이 된다. `::text` 비교는 어느 경우든 안전하다.
- 기존 사례: `drizzle/0005_animal_farm.sql`은 `ledger_reason`에 값 3개를 더했지만 같은 파일에서 그 값을 쓰지 않았다 [코드].
- `type::text` 비교는 Drizzle `check()`에 `sql` 조각으로 그대로 쓸 수 있고, `drizzle-kit generate`가 그 글자를 그대로 SQL로 옮긴다 (기존 `check(... sql\`...\`)` 사용례) [코드].

**Alternatives considered**:
- `type = 'growth'` 그대로 쓰고 "마이그레이션을 두 번에 나눠 적용"하라고 안내: 팀원·배포 DB마다 순서를 지켜야 해서 깨지기 쉽다.
- 열거형 대신 `text` + CHECK로 종류를 바꾸기: 기존 컬럼 타입 변경이 커지고 다른 spec의 쿼리(`eq(items.type, "character")`)에도 영향이 간다.

## R3. 판매하지 않는 아이템(기본 아이템·과거 캐릭터)을 어떻게 구분하나

**Decision**: `items.is_on_sale boolean NOT NULL DEFAULT true`를 더한다. 바로 뒤의 직접 쓴 데이터 마이그레이션이 `UPDATE items SET is_on_sale = false WHERE is_starter OR type = 'character'`를 한다(여러 번 실행해도 같다). 시드는 모든 아이템에 `isOnSale`을 명시한다. 상점 목록은 `is_on_sale = true`만, `buyItem`은 `!item || !item.isOnSale || item.isStarter`면 `살 수 없는 아이템이에요`. 과거 캐릭터 9종의 행은 지우지 않는다.

**Rationale**:
- spec Key Entities가 "판매 중 여부"를 아이템 속성으로 적었고, FR-009가 "판매 중이 아닌" 아이템을 거부하라고 한다. 운영 중 아이템을 내릴 때도 같은 컬럼을 쓴다.
- 지금은 "보이지 않게"를 `is_starter`로만 한다(`src/app/shop/page.tsx`의 `!i.isStarter`) [코드]. 과거 캐릭터를 `is_starter = true`로 바꾸면 기본 캐릭터 고르기 목록(지금은 온보딩 `src/app/onboarding/page.tsx`·`actions.ts`의 `eq(items.isStarter, true)`, auth 단계 1 뒤에는 가입 폼)에 나타나므로 쓸 수 없다 [코드].
- 행을 지우면 `user_items.item_id` FK(삭제 동작 없음)와 `profiles` 복합 FK가 막고, D12("그대로 보유·장착")와도 어긋난다.
- 백필을 마이그레이션에 두는 이유: 시드를 잊어도 `db:migrate`만으로 캐릭터 판매가 멈춘다(SC-010). 판매 중단은 `user_items`·`profiles`·`point_ledger`를 건드리지 않으므로 환불·장착 변경이 0건이다.

**Alternatives considered**:
- 종류로만 거르기(`type <> 'character'`): 지금 결정에는 맞지만 "판매 중 여부"라는 spec 속성이 없어지고 다른 종류를 내릴 방법이 없다.
- 시드에만 맡기기: 배포 DB에서 시드를 돌리기 전까지 캐릭터가 계속 팔린다.

## R4. 성장 아이템을 여러 개 갖게 하는 방법

**Decision**: `user_items.quantity integer NOT NULL DEFAULT 1` + CHECK `quantity >= 0` (ERD 3.11 그대로). 성장 아이템을 사면 `INSERT … ON CONFLICT (user_id, item_id) DO UPDATE SET quantity = user_items.quantity + 1`. 다 써서 0이 돼도 행은 남긴다. "보유"는 어디서나 `quantity > 0`으로 판단한다. 꾸미기 아이템은 늘 1이다(코드 규칙: 꾸미기는 upsert하지 않고 그냥 insert).

**Rationale**:
- ERD가 "같은 먹이를 또 사면 줄을 늘리지 않고 수량 +1, 쓰면 −1. 0이 돼도 줄은 남긴다. 복합 PK는 그대로 아이템마다 한 줄"로 정했다 [ERD 3.11].
- 컬럼을 `DEFAULT 1 NOT NULL`로 더하면 기존 행이 모두 1이 되어 백필이 따로 필요 없다 (ERD 7장 6-2 "기존 보유 아이템은 수량 1").
- 복합 PK가 그대로라 `profiles`·`blogs`의 장착 복합 FK도 그대로다.
- 동시성: 같은 회원의 구매는 `lockUser`로 한 줄로 서고, upsert 자체도 원자적이라 "늘어난 수량 × 가격 = 빠진 코인"(FR-045)이 깨지지 않는다.
- Drizzle: `onConflictDoUpdate({ target: [userItems.userId, userItems.itemId], set: { quantity: sql\`${userItems.quantity} + 1\` } })` 형태. `onConflictDoUpdate`는 `scripts/seed.ts`에서 이미 쓴다 [코드]. 복합 target과 `sql` 증가식은 구현 전 확인.

**Alternatives considered**:
- 개수만큼 행 여러 개: 복합 PK를 깨야 하고 장착 FK 대상이 바뀐다.
- 성장 아이템 전용 표: 같은 "보유" 개념이 두 곳으로 갈라진다.

## R5. 구매 처리 순서와 거부 문구

**Decision**: `buyItem(itemId)`는 다음 순서로 판단한다. 모두 한 트랜잭션, `lockUser` 뒤.

1. `parseId(itemId)`가 null → `살 수 없는 아이템이에요` (없는 아이템)
2. 아이템이 없거나 `is_on_sale = false`거나 `is_starter` → `살 수 없는 아이템이에요`
3. 꾸미기 아이템(아바타·가구·배경)이고 내 `user_items`에 이미 있음 → `이미 가지고 있는 아이템이에요`
4. `getWallet(userId, tx)`의 레벨 < 필요 레벨 → `레벨 N부터 살 수 있어요`
5. 잔액 < 가격 → `코인이 N개 부족해요` (N = 가격 − 잔액)
6. 꾸미기: `user_items` insert / 성장: upsert 수량 +1 → `point_ledger`에 `reason = 'purchase'`, `coin_delta = −가격`, `ref_id = 아이템 ID`
7. 성공하면 `revalidatePath("/", "layout")`

`user_items` PK 위반(23505)은 지금처럼 잡아서 `이미 가지고 있는 아이템이에요`로 바꾼다(잠금 밖의 쓰기와 겹칠 때의 안전망).

**Rationale**:
- 화면 버튼 우선순위 `보유 중` → `🔒` → `코인 부족`(FR-006)과 서버 거부 순서를 맞춘다. 지금 코드는 보유를 잔액 다음의 PK 위반으로만 알아서, 이미 가진 아이템을 잔액이 모자랄 때 다시 사면 `코인이 N개 부족해요`가 나온다 [코드]. AC 2-6은 `이미 가지고 있는 아이템이에요`를 기대한다.
- SC-002(10번 동시 요청 → 성공 1, 거부 9): 잠금 덕분에 두 번째부터는 3번에서 멈춘다.
- 지금 `buyItem`은 `parseId`를 거치지 않아 범위 밖 숫자가 DB 오류가 된다 [코드, 공통 맥락 4.6]. 없는 아이템이므로 spec 문구 `살 수 없는 아이템이에요`를 쓴다(새 문구를 만들지 않음).
- 원장 `ref_id`에 아이템 ID를 넣으면 내역 화면이 이미 `purchase` 기록에 아이템 이름을 붙인다(`listLedger`의 `items.id::text = ref_id` 조인) [코드]. 성장 아이템도 `🏪 아이템 구매 · 동물 먹이`로 보인다(FR-049). 새 원장 사유가 필요 없다.

**Alternatives considered**: 보유 확인을 PK 위반에만 맡기기(지금 방식) — 문구 순서가 화면과 어긋난다.

## R6. 아바타 착용 상태를 어디에 저장하나

**Decision**: 새 표 `avatar_equips(user_id, slot avatar_slot, item_id, equipped_at)`. PK (`user_id`, `slot`) = 부위마다 하나. 복합 FK (`user_id`, `item_id`) → `user_items`(`user_id`, `item_id`) `ON DELETE CASCADE` = 가진 것만. `user_id` → `users` `ON DELETE CASCADE`. "그 아이템이 그 부위의 아바타 아이템인지"는 서버 코드가 `items.avatar_slot`에서 `slot`을 정해 넣으므로 코드로 지킨다.

**Rationale**:
- ERD 5장이 "profiles 컬럼 3개 vs 별도 표"를 남겼고, 공통 맥락이 별도 표를 권장했다. 별도 표면 auth 담당 `profiles`를 건드리지 않고 shop 안에서 끝난다.
- 원본 SHOP-06 구현 제안 `avatar_equips(user_id, slot, item_id)`, "`user_items`를 가리키는 복합 외래 키로 가진 것만 입기"와 같다 [원본].
- 벗기 = 행 삭제라 NULL 컬럼이 없다. 부위를 더해도 표 구조가 그대로다.
- 종류 검사를 코드에 두는 것은 ERD 3.4 선례("캐릭터 칸에는 캐릭터 아이템만이라는 종류 검사는 서버 코드에서 한다")와 같다 [ERD].

**Alternatives considered**:
- `profiles` 컬럼 3개 + 복합 FK 3개: auth에 구조 변경 요청이 필요하고 부위가 늘 때마다 컬럼이 는다.
- `items`에 UNIQUE (`id`, `avatar_slot`)를 두고 `avatar_equips`(`item_id`, `slot`) → `items`(`id`, `avatar_slot`) 복합 FK로 부위 일치까지 DB가 막기: 가능하지만 제약이 하나 더 늘고, `slot`을 사용자가 고르지 않으므로 얻는 것이 적다.
- JSON 컬럼 하나: 보유 여부를 FK로 막을 수 없다 (constitution V 위반).

## R7. 입기·벗기 Server Action 모양

**Decision**: `equipItem(itemId)`가 캐릭터·배경·아바타를 모두 "장착/입기"로 받는다(아바타는 `ON CONFLICT (user_id, slot) DO UPDATE SET item_id`). 벗기는 따로 `unequipAvatar(itemId)` (`DELETE … WHERE user_id = 나 AND item_id = ?`). 입은 아이템을 다시 누르면 화면이 `unequipAvatar`를 부른다. 서버는 토글하지 않는다.

**Rationale**:
- 서버가 "지금 입었으면 벗기"로 토글하면, 빠르게 두 번 누르거나 탭 두 개에서 누를 때 결과가 요청 순서에 따라 뒤집힌다. 의도를 담은 두 동작은 여러 번 보내도 결과가 같다(멱등).
- 캐릭터·배경은 지금처럼 `profiles.character_item_id`·`blogs.background_item_id` UPDATE. "항상 하나씩"(FR-028)은 NOT NULL 컬럼 하나라 그대로 지켜진다 [코드].
- 가진 것 확인은 `user_items` + `items` 조인에서 `quantity > 0`. 성장 아이템과 가구는 `아직 장착할 수 없는 종류예요` (가구는 R10의 배치 동작으로만 미니룸에 들어간다).

**Alternatives considered**: `equipItem(itemId, { off: true })` 하나로 합치기 — 인자 조작 면이 넓어지고 이름이 동작을 가린다.

## R8. 차림을 그리는 방법과 데이터 모양

**Decision**:
- 그림: `src/lib/art/avatar.ts`에 부위별 겹침 SVG 조각(`asset_key` → `{ slot, svg }`)을 두고, `characterSvg(assetKey, size, outfit = [])`가 몸 위에 **옷 → 소품 → 모자** 순서로 겹친다(FR-040의 몸 → 옷 → 소품 → 모자). 순서는 데이터 순서가 아니라 그림 모듈이 부위로 정한다. 모르는 키는 무시한다.
  - 끼우는 자리: 지금 `draw()`(`src/lib/art/characters.ts`)는 그림자 → 발 → 몸 → 팔 → 머리 → 얼굴 → 머리카락(`overHead`) → `extra` 순서로 그린다 [코드]. 몸통 윗부분(y≈36부터)이 머리 원 아래(턱, y≈43.5까지)와 겹치므로 옷 조각을 맨 뒤에 덧붙이면 턱을 덮는다. 그래서 옷은 `draw()` 안에서 팔 다음·머리 앞에 끼우고, 소품·모자는 맨 뒤(`extra` 뒤)에 덧붙인다. 모자는 머리카락·귀 위에 그려진다.
- 데이터: 차림은 착용한 아이템의 `asset_key` 배열(`outfit: string[]`). 서버 쿼리에는 shop이 제공하는 SQL 조각 `outfitOf(userIdColumn)` = `ARRAY(SELECT i.asset_key FROM avatar_equips ae JOIN items i ON i.id = ae.item_id WHERE ae.user_id = …)`을 필드로 더한다. node-postgres는 `text[]`를 JS 배열로 바꿔 준다 [추측, 구현 전 확인].
- 컴포넌트: `CharacterArt`·`CharacterBadge`에 선택 prop `outfit`. 넘기지 않으면 지금과 똑같다.
- 광장(Phaser): 텍스처 키가 `char:{asset}`이라 같은 캐릭터면 차림이 달라도 텍스처를 같이 쓴다 [코드 `scene.ts`의 `charKey`]. `lookKey(asset, outfit)` = `char.boy+hat.straw+outfit.hoodie`처럼 차림을 키에 넣고 `characterDataUri(asset, size, outfit)`로 그린다.

**Rationale**:
- 모든 캐릭터가 같은 몸 좌표(viewBox 64, 2등신, 발 y≈59)를 쓰므로 겹침 조각 하나가 남녀 주민과 과거 캐릭터 모두에 맞는다 [코드 `characters.ts` 주석].
- 같은 함수 하나로 헤더·광장·미니룸을 그리면 세 곳이 다를 수 없다 (SC-006).
- `characterAsset`(문자열)에 차림을 이어 붙여 기존 필드를 재활용하는 방법은 기존 필드의 뜻을 바꾸므로 다른 spec 소유 쿼리의 동작 변경이 된다. 새 필드 추가는 "공통 모듈 추가"로 할 수 있다.

**Alternatives considered**:
- 합친 문자열 키 하나(`characterAsset = "char.boy|hat.straw"`): 필드 추가가 없지만 위 이유로 기각.
- 차림별 그림 파일: 외부 그림 금지 규칙에 어긋난다.

## R9. 차림이 보여야 하는 곳의 범위

**Decision**: FR-040·SC-006이 적은 세 곳과 꾸미기 미니룸에 보인다: 헤더 캐릭터 얼굴(`src/components/site-header.tsx`), 광장의 내 캐릭터와 집 문 옆 주인 캐릭터(`scene.ts`, 휴대폰 간단 메뉴 `town-menu.tsx`의 얼굴 포함), 블로그 홈 미니룸(`blog-header.tsx`), 꾸미기 미니룸. 댓글·글 카드·글 상세의 작은 얼굴(`comment-section.tsx`, `post-card.tsx`, `blog/[slug]/[postId]/page.tsx`)은 이번에는 기본 캐릭터만 그린다.

**Rationale**: 작은 얼굴은 social·post 소유 쿼리 여러 개를 바꿔야 하고 spec이 요구하지 않는다. 나중에 같은 `outfitOf` 필드를 더하면 된다. 집 문 옆 캐릭터는 "광장의 내 캐릭터"와 나란히 보여 차림이 다르면 어긋나 보이므로 포함한다.

**Alternatives considered**: 모든 얼굴에 차림 — 범위가 커지고 쿼리마다 하위 쿼리가 하나씩 는다.

## R10. 가구 배치 저장과 5개 상한

**Decision**: 새 표 `room_furniture(user_id, item_id, slot smallint, x real, y real, placed_at, updated_at)`. `updated_at`은 지금 스키마의 `updatedAt()` 도우미(고칠 때 갱신)를 그대로 쓴다.
- PK (`user_id`, `item_id`): 같은 가구는 한 번만 놓인다 (같은 가구는 하나만 가짐, spec 기본값).
- UNIQUE (`user_id`, `slot`) + CHECK `slot BETWEEN 1 AND 5`: 한 회원(= 한 미니룸)에 최대 5줄을 **DB가** 막는다 (FR-033).
- CHECK `x BETWEEN 0 AND 100`, `y BETWEEN 0 AND 100`: 미니룸 기준 비율(FR-035). `x`는 가구 아랫변 가운데의 가로 위치, `y`는 아랫변의 세로 위치(위에서부터 %).
- 복합 FK (`user_id`, `item_id`) → `user_items` `ON DELETE CASCADE`, `user_id` → `users` `ON DELETE CASCADE`.
- 놓기는 `lockUser` 트랜잭션 안에서: 이미 놓인 가구면 `x`·`y`만 고치고(옮기기), 아니면 빈 자리 번호(1~5 중 가장 작은 것)를 골라 넣는다. 빈 자리가 없으면 `가구는 5개까지 놓을 수 있어요`. 빼기는 행 삭제(그 자리 번호가 다시 빈다, spec Edge "하나를 빼면 다시 놓을 수 있다").

**Rationale**:
- constitution V는 이런 규칙을 앱 코드뿐 아니라 DB 제약으로도 막으라고 한다. 동물 5마리는 상태별 개수라 CHECK로 못 막아 코드가 지키지만(ERD 3.11), 가구는 "자리 번호 1~5"로 바꿔 쓰면 UNIQUE + CHECK로 표현된다.
- 잠금을 거는 이유: 잠금 없이 둘이 동시에 같은 빈 자리를 고르면 하나가 UNIQUE에 막혀, 자리가 남았는데도 상한 문구가 나올 수 있다.
- 비율은 실수(`real`)로 둔다. 정수 %면 375px 화면에서 칸이 약 3.8px, 760px에서 약 7.6px씩 뛰어 놓은 자리와 어긋나 보인다. Drizzle `real()`은 JS number로 돌려준다 [추측, 구현 전 확인. 지금 코드에 `real` 사용례 없음]. 서버는 소수 둘째 자리에서 반올림해 저장한다.
- 표 이름은 원본 SHOP-05 열린 질문의 예 `room_furniture(user_id, item_id, x, y)`를 따른다 [원본]. 회원과 블로그가 1:1이고 복합 FK가 `user_id`를 써야 하므로 `blog_id`가 아니라 `user_id`로 묶는다.

**Alternatives considered**:
- 상한을 코드로만(농장 방식): constitution V의 "DB 제약으로도"를 지키지 못한다.
- 미니룸 하나를 JSON 배열로 저장: 보유 FK·상한을 DB가 막지 못한다.
- `numeric(5,2)`: Drizzle이 문자열로 돌려줘 변환 코드가 는다.

## R11. 끌어다 놓기 구현

**Decision**: 라이브러리 없이 Pointer Events로 만든다 (`src/app/closet/furniture-room.tsx`, 클라이언트).
- 아래 `🪑 가구` 카드에서 누른 채 끌면 그림이 손가락·마우스를 따라오고(`setPointerCapture`), 미니룸 안에서 놓으면 그 지점의 비율로 `placeFurniture`, 밖에서 놓으면 아무 일 없음.
- 미니룸 안의 가구를 끌면 옮기기, 미니룸 밖에서 놓으면 `removeFurniture` (AC 7-2).
- 끌 수 있는 요소에 `touch-action: none`을 줘 휴대폰에서 끌 때 화면이 스크롤되지 않게 한다.
- 키보드(FR-047, NF-18): 가구 카드는 버튼이라 Enter(또는 끌지 않고 누르기)로 "놓기 / 빼기"를 번갈아 한다. 놓는 자리는 `src/lib/shop.ts`의 `DEFAULT_SPOTS` 중 비어 있는 첫 곳이다(가운데 캐릭터를 피한 아래쪽 5곳, [contracts/closet.md](contracts/closet.md) 1.3). 키보드로 **옮기기**는 FR-047("Tab, Enter로 사고 장착")이 요구하지 않아 이번 범위에 넣지 않는다 (아래 Alternatives, 팀 확인 항목).
- 낙관적 갱신: 놓자마자 화면을 바꾸고 실패하면 되돌린다(지금 `ClosetView`의 장착 방식) [코드].

**Rationale**: HTML5 drag and drop(`draggable`, `dragstart`)은 모바일 브라우저 터치에서 동작하지 않아 375px 휴대폰(AC 7-6, constitution VI)을 만족하지 못한다. Pointer Events는 마우스·터치·펜을 한 코드로 받는다. 끌기 라이브러리(dnd-kit 등)는 이 정도 크기에서 의존성 추가 이유가 약하다(VII).

**Alternatives considered**: dnd-kit, react-draggable — 의존성 추가. 칸(격자) 배치 — "자유롭게 끌어다 놓는다"(FR-032)와 다르다. 놓인 가구에 포커스를 두고 방향키로 2%씩 옮기기 — 접근성에는 좋지만 spec 범위 밖이라(constitution VII "범위 밖 기능은 열린 질문으로") 팀이 정하면 같은 `placeFurniture`로 더한다.

## R12. 미니룸 안 그리기 순서와 크기

**Decision**: 가구는 `y`가 작은 것부터 그려 아래쪽 가구가 앞에 보인다(spec 기본값). 캐릭터는 늘 가구 앞에 그린다. 가구 그림 크기는 미니룸 높이에 대한 비율(가구마다 정한 값)로 정해 화면 너비가 달라도 같은 비율로 보인다. 광장(Phaser) 집 겉모습에는 가구를 그리지 않는다(FR-036, 집 그림은 `houseSvg`로 따로 그린다 [코드]).

**Rationale**: spec은 가구끼리의 겹침만 정했다. 캐릭터가 침대 뒤로 숨으면 "내 캐릭터"가 안 보이므로 캐릭터를 앞에 둔다. 미니룸은 `bg-cover bg-bottom` 배너라 너비에 따라 보이는 배경 폭은 달라도, 가구 위치는 상자 기준 비율이라 같은 자리에 있다(FR-035).

**Alternatives considered**: 캐릭터도 `y`로 함께 정렬 — 캐릭터 위치가 고정이라 얻는 것이 적고 가려질 수 있다.

## R13. 정렬 6가지를 어디서 하나

**Decision**: 상점 화면(`ShopView`, 클라이언트)의 `useState`로 정렬을 들고, `src/lib/shop.ts`의 순수 함수 `sortShopItems(items, sort)`로 모든 구역을 다시 정렬한다. 주소(`?sort=`)나 저장소에 남기지 않는다 (spec 기본값: 새로 열면 늘 레벨순). 기준(원본 정렬표 그대로, 마지막 동률은 `id` 작은 순으로 고정):

| 정렬 | 1순위 | 같으면 |
|---|---|---|
| `레벨순` (기본) | 필요 레벨 낮은 순 | 가격 낮은 순 |
| `인기순` | 가진 회원 수 많은 순 | 가격 낮은 순 |
| `비싼 순` | 가격 높은 순 | 필요 레벨 낮은 순 |
| `싼 순` | 가격 낮은 순 | 필요 레벨 낮은 순 |
| `최신순` | `id` 큰 순 (원본 표 "`items.id` 큰 순") | - |
| `보유순` | 내가 가진 것 먼저 (`quantity > 0`) | 가격 낮은 순 |

"가진 회원 수"는 `SELECT item_id, COUNT(*) FROM user_items WHERE quantity > 0 GROUP BY item_id` 한 번으로 구해 목록에 붙인다 (성장 아이템도 "지금 1개 이상 가진 회원 수", spec 기본값).

**Rationale**: 정렬을 바꿀 때 서버 왕복이 없고, 기억하지 않는다는 기본값이 저절로 지켜진다. 순수 함수라 SC-008을 단위 테스트(`scripts/test-shop.ts`)로 100% 확인할 수 있다. 정렬 고르기는 `<select aria-label="정렬">` 하나로 해 375px에서 넘치지 않는다.

**Alternatives considered**: 서버에서 `ORDER BY` + `?sort=` — 새로고침해도 유지되어 기본값과 다르고, 6가지 ORDER BY를 쿼리마다 만들어야 한다. 아이템마다 상관 하위 쿼리로 회원 수 세기 — `user_items` PK가 (`user_id`, `item_id`)라 `item_id`만으로는 인덱스를 못 타 아이템 수만큼 전체를 읽는다 [코드].

## R14. 버튼 상태와 카드 표시

**Decision**: `buttonState(item, { level, coins })` 순수 함수가 `owned` → `locked` → `short` → `buy` 순서로 하나를 고른다. 성장 아이템은 `owned`를 건너뛴다(FR-006). 버튼 글자 `보유 중` / `🔒 Lv.N` / `코인 부족` / `사기`, 보유 중이면 카드 흐림(지금 `opacity-70`). 가격 옆 `Lv.N+`는 필요 레벨 2 이상만. 버튼은 `min-h-11`(44px)과 `whitespace-nowrap`, 눌러도 되는 상태가 아니면 `disabled`. 카드 칸 수는 지금 클래스(`grid-cols-2 sm:grid-cols-3 lg:grid-cols-4`)가 FR-005와 같아 그대로 둔다. 구매 결과 문구는 지금처럼 그 구역(`ShopGrid`)마다 따로 들고 그 구역 위에 보인다(FR-008).

**Rationale**: 지금 버튼은 `py-1.5 text-sm`이라 높이가 약 32px로 FR-046(44px)에 못 미친다 [코드]. 우선순위를 함수로 빼면 화면과 단위 테스트가 같은 규칙을 쓴다.

## R15. 새 그림 자산

**Decision**: `src/lib/art/avatar.ts`(모자 3·옷 3·소품 3), `src/lib/art/furniture.ts`(화분·의자·램프·책상·침대), `src/lib/art/growth.ts`(동물 먹이·고급 먹이·성장 촉진제)를 코드 SVG로 만든다. `asset_key`는 `hat.straw`, `outfit.hoodie`, `acc.glasses`, `furn.chair`, `growth.feed`처럼 부위·종류 앞말을 붙인다. 상점·꾸미기 카드(`ItemArt`)에서 아바타 아이템은 회색 방문자 몸(`VISITOR_CHARACTER`) 위에 겹쳐 입은 모습을 보여 준다.

**Rationale**: CLAUDE.md 규칙 "DB에는 `asset_key`만, 실제 모양은 `src/lib/art/`의 SVG, 외부 그림 파일을 쓰지 않는다" [코드]. 직접 그린 그림이라 constitution VII의 에셋 출처·상업 이용 걱정이 없다. 아이소메트릭(TOWN-05)으로 바뀌면 같은 키로 다시 그린다(spec 기본값).

## R16. 성장 아이템을 쓰는 동작과의 경계 (town)

**Decision**: shop은 `consumeGrowthItem(tx, userId, itemId)`를 `src/server/inventory.ts`에 제공한다. `UPDATE user_items SET quantity = quantity − 1 WHERE user_id = ? AND item_id = ? AND quantity > 0 AND <아이템이 growth> RETURNING`으로 하나 줄이고, 줄였으면 `{ name, growthValue }`, 못 줄였으면 `null`을 돌려준다. town의 농장 Server Action이 자기 `lockUser` 트랜잭션 안에서 이것을 부르고 `addGrowth`로 동물을 키운다. 사용 기록 표는 두지 않는다(ERD 3.11).

**Rationale**: `user_items`는 shop 담당 표라 수량 규칙(0 아래 금지, 성장 아이템만)을 shop 코드 한 곳에 둔다. 조건부 UPDATE + CHECK `quantity >= 0`이 동시에 써도 0 아래로 내려가지 않게 이중으로 막는다(FR-044, SC-009). 하루 사용 횟수 제한이 없으므로 날짜 기록이 필요 없다. 이름을 `use…`로 시작하지 않는 것은 `eslint-config-next`의 React Hooks 규칙이 `use`로 시작하는 함수를 훅으로 보고 Server Action 안에서 부르면 오류를 내기 때문이다.

**Alternatives considered**: town이 `user_items`를 직접 UPDATE — 같은 규칙이 두 곳에 생긴다.

## R17. 화면 갱신

**Decision**: `buyItem`(성공 때), `equipItem`, `unequipAvatar`, `placeFurniture`, `removeFurniture`는 모두 `revalidatePath("/", "layout")`을 부른다.

**Rationale**: 루트 레이아웃(헤더)은 화면 이동만으로 다시 그려지지 않으므로 코인·캐릭터가 바뀌는 Server Action은 이렇게 부르는 것이 프로젝트 규칙이다 [코드 CLAUDE.md]. 가구는 헤더를 바꾸지 않지만 같은 규칙으로 두어 블로그 홈의 이전 화면 캐시도 비운다. 이 앱에는 `use cache`·`unstable_cache`가 없어 모든 화면이 요청마다 그려진다 [코드]. Next.js 16 `revalidatePath` 동작은 구현 전 확인.

## R18. 검증 방법

**Decision**:
- 단위: `scripts/test-shop.ts`(프로젝트 관례: `expect(name, got, want)`, 실패 시 `process.exit(1)`) — `buttonState` 우선순위, `shortage`, `sortShopItems` 6가지와 동률, `sectionOf`, 구매 문구, `orderOutfit`(겹침 순서), `nextFreeSlot`·상한. `package.json`에 `test:shop`을 더하고 `test` 체인 끝에 붙인다.
- E2E: 새 `e2e/shop.mjs`, `e2e/closet.mjs`. 실행마다 새 회원을 만들고(`` `shop${Date.now() % 100_000_000}` ``), `pg`로 코인·레벨을 준비하고 결과 행을 센다. game의 자동 출석(game research R1: `getViewer()`가 그날 첫 화면에서 출석 보상을 넣는다)이 들어간 뒤에 원장 합계를 목표 값으로 맞춰야 "잔액 = 가격" 같은 준비가 어긋나지 않는다. 조작 요청과 동시 요청은 `e2e/params.mjs`의 "진짜 Server Action 요청을 한 번 잡아 인자만 바꿔 다시 보내기"(`next-action` 헤더 재사용) 방식을 쓴다 [코드].
- SC-003(1,000번): 성장 아이템 구매 요청 재생을 동시에 여러 개씩 보내고, 그 사이 보상은 e2e가 같은 advisory lock을 건 트랜잭션으로 `point_ledger`에 넣는다. 끝나면 회원별 `SUM(coin_delta) OVER (ORDER BY id)`의 최솟값 ≥ 0, 헤더 코인 = 원장 합계, `quantity × 가격 = −구매 합계`를 확인한다.
- 광장은 canvas라 DOM으로 차림을 읽을 수 없다. `data-look` 속성(그림 컴포넌트가 `lookKey`를 붙임)으로 헤더·미니룸을 비교하고, 광장은 town에 요청한 `data-player-look`이 있으면 그것으로, 없으면 스크린샷으로 확인한다.

**Rationale**: 프로젝트에 테스트 러너가 없고 위 방식이 기존 관례다 [코드 `scripts/test-*.ts`, `e2e/*.mjs`]. 원장 id는 같은 회원 쓰기를 모두 같은 잠금으로 줄 세우면 커밋 순서와 같아져, 누적합으로 "0 아래로 내려간 순간"을 셀 수 있다.

## R19. spec에 문구가 없는 곳의 기본값

constitution III("모르는 것은 지어내지 않고 열린 질문에 적는다", "사용자에게 보여줄 문구는 spec에 그대로 적는다")에 따라 spec·원본에 있는 문구를 먼저 다시 쓰고, 없으면 기본값을 표시해 팀 확인으로 넘긴다.

| 자리 | 기본값 | 근거 |
|---|---|---|
| 벗기·가구 놓기·옮기기·빼기 성공 | `{아이템 이름} 장착을 저장했어요 ✓` (FR-024 문구를 그대로) | spec이 저장 성공 문구를 이것 하나만 정했고 SC-005가 "누르면 저장 문구 2초"를 요구한다. FR-027이 가구 배치를 "장착"의 한 종류로 부른다 |
| 꾸미기의 네트워크·서버 오류 (배치 포함) | `장착하지 못했어요. 잠시 뒤 다시 시도해 주세요` | FR-025, FR-041 |
| 가구가 아닌 아이템 배치 요청 (조작) | `아직 장착할 수 없는 종류예요` | FR-027 문구 재사용 |
| 범위 밖·숫자가 아닌 아이템 ID | 구매 `살 수 없는 아이템이에요`, 장착·배치 `가지고 있지 않은 아이템이에요` | 없는 아이템 = 살 수 없고 가지고 있지 않다 (FR-009, FR-026) |
| 잘못된 좌표 값 (조작, `placeFurniture`의 `x`·`y`) | `잘못된 요청이에요` | 기존 코드 관례 (`src/app/farm/actions.ts`, `e2e/params.mjs`). 부위는 서버가 아이템에서 정하므로 받는 인자가 없다 |
| 상점 안내 문구 (FR-002 "안내 문구") | `글을 쓰고 출석해서 모은 코인으로 내 캐릭터와 미니룸을 꾸며 보세요.` | 지금 코드 문구(`src/app/shop/page.tsx`의 `…새 친구와 배경을 데려가세요.`)는 캐릭터 판매 중단과 맞지 않는다 [코드]. 원본 SHOP-01에는 안내 문구 글자가 없다. `specs/README.md` "원본 문서에서 고칠 곳"의 "캐릭터 판매 중단 뒤 남은 문구"와 같은 종류 |
| 성장 아이템 카드의 가진 개수 (FR-043) | `보유 N개`, 0개여도 늘 보인다(`보유 0개`) | spec에 형식이 없다. FR-043은 "지금 가진 개수를 보여 주어야"라고만 적어 0개일 때 숨길 근거가 없다 |
| 상점 구매 중 네트워크·서버 오류 (Server Action이 throw) | 문구가 정해지기 전까지 지금 동작 그대로(잡지 않아 오류 화면 `src/app/error.tsx`가 뜬다). 트랜잭션이라 코인·보유는 바뀌지 않는다(AC 2-10) | spec에 상점 오류 문구도, "오류 화면으로 넘어가지 않아야"라는 요구도 없다(꾸미기만 FR-025). 꾸미기처럼 잡을지와 그 문구는 팀 확인 |
| 첫 출시 아이템 17종의 설명(`description`) | 구현 때 카드 2줄 안으로 쓰고 PR에서 팀이 확인한다 | 화면에 보이는 글자지만 spec·원본에 설명 문구가 없다 (기존 17종은 지금 시드 설명 그대로) |
| 성장 아이템 구역 이름 | `🌱 성장 아이템` | spec 기본값 |
| 성장 아이템 구매 성공 | `🎉 {이름}을(를) 샀어요! 동물 농장에서 써 보세요.` | spec 기본값 |
| 꾸미기 맨 아래 | `더 많은 꾸미기 아이템과 배경은 상점에서 만날 수 있어요.` | spec 기본값 (FR-030) |
| 꾸미기에서 가진 것이 없는 구역 (`👕 아바타 꾸미기`, `🪑 가구`) | 구역 제목은 늘 보이고, 카드 자리는 비워 둔다. 빈 구역 안내 문구는 지어내지 않는다 (`🖼 내 배경`은 늘 1개 이상) | FR-022는 `👕`·`🖼`·`🪑` 세 구역을 조건 없이 적고 `🐾 내 캐릭터`만 조건부로 적었다. 맨 아래 안내가 상점으로 이끈다. 빈 구역 안내 문구를 둘지는 팀 확인 |
| `🐾 내 캐릭터` 구역 위치 | 가진 캐릭터가 2개 이상일 때 `👕 아바타 꾸미기` 앞 | FR-022는 조건만 정했다 |
| 놓인 가구 카드 | 장착 카드와 같이 `✓` + 노란 테두리 | FR-023 |

## R20. 함께 고칠 문서

**Decision**: `docs/02-erd.md`의 shop 담당 부분을 같은 PR에서 고친다 — 1장 관계도(`items`·`user_items` 컬럼, `avatar_equips`·`room_furniture`와 관계), 2장 아이템 그룹, 3.4(착용·배치도 같은 복합 FK), 3.7 복합 PK 표, 3.11 성장 아이템 ⏳ 해제, 3.14 삭제 규칙(회원 삭제 → 착용·배치), 3.15 열거형(`item_type`, `avatar_slot`), 3.17 2NF 표, 5장 "아바타 꾸미기 테이블 설계 전" 해결, 7장 6-2의 shop 부분, 부록 NULL 허용(`items.avatar_slot`)과 글자 길이. `docs/erdcloud-import.sql`(ERD 목표 설계를 담은 가져오기 SQL, "테이블 25개")에도 두 표와 컬럼을 더한다. `docs/erdcloud-final.sql`은 제출본 내보내기라 건드리지 않는다. 코드 저장소 `README.md` 스크립트 표에 `test:shop`, `e2e/shop.mjs`, `e2e/closet.mjs`를 한 줄씩 더한다.

**Rationale**: 테이블 담당 규칙 5(담당 spec이 ERD의 그 테이블 부분을 함께 고친다), CLAUDE.md "스키마 변경 → `docs/02-erd.md`도 함께".

**Alternatives considered**: ERDCloud 가져오기 SQL을 그대로 두기 — ERD 문서와 표 개수가 어긋난다. 갱신할지는 팀이 정할 수 있게 열린 항목으로도 남긴다.
