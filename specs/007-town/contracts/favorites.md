# Contract: 즐겨찾는 이웃과 광장의 이웃집 (TOWN-04, TOWN-08)

**Feature**: `007-town` | 관련: [../data-model.md](../data-model.md) 2.4·4.2, [../research.md](../research.md) R-03~R-07, social [`specs/004-social/contracts/follows-feed.md`](../../004-social/contracts/follows-feed.md)

## 1. 내 이웃 목록 (블로그 홈, 주인에게만)

| 항목 | 계약 |
|---|---|
| 자리 | `/@{내 주소}` 블로그 홈 왼쪽 `<aside>`의 카테고리 카드 아래 (blog 소유 `src/app/blog/[slug]/page.tsx`에 `{isOwner && <MyNeighbors userId={…} />}` 한 줄 — 공통 모듈 추가) |
| 보이는 사람 | 블로그 주인만. 다른 회원·방문자에게는 **서버가 그리지 않는다** (FR-032, US4-2) |
| 컴포넌트 | `MyNeighbors` (서버, `src/components/town/my-neighbors.tsx`) → 줄마다 `FavoriteButton` (클라이언트, `src/components/town/favorite-button.tsx`) |
| 제목 | `🏘 내 이웃` + `⭐ {즐겨찾기 수} / 10` *(둘 다 plan 임시 — spec은 "내 이웃 목록"이라는 이름만 정함)* |
| 줄 | 캐릭터 얼굴 · 블로그 이름(굵게) · 닉네임 → 그 블로그 링크, 오른쪽 ☆(즐겨찾기 아님) / ⭐(즐겨찾기) 버튼. 버튼 접근 이름 `{닉네임} 즐겨찾기` / `{닉네임} 즐겨찾기 취소` *(plan 임시)*, 44×44px 이상 |
| 순서 | 즐겨찾기 먼저 → 최근 공개 글 최신순(공개 글 없으면 뒤) → 이웃 추가 최신순 |
| 빈 목록 | `아직 이웃이 없어요. 마을 소식에서 마음에 드는 블로그를 이웃으로 추가해 보세요.` (FR-034, US4-10) |
| 누르면 | `toggleFavorite(대상 회원 ID)` → 성공 결과의 `favorite` 값으로 ☆/⭐ 바꿈 (미리 바꾸지 않음). 실패하면 목록 위 한 줄 문구(`role="status"`, 빨간 글씨) |
| 375px | 가로 스크롤 없음, 블로그 이름·닉네임은 `truncate` |

## 2. Server Action `toggleFavorite`

- 파일: `src/app/town/actions.ts` (새, `"use server"`)
- 모양: `toggleFavorite(followeeId: unknown): Promise<{ ok: true; favorite: boolean } | { ok: false; error: string }>`

| 순서 | 조건 | 결과 |
|---|---|---|
| 1 | 로그인 안 함 | `requireMember()` → `/`로 이동 |
| 2 | `followeeId`가 문자열이 아니거나 1~64자가 아님, 또는 나 자신 | `{ ok: false, error: "이웃으로 추가한 블로그만 즐겨찾기할 수 있어요" }` *(plan 임시)* |
| 3 | 트랜잭션 시작 → `lockUser(tx, 나)` | |
| 4 | `follows`에 (나, 대상) 행 없음 (이웃 아님, 조작 요청, 방금 이웃 취소) | `{ ok: false, error: "이웃으로 추가한 블로그만 즐겨찾기할 수 있어요" }` *(plan 임시)* (FR-034, Edge Cases) |
| 5 | 지금 즐겨찾기 → 끄기 | `UPDATE … SET is_favorite = false` → `{ ok: true, favorite: false }` |
| 6 | 지금 아님 → 켜기, 내 즐겨찾기 수 ≥ 10 | `{ ok: false, error: "즐겨찾기할 이웃은 최대 10명이에요" }` (FR-033, US4-4) |
| 7 | 켜기, 10 미만 | `UPDATE … SET is_favorite = true` → `{ ok: true, favorite: true }` |
| 8 | 성공 뒤 | `revalidatePath("/", "layout")` (social `toggleFollow`와 같음) |

- 보장: 같은 회원의 요청은 회원 잠금으로 차례로 처리 → 동시에 와도 10명을 넘지 않는다 (FR-033, US4-5, SC-006).
- 경험치·코인 변화 없음. 원장 기록 없음.
- 쿼리는 값 바인딩만 쓴다 (Drizzle 조건식).

## 3. 🏘 이웃집 패널 (`/town` 왼쪽 위)

| 보는 사람 | 내용 |
|---|---|
| 회원, 이웃 1명 이상 | 제목 `🏘 이웃집 {이웃 수}`, 목록 = 내 모든 이웃(1절 순서), 즐겨찾기는 이름 앞 ⭐, 각 줄은 `/@{주소}` 링크(`{블로그 이름} · {닉네임}`). 즐겨찾기가 0명이면 목록 위에 `마음에 드는 블로그를 즐겨찾기하면 광장에 집이 생겨요` (FR-028, US4-9) |
| 회원, 이웃 0명 | 제목 `🏘 이웃집`, 내용 `마음에 드는 블로그를 즐겨찾기하면 광장에 집이 생겨요` |
| 방문자 | 제목 `🏘 이웃집 {N}`, 목록 = 광장에 놓인 집(최대 10)과 같은 블로그 |

- `<details open>`으로 접고 펼 수 있고 처음에는 펼쳐짐, 목록이 길면 패널 안에서 스크롤(`max-h-[40dvh]`) (FR-006).
- 접기 제목과 링크 한 줄의 누르는 높이는 44px 이상 (constitution VI, SC-004). 지금 링크는 `py-1.5 text-sm`이라 44px보다 낮다 → 줄 높이를 키운다.
- 광장을 걷지 않고 블로그에 들어가는 유일한 일반 링크 목록이다 (FR-030). 즐겨찾기하지 않은 이웃도 여기서 들어간다 (US4-7).

## 4. 광장에 놓는 집 (서버 함수 계약, `src/server/town.ts`)

| 함수 | 모양 | 규칙 |
|---|---|---|
| `getTownHouses(userId)` | `Promise<TownHouse[]>` (≤ 10) | 내가 즐겨찾기한 이웃의 블로그. 정렬 최근 공개 글 DESC(없으면 뒤) → `blogs.created_at` DESC → `blogs.id` DESC. 공개 글 없는 블로그도 기본 집으로 (FR-025·026, US4-6, SC-007) |
| `getMyHouse(userId)` | `Promise<TownHouse \| null>` | 내 블로그 (FR-027) |
| `listMyNeighbors(userId)` | `Promise<MyNeighbor[]>` ([../data-model.md](../data-model.md) 3.1: `TownLink` + 이웃 회원 `userId`·`characterAsset`·`outfit`) | 내 모든 이웃, 1절 순서. 내 이웃 목록은 그대로 쓰고(☆ 버튼이 `userId`로 `toggleFavorite`), `/town` 패널은 `TownLink` 필드만 골라 넘긴다 |
| `getVisitorHouses()` | `Promise<TownHouse[]>` (≤ 10) | 후보 = 공개 글이 있는 블로그를 이웃 수 DESC → 최근 공개 글 DESC → `blogs.id` ASC로 상위 100. 후보에서 `sampleDistinct(후보, 10, randomInt)`로 겹치지 않게 고른다. 후보가 10보다 적으면 전부, 남는 자리는 빈칸 (FR-029, US5, SC-013) |

- 모든 `TownHouse`는 `stage`(주인 레벨 → 1~3)와 `roof`(지붕 색)를 서버에서 채운다 ([../data-model.md](../data-model.md) 3.2).
- 즐겨찾기 블로그를 고른 순서대로 자리 1~10에 놓는다 (위 줄 왼쪽부터 5, 아래 줄 5). 방문자는 뽑힌 순서대로.
- 이웃을 취소하거나 상대가 탈퇴하면 행이 없어져 다음 광장 방문 때 집이 사라진다 (FR-035, US4-8).

## 5. 💛 이웃 새 글 순서 (FR-036 — social이 구현)

town spec FR-036의 정렬은 social의 `listFeed({ followerId })`가 만든다 (`specs/004-social/contracts/follows-feed.md` 3절): 최근 7일(한국 시간, 오늘 포함) 안에 쓴 즐겨찾은 이웃의 공개 글을 맨 위 최신순, 그다음 나머지를 최신순. town은 `is_favorite` 값만 바꾼다. 즐겨찾기를 풀면 다음 목록부터 일반 순서다.
