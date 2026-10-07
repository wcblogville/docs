# Quickstart: 글 (POST) 검증 시나리오

**Feature**: `003-post` | **Date**: 2026-10-07 | **Plan**: [plan.md](plan.md)

이 기능이 끝까지 동작하는지 확인하는 순서다. 명령은 코드 저장소 루트에서 실행한다. 구현 코드는 tasks·구현 단계에서 쓴다.
계약은 [contracts/](contracts/), 데이터 규칙은 [data-model.md](data-model.md)를 본다.

## 0. 전제

| 전제 | 이유 |
|---|---|
| auth 단계 1이 끝남: 가입하면 바로 블로그가 생기고 `e2e/helpers.mjs`의 `loginDev`가 온보딩 없이 동작 | 모든 e2e가 `loginDev`로 회원을 만든다 |
| blog 단계 2가 끝남: `subcategories` 표 (UK (`category_id`, `id`)) | 소분류 마이그레이션(D)과 `e2e/post-categories.mjs` |
| social 단계 5(D15 이웃 새 글 순서, `replies` 분리)가 끝남 | US4-10, 카드 댓글 수(답글 포함) 확인. 없으면 그 항목만 건너뛴다 |
| PostgreSQL 15 이상 (README 안내는 17) | `ON DELETE SET NULL (subcategory_id)` |
| 의존성 설치(README 절차), `.env.local`에 `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `ADMIN_USERNAME`, `ADMIN_PASSWORD`, `UPLOAD_DIR`(선택) | 값은 문서에 적지 않는다 |
| 로컬 DB에서만 | `npm run db:reset`은 로컬 주소에서만 돈다 |

## 1. 정적 검사와 단위 테스트

```bash
npx tsc --noEmit
npx eslint
npm test
```

기대 결과:

- 타입·린트 오류 0.
- `npm test` = `test:game` → `test:ids` → `test:sanitize`(XSS 6가지 + 첨부 주소 + 글자 수 24경우) → **`test:post`(새)** 가 모두 `✅`, 종료 코드 0.
- `test:post`(`scripts/test-post.ts`)가 확인하는 것:

  | 묶음 | 확인 | spec |
  |---|---|---|
  | `postInputSchema` | 빈 제목·공백 제목 `제목을 적어 주세요`, 101자 `제목은 100자까지예요`, 200,001자 `글이 너무 길어요`, 태그 301자 `태그는 모두 합쳐 300자까지예요`, 공개 설정 `x`·카테고리 `0`·`abc`·`1.5`·`99999999999`·`1e1` `잘못된 요청이에요`, 여러 오류면 정해진 순서의 첫 번째 | FR-006, FR-009, FR-018, Edge Cases |
  | 본문 크기 판단 | 900,000바이트 넘는 본문은 브라우저에서 검사, 그 아래는 서버로 보냄 | Assumptions (요청 크기) |
  | `parseTags` | `git, #회고, Git` → `git`, `회고` / 21자 버림 / 11개 → 앞 10개 / `c#` → `c` / 가운데 공백 한 칸 / 줄바꿈 구분 | FR-041, US6-1·2, Edge Cases |
  | `attachmentAccess` | 공개 글 첨부 누구나, 비공개 글 첨부 주인만(관리자 거부), 붙지 않은 첨부 올린 사람만, 프로필 사진 누구나 | FR-029, FR-059 |
  | `pasteAction` | 내 것·안 붙음 keep, 내 것·이 글 keep, 내 것·다른 글 reupload, 남의 것·없는 키 drop | FR-047, D7 |

## 2. DB 준비

개발 서버를 한 터미널에서 띄운다.

```bash
npm run dev
```

다른 터미널에서, 먼저 마이그레이션만 적용한다 (기존 개발 DB가 있으면 데이터 이전 B의 결과를 아래 표로 확인할 수 있게 `db:reset` 전에 멈춘다):

```bash
npm run db:migrate
```

확인한 뒤 나머지를 실행한다:

```bash
npm run db:seed && npm run db:reset && npm run admin:create
```

마이그레이션 뒤 확인 (Drizzle Studio `npm run db:studio` 또는 psql):

| 확인 | 기대 |
|---|---|
| `posts`에 `subcategory_id`, FK `posts_subcategory_fk`(`ON DELETE SET NULL (subcategory_id)`), CHECK `posts_subcategory_check`, 트리거 `posts_clear_subcategory` | 모두 있음 |
| `attachments`에 `post_id`(FK `ON DELETE SET NULL`), `detached_at`, 인덱스 (`post_id`), 트리거 `attachments_track_detached` | 모두 있음 |
| `post_views` 표, PK (`post_id`, `date`, `visitor_id`) | 있음 |
| 데이터 이전 B 뒤: 본문에 `/files/키`가 있는 기존 글의 첨부 `post_id` | 채워짐 (`db:reset` 전에 확인) |

이전 B의 개수는 [data-model.md §7.2](data-model.md#72-이전-뒤-확인-quickstart에서-실행)를 본다.

## 3. E2E 실행 순서

스크린샷 폴더를 `shots`로 쓴다. 새 시나리오는 실패가 있으면 종료 코드 1이다.

```bash
node e2e/blog.mjs shots              # tester1·tester2와 글 준비 (다른 시나리오의 전제). 카테고리 칸 이름 `대분류`로 고침
node e2e/post-write.mjs shots        # 새
node e2e/post-lists.mjs shots        # 새
node e2e/post-categories.mjs shots   # 새 (blog 단계 2 필요)
node e2e/post-views.mjs shots        # 새
node e2e/attachment-links.mjs shots  # 새 (안에서 npm run posts:cleanup 실행)
node e2e/post-drafts.mjs shots       # 새
```

회귀 (이 기능이 고친 파일 포함):

```bash
node e2e/write-count.mjs shots       # 보상 판단을 안내 문구로, 글 ID를 주소 경로에서 읽게 고침
node e2e/attachments.mjs shots
node e2e/params.mjs shots            # 카테고리 칸 이름 `대분류`로 고침
node e2e/visits.mjs shots            # bv_visitor 쿠키를 같이 써도 방문자 수 그대로
node e2e/social.mjs shots
node e2e/farm.mjs shots              # 보상 판단을 안내 문구로 읽게 고침
```

정리 작업은 손으로도 돌려 본다 (`posts:cleanup`은 이 기능이 `package.json`에 **새로 추가**하는 명령):

```bash
npm run posts:cleanup -- --dry-run
npm run posts:cleanup
```

## 4. 시나리오별 기대 결과

### `e2e/post-write.mjs` — 글쓰기·수정·삭제·공개 범위 (실행마다 새 회원 A·B)

| 단계 | 기대 | spec |
|---|---|---|
| A가 새 글쓰기를 엶 | 본문 첫 줄에 흐린 `오늘 배운 것, 생각한 것, 무엇이든 적어 보세요 ✏️`(글자를 쓰면 사라짐), [🌍 공개]가 골라져 있음 | US1-9, US3-1 |
| [🔒 비공개]를 고름 | 아래 상자 `N자 · 비공개 글은 보상이 없어요` (100자 이상이어도) | US3-2 |
| A가 100자 이상 공개 글 발행 | 그 글 상세로 이동, 맨 위 `🎉 글을 발행했어요! ✨ 경험치 30 · 🪙 30 코인을 받았어요`, 서식 그대로 | US1-1, US1-3 |
| 새로고침 / B가 같은 주소(`?new=1` 포함)를 엶 | 안내 없음, 주소에서 `?new` 사라짐 | FR-013 |
| A가 비공개 글·99자 글 발행 | `🎉 글을 발행했어요! (비공개 글, 짧은 글, 또는 오늘 글쓰기 보상 3번을 다 받아서 보상은 없어요)` | US1-2, US1-8 |
| 공백 제목 / 빈 본문 | `제목을 적어 주세요` / `본문을 적어 주세요`, 제목·카테고리·공개 설정·본문·태그 그대로 | US1-5, US1-6 |
| 요청 조작: 공개 설정 `x`, 카테고리 `abc`·`0`·`1.5` | `잘못된 요청이에요` (영어 문구 없음), 저장 안 됨 | Edge Cases, SC-008 |
| 요청 조작: 제목 101자, 본문 200,001자, 태그 301자 | 각각 `제목은 100자까지예요`, `글이 너무 길어요`, `태그는 모두 합쳐 300자까지예요` | Edge Cases |
| 화면에서 한글 40만 자를 붙여 넣고 발행 | 오류 화면 없이 `글이 너무 길어요`, 입력값 그대로 | Assumptions, R11 |
| A [수정] | 저장값(대분류·소분류 포함) 채움, 버튼 [수정 완료], `글을 고치고 있어요` → 저장 후 안내 없음, 보상 없음, `created_at`·목록 순서 그대로 | US2-1, US2-2 |
| B가 `/write/{A의 글}`, `/write/0`, `/write/abc` | 404 | US2-4 |
| B가 A의 글 번호로 저장 요청 조작 | 오류 화면 `잠깐 문제가 생겼어요`, DB 변화 0 | US2-5, SC-006 |
| B가 A의 글·범위 밖 번호로 삭제 요청 조작 | 지워진 글 0 | US2-6, SC-006 |
| B가 A의 글 상세 | [수정] [삭제] 없음 | US2-7 |
| A [삭제] | 확인 창 `이 글을 삭제할까요? 댓글과 공감도 함께 지워져요.`, 확인 → 블로그 홈, 댓글·공감·태그 연결 없음, 원장 그대로(레벨·잔액 줄지 않음) / [취소] → 그대로 | US2-3, US2-8 |
| A가 오늘 보상 3번 받은 뒤 한 글을 지우고 다시 발행 | 보상 없음 | US2-9 |
| A의 비공개 글을 B·방문자·관리자가 엶 | 404 (`길을 잃었어요`), 탭 제목 `글 \| Blogville` | US3-3, FR-028 |
| B가 보던 글을 A가 비공개로 바꾼 뒤 B가 공감·댓글 | 공감 저장 안 됨, 댓글 `글을 찾을 수 없어요` | US3-6 (social 함수) |
| 공개→비공개→공개 | 댓글·공감·보상 그대로, 새 보상 없음 | US3-7 |
| 로그아웃 상태로 `/write` | 첫 화면(`/`) | US1-10 |

### `e2e/post-lists.mjs` — 목록·태그·화면 폭 (새 회원 + 글 9개)

| 단계 | 기대 | spec |
|---|---|---|
| 글 9개 블로그 홈·`/feed`·`/tags/*` | 1페이지 8개·2페이지 1개, 최신순(같은 시각이면 나중 글 먼저) | US4-1 |
| 글 8개 이하 | 페이지 번호 없음 | US4-2 |
| 20페이지 중 10페이지 (pg로 글을 넣어 만듦) | `1 … 8 9 10 11 12 … 20`, 10만 `aria-current="page"`, [이전] [다음] 없음 | US4-3 |
| 대분류를 고른 블로그 홈 2페이지 | 그 대분류 글만 | US4-5 |
| 글 수정 뒤 목록 | 순서·카드 날짜 그대로 | US4-6 |
| 카드 | `/feed`·`/feed/following`·`/tags/*`는 작성자 줄, 블로그 홈은 없음. `♥ N` `💬 N` `👀 N` 0도 보임 | US4-7, FR-036 |
| 글 없는 목록 4곳 | 빈 화면 문구, 페이지 번호 숨김 | US4-8 |
| 로그아웃 상태로 `/feed/following` | `/` | US4-9 |
| (social 단계 5 뒤) pg로 즐겨찾은 이웃의 3일 전·10일 전 공개 글, 일반 이웃의 어제 공개 글을 준비 → `/feed/following` | 3일 전 → 어제 → 10일 전 순, 페이지를 넘겨도 이어짐 (정렬은 social 구현) | US4-10, FR-035 |
| (social 단계 5 뒤) 댓글 2개(1개 삭제) + 답글 1개인 글의 카드 | `💬 2` (삭제 뺌, 답글 포함) | FR-036 |
| A의 비공개 글을 B·방문자가 찾음: A 블로그 홈·`/feed`·`/feed/following`·`/tags/*`·A의 다른 공개 글 상세의 이전/다음 글·인기 태그 수·블로그 `글 N` | 어디에도 없고 수에도 안 들어감 (광장 이웃집 순서는 town 시나리오) | US3-4, FR-024, SC-004 |
| A가 자기 블로그 홈·이전/다음 글을 봄 | 비공개 글이 `🔒 비공개` 배지와 함께 목록에 있고, 이전/다음 글에도 나옴(배지 없음) | US3-5, FR-025 |
| 태그 `git, #회고, Git` | `#git` `#회고` 2개 | US6-1 |
| 21자 태그 + 11개 | 21자 빠지고 앞 10개 | US6-2 |
| 태그 고치기·비우기 | 빠진 태그 연결 없음, 비우면 태그 줄 없음 | US6-3 |
| 태그 칸에 300자 넘게 입력 | 300자에서 더 들어가지 않음 (조작 요청은 `e2e/post-write.mjs`) | US6-4 |
| 로그아웃 상태 `/tags/{태그}` | 공개 글 8개씩, 없으면 `이 태그가 달린 글이 없어요.` (200) | US6-5 |
| 인기 태그 | 많은 순 → 이름순, 최대 30, `#태그 N`, 비공개 글 태그 수에 없음 | US6-6, US6-8 |
| 태그 `100%` | 목록·탭 제목 `#100% \| Blogville` 정상 | US6-7 |
| 375px: 글쓰기(대분류·소분류 두 칸 포함), 사진·파일 카드가 있는 글 상세, 목록 4곳 | 가로 스크롤 0 (`scrollWidth - clientWidth ≤ 0`), 스크린샷 | SC-009, FR-066 |
| 375px: 페이지 번호, 에디터 도구·[🖼 사진]·[📎 파일], 공개 토글, 대분류·소분류 칸, 상세 배지·`#태그`·[수정]·[삭제], 마을 소식 탭·인기 태그 칩, 카드 작성자 줄 | `boundingBox()` 너비·높이 모두 44px 이상 | constitution VI, R21 |

### `e2e/post-categories.mjs` — 대분류·소분류 (pg로 소분류를 넣어 준비)

| 단계 | 기대 | spec |
|---|---|---|
| 새 글 화면 | 대분류 `카테고리 없음` 선택됨, 내 대분류만 관리 순서대로 | US5-1 |
| 대분류 "개발" 고름 | 소분류 칸에 "개발" 아래 소분류만, 안 골라도 발행됨 | US5-2 |
| 대분류를 바꿈 | 고른 소분류 풀림, 새 대분류 소분류만 | US5-3 |
| 카테고리 없이 발행 | 카드·상세 배지 없음 | US5-4 |
| "개발 + Git" 발행 | 상세·카드 배지 `개발 › Git`, 블로그 홈 "개발"로 거르면 그 글 포함 | Independent Test, FR-033 |
| 다른 대분류의 소분류를 섞어 조작 | `잘못된 요청이에요`, 저장 안 됨, 입력값 그대로 | US5-5 |
| 남의 블로그 대분류·없는 대분류 조작 / 화면을 연 뒤 대분류를 지움 | 오류 없이 `카테고리 없음`으로 저장 | US5-6 |
| 카테고리 번호 `99999999999` | `잘못된 요청이에요`, 제목·본문·태그 그대로 | US5-9 |
| 상세 배지 누름 | 그 블로그의 해당 카테고리 목록 | US5-7 |
| 대분류 이름 바꿈 | 배지 새 이름 | US5-8 |
| 소분류 삭제 | 글은 대분류만 남음 | Edge Cases |
| 소분류 글이 있는 대분류를 블로그 관리에서 삭제 | 오류 없이 글이 `카테고리 없음` (blog `deleteCategory`가 먼저 비움) | Edge Cases |
| 소분류 글이 있는 대분류 행을 pg로 직접 `DELETE` (앱 처리 없이 트리거만) | 오류 없이 글이 `카테고리 없음`, 소분류 CASCADE | Edge Cases, R4 |
| 소분류 글이 있는 회원을 pg로 삭제 | 오류 없이 지워짐 (트리거) | R4 (탈퇴 경로) |

### `e2e/post-views.mjs` — 조회수 하루 1번 (`blog.mjs` 다음)

| 단계 | 기대 | spec |
|---|---|---|
| 새 브라우저(쿠키 없음)가 A의 공개 글을 엶 | 화면 `👀` = 이전+1, DB `view_count` +1, `post_views` 1행 | US7-1 |
| 같은 브라우저로 새로고침 9번, 다른 탭, 다른 화면 갔다 오기 | 더 오르지 않음, 화면은 저장값 (첫 방문 쿠키 경쟁 없음 확인 포함) | US7-6, SC-012, R3 |
| 같은 날 다른 글 B 처음 엶 | B만 +1 | US7-8 |
| `post_views.date`를 어제로 바꾼 뒤 다시 엶 | +1 | US7-7, SC-012 |
| 주인이 자기 글(비공개 포함)을 엶 | 오르지 않음 | US7-2 |
| 로그아웃한 주인·다른 회원·관리자 | +1 (브라우저마다) | US7-1 |
| 없는 블로그·없는 글·`abc`·다른 블로그 주소·남의 비공개 글 | 404, 오르지 않음 | US7-3 |
| 조회수가 오른 뒤 | `updated_at` 그대로 | US7-4 |
| 상세에서 공감·댓글 | 조회수 그대로 | US7-5 |
| 쿠키 `bv_visitor` | `HttpOnly`, `SameSite=Lax`, `Path=/`, 1년 | R3, BLOG-06 |
| 서버 렌더만 하는 요청(`page.request.get`) | 오르지 않음 | 기본값(브라우저에서 열렸을 때만) |

### `e2e/attachment-links.mjs` — 첨부가 글에 붙음 (실행마다 새 회원 A·B)

| 단계 | 기대 | spec |
|---|---|---|
| A가 사진·파일을 넣어 공개 글 P1 발행 | 두 첨부 `post_id` = P1 | FR-047 |
| A가 P1 본문에서 사진을 빼고 저장 | 사진 `post_id` NULL, `detached_at` 채워짐 | FR-047, FR-059 |
| A가 P1의 사진·파일 카드를 복사해 새 글 P2에 붙여 넣음 | `올리는 중... (1/2)`, 새 키로 들어감 → 발행 후 새 키 `post_id` = P2, P1 첨부 그대로, 원래 이름·크기·내용(바이트) 같음 | US8-10, US9-9, SC-013 |
| 다시 올리는 동안 | [🖼 사진]·[📎 파일]이 눌리지 않고 발행 버튼이 `첨부를 올리는 중...`. 그 사이 다른 파일을 붙여 넣으면 넣지 않고 `다른 파일을 올리는 중이에요. 끝난 뒤 다시 넣어 주세요` | US8-2, FR-049 |
| 원본 파일을 저장소에서 지워 다시 올리기를 실패시킴 (pg·파일 시스템으로 준비) | 그 사진·파일은 본문에 들어가지 않고 `파일 이름: 올리지 못했어요. 다시 시도해 주세요`, [✕]로 닫힘, 제목·본문 그대로 | FR-047, US8-6, Edge Cases |
| A가 P1을 지운 뒤 P2를 엶 | P2의 다시 올린 사진·파일은 그대로 보이고 내려받아짐 (`post_id` = P2) | Edge Cases, SC-013 |
| [📎 파일]로 PNG를 고름 | 사진으로 들어감 | US9-2 |
| 파일 카드의 `data-name`·`data-size`를 조작해 저장 | 처음 올린 이름·크기로 저장, 내려받는 이름도 원래 이름 | US9-6 |
| 같은 글 수정 화면 안에서 잘라 붙이기 | 다시 올리지 않음 (keep) | R8 |
| 저장 요청 조작: P1 키·B의 키·없는 키·다른 사이트 사진 | 저장된 본문에서 빠짐, P1 첨부는 P1에 그대로 | US8-8, SC-003 |
| B가 A의 사진 HTML을 B 글쓰기에 붙여 넣음 | 에디터에 들어가지 않음 | R8 (drop) |
| A의 비공개 글 P4의 사진 주소를 B·방문자·관리자가 엶 | 404 `파일을 찾을 수 없어요`. A는 열림 | US3-8, FR-029, SC-004 |
| A가 올리기만 한 첨부 주소를 B·방문자가 엶 | 404. A는 열림 | FR-059 기본값 |
| 응답 머리글 | `Cache-Control: private, no-cache`, `ETag`, 같은 `If-None-Match`면 허용된 사람만 304 | R9 |
| A가 P1 삭제 | P1 첨부 `post_id` NULL + `detached_at` (트리거) | FR-016, FR-059 |
| pg로 붙지 않은 첨부의 `created_at`·`detached_at`을 이틀 전으로 → `npm run posts:cleanup` | 그 행과 파일 0개 남음, 방금 올린 붙지 않은 첨부·붙은 첨부는 남음 | SC-011 |
| pg로 회원 하나 삭제 → 2시간 전으로 파일 시각 바꿈 → `npm run posts:cleanup` | 주인 없는 파일 지워짐 | ERD 3.14 |
| (auth의 `profiles.photo_key`가 들어온 뒤) pg로 A의 붙지 않은 첨부를 A의 `photo_key`로 지정 → 이틀 전으로 → `npm run posts:cleanup` | 그 행·파일이 남음, B·방문자도 열림. A가 그 주소를 글 본문에 넣어 저장하면 본문에서 빠짐 | FR-047, FR-059, SC-011, R19 |
| 화면 오류 | 없음 (일부러 거부시킨 응답 제외) | - |

### `e2e/post-drafts.mjs` — 임시 저장

| 단계 | 기대 | spec |
|---|---|---|
| 새 글에 제목·서식 본문·태그·대분류·비공개 입력 후 2초 이상 멈춤 | 아래 상자 `임시 저장됨 HH:mm` (처음엔 `작성 중인 글은 이 브라우저에 자동 저장됩니다.`) | US10-1, FR-061 |
| 새로고침 → `작성 중이던 글이 있어요. 불러올까요?` 확인 | 모든 칸 서식 포함 그대로 | US10-2, SC-010 |
| 새로고침 → 취소 | 빈 화면 | US10-3 |
| 제목·본문 비우고 대분류만 바꿈 → 2초 → 새로고침 | 묻지 않음 | US10-4 |
| 마지막 입력 뒤 2초 안에 발행 성공 → 글쓰기 다시 열기 | 묻지 않음 | US10-5 |
| 빈 줄뿐인 본문으로 발행 실패 → 다시 열기 | 임시 글 남음 | US10-6 |
| 같은 브라우저에서 회원 B로 로그인해 글쓰기 | A의 임시 글 안 보임 | US10-7 |
| `localStorage`가 오류를 내게 만든 브라우저 | 글쓰기·발행 정상, 오류 없음 | US10-8 |
| 수정 화면 | 임시 저장 줄·질문 없음 | US10-9 |
| A가 로그아웃 → 다시 로그인해 글쓰기 | 묻지 않음 (로그아웃 때 지움) | 기본값 (POST-08) |
| 불러온 임시 글의 대분류가 그 사이 지워짐 | `카테고리 없음`으로 채움 | Edge Cases, FR-062 |

## 5. 성공 기준(SC) 확인 표

| SC | 확인 방법 |
|---|---|
| SC-001 (처음 쓰는 회원 3분 안 발행) | 팀원이 아닌 사람 1명에게 안내 없이 글쓰기를 맡겨 시간을 잰다 (수동) |
| SC-002 (목록 1초, 글 1,000개) | pg로 `normal01`(`e2e/auth.mjs`가 만든 회원, `nonfunctional.mjs`가 로그인하는 계정)의 블로그에 태그 하나를 단 공개 글 1,000개를 넣고 `npm run build && npm run start` 뒤 `node e2e/nonfunctional.mjs shots`. `/feed`·`/@normal01`·`/tags/{태그}`(이 기능이 측정 경로에 더함)를 본다. 배포 후 다시 잰다 (NF-07) |
| SC-003 (위험한 본문 0건) | `npm run test:sanitize`, `e2e/attachments.mjs`, `e2e/attachment-links.mjs` |
| SC-004 (비공개 글·첨부 노출 0건) | `e2e/post-write.mjs`, `e2e/post-lists.mjs`, `e2e/attachment-links.mjs` |
| SC-005 (글자 수 일치 100%) | `npm run test:sanitize`(24경우), `e2e/write-count.mjs` |
| SC-006 (남의 글 수정·삭제 0건) | `e2e/post-write.mjs`, `e2e/params.mjs` |
| SC-007 (사진·파일 100%, 바이트 같음) | `e2e/attachments.mjs`, `e2e/attachment-links.mjs` |
| SC-008 (오류 문구 100% 한국어) | `npm run test:post`, `e2e/post-write.mjs`, `e2e/attachments.mjs` + 화면 오류 문구 스크린샷 확인 |
| SC-009 (375px 가로 스크롤 0) | `e2e/post-lists.mjs`(누르는 영역 44px 포함), `e2e/attachments.mjs` |
| SC-010 (임시 글 100% 복원) | `e2e/post-drafts.mjs` |
| SC-011 (하루 지난 붙지 않은 첨부 0%) | `e2e/attachment-links.mjs` (`npm run posts:cleanup`) |
| SC-012 (하루 10번 → 1, 다음 날 +1) | `e2e/post-views.mjs` |
| SC-013 (한 첨부 두 글 0건, 원래 글 100%) | `e2e/attachment-links.mjs` + pg 확인: 첨부마다 `post_id`는 하나뿐(구조), 원래 글 첨부 열림 |

## 6. 화면 확인 (스크린샷)

`shots/`에서 눈으로 본다: 글쓰기(대분류·소분류 두 칸, PC·375px), 아래 상자의 임시 저장 줄, 발행 안내, 상세 배지 `대분류 › 소분류`, 붙여 넣기 다시 올리기 중 `올리는 중... (1/2)`, 비공개 글 첨부 404 화면.

## 7. 결과 기록

CI가 없으므로 PR 작성자가 위 명령의 결과(✅/❌ 줄)를 PR 설명에 붙인다 (constitution 품질 관문). 비밀값은 붙이지 않는다.
