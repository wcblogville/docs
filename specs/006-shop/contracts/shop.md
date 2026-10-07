# Contract: 상점 (`/shop`, `buyItem`)

SHOP-01·02·03, SHOP-05(가구 구매), SHOP-06(아바타 구매), FR-001~020, FR-031, FR-037, FR-042~049. 결정 근거는 [research.md](../research.md) R3·R5·R13·R14·R19.

이 기능에는 Route Handler(HTTP API)가 없다. 바깥 인터페이스는 화면 라우트 1개와 Server Action 1개다.

## 1. 화면 라우트 `/shop`

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/shop/page.tsx` (Server Component) → `src/app/shop/shop-view.tsx` (클라이언트, 정렬) → `src/app/shop/shop-grid.tsx` (클라이언트, 구역마다 하나) |
| 접근 | 회원 전용. 페이지가 `requireMember()`를 부른다 (레이아웃에서 검사하지 않음) |
| 로그인하지 않음 | `requireUser()` → `redirect("/")` (첫 화면, 로그인·회원가입). 광장 상점 입구도 방문자에게는 `Space 로그인하고 이용하기` (town FR-022) |
| 프로필 없는 회원 | 지금 `requireMember()`가 `/onboarding`으로 보낸다. auth 단계 1(온보딩 없애기) 뒤에는 이런 회원이 없다 |
| 주소 인자 | 없음. 정렬은 주소에 남기지 않는다 (`?sort=` 없음, spec 기본값) |
| 들어오는 길 | 광장 🏪 상점 입구(`src/components/town/scene.ts`의 `entrances`, 휴대폰 간단 메뉴 `town-menu.tsx`), 레벨업 팝업 [상점 가기]·레벨업 알림(game) |
| 나가는 길 | 헤더 `← 광장으로 나가기` (`src/components/exit-button.tsx`) |
| 페이지 제목 | `metadata.title = "상점"` (지금 그대로) |

### 1.1 서버가 읽는 데이터

| 함수 (`src/server/inventory.ts`, `src/server/points.ts`) | 돌려주는 것 |
|---|---|
| `listShopItems(userId)` | `is_on_sale = true`인 아이템마다 `{ id, type, avatarSlot, name, description, price, requiredLevel, assetKey, growthValue, quantity(내 보유 수량, 없으면 0), ownerCount(보유 수량 1 이상인 회원 수) }` |
| `getWallet(userId)` | `{ level, coins, … }` — 원장 합계 |

### 1.2 화면 구성 (위에서 아래로)

| 자리 | 내용 | 근거 |
|---|---|---|
| 제목 | `🏪 마을 상점` | FR-002 |
| 안내 | 기본값 `글을 쓰고 출석해서 모은 코인으로 내 캐릭터와 미니룸을 꾸며 보세요.` (팀 확인 대상, R19) | FR-002 |
| 오른쪽 | `Lv.N 🪙 N` (코인 천 단위 쉼표) | FR-002, game FR-013 |
| 정렬 | `<select aria-label="정렬">` 선택지 순서 `레벨순`(처음 값) · `인기순` · `비싼 순` · `싼 순` · `최신순` · `보유순`. 바꾸면 모든 구역을 다시 정렬. 새로 열면 늘 `레벨순` | FR-014, FR-015 |
| 구역 1 | `👕 아바타 꾸미기` (type `avatar`: 모자·옷·소품) | FR-003, FR-037 |
| 구역 2 | `🪑 가구` (type `furniture`) | FR-003, FR-031 |
| 구역 3 | `🖼 배경` (type `background`, 초원 제외) | FR-003, FR-004 |
| 구역 4 | `🌱 성장 아이템` (type `growth`) | FR-003, spec 기본값 |
| 없음 | 캐릭터 구역, 되팔기·환불 버튼 | FR-003, FR-012 |

### 1.3 아이템 카드

| 요소 | 규칙 |
|---|---|
| 그림 | `ItemArt` 높이 112px. 아바타는 회색 몸 위에 겹친 모습, 가구·성장 아이템은 그 그림 |
| 이름 | `name` |
| 설명 | `description`, 2줄까지 (`line-clamp-2`) |
| 가격 | `🪙 {가격}`(천 단위 쉼표), 필요 레벨이 2 이상이면 옆에 `Lv.N+` |
| 가진 개수 | 성장 아이템만, 늘 기본값 `보유 N개` (0개면 `보유 0개`, R19) |
| 버튼 | 아래 표. `min-h-11`(44px), 글자 한 줄 |
| 흐림 | 버튼이 `보유 중`이면 카드가 흐려진다 |
| 칸 수 | 모바일 2 · 640px 이상 3 · 1024px 이상 4 |

**버튼 상태** (위가 먼저, `src/lib/shop.ts`의 `buttonState`)

| 상태 | 조건 | 버튼 | 누를 수 있나 |
|---|---|---|---|
| `owned` | 꾸미기 아이템이고 `quantity > 0` (성장 아이템은 이 상태 없음) | `보유 중` | 아니오 |
| `locked` | 지금 레벨 < 필요 레벨 | `🔒 Lv.{필요 레벨}` | 아니오 |
| `short` | 지금 코인 < 가격 | `코인 부족` | 아니오 |
| `buy` | 그 밖 (코인 = 가격 포함) | `사기` | 예 |

처리 중(`useTransition`의 pending)에는 그 구역 버튼을 모두 잠근다. 버튼 상태는 화면을 연 때의 값이고, 판단은 서버가 다시 한다.

### 1.4 구매 결과 문구

결과는 산 아이템이 있는 **그 구역 위**에 `role="status"`로 보인다. 다른 구역의 문구는 그대로 둔다.

| 결과 | 문구 |
|---|---|
| 꾸미기 아이템 성공 | `🎉 {이름}을(를) 샀어요! 꾸미기에서 장착해 보세요.` |
| 성장 아이템 성공 | `🎉 {이름}을(를) 샀어요! 동물 농장에서 써 보세요.` (spec 기본값) |
| 실패 | 서버가 돌려준 `error` 그대로 (2장) |
| 네트워크·서버 오류 (Server Action이 throw) | spec에 문구·요구가 없어 지금 동작 그대로 둔다: 잡지 않으므로 오류 화면(`src/app/error.tsx`)이 뜨고, 트랜잭션이라 코인·보유는 바뀌지 않는다. 꾸미기(FR-025)처럼 잡아서 문구로 보여 줄지는 열린 항목 (R19) |

성공하면 서버의 `revalidatePath("/", "layout")`로 헤더 코인·카드 상태(보유 중, 가진 개수)가 바로 새 값이 된다 (FR-008).

## 2. Server Action `buyItem`

```ts
// src/app/shop/actions.ts ("use server")
export type BuyResult =
  | { ok: true; name: string; kind: "decor" | "growth"; quantity: number } // quantity: 산 뒤 보유 수량
  | { ok: false; error: string };
export async function buyItem(itemId: number): Promise<BuyResult>;
```

| 항목 | 내용 |
|---|---|
| 권한 | `requireMember()`. 로그인하지 않았으면 `redirect("/")` |
| 입력 검증 | `parseId(itemId)` (1~2147483647 정수, 문자열 숫자도 받음). null이면 실패 |
| CSRF | Server Action의 Origin 검사(Next.js 기본)에 맡긴다. 새 Route Handler가 없으므로 따로 검사하지 않는다 |
| 트랜잭션 | `db.transaction` 안에서 `lockUser(tx, userId)` 먼저 |

**처리 순서와 결과** (한 트랜잭션, 위에서 멈추면 아무것도 바뀌지 않음)

| # | 확인 | 실패 문구 (`ok: false`) | FR |
|---|---|---|---|
| 1 | `parseId` 통과 | `살 수 없는 아이템이에요` | FR-009 (없는 아이템) |
| 2 | 아이템이 있고 `is_on_sale = true`, `is_starter = false` | `살 수 없는 아이템이에요` | FR-004, FR-009, FR-011 |
| 3 | 꾸미기 아이템(avatar·furniture·background)이면 내 `user_items`에 `quantity > 0` 행이 없음 | `이미 가지고 있는 아이템이에요` | FR-009, FR-010 |
| 4 | `getWallet(userId, tx).level >= required_level` | `레벨 {필요 레벨}부터 살 수 있어요` | FR-018 |
| 5 | `getWallet(userId, tx).coins >= price` | `코인이 {가격 − 잔액}개 부족해요` | FR-019 |
| 6 | 지급: 꾸미기 `INSERT user_items` / 성장 `INSERT … ON CONFLICT (user_id, item_id) DO UPDATE SET quantity = quantity + 1` | (PK 위반 23505 → `이미 가지고 있는 아이템이에요`) | FR-007, FR-042 |
| 7 | 원장: `point_ledger` (`reason 'purchase'`, `exp_delta 0`, `coin_delta −price`, `ref_id = String(itemId)`) | - | FR-007, FR-049 |

**성공** `{ ok: true, name, kind, quantity }` 뒤 `revalidatePath("/", "layout")`.

**보장**

| 보장 | 방법 | 확인 |
|---|---|---|
| 지급과 차감이 함께, 하나라도 실패하면 둘 다 없음 | 한 트랜잭션 | FR-007, AC 2-10 |
| 동시 요청에도 꾸미기 아이템 1개·차감 1번 | 잠금 + 3번 확인 + `user_items` PK | FR-010, SC-002 (성공 1, 거부 9) |
| 동시 구매가 같은 잔액을 보지 않음, 잔액 ≥ 0 | 잠금 안에서 원장 합계 | FR-020, AC 2-9, SC-003 |
| 성장 아이템 "늘어난 수량 × 가격 = 빠진 코인" | 잠금 + upsert + 같은 트랜잭션의 원장 | FR-045, AC 5-6, SC-009 |
| 그 순간의 레벨·잔액으로 판단 | 트랜잭션 안에서 `getWallet(…, tx)` | FR-018, AC 3-4 |
| 되팔기·환불 없음 | 그런 Server Action이 없다 | FR-012 |

**거부되면 바뀌지 않는 것**: `user_items`, `point_ledger`, 장착 상태 (SC-004).
