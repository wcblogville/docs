# Contract: 꾸미기 (`/closet`, 장착·착용·가구 배치)

SHOP-04·05·06, FR-021~041, FR-046~048. 결정 근거는 [research.md](../research.md) R6~R12·R17·R19.

이 기능에는 Route Handler가 없다. 화면 라우트 1개와 Server Action 4개다.

## 1. 화면 라우트 `/closet`

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/closet/page.tsx` (Server Component) → `closet-view.tsx` (클라이언트) + `furniture-room.tsx` (클라이언트, 가구 끌기) |
| 접근 | 회원 전용, `requireMember()`. 로그인하지 않았으면 `redirect("/")` |
| 주소 인자 | 없음 |
| 들어오는 길 | 내 블로그 홈 정보 줄의 [🎨 꾸미기] (`src/components/blog/blog-header.tsx`, 주인에게만, blog 소유) |
| 나가는 길 | 헤더 `← 광장으로 나가기` |
| 페이지 제목 | `metadata.title = "꾸미기"` |

### 1.1 서버가 읽는 데이터 (`src/server/inventory.ts`, `src/server/points.ts`)

| 함수 | 돌려주는 것 |
|---|---|
| `listOwnedItems(userId)` | 내가 가진(`quantity > 0`) 아이템 `{ id, type, avatarSlot, name, assetKey }` (성장 아이템 제외) |
| `getEquipped(userId)` | `{ characterItemId, backgroundItemId, avatar: { hat?, outfit?, accessory? }(아이템 ID), furniture: [{ itemId, x, y }] }` |
| `getWallet(userId)` | 레벨 진행 막대용 |

### 1.2 화면 구성

| 자리 | 내용 | 근거 |
|---|---|---|
| 제목 | `🎨 꾸미기` | FR-022 |
| 레벨 | `Lv.N`, `현재 / 필요 EXP`(최고 레벨은 `MAX`), 초록 막대, [경험치·코인 내역 보기] → `/wallet` (지금 그대로) | FR-022, game FR-011·032 |
| 미니룸 | 장착 배경 + 놓인 가구(아래쪽이 앞) + 장착 캐릭터(차림 겹침, 늘 맨 앞) + 닉네임. 높이 240px | FR-022, FR-040 |
| 상태 줄 | `role="status" aria-live="polite"` 한 줄. 성공 문구는 2초 뒤 사라짐 | FR-024, SC-005 |
| 구역 `🐾 내 캐릭터` | 가진 캐릭터가 **2개 이상**일 때만, 가입 때 고른 것 포함 모두 | FR-022, FR-011 |
| 구역 `👕 아바타 꾸미기` | 가진 아바타 아이템 (모자·옷·소품). 가진 것이 없어도 구역 제목은 보이고 카드 자리는 비어 있다 (빈 구역 안내 문구는 spec에 없어 넣지 않음, R19) | FR-022, FR-038 |
| 구역 `🖼 내 배경` | 가진 배경 (초원 포함, 늘 1개 이상) | FR-022 |
| 구역 `🪑 가구` | 가진 가구. 가진 것이 없어도 구역 제목은 보인다 (위와 같음) | FR-022, FR-032 |
| (town이 끼움) | `🏠 지붕 색` 칸 (TOWN-07, town 소유 컴포넌트) | town FR-039 |
| 맨 아래 | `더 많은 꾸미기 아이템과 배경은 상점에서 만날 수 있어요.` (`상점`은 `/shop` 링크) | FR-030 (spec 기본값) |
| 칸 수 | 모바일 3 · 640px 이상 4 · 1024px 이상 6 | FR-022 |

**카드**: `<button>` 하나 = 그림 + 이름. 장착·입은·놓인 카드는 이름 앞 `✓`, 노란 테두리·배경, `aria-pressed="true"` (FR-023). 키보드 포커스가 눈에 보인다(`focus-visible` 테두리, FR-047). 누르는 영역 44×44px 이상. 카드도 버튼이므로 이름 글자는 한 줄로 두고(`whitespace-nowrap`, 375px 3칸에서 넘치면 `truncate` + `title`에 전체 이름) 두 줄로 쪼개지지 않게 한다 (FR-046, SC-007. 예: `✓ 밤의 도시`).

### 1.3 누르면 일어나는 일 (낙관적 갱신)

| 누른 카드 | 화면 먼저 | 부르는 Server Action |
|---|---|---|
| 캐릭터 / 배경 (장착 중 아님) | 미니룸을 바로 바꿈 | `equipItem(itemId)` |
| 캐릭터 / 배경 (장착 중) | 그대로 (벗기 없음) | 부르지 않음 |
| 아바타 (그 부위에 안 입음 또는 다른 것 입음) | 미니룸 캐릭터에 바로 입힘 (같은 부위는 바뀜) | `equipItem(itemId)` |
| 아바타 (입은 것) | 바로 벗김 | `unequipAvatar(itemId)` |
| 가구 (안 놓임) | 기본 자리에 놓음 (키보드·클릭 대안). 기본 자리는 `src/lib/shop.ts`의 `DEFAULT_SPOTS` 중 비어 있는 첫 곳 — `y 92`에 `x` 15 · 30 · 70 · 85 · 50 순서 (가운데 캐릭터를 피함) | `placeFurniture(itemId, x, y)` |
| 가구 (놓임) | 미니룸에서 뺌 | `removeFurniture(itemId)` |

- 성공(`ok: true`): 상태 줄에 `{아이템 이름} 장착을 저장했어요 ✓`를 2초 (벗기·놓기·빼기도 같은 문구, R19 기본값).
- 실패(`ok: false`): 화면을 누르기 전으로 되돌리고 서버 `error`를 보여 준다.
- 예외(네트워크·서버 오류로 Server Action이 throw): 되돌리고 `장착하지 못했어요. 잠시 뒤 다시 시도해 주세요`. 오류 화면(`error.tsx`)으로 넘어가지 않는다 (FR-025, FR-041).
- 처리 중에는 카드를 잠근다 (지금처럼). 문구가 떠 있는 동안 다른 카드를 누르면 마지막으로 누른 것 기준으로 바뀐다.

### 1.4 가구 끌어다 놓기 (`furniture-room.tsx`)

| 동작 | 결과 | Server Action |
|---|---|---|
| `🪑 가구` 카드를 눌러 끌고 미니룸 안에서 놓기 | 놓은 지점(아랫변 가운데)을 미니룸 상자 기준 %로 바꿔 놓음 | `placeFurniture(itemId, x, y)` |
| 같은 동작을 미니룸 밖에서 놓기 | 아무 일 없음 | - |
| 놓인 가구를 끌어 미니룸 안에서 놓기 | 옮김 | `placeFurniture(itemId, x, y)` |
| 놓인 가구를 끌어 미니룸 밖에서 놓기 | 뺌 | `removeFurniture(itemId)` |
| 6번째 놓기 | 놓이지 않음, `가구는 5개까지 놓을 수 있어요` | 서버가 거부 (화면도 5개면 미리 막음) |

- 마우스·터치·펜 모두 Pointer Events(`pointerdown`/`pointermove`/`pointerup`, `setPointerCapture`), 끄는 요소는 `touch-action: none` (R11). 누른 뒤 움직이지 않고 떼면 끌기가 아니라 1.3의 "누르기"(기본 자리에 놓기 / 빼기)로 처리한다.
- 키보드는 1.3의 Enter(놓기·빼기)까지만 한다. 방향키로 옮기기는 spec 범위 밖이라 넣지 않는다 (R11, 팀 확인 항목).
- 화면은 좌표를 0~100으로 잘라 보낸다. 서버는 범위를 다시 검사한다.

## 2. Server Actions (`src/app/closet/actions.ts`, `"use server"`)

공통

```ts
export type ClosetResult = { ok: true; name: string } | { ok: false; error: string };
```

- 권한: 모두 `requireMember()` (로그인하지 않았으면 `redirect("/")`). 대상은 언제나 **나**(`viewer.userId`)이고 다른 회원 ID를 받지 않는다.
- 아이템 ID: `parseId()`. null이면 `가지고 있지 않은 아이템이에요` (없는 아이템은 가지고 있지 않다, R19).
- "가졌다" = 내 `user_items` 행의 `quantity > 0`.
- 성공하면 `revalidatePath("/", "layout")` (R17).

### 2.1 `equipItem(itemId: number): Promise<ClosetResult>`

| # | 확인 | 실패 문구 | FR |
|---|---|---|---|
| 1 | `parseId` 통과, 내가 가짐 | `가지고 있지 않은 아이템이에요` | FR-026, FR-041 |
| 2 | 종류가 character / background / avatar | `아직 장착할 수 없는 종류예요` (furniture, growth) | FR-027 |
| 3 | character → `UPDATE profiles SET character_item_id` / background → `UPDATE blogs SET background_item_id` / avatar → `INSERT avatar_equips (user_id, slot = items.avatar_slot, item_id) ON CONFLICT (user_id, slot) DO UPDATE SET item_id, equipped_at` | (복합 FK 위반이면 throw → 화면이 1.3 예외 처리) | FR-028, FR-038~039 |

성공: `{ ok: true, name }`. 판매가 끝난 캐릭터도 가지고 있으면 장착된다 (FR-011, AC 1-8).

### 2.2 `unequipAvatar(itemId: number): Promise<ClosetResult>`

| # | 확인 | 실패 문구 |
|---|---|---|
| 1 | `parseId` 통과, 내가 가짐 | `가지고 있지 않은 아이템이에요` |
| 2 | 종류가 avatar | `아직 장착할 수 없는 종류예요` |
| 3 | `DELETE FROM avatar_equips WHERE user_id = 나 AND item_id = ?` | - (입고 있지 않았어도 성공, 여러 번 보내도 같음) |

FR-039 "입은 아이템을 다시 누르면 벗는다". 서버는 토글하지 않는다 (R7).

### 2.3 `placeFurniture(itemId: number, x: number, y: number): Promise<ClosetResult>`

입력 검증: zod `x`, `y` = 유한한 숫자, 0 이상 100 이하. 아니면 `잘못된 요청이에요`. 소수 둘째 자리에서 반올림.

| # | 확인 (트랜잭션, `lockUser` 뒤) | 실패 문구 | FR |
|---|---|---|---|
| 1 | `parseId` 통과, 내가 가짐 | `가지고 있지 않은 아이템이에요` | FR-034 |
| 2 | 종류가 furniture | `아직 장착할 수 없는 종류예요` (R19) | - |
| 3 | 이미 놓인 가구면 `UPDATE room_furniture SET x, y, updated_at` (옮기기) | - | FR-032 |
| 4 | 아니면 1~5 중 비어 있는 가장 작은 자리 번호. 없으면 거부 | `가구는 5개까지 놓을 수 있어요` | FR-033 |
| 5 | `INSERT room_furniture (user_id, item_id, slot, x, y)` | (UNIQUE 위반 23505 → `가구는 5개까지 놓을 수 있어요`) | FR-033 |

### 2.4 `removeFurniture(itemId: number): Promise<ClosetResult>`

| # | 확인 | 실패 문구 |
|---|---|---|
| 1 | `parseId` 통과, 내가 가짐 | `가지고 있지 않은 아이템이에요` |
| 2 | `DELETE FROM room_furniture WHERE user_id = 나 AND item_id = ?` | - (놓여 있지 않았어도 성공) |

## 3. 결과가 보이는 다른 화면 (다른 spec 소유, 이 spec은 데이터·그림만 제공)

| 화면 | 보이는 것 | 소유 | 연결 |
|---|---|---|---|
| `/@{주소}` 블로그 홈 미니룸 (누가 보든) | 배경 + 가구(같은 비율 위치) + 캐릭터·차림 + 닉네임 | blog | `getMiniRoomDecor(ownerId)` → `MiniRoom`의 `outfit`·`furniture` prop ([shared-modules.md](shared-modules.md)) |
| 헤더 캐릭터 얼굴 (모든 화면) | 캐릭터·차림 | town (헤더), auth (`getViewer`) | `getViewer().profile.outfit` → `CharacterBadge outfit` |
| `/town` 광장 내 캐릭터, 집 문 옆 주인 캐릭터 | 캐릭터·차림. 집 지붕 색은 지붕 색을 고르지 않았으면 배경 색 | town | `TownHouse.outfit`, `player.outfit`, `lookKey`·`characterDataUri(asset, size, outfit)` |
| `/town` 광장 집 겉모습 | 가구 없음 | town | (그리지 않음, FR-036) |

## 4. 거부되면 바뀌지 않는 것

`profiles.character_item_id`, `blogs.background_item_id`, `avatar_equips`, `room_furniture`, `user_items`, `point_ledger` (SC-004).
