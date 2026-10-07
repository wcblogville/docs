# Data Model: 광장 (TOWN)

**Feature**: `007-town` | **Spec**: [spec.md](spec.md) | **Research**: [research.md](research.md) | **Date**: 2026-10-07

테이블 담당 규칙(공통 맥락 3.1)에 따라 **town이 담당하는 표**는 `animal_species`, `user_animals`, `animal_cares`와 열거형 `animal_status`, `egg_source`, `care_action`, 시드 `SPECIES`다.
담당 표의 구조 변경은 **변경**, 남의 표는 **참조**로 적는다. 현재 = 코드 저장소 `main` `feb4c05`의 `src/db/schema.ts`, 목표 = 이 기능과 다른 spec의 결정이 모두 들어간 뒤.

## 1. 한눈에

| 테이블 | 구분 | 내용 |
|---|---|---|
| `user_animals` | 변경 (요청: blog, BLOG-04) | UNIQUE(`user_id`, `id`) 추가 — 전시 동물 복합 FK의 대상. 데이터 이전 없음 |
| `animal_species` | 담당, 구조 변경 없음 | 시드 `SPECIES` 4종 그대로 (병아리·토끼·아기 돼지·송아지) |
| `animal_cares` | 담당, 구조 변경 없음 | 돌보기 하루 한 번(복합 PK) 그대로 |
| `follows` | 참조 (요청: social이 `is_favorite` 추가) | 즐겨찾기 여부를 town이 읽고 바꾼다. 최대 10명은 town 코드 규칙 |
| `blogs` | 참조 (요청: blog가 `roof_color` 추가) | 지붕 색을 town이 읽고 바꾼다. `showcase_animal_id`(전시)는 blog가 추가·사용 |
| `items` | 참조 (요청: shop이 `type = 'growth'`, `growth_value` 추가) | 성장 아이템과 성장치를 읽는다 |
| `user_items` | 참조 (요청: shop이 `quantity` 추가) | 성장 아이템 수량을 shop 도우미 `consumeGrowthItem`으로 줄인다 |
| `point_ledger` | 참조 | 돌보기·다 키움·알 구매 기록(지금과 같음), 집 단계 계산용 경험치 합계 |
| `profiles` | 참조 (요청: auth가 `photo_key` 추가. 닉네임 2~20자 변경은 auth 결정) | 집 이름표 닉네임, 집 문 옆·광장 캐릭터, 헤더 프로필 사진 |
| `posts` | 참조 | 최근 공개 글 시각(집 순서, 방문자 후보) |
| `attendances` | 참조 | 오늘 출석 여부(게시판 부제). game 작업 뒤에는 `getViewer()`의 값 |
| `users`, `attachments` | 참조 | 회원 기준, 프로필 사진 파일 주소 `/files/{key}` |

새 표·새 열거형 값·새 원장 사유는 없다.

## 2. 테이블별 현재와 목표

### 2.1 `user_animals` — 변경 (요청: blog, BLOG-04)

| 항목 | 현재 | 목표 |
|---|---|---|
| 컬럼 | `id` PK, `user_id` FK→users CASCADE, `species_id` FK NULL, `status` animal_status 기본 egg, `growth` 기본 0, `source` egg_source, `source_level` NULL, `created_at`, `hatched_at`, `grown_at` | 같음 |
| CHECK | `user_animals_species_check` (알이면 종류 없음), `user_animals_growth_check` (≥ 0), `user_animals_level_check` (level이면 레벨 있음) | 같음 |
| 고유 | 부분 고유 인덱스 `user_animals_starter_uq` (`user_id`) WHERE starter, `user_animals_level_uq` (`user_id`, `source_level`) WHERE level | 같음 + **UNIQUE `user_animals_user_id_id_uq` (`user_id`, `id`)** |
| 인덱스 | `user_animals_user_status_idx` (`user_id`, `status`) | 같음 (도감 조회도 이 인덱스) |
| 가리키는 표 | `animal_cares.animal_id` → `id` CASCADE | 같음 + `blogs` (`owner_id`, `showcase_animal_id`) → (`user_id`, `id`) `ON DELETE SET NULL (showcase_animal_id)` (blog가 추가) |

- **왜**: 전시 동물을 "내 동물만"으로 DB가 막으려면 복합 FK가 가리킬 (`user_id`, `id`) UNIQUE가 있어야 한다 (ERD 3.11, 7장 6-2, research R-20). 부분 고유 인덱스는 대상이 될 수 없다.
- **검증 규칙** (앱, 지금과 같음 + 성장 아이템):
  - 회원의 알·키우는 동물(`status IN ('egg','growing')`)은 최대 5마리 — `countActive()`를 `lockUser` 안에서 센다 (ERD 3.11 "코드가 지킨다").
  - 무료 알은 같은 것을 한 번 (부분 고유 인덱스 → `이미 받은 알이에요`), 레벨 알은 `levelEggLevels(레벨)` 안에서만 (`아직 받을 수 없는 알이에요`).
  - 돌보기·성장 아이템은 내 동물이고 `growing`일 때만 (`돌볼 수 있는 동물이 아니에요`). 부화는 내 알일 때만 (`부화시킬 수 있는 알이 아니에요`).
  - 성장치가 `animal_species.grow_exp` 이상이 되면 `status = 'grown'`, `growth = grow_exp`(넘친 양은 버림), `grown_at = now()`, 원장 `farm_grown` 한 줄.
- **삭제**: 회원 삭제 → CASCADE(돌보기 기록도). 동물 행이 지워지면 전시하던 블로그는 `showcase_animal_id`만 NULL (blog의 FK 동작). 동물만 지우는 기능은 없다.

### 2.2 `animal_species` — 담당, 구조 변경 없음

| 항목 | 현재 = 목표 |
|---|---|
| 컬럼 | `id` PK, `code` UK, `name`, `asset_key`, `grow_exp` > 0, `reward_exp`·`reward_coins` ≥ 0, `hatch_weight` > 0 |
| 시드 (`scripts/seed.ts` `SPECIES`, town 소유) | `chick` 병아리 60 / ✨50 🪙20 / 40, `bunny` 토끼 90 / ✨80 🪙30 / 30, `piglet` 아기 돼지 120 / ✨110 🪙50 / 20, `calf` 송아지 160 / ✨160 🪙80 / 10 — spec FR-045 표와 같다 (**코드 확인**) |

### 2.3 `animal_cares` — 담당, 구조 변경 없음

PK(`animal_id`, `action`, `date`) — 같은 동물·같은 돌보기·같은 한국 날짜는 한 번 (FR-046, SC-008). 동물 삭제 시 CASCADE.

### 2.4 `follows` — 참조 (요청: social이 `is_favorite` 추가)

| 항목 | 현재 | 목표 (social `data-model.md` U4) |
|---|---|---|
| 컬럼 | `follower_id`, `followee_id` (PK, 둘 다 FK→users CASCADE), `created_at` | + `is_favorite boolean NOT NULL DEFAULT false` (기존 행 false, 백필 없음) |
| 제약 | CHECK 자기 자신 불가, 인덱스 `follows_followee_idx` | 같음 |

- town이 하는 일: `is_favorite`를 읽어 광장 집·패널·내 이웃 목록을 만들고, `toggleFavorite`로 바꾼다.
- **검증 규칙** (town 서버 코드, research R-03): 이웃 행이 있어야 바꿀 수 있다(없으면 거부). 켤 때 `lockUser(나)` 안에서 `is_favorite = true`인 내 행이 10개 미만이어야 한다. DB 제약은 없다 ([plan.md](plan.md) Complexity Tracking).
- 이웃 취소(social `toggleFollow`)는 행을 지우므로 즐겨찾기도 사라진다. 회원 탈퇴도 CASCADE.

### 2.5 `blogs` — 참조 (요청: blog가 `roof_color` 추가)

| 항목 | 현재 | 목표 (blog `data-model.md` B-M2, B-M3) |
|---|---|---|
| 지붕 색 | 없음 (장착 배경의 강조색만) | `roof_color text NULL` + CHECK `blogs_roof_color_check`: NULL 또는 `red`·`orange`·`yellow`·`green`·`sky`·`blue`·`purple`·`brown`. NULL = 배경 색 따라가기 |
| 전시 동물 | 없음 | `showcase_animal_id` NULL + 복합 FK → `user_animals` (`user_id`, `id`) — 2.1의 UNIQUE가 먼저 |
| town이 쓰는 컬럼 | `slug`, `title`, `owner_id`, `background_item_id`, `created_at` | + `roof_color` (읽고 `setRoofColor`로 바꿈) |

- town이 `roof_color`를 바꿀 때 대상은 요청 값이 아니라 로그인한 회원의 블로그(`owner_id = 나`)다.
- 코드값 → 색은 town의 `src/lib/town.ts` `ROOF_PALETTE`가 정한다(DB에는 코드값만).

### 2.6 `items`·`user_items` — 참조 (요청: shop)

| 항목 | 현재 | 목표 (shop `data-model.md`) |
|---|---|---|
| `items.type` | character / background / furniture | + `avatar`, `growth` |
| `items.growth_value` | 없음 | integer NULL, CHECK (`type = 'growth'`) = (`growth_value IS NOT NULL`), `> 0` |
| `user_items.quantity` | 없음 | integer 기본 1, CHECK ≥ 0 (기존 행 1) |
| 성장 아이템 시드 (shop) | 없음 | `growth_feed` 동물 먹이 🪙20 Lv.1 +20, `growth_premium` 고급 먹이 🪙60 Lv.1 +50, `growth_booster` 성장 촉진제 🪙150 Lv.3 +100 (spec FR-048, D13) |

- town은 농장 화면에서 `type = 'growth' AND quantity > 0`인 내 아이템을 읽고, 쓸 때 shop의 `consumeGrowthItem(tx, 나, itemId)`로 수량을 1 줄인다. 0이 돼도 행은 남는다 (shop 규칙).

### 2.7 `point_ledger` — 참조 (game)

| 사유 | town이 쓰는 경우 | 값 |
|---|---|---|
| `farm_care` | 돌보기 (하루 15번, `grantReward`) | ✨2 |
| `farm_grown` | 다 키움 (종류 표의 값, `ref_id` = 동물 ID) | ✨`reward_exp` 🪙`reward_coins` — game의 `addLedgerEntry`로 기록 방식만 바뀐다 |
| `egg_purchase` | 알 사기 | 🪙 −100 |
| (없음) | 성장 아이템 사용, 지붕 색, 즐겨찾기 | 원장 기록 없음 |

- 집 단계: 집 주인의 `SUM(exp_delta)` → `levelFromExp` → `houseStage` (research R-08). 저장하지 않는다.

### 2.8 `profiles` — 참조 (auth)

| 컬럼 | town이 쓰는 곳 |
|---|---|
| `nickname` (목표 2~20자, auth) | 광장 내 이름표, 이웃집 이름표 `{닉네임}의 집`(12자 넘으면 `…`), 헤더 상태창, 패널·내 이웃 목록 |
| `character_item_id` | 광장 내 캐릭터, 집 문 옆 주인 캐릭터, 헤더 프로필(사진이 없을 때) |
| `photo_key` (auth가 추가, NULL) | 헤더 상태창 프로필 사진 `/files/{key}` |

### 2.9 `posts`·`attendances`·`users`·`attachments` — 참조

- `posts`: `MAX(created_at) WHERE blog_id = ? AND visibility = 'public'` — 즐겨찾기 집 순서(FR-025·026), 방문자 후보 조건·순서(FR-029). 인덱스 `posts_blog_created_idx`.
- `attendances`: 오늘(`todayKST()`) 행이 있으면 게시판 `(오늘 완료 ✅)`. game의 구조 변경(`cycle_day` 등) 뒤에도 PK(`user_id`, `date`)는 그대로라 지금 쿼리가 계속 맞다. game이 `viewer.attendance`를 주면 그 값으로 바꾼다.
- `attachments`: 프로필 사진 파일. town은 주소(`/files/{key}`)만 만든다.

## 3. 저장하지 않는 엔터티 (spec Key Entities → 계산·화면)

| spec 엔터티 | 어디서 | 모양 |
|---|---|---|
| 광장(마을) | `src/components/town/scene.ts` `layout()`, 상수 `WORLD` 1800 × 1400 | 건물(게시판·상점·농장), 분수·가로등·나무, 내 집 자리 1, 이웃집 자리 10 (위 5, 아래 5). 나무·바닥은 고정 시드 |
| 건물 입구 | `layout()`의 `entrances` | `{ label, emoji, x, y, target: { kind: "link", href } \| { kind: "login" }, area, promptY }` — 게시판은 입구 2개. 방문자 이용 여부는 `target.kind` |
| 집 | `TownHouse` (`src/components/town/types.ts`) | 아래 3.1 |
| 이웃 관계·즐겨찾기 | `follows` 행 | 2.4 |
| 방문자 광장 후보 | `getVisitorHouses()` 안에서 매번 계산 | 공개 글 있는 블로그, 이웃 수 DESC → 최근 공개 글 DESC → `blogs.id` ASC, 상위 100 → 무작위 10 |
| 동물 종류, 회원의 동물, 돌보기 기록 | 2.1~2.3 | |
| 성장 아이템 | `items`(`growth`) + `user_items.quantity` | 2.6. 사용 기록은 남기지 않는다 (ERD 3.11) |
| 블로그 도감·전시 | `user_animals.status = 'grown'` + `blogs.showcase_animal_id` | 도감 = 주인의 다 키운 동물 (blog가 그림). 전시 0~1마리 |
| 원장 기록 | 2.7 | |
| 유저 상태창 | `getViewer()` 프로필 (`nickname`, `blogTitle`, `blogSlug`, `characterAsset`, + `photoKey`, + shop `outfit`) | 헤더가 그릴 때 읽음 |
| 환영 문구 표시 | 브라우저 쿠키 `bv_welcome` | 값 `1`, Path `/town`, Max-Age 600초, SameSite=Lax, 배포 Secure, HttpOnly 아님 (research R-02) |

### 3.1 광장 화면 데이터 (`src/components/town/types.ts`)

```ts
type TownHouse = {
  slug: string;            // 들어가면 /@{slug}
  title: string;           // 부제 = 블로그 이름
  nickname: string;        // 이름표 {닉네임}의 집
  characterAsset: string;  // 문 옆 주인 캐릭터
  outfit: string[];        // 차림 (shop이 추가)
  stage: 1 | 2 | 3;        // houseStage(주인 레벨)
  roof: string;            // 지붕 색 "#rrggbb" = houseRoof(roof_color, 배경)
};
type TownLink = { slug: string; title: string; nickname: string; favorite: boolean };   // 🏘 패널 한 줄
// listMyNeighbors()가 돌려주는 줄. 내 이웃 목록(☆ 버튼이 toggleFavorite(userId)를 부름, 캐릭터 얼굴)이 쓴다.
// /town 패널에는 TownLink 필드만 골라 넘긴다 (게임 데이터에 회원 ID를 싣지 않는다)
type MyNeighbor = TownLink & { userId: string; characterAsset: string; outfit: string[] };
type TownData = {
  player: { nickname: string; characterAsset: string; outfit: string[] } | null; // null = 방문자
  myHouse: TownHouse | null;     // 방문자는 null
  neighbors: TownHouse[];        // 광장에 놓을 집, 최대 10 (회원: 즐겨찾기, 방문자: 인기 무작위)
  panel: TownLink[];             // 🏘 이웃집 패널 (회원: 모든 이웃, 방문자: neighbors와 같은 블로그)
  attendedToday: boolean;        // 게시판 부제
  attendanceDay: number | null;  // game 작업 뒤 (문구 결정용)
};
```

- 지금(`attendedToday`, `neighbors`, `myHouse`, `player`)에서 `stage`, `roof`, `outfit`, `panel`, `attendanceDay`가 늘고 `backgroundAsset`이 빠진다 (지붕 색을 서버가 정하므로).
- 환영 여부는 `TownData`에 넣지 않는다 — 게임 데이터가 바뀌면 게임이 다시 만들어지기 때문 (research R-02).

### 3.2 계산 규칙 (`src/lib/town.ts`, 순수 함수)

| 이름 | 규칙 | 근거 |
|---|---|---|
| `houseStage(level)` | 1~9 → 1, 10~29 → 2, 30 이상 → 3 | FR-059, D16 |
| `houseRoof(roofColor, backgroundAsset)` | `roofColor`가 있으면 `ROOF_PALETTE[roofColor].hex`, 없으면 `backgroundAccent(backgroundAsset)` | FR-040 |
| `ROOF_PALETTE` | 8개 코드값 → `{ hex, label }` (빨강·주황·노랑·초록·하늘·파랑·보라·갈색) | FR-039 |
| `houseLabel(nickname)` | 코드 포인트 12자 초과면 12자 + `…`, 뒤에 `의 집` | FR-024 |
| `FAVORITE_LIMIT` = 10, `canFavorite(count)` = count < 10 | | FR-033 |
| `VISITOR_POOL` = 100, `TOWN_HOUSE_SLOTS` = 10, `sampleDistinct(list, n, randInt)` | 겹치지 않게 `min(n, list.length)`개 | FR-029 |
| `toScreen(p)`, `toLogical(p)` (아이소메트릭 단계) | P(x, y) = (x − y, (x + y) / 2) + 원점, 역변환 | FR-037, research R-16 |

## 4. 상태 전이

### 4.1 회원의 동물 (`user_animals.status`)

```text
(없음) ──[첫 알 / 레벨 N 알 / 🪙100 알 사기]──▶ egg            (5마리 미만, 무료 알은 한 번, 알 사기는 원장 −100)
egg ──[부화시키기 (내 알)]──▶ growing                          (species = 비중 무작위, hatched_at)
growing ──[돌보기 (하루 종류별 1번) / 공개 글 보상 / 성장 아이템 (수량 −1)]──▶ growing (growth + n)
growing ──[growth ≥ grow_exp]──▶ grown                         (growth = grow_exp, grown_at, 원장 farm_grown 1줄, 자리 하나 빔)
grown ──[블로그 주인이 전시 고르기 (blog)]──▶ grown + blogs.showcase_animal_id = id
```

- 한 방향이다. `grown`에서 되돌리는 코드가 없다 → 전시 확인과 저장 사이 경쟁이 없다 (blog research R-19와 같은 근거).
- 알(`egg`)에는 돌보기·성장 아이템을 쓸 수 없다. `grown`에도 쓸 수 없다.

### 4.2 즐겨찾기 (`follows`)

```text
이웃 아님 ──[+ 이웃 추가 (social)]──▶ 이웃(is_favorite=false)
이웃(false) ──[☆ (town, 내 즐겨찾기 < 10)]──▶ 이웃(true)      ← 10명이면 거부: 즐겨찾기할 이웃은 최대 10명이에요
이웃(true) ──[⭐ (town)]──▶ 이웃(false)
이웃(true/false) ──[✓ 이웃 취소 (social) / 회원 탈퇴]──▶ 행 없음 (즐겨찾기도 사라짐, 다음 광장에서 집 사라짐)
```

### 4.3 집

- 단계: 주인 레벨이 오르면 다음 광장 방문 때 1 → 2 → 3. 내려가지 않는다 (보상 회수 없음, D6).
- 지붕 색: `roof_color` NULL(배경 색) ⇄ 8색 중 하나. [배경 색 따라가기] = NULL로.
- 광장에 보이는지: 회원 광장 = 즐겨찾기 행이 있을 때, 방문자 광장 = 그때그때 무작위.

### 4.4 환영 문구 쿠키 (`bv_welcome`)

```text
가입 성공 (auth signUp) ──▶ 쿠키 있음 (10분)
쿠키 있음 ──[/town 서버가 읽음 → 브라우저에서 아직 있으면 문구 열고 바로 지움]──▶ 쿠키 없음 (문구는 [✕]까지 화면에 남음)
쿠키 있음 ──[10분 지남]──▶ 쿠키 없음 (문구 안 보임)
쿠키 없음 ──[새로고침 / 뒤로 가기 / 다른 화면에서 돌아옴]──▶ 문구 안 보임
```

## 5. 마이그레이션 순서와 데이터 이전

번호는 정하지 않는다 (공통 맥락 3.6 — 먼저 merge된 쪽이 생기면 최신 `main`에서 `npm run db:generate`를 다시 돌린다).

| 순서 | 담당 | 마이그레이션 (내용) | 선행 | 데이터 이전 |
|---|---|---|---|---|
| T-M1 | **town** | **`user_animals` UNIQUE(`user_id`, `id`)** — `src/db/schema.ts` `userAnimals` 블록에 `unique("user_animals_user_id_id_uq").on(t.userId, t.id)` → `npm run db:generate` → 생성 SQL 맨 위에 `-- TOWN-09 (요청: blog, BLOG-04): 전시 동물 복합 FK가 가리킬 UNIQUE (ERD 3.11, 7장 6-2)` → `npm run db:migrate` | 없음 | 없음 — `id`가 PK라 모든 행이 이미 고유. 적용 전 점검 쿼리도 필요 없다 |
| (참조) | social | `follows.is_favorite` (U4) | 없음 | 기존 행 false |
| (참조) | shop | `item_type` + `growth`, `items.growth_value`, `user_items.quantity` (shop 1) | 없음 | 기존 보유 행 수량 1 |
| (참조) | blog | `blogs.roof_color` + CHECK (B-M2) | 없음 | 모두 NULL(배경 색) |
| (참조) | blog | `blogs.showcase_animal_id` + 복합 FK (B-M3) | **T-M1** | 모두 NULL(전시 없음) |
| (참조) | auth | `profiles.photo_key` (R16) | 없음 | 모두 NULL(캐릭터 얼굴) |

- T-M1은 다른 것에 기대지 않으므로 town 작업 중 가장 먼저(또는 blog B-M3 바로 앞) merge한다.
- `scripts/reset-dev.ts`(`TRUNCATE users, tags … CASCADE`)는 `user_animals`·`animal_cares`를 함께 비우고 `animal_species`는 남긴다 (지금과 같음). 시드 `npm run db:seed`는 `SPECIES`를 `code` 기준으로 넣거나 덮어쓴다.
- 적용 뒤 확인: `\d user_animals`에 `user_animals_user_id_id_uq` UNIQUE가 보인다. DB에 같은 (`user_id`, `id`)를 직접 넣을 수 없음은 PK로 이미 막히므로 따로 시험하지 않는다.

## 6. 조회와 인덱스

| 조회 | 어디서 | 쓰는 인덱스 |
|---|---|---|
| 내 즐겨찾기 집 (≤ 10) | `getTownHouses(나)` | `follows` PK (`follower_id`, …) |
| 내 모든 이웃 (패널·내 이웃 목록) | `listMyNeighbors(나)` | `follows` PK |
| 이웃 수 (방문자 후보) | `getVisitorHouses()` | `follows_followee_idx` |
| 최근 공개 글 | 위 셋 | `posts_blog_created_idx` (`blog_id`, `created_at` DESC) |
| 주인 경험치 합계 (집 단계, ≤ 11명) | 집 조회 | `point_ledger_user_reason_created_idx` (앞 열 `user_id`) |
| 농장 동물·5마리 세기 | `getFarm`, `countActive` | `user_animals_user_status_idx` |
| 내 성장 아이템 | `getFarm` | `user_items` PK (`user_id`, `item_id`) |

새 인덱스는 만들지 않는다.

## 7. `docs/02-erd.md`에서 town이 고칠 곳 (테이블 담당 규칙 5)

| 위치 | 고칠 내용 |
|---|---|
| 1장 관계도 `user_animals` | 이미 `UK(user_id, id): 전시 FK용`이 적혀 있음 → 그대로 |
| 3.11 동물 농장 | "블로그에 한 마리 전시"의 UNIQUE 부분 ⏳ 해제(구현 뒤). "성장 아이템으로 키우기"는 shop 컬럼과 town 사용이 모두 끝나면 ⏳ 해제, 사용 순서(내 키우는 동물 확인 → 수량 −1 → 성장)를 한 줄 |
| 5장 남은 확인 사항 | 지금 한 줄 "**아바타 꾸미기(SHOP-06)·집 성장(TOWN-11)**: 결정은 됐지만 테이블 설계 전이다"에서 집 성장 부분만 뗀다 (아바타 부분은 shop이 고친다) → "집 단계는 저장하지 않는다: 주인 레벨(Lv.1~9 / 10~29 / 30+)에서 계산 (3.5와 같은 이유, D16)" |
| 7장 할 일 표 6-2 | `user_animals` UNIQUE(`user_id`, `id`) 완료 표시 (같은 줄의 다른 항목은 각 담당) |
| 부록 | 바뀌는 타입·NULL 허용 없음 |
