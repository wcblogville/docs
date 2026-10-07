# Contract: 글 상세·조회수·글 목록

**Feature**: `003-post` | 관련: POST-01, POST-02, POST-03, POST-04, POST-05, POST-06, BLOG-06, TOWN-08(이웃 새 글 순서)

코드 위치는 코드 저장소 기준. `/@주소`, `/@주소/글ID`는 `next.config.ts`의 `rewrites()`로 `/blog/[slug]`, `/blog/[slug]/[postId]`에 연결된다 (blog 소유, 변경 없음).

## 1. 글 상세 `/@{주소}/{글ID}`

파일: `src/app/blog/[slug]/[postId]/page.tsx` (post 소유. 댓글·공감 영역은 social, `RecordVisit`는 blog).

**접근**

| 경우 | 결과 |
|---|---|
| 글 번호가 `parseId` 실패 / 없는 블로그 / 없는 글 / 다른 블로그의 글 번호 | 404 (🧭 `길을 잃었어요` / `찾는 블로그나 글이 없거나, 비공개 글이에요.` / [광장으로 돌아가기]) |
| 비공개 글 + 주인이 아님 (다른 회원·방문자·관리자) | 404 (같은 화면, FR-023) |
| 공개 글, 또는 내 비공개 글 | 200 |

**탭 제목**: 공개 글 `{제목} | Blogville` / 비공개 글은 주인이 봐도 `글 | Blogville` (FR-028) / 404면 `글 | Blogville`.

**머리** (FR-021, FR-033): [카테고리 배지(노랑, `대분류` 또는 `대분류 › 소분류`)] [`🔒 비공개`] / 제목 / `YYYY. MM. DD. HH:mm`(한국 시간) `👀 N` …… 주인만 [수정] [삭제].

- 배지 링크: 소분류가 있으면 `/@{주소}?category={대분류ID}&sub={소분류ID}`(blog R-13: `sub`가 올바르면 소분류로 거르고 `category`는 무시. blog 단계 4 전에는 대분류로 거른다), 없으면 `/@{주소}?category={대분류ID}` (R16).
- 배지·`#태그`·[수정]·[삭제]는 누르는 영역 44×44px (R21).
- 본문 아래(공감 버튼 위) `#태그` 배지 이름순, 누르면 `/tags/{인코딩한 태그}` (FR-043).
- 이전/다음 글: 주인은 비공개 포함, 다른 사람은 공개 글만, 배지 없음 (FR-024·025).

**발행 안내** (FR-013, R13)

| 조건 (모두) | 보이는 것 |
|---|---|
| 주소에 `?new`가 있음 + 보는 사람이 주인 + 글이 만들어진 지 10분 안 + 원장에 (주인, `post`, 이 글 ID) 행이 있음 | 맨 위 `🎉 글을 발행했어요! ✨ 경험치 30 · 🪙 30 코인을 받았어요` |
| 위와 같으나 원장 행이 없음 | `🎉 글을 발행했어요! (비공개 글, 짧은 글, 또는 오늘 글쓰기 보상 3번을 다 받아서 보상은 없어요)` |
| 그 밖 (다른 사람, 10분 지남, `?new` 없음) | 안내 없음 |

안내를 그린 `PublishNotice`(`src/components/blog/publish-notice.tsx`, 새)는 처음 그린 뒤 주소에서 `?new`를 지우고(`history.replaceState`), 주인의 임시 글을 지운다.

**조회수 표시** `👀 N` (FR-046, R2): `ViewCount`(`src/components/blog/view-count.tsx`, 새)가 그린다.

| 보는 사람 | 처음 그리는 값 | 화면이 열린 뒤 |
|---|---|---|
| 주인 | 저장값 | 부르지 않음 |
| 주인이 아님 + 이 브라우저(`bv_visitor`)로 오늘 이 글을 센 기록 없음 (쿠키 없음 포함) | 저장값 + 1 | `recordPostView(글ID)` 한 번 → 돌려받은 값으로 바꿈 |
| 주인이 아님 + 오늘 이미 셈 | 저장값 | `recordPostView` 한 번 → 0행 추가, 저장값 그대로 |

- `recordPostView`는 컴포넌트가 처음 그려질 때만 부른다 (의존성 `글ID`). 공감·댓글 뒤 화면을 다시 그려도 다시 부르지 않는다 (US7-5).
- 서버 렌더는 조회수를 바꾸지 않는다 (`incrementViewCount` 삭제).

## 2. `recordPostView(postId: number)` — 조회 기록 (새)

파일: `src/app/blog/[slug]/[postId]/actions.ts` (`"use server"`, 새).

| 항목 | 내용 |
|---|---|
| 권한 | 로그인 없이 부를 수 있다 (`recordBlogVisit`와 같음). 대신 아래를 함수 안에서 직접 확인한다 |
| 입력 | `parseId(postId)` 실패 → `null` |
| 확인 | 글이 없음 → `null` / 비공개 글 → `null` (주인은 세지 않으므로 비공개 글은 늘 세지 않음) / 보는 사람이 그 블로그 주인 → `{ viewCount: 저장값 }` |
| 쿠키 | `ensureVisitorId()`(`src/server/visitor.ts`, 새): `bv_visitor`가 UUID 형식이 아니면 새 UUID로 만든다 (`HttpOnly`, `SameSite=Lax`, `Path=/`, `Max-Age` 1년, 배포 `Secure`) |
| 처리 (트랜잭션) | `post_views` (`post_id`, 한국 오늘, `visitor_id`) INSERT `ON CONFLICT DO NOTHING RETURNING` → 들어갔으면 `posts.view_count + 1` (`updated_at` 그대로) |
| 반환 | `{ viewCount: 현재 저장값 }` |
| 하지 않는 것 | IP·회원 ID 저장, 검색 로봇 가려내기 (spec 기본값) |

## 3. 글 목록 화면 (목록 규칙은 post, 화면 소유는 표 참고)

| 주소 | 파일 (소유) | 접근 | 대상 글 | 빈 화면 문구 |
|---|---|---|---|---|
| `/@{주소}` | `src/app/blog/[slug]/page.tsx` (blog) | 누구나 | 그 블로그 글 (주인은 비공개 포함), `?category=` 거르기 (blog 단계 4 뒤 `?sub=`, blog R-13) | `🌱` `아직 글이 없어요.` + 주인만 [첫 글 쓰기] |
| `/feed` | `src/app/feed/page.tsx` (social) | 누구나 | 모든 공개 글 | `아직 마을에 글이 없어요. 첫 글의 주인공이 되어 보세요! ✏️` |
| `/feed/following` | `src/app/feed/following/page.tsx` (social) | 회원 (아니면 `/`, FR-040) | 이웃의 공개 글. 순서는 social이 D15로 구현 | `아직 이웃이 없거나 이웃의 새 글이 없어요.` / `마을 소식에서 마음에 드는 블로그를 이웃으로 추가해 보세요.` |
| `/tags/{태그}` | `src/app/tags/[name]/page.tsx` (post) | 누구나 | 그 태그의 공개 글 (주인에게도 공개 글만) | `이 태그가 달린 글이 없어요.` (404 아님) |

**공통 목록 규칙** (FR-035~039, `src/server/blog.ts`의 post 소유 함수 + `src/components/blog/post-card.tsx`, `src/components/pagination.tsx`)

- 한 페이지 8개 (`PAGE_SIZE`), `created_at` 최신순, 같으면 `id` 큰 것 먼저. 수정해도 순서·날짜 그대로.
- `?page=`가 `parseId` 실패(`0`, `-1`, `abc`, `2.5`, `012`, `1e1`, `99999999999999999999`) → 1페이지 (`parsePage`, `src/components/pagination.tsx`, 변경 없음). 마지막보다 큰 페이지 → 빈 목록 + 빈 화면 문구.
- 페이지 번호: 첫·마지막·현재 앞뒤 2개, 사이는 `…`, 현재는 강조 + `aria-current="page"`, 1페이지뿐이면 숨김. [이전] [다음] 없음. 번호 링크는 누르는 영역 44×44px (바뀜, R21. blog·social 요청).
- 마을 소식 탭 [🏘 마을 전체] [💛 이웃 새 글]·인기 태그 칩(`feed-view.tsx`)·카드 작성자 줄(`post-card.tsx`)도 누르는 영역 44×44px (바뀜, R21. social D-10 요청).
- 카드: [배지] [`🔒 비공개`] / 제목(자르지 않음) / 요약(본문 글자 앞 160자, 2줄, 넘치면 `…`) / `YYYY. MM. DD.` `♥ N` `💬 N` `👀 N` (쉼표 없음, 0도 보임). 사진 없음. 한 줄에 1개.
- `/feed`, `/feed/following`, `/tags/*` 카드는 위에 작성자 줄 `{캐릭터} {닉네임} · {블로그 이름}`(→ 블로그 홈), 나머지는 글 상세로. 블로그 홈 카드는 작성자 줄 없음.
- 댓글 수는 삭제 안 된 댓글 + 답글 (정의는 social이 `listColumns.commentCount`에 둔다).
- "🏷 인기 태그" (`/feed`, `/feed/following`, `/tags/*`): 공개 글 기준 사용 횟수 많은 순 → 이름순, 최대 30개, `#태그 N`, 없으면 `아직 태그가 없어요`. 768px 이상 오른쪽 따라다님, 좁으면 목록 아래 (`getAllTags`, 변경 없음).
- 태그 목록 탭 제목 `#{태그} | Blogville`, 제목 `🏷 #{태그}`. `%`가 든 태그도 정상 (#20, 변경 없음).

**post가 다른 spec에 여는 함수 인자** (`src/server/blog.ts`)

| 함수 | 인자 | 반환에 더하는 것 | 쓰는 곳 |
|---|---|---|---|
| `listBlogPosts` | `{ blogId, isOwner, categoryId?, subcategoryId?, page }` (`subcategoryId` 새로 추가: `WHERE subcategory_id = ?`) | 카드마다 `subcategoryName` | blog 단계 4 (블로그 홈 트리) |
| `listFeed` | post는 바꾸지 않음 (`{ page, followerId?, tag? }`). 이웃 거르기·즐겨찾기 정렬은 social, 선택 인자 `search`는 blog가 "추가만" (blog plan B9) | 카드마다 `subcategoryName` | social, blog, post |
| `paged`, `baseList` | 선택 정렬 인자 `orderFirst`를 social이 "추가만" (social plan D-5, post 동의). 없으면 지금 정렬 `created_at` desc, `id` desc | `subcategories` LEFT JOIN으로 `subcategoryId`·`subcategoryName` | 모든 목록 |
| `getPost` | 변경 없음 (`blogId`, `postId`) | `subcategoryId`, `subcategoryName` | 글 상세, 수정 화면 |

같은 함수를 여러 spec이 고치므로 나중에 merge하는 쪽이 최신 `main`에 맞춘다 (social plan D-5와 같은 약속).
