# Contract: 다른 spec이 쓰는 모듈 (서버 도우미·순수 규칙·그림·컴포넌트)

화면·Server Action이 아니라 **spec 사이의 경계**다. 소유는 공통 모듈 소유 규칙을 따른다: shop 소유 파일은 shop이 바꾸고, 다른 spec 소유 파일에는 필드·prop·호출을 "추가만" 한다. 모든 새 인자·prop은 선택이며, 넘기지 않으면 지금과 똑같이 동작한다(기존 호출을 고치지 않아도 깨지지 않는다).

## 1. 서버 도우미 — `src/server/inventory.ts` (shop 소유, `import "server-only"`)

| 함수 | 모양 | 쓰는 곳 | 약속 |
|---|---|---|---|
| `listShopItems` | `(userId: string) => Promise<ShopItem[]>` | shop `/shop` | `is_on_sale = true`만. `quantity`(내 보유 수량, 없으면 0), `ownerCount`(보유 수량 ≥ 1인 회원 수, `GROUP BY` 한 번) |
| `listOwnedItems` | `(userId: string) => Promise<OwnedItem[]>` | shop `/closet` | `quantity > 0`, 성장 아이템 제외. 판매 중단 아이템도 포함 (D12) |
| `getEquipped` | `(userId: string) => Promise<{ characterItemId; backgroundItemId; avatar: Partial<Record<"hat" \| "outfit" \| "accessory", number>>; furniture: { itemId; x; y }[] }>` | shop `/closet` | 지금 함수에 착용·배치를 더함 |
| `getMiniRoomDecor` | `(userId: string) => Promise<{ outfit: string[]; furniture: { assetKey: string; name: string; x: number; y: number }[] }>` | **blog** 블로그 홈 (`src/app/blog/[slug]/page.tsx`), shop `/closet` | `furniture`는 `y` 오름차순(그리는 순서). 누구나 볼 수 있는 정보만 (가진 아이템 목록·코인은 주지 않음) |
| `outfitOf` | `(userIdColumn: AnyPgColumn) => SQL<string[]>` | **auth** `getViewer`, **town** `getTownHouses`·`getMyHouse` | select 필드로 끼우는 SQL 조각: `ARRAY(SELECT i.asset_key FROM avatar_equips ae JOIN items i ON i.id = ae.item_id WHERE ae.user_id = <col>)`. 순서는 보장하지 않는다 (그리는 순서는 그림 모듈이 정함) |
| `consumeGrowthItem` | `(tx: Tx, userId: string, itemId: number) => Promise<{ name: string; growthValue: number } \| null>` | **town** 농장 성장 아이템 사용 (TOWN-09 FR-048) | 부르는 쪽이 `lockUser(tx, userId)`를 건 트랜잭션 안에서 부른다. `type = 'growth'`이고 `quantity > 0`일 때만 1 줄이고 성장치를 돌려준다. 아니면 `null` (부르는 쪽이 거부 문구를 정함). 원장은 쓰지 않는다 (사용에는 보상 없음) |
| `listItemsUnlockedBetween` | `(fromLevel: number, toLevel: number) => Promise<{ id; name; type; assetKey; price; requiredLevel }[]>` | **game** 레벨업 팝업 (GAME-06, game research R12 "안 본 레벨 L1~L2 사이(포함)에 필요 레벨이 있는 판매 중 아이템") | `is_on_sale = true AND required_level BETWEEN fromLevel AND toLevel`, 필요 레벨 → 가격 → ID 순. 캐릭터는 판매 중이 아니라 저절로 빠진다. 한 레벨만 올랐으면 `from = to`. game이 `items.is_on_sale`을 직접 읽어도 된다 (이 함수는 선택) |

`ShopItem`·`OwnedItem` 타입은 같은 파일에서 내보낸다. 필드는 [shop.md](shop.md) 1.1, [closet.md](closet.md) 1.1.

## 2. 순수 규칙 — `src/lib/shop.ts` (shop 소유, DB 없음, 화면·서버 공용)

| 이름 | 모양 | 약속 (단위 테스트 `scripts/test-shop.ts`) |
|---|---|---|
| `SHOP_SECTIONS` | `[{ type: "avatar", title: "👕 아바타 꾸미기" }, { type: "furniture", title: "🪑 가구" }, { type: "background", title: "🖼 배경" }, { type: "growth", title: "🌱 성장 아이템" }]` | 상점 구역 순서와 제목 (FR-003) |
| `SHOP_SORTS` | `"level" \| "popular" \| "priceDesc" \| "priceAsc" \| "newest" \| "owned"`과 화면 이름 `레벨순`·`인기순`·`비싼 순`·`싼 순`·`최신순`·`보유순` | 기본 `level` (FR-014~015) |
| `sortShopItems` | `(items, sort) => items` (새 배열) | research R13 표. 마지막 동률은 `id` 작은 순 |
| `buttonState` | `(item, { level, coins }) => "owned" \| "locked" \| "short" \| "buy"` | `owned` → `locked` → `short` → `buy`, 성장 아이템은 `owned` 없음, 코인 = 가격이면 `buy` (FR-006) |
| `shortage` | `(price, coins) => number` | `price − coins` (FR-019 문구의 N) |
| `purchaseMessage` | `(kind: "decor" \| "growth", name) => string` | [shop.md](shop.md) 1.4 문구 |
| `FURNITURE_LIMIT` | `5` | FR-033 |
| `nextFreeSlot` | `(usedSlots: number[]) => number \| null` | 1~5 중 비어 있는 가장 작은 번호, 없으면 null |
| `DEFAULT_SPOTS` | 키보드·클릭으로 놓을 때 기본 자리 5곳 | [closet.md](closet.md) 1.3 |

## 3. 그림 — `src/lib/art/` (shop 소유)

| 파일·이름 | 모양 | 쓰는 곳 | 약속 |
|---|---|---|---|
| `characters.ts` `characterSvg` | `(assetKey, size = 64, outfit: string[] = []) => string` | 모든 캐릭터 그림 | `outfit`이 비면 지금과 같은 SVG. 있으면 몸 위에 옷 → 소품 → 모자 순서로 겹친다 (FR-040). 모르는 키는 무시 |
| `characters.ts` `characterDataUri` | `(assetKey, size = 64, outfit: string[] = []) => string` | `CharacterArt`·`CharacterBadge`, **town** `scene.ts` 텍스처 | 위와 같음 |
| `characters.ts` `lookKey` | `(assetKey, outfit: string[] = []) => string` | **town** `scene.ts` 텍스처 키, 검증용 `data-look` | 차림 순서와 상관없이 같은 차림이면 같은 문자열 (정렬해서 이음). 차림이 없으면 `assetKey` 그대로 |
| `avatar.ts` `AVATAR_PARTS`, `orderOutfit` | `asset_key → { slot, svg }`, `(outfit) => outfit`(겹치는 순서) | `characterSvg`, `ItemArt` | 9종 (`hat.straw` … `acc.bag`) |
| `furniture.ts` `furnitureSvg` | `(assetKey) => string` + 가구마다 미니룸 높이 대비 크기 | `MiniRoom`, `ItemArt` | 5종 (`furn.pot` … `furn.bed`) |
| `growth.ts` `growthSvg` | `(assetKey) => string` | `ItemArt`, **town** 농장 화면(쓰고 싶으면) | 3종 (`growth.feed`, `growth.premium`, `growth.booster`) |
| `src/lib/assets.ts` | 위 함수를 다시 내보냄 | 공통 | 지금 내보내기 유지 |

## 4. 컴포넌트

| 컴포넌트 (파일) | 소유 | 바뀌는 점 | 쓰는 곳 |
|---|---|---|---|
| `CharacterArt`, `CharacterBadge` (`src/components/character.tsx`) | 그리기는 shop | 선택 prop `outfit?: string[]`. 그린 `<img>`에 `data-look={lookKey(asset, outfit)}` (검증용) | 헤더(town), 휴대폰 간단 메뉴(town), 미니룸 |
| `MiniRoom` (`src/components/character.tsx`) | **blog** (shop은 추가만) | 선택 prop `outfit?: string[]`, `furniture?: { assetKey; name; x; y }[]`(아래쪽이 앞), `children?`(꾸미기의 끌기 층). 캐릭터는 가구 앞. 넘기지 않으면 지금과 같음. blog가 더하는 `showcase`(전시 동물, 캐릭터 오른쪽, blog research)와 겹치는 순서·자리는 blog와 같은 PR에서 맞춘다 (가구는 맨 뒤 층, 전시 동물·캐릭터는 그 앞) | 블로그 홈(blog), 꾸미기(shop) |
| `ItemArt` (`src/components/item-art.tsx`) | shop | `type`이 `avatar`(회색 몸 + 겹침), `furniture`, `growth`일 때 그림 | 상점, 꾸미기 |

## 5. 다른 spec 소유 파일에 끼울 줄 (공통 모듈 추가)

| 파일 | 소유 | 끼울 것 |
|---|---|---|
| `src/server/dal.ts` `getViewer` | auth | select에 `outfit: outfitOf(profiles.userId)` → `viewer.profile.outfit` |
| `src/server/town.ts` `getTownHouses`, `getMyHouse` | town | select에 `outfit: outfitOf(profiles.userId)` |
| `src/components/town/types.ts` | town | `TownHouse.outfit: string[]`, `TownData.player.outfit: string[]` |
| `src/app/town/page.tsx` | town | `player`에 `outfit: member.profile.outfit` |
| `src/components/site-header.tsx`, `src/components/town/town-menu.tsx` | town | `<CharacterBadge … outfit={…} />` |
| `src/app/blog/[slug]/page.tsx`, `src/components/blog/blog-header.tsx` | blog | `getMiniRoomDecor(blog.ownerId)`를 부르고 `MiniRoom`에 `outfit`·`furniture` 전달 |
| `src/components/town/scene.ts` | town (**요청**: 함수 모양 변경) | `charKey(asset)` → `charKey(lookKey(asset, outfit))`, `characterDataUri(asset, size, outfit)`. 플레이어와 집 문 옆 캐릭터 모두. 검증용으로 `TownGame` 바깥 요소에 `data-player-look` 속성 하나 |
