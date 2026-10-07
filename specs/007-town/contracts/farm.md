# Contract: 동물 농장 `/farm` (TOWN-09)

**Feature**: `007-town` | 관련: [../data-model.md](../data-model.md) 2.1~2.3·2.6·4.1, [../research.md](../research.md) R-14·R-15

## 1. 화면 라우트

| 주소 | 파일 | 접근 | 리다이렉트 | 탭 제목 |
|---|---|---|---|---|
| `/farm` | `src/app/farm/page.tsx` (서버) + `farm-view.tsx` (클라이언트) | 회원만 | 방문자 → `/` (`requireMember()`, Edge Cases) | `동물 농장 \| Blogville` |

| 구역 | 내용 (바뀌는 것만 굵게) |
|---|---|
| 머리 | `🐮 동물 농장`, 안내 문장(돌보기 하루 한 번, 공개 글 성장 +10, **상점의 성장 아이템으로도 자라요** *(plan 임시)*), `키우는 중 N / 5` |
| 상태 줄 | `role="status"` 한 줄 — Action 결과 문구 (성공 초록, 실패 빨강) |
| 🥚 알 받기 | [🎁 농장 첫 알 (무료)], [🎁 레벨 N 보상 알 (무료)] (받을 수 있는 것만), [🪙 100으로 알 사기]. 5마리면 모두 막히고 `자리가 꽉 찼어요. 다 키운 뒤에 받을 수 있어요`, 코인이 100 미만이면 알 사기 막힘 (FR-042~044) |
| 🌾 키우는 중 | 알 카드: [🐣 부화시키기]. 동물 카드: 이름·단계(아기/청소년/어른), 성장 막대 `성장 N / M · 다 키우면 ✨ X · 🪙 Y`, [🥕 밥 주기] [💧 물 주기] [🤲 쓰다듬기] (오늘 했으면 `완료`로 막힘), **🌱 성장 아이템 줄: 내가 가진(수량 ≥ 1) 성장 아이템마다 [{이름} ×{수량}] 버튼. 하나도 없으면 `상점에서 성장 아이템을 살 수 있어요`** *(plan 임시)* |
| 🏅 다 키운 동물 | 다 키운 시각 최신순 카드. **blog 단계 12 뒤 `내 블로그 도감에서도 볼 수 있어요` 링크** *(plan 임시)* |
| 헤더 | `← 광장으로 나가기` → `/town` (US6-12) |

- 버튼 44×44px 이상, 375px에서 가로 스크롤 없음 (SC-004). 지금 돌보기 버튼(`farm-view.tsx`, `btn … px-2.5 py-1.5 text-xs`)은 44px보다 낮으므로 **높이를 키운다** (새 🌱 버튼도 같은 크기).
- 동물 그림: `src/lib/art/animals.ts` (4종 × 아기·청소년·어른 + 알, 단계가 오를수록 크고 어른은 리본 — FR-049). blog의 도감 카드도 이 그림을 쓴다.

## 2. Server Action (`src/app/farm/actions.ts`)

결과 타입은 모두 `FarmResult = { ok: true; text: string } | { ok: false; text: string }`. 모든 Action은 첫 줄 `requireMember()`(방문자 → `/`), 회원 잠금(`lockUser`)을 건 한 트랜잭션이다 — 농장의 모든 동작은 서버에서 권한과 규칙을 확인한다 (FR-041). 숫자 ID 인자(`animalId`, `itemId`)는 `parseId()`를 거친다. `claimEgg`의 `level`은 `parseId()`를 거치지 않고 `levelEggLevels(내 레벨)`에 들어 있는지로만 확인한다(지금 코드, DB에 닿기 전에 걸러짐). Next.js Server Action이라 다른 사이트에서 보낸 요청은 Origin 확인으로 거부된다.

| Action | 입력 | 성공 | 실패 (문구 그대로) | 바뀌는 것 |
|---|---|---|---|---|
| `claimEgg(kind, level?)` (기존) | `kind`: `"starter"` \| `"level"`, `level`: 레벨 알이면 5의 배수 | `🥚 알을 받았어요! [부화시키기]를 눌러 보세요` | 5마리 → **`자리가 꽉 찼어요. 다 키운 뒤에 받을 수 있어요`** (바뀜, FR-043) / 아직 안 된 레벨 → `아직 받을 수 없는 알이에요` / 같은 무료 알 두 번(동시 포함) → `이미 받은 알이에요` (부분 고유 인덱스 23505) / 그 밖 → `잘못된 요청이에요` | `user_animals` 1행 (egg) |
| `buyEgg()` (기존) | 없음 | `🥚 알을 샀어요! (🪙 −100)` | 5마리 → 위와 같은 꽉 참 문구 / 코인 부족 → `코인이 N개 부족해요` (N = 100 − 코인) | `user_animals` 1행 + 원장 `egg_purchase` −100. `revalidatePath("/", "layout")` |
| `hatchEgg(animalId)` (기존) | 숫자 | `🐣 {이름}가 태어났어요!` (조사 자동) | 남의 알·이미 부화·없는 ID → `부화시킬 수 있는 알이 아니에요` / 이상한 숫자 → `잘못된 요청이에요` | `status = growing`, 종류 = 부화 비중 무작위(`randomInt`) |
| `careAnimal(animalId, action)` (기존) | 숫자, `"feed"` \| `"water"` \| `"pet"` | `{🥕\|💧\|🤲} {돌보기 이름} 완료! 성장 +{10\|10\|5}` 또는 다 자라면 `🎉 {이름}가 다 자랐어요! 경험치 +X, 🪙 +Y` | 남의·알·다 자란 동물 → `돌볼 수 있는 동물이 아니에요` / 오늘 이미 함 → `오늘은 이미 {밥 주기\|물 주기\|쓰다듬기}를 했어요` (PK 23505) | `animal_cares` 1행, 원장 `farm_care` ✨2(하루 15번까지), 성장, 다 자라면 `grown` + 원장 `farm_grown`. `revalidatePath("/", "layout")` |
| **`applyGrowthItem(animalId, itemId)` (새)** | 숫자, 숫자 | `🌱 {아이템 이름} 사용! 성장 +{성장치}` *(plan 임시)* 또는 다 자라면 `🎉 {이름}가 다 자랐어요! 경험치 +X, 🪙 +Y` | 이상한 숫자 → `잘못된 요청이에요` / 남의 동물·알·다 자란 동물·없는 ID → `돌볼 수 있는 동물이 아니에요` / 성장 아이템이 아니거나 수량 0·가지지 않음 → `가지고 있는 성장 아이템이 없어요` *(plan 임시)* | 순서: 동물 확인 → shop `consumeGrowthItem`(수량 −1) → `addGrowth`(성장 + 성장치, 넘치면 버림, 다 자라면 보상). 경험치·원장 기록 없음(다 자랄 때의 `farm_grown`만). `revalidatePath("/", "layout")` |

- `applyGrowthItem`은 하루에 여러 번 쓸 수 있고, 막는 것은 수량뿐이다 (FR-048, US6-8·9). 수량 1개로 두 탭에서 동시에 쓰면 하나만 성공한다.
- 다 키움 보상은 동물마다 한 번이다 (`status`가 `growing`일 때만 성장, `grown`으로 바뀌면 다시 오르지 않음 — FR-050).
- 원장 경로가 game의 `addLedgerEntry`로 바뀌어도 기록 한 줄과 값은 같다 (SC-008).

## 3. 공개 글 보상과 농장 (기존, post가 부름)

post의 `savePost`가 글쓰기 보상을 받으면 같은 트랜잭션에서 `growForPost(tx, 나)` → 키우는 동물(`growing`) 모두 성장 +10 (FR-047, US6-7). 키우는 동물이 없으면 0행으로 지나간다 (Edge Cases). 이 함수의 모양은 바꾸지 않는다.

## 4. 블로그 도감·전시 (US7 — blog가 구현)

| spec | 구현하는 곳 | 계약 |
|---|---|---|
| FR-051 도감 (누구나 봄) | blog `getGrownAnimals(ownerId)` + 블로그 홈 `animal-collection.tsx` | `user_animals.status = 'grown'`인 주인의 동물, 다 키운 시각 최신순 ([`specs/002-blog/contracts/profile-showcase.md`](../../002-blog/contracts/profile-showcase.md) 4절) |
| FR-052 전시 0~1마리 | blog `setShowcaseAnimal(animalId \| null)` | 내 동물(복합 FK, **town의 UNIQUE(`user_id`, `id`)가 대상**) AND `grown`(앱)일 때만. 조작 요청은 아무것도 바꾸지 않음 (같은 문서 2절) |

town이 주는 것: `user_animals` UNIQUE(`user_id`, `id`) 마이그레이션, 동물 그림(`src/lib/art/animals.ts`), "`grown`은 되돌아가지 않는다"는 상태 규칙 ([../data-model.md](../data-model.md) 4.1).
