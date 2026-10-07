# Contract: 이웃·마을 소식·이웃 새 글 (SOC-04, SOC-05)

**Feature**: `004-social` | 관련: [data-model.md](../data-model.md) 2.4, [research.md](../research.md) R10·R11·R16

## 1. 화면 라우트

| 주소 | 실제 파일 | 접근 | 리다이렉트 | 탭 제목 | 내용 |
|---|---|---|---|---|---|
| `/feed` | `src/app/feed/page.tsx` | 누구나 (방문자·회원·관리자) | 없음 | `마을 소식 \| Blogville` | 제목 `📋 마을 소식`, 탭 [🏘 마을 전체](진하게) + 회원만 [💛 이웃 새 글], 공개 글만 최신순 8개, 오른쪽(768px 미만은 아래) 인기 태그 30개 |
| `/feed?page={n}` | 같음 | 같음 | 없음 (`n`이 1~2147483647 정수가 아니면 1페이지) | 같음 | 마지막보다 큰 페이지는 빈 목록 문구 |
| `/feed/following` | `src/app/feed/following/page.tsx` | 회원만 | 방문자 → `/` (`requireMember()`) | `이웃 새 글 \| Blogville` | 제목 `💛 이웃 새 글`, [💛 이웃 새 글] 탭 진하게, 이웃 블로그의 공개 글만, 순서는 3절 |
| `/@{slug}` (정보 줄 오른쪽) | `src/components/blog/blog-header.tsx`의 이웃 버튼 부분 (blog 소유 파일) | 버튼은 회원이면서 주인이 아닐 때만 | 없음 | (blog 규칙) | `+ 이웃 추가`(하늘색 바탕 흰 글씨) / `✓ 이웃`(흰 바탕). 주인에게는 [✏️ 글쓰기] [🎨 꾸미기] [⚙️ 관리](blog). 정보 줄 `이웃 N`은 누구에게나 |

빈 화면 문구 (그대로):

| 화면 | 문구 |
|---|---|
| 마을 소식 (공개 글 0개, 또는 마지막보다 큰 페이지) | `아직 마을에 글이 없어요. 첫 글의 주인공이 되어 보세요! ✏️` |
| 이웃 새 글 (이웃 0명, 이웃 공개 글 0개, 또는 마지막보다 큰 페이지) | `아직 이웃이 없거나 이웃의 새 글이 없어요.` / `마을 소식에서 마음에 드는 블로그를 이웃으로 추가해 보세요.` ("마을 소식"은 `/feed` 링크) |
| 인기 태그 0개 | `아직 태그가 없어요` |

지금 코드가 이미 위 화면을 만든다(`src/components/blog/feed-view.tsx`, post 소유). 이 기능에서 바뀌는 것은 이웃 새 글의 순서(3절)와 카드의 `💬 N` 정의(댓글 + 답글)다. 탭·인기 태그 칩·페이지 번호의 누르는 영역 44×44px(constitution VI)는 post에 요청한다(plan 의존성 D-10).

정보 줄의 `이웃 N`(FR-037) = `follows`에서 `followee_id = 블로그 주인`인 행 수. 지금 `getBlogBySlug`(`src/server/blog.ts`, blog 소유)의 `followerCount`가 이 값이고, 방문자·주인 모두에게 보인다. social은 이 식을 바꾸지 않는다.

## 2. Server Action: `toggleFollow(followeeId: unknown): Promise<void>`

`src/app/blog/actions.ts`. 블로그 머리의 폼이 `toggleFollow.bind(null, blog.ownerId)`로 부른다(지금과 같음).

| 판정 순서 | 조건 | 결과 |
|---|---|---|
| 1 | 로그인 없음 | `/`로 이동 (FR-002) |
| 2 | `followeeId`가 문자열이 아니거나 1~64자가 아님 | 아무것도 안 함 |
| 3 | `followeeId` = 나 | 아무것도 안 함 (DB CHECK도 거부, FR-038) |
| 4 | 이미 이웃 | 이웃 행 삭제 (확인 창 없음, FR-035). 즐겨찾기도 함께 사라짐 |
| 5 | 이웃 아님 | `INSERT INTO follows ... SELECT ... FROM blogs WHERE owner_id = followeeId ON CONFLICT DO NOTHING` — 블로그가 없는 대상이면 0행 (FR-040), 동시 요청이면 1행만 (FR-039) |

- 문구·완료 안내·처리 중 표시 없음 (FR-036). 처리 뒤 `revalidatePath("/", "layout")`로 버튼과 `이웃 N`이 바뀐다(미리 바꾸지 않음).
- 경험치·코인 변화 없음 (FR-041).
- 그 순간 대상 회원이 지워져 FK 위반(23503)이 나면 무시한다 (`src/server/db-errors.ts`에 새로 더하는 `foreignKeyViolation(err)`로 가림, 다른 오류는 그대로 던짐, research R10).

## 3. 목록 순서 (서버 함수 계약)

`src/server/blog.ts`의 `listFeed(opts: { page: number; followerId?: string; tag?: string })` — 공통 부분은 post 소유, 이웃 거르기·정렬은 social.

| 목록 | 거르기 | 정렬 |
|---|---|---|
| 마을 소식 (`followerId` 없음) | `posts.visibility = 'public'` | `created_at DESC, id DESC` (FR-047) |
| 이웃 새 글 (`followerId` 있음) | 공개 글 AND 글 블로그 주인 ∈ 내가 이웃 추가한 회원 | ① `즐겨찾는 이웃의 글 AND created_at >= 시작 날짜의 한국 0시` 인 글 먼저 ② `created_at DESC` ③ `id DESC` (FR-042, D15) |

- "최근 7일(한국 시간)" = 오늘을 포함한 한국 날짜 7일 (research R11). 시작 날짜는 새 파일 `src/lib/social.ts`의 `favoriteWindowStart(todayKST())`가 정하고, 쿼리는 `(${시작 날짜}::date::timestamp AT TIME ZONE 'Asia/Seoul')`와 비교한다. 예: 오늘이 10월 7일이면 10월 1일 0시(한국) 이후에 쓴 즐겨찾은 이웃 글이 위로 간다.
- 한 페이지 8개, 페이지를 넘겨도 같은 정렬이 이어진다. 인기 태그는 이웃 글이 아니라 마을 전체 공개 글 기준 (FR-042).
- 공통 모듈 추가: `paged(where, page, extra, orderFirst?: SQL)`와 `baseList(where, orderFirst?: SQL)` — `listFeed`는 `paged`만 부르므로 `paged`가 받아 `baseList`로 넘기고, `baseList`는 `orderFirst`가 있으면 기존 정렬 앞에 붙인다. 없으면 지금과 같다 (post 소유 함수, post와 합의).
- blog가 `listFeed`에 `search` 선택 인자를 더한다(blog plan B9). 이웃 거르기·즐겨찾기 정렬과 독립이다.

## 4. `follows.is_favorite` (town이 쓰는 데이터 계약)

| 항목 | 계약 |
|---|---|
| 컬럼 | `follows.is_favorite boolean NOT NULL DEFAULT false` (social이 마이그레이션) |
| 쓰는 쪽 | town (TOWN-08 "내 이웃 목록"의 ☆). 이웃 행이 있는 경우에만 바꿀 수 있다(행이 없으면 갱신 0행) |
| "최대 10명" | town이 `lockUser(나)` 안에서 세어 지킨다 (town FR-033). social은 제약을 두지 않는다 |
| 이웃 취소 | social의 `toggleFollow`가 행을 지우므로 즐겨찾기도 함께 사라진다 |
| 읽는 쪽 | social: 이웃 새 글 정렬. town: 광장 집 목록 |
