# Contract: 글쓰기·수정·삭제 (화면 + Server Action)

**Feature**: `003-post` | 관련: POST-01, POST-02, POST-03, POST-04, POST-07, POST-08, POST-09, GAME-05

코드 위치는 코드 저장소 기준. Server Action은 모두 `"use server"` 파일에 있고, Next.js가 출처(Origin)를 확인한다.
화면 문구·오류 문구는 spec 그대로다.

## 1. 화면 라우트

| 주소 | 파일 | 접근 | 실패 시 | 탭 제목 |
|---|---|---|---|---|
| `/write` | `src/app/write/page.tsx` | 로그인한 회원 (`requireMember()`) | 로그인 안 함 → `/` (FR-019) | `글쓰기 \| Blogville` |
| `/write/[postId]` | `src/app/write/[postId]/page.tsx` | 로그인한 회원 + 내 블로그의 글 | 로그인 안 함 → `/` / 남의 글·없는 글·`parseId` 실패(`0`, `abc`, `1e3`, `2147483648`) → 404 (FR-017) | `글 고치기 \| Blogville` |

두 화면이 `PostForm`(`src/components/editor/post-form.tsx`)에 넘기는 값:

| prop | 새 글 | 수정 |
|---|---|---|
| `categories` | `getCategoryOptions(blogId)`: `{ id, name, subcategories: { id, name }[] }[]`, 순서 `position`, `id` | 같음 |
| `initial` | `{ title: "", contentHtml: "", categoryId: null, subcategoryId: null, tags: [], visibility: "public" }` | 저장된 값. 지워진 대분류·소분류는 비어 있음 (FR-031) |
| `draftOwnerId` | 로그인한 회원 ID (임시 저장 켜짐) | 없음 (임시 저장·불러오기 없음, FR-064) |

화면 순서 (FR-020): 제목 칸(`제목`) → [대분류 ▼] [소분류 ▼] + [🌍 공개] [🔒 비공개] → 도구 모음 + 본문(최소 360px, 비어 있으면 흐린 `오늘 배운 것, 생각한 것, 무엇이든 적어 보세요 ✏️`) → 태그 칸(`태그 (쉼표로 구분, 최대 10개)  예: git, 회고`, 300자에서 입력 멈춤) → 아래 상자.

- 에디터 도구·링크 창(`링크 주소 (비우면 링크 해제)`)·[🖼 사진]·[📎 파일]은 지금 그대로다 (FR-002~005, FR-048). 바뀌는 것은 누르는 영역뿐이다: 도구 버튼·[🖼 사진]·[📎 파일]·안내 [✕]·공개 토글·두 선택 칸은 44×44px 이상 (R21).
- 선택 칸 이름(보조 기술·e2e): `대분류`, `소분류`. 대분류 첫 항목 `카테고리 없음`, 소분류 첫 항목 `소분류 없음`(spec에 없는 문구, plan 남은 문제 8). 대분류가 `카테고리 없음`이면 소분류 칸은 눌리지 않는다.
- 아래 상자 첫 줄: `N자 · ` + 안내 하나 (FR-010). `N`은 `toLocaleString("ko-KR")`.
  - 새 공개 글 100자 이상: `저장하면 ✨ 경험치 30 · 🪙 30 보상 (하루 3번까지)`
  - 비공개: `비공개 글은 보상이 없어요`
  - 100자 미만: `100자 이상 쓰면 보상을 받아요`
  - 수정: `글을 고치고 있어요`
- 아래 상자 둘째 줄 (새 글만): 처음 `작성 중인 글은 이 브라우저에 자동 저장됩니다.`, 저장 뒤 `임시 저장됨 HH:mm` (한국 시간) (FR-061).
- 버튼: 새 글 `발행하기` / 수정 `수정 완료` / 처리 중 `저장하는 중...` / 첨부 올리는 중 `첨부를 올리는 중...` (눌리지 않음) (FR-014, FR-049).
- 오류 문구: 버튼 왼쪽 빨간 글씨, 한 번에 하나 (FR-009).

## 2. `savePost(prev, formData)` — 글 저장 (새 글·수정)

`src/app/write/actions.ts`. `useActionState`로 부른다.

**권한**: `requireMember()`. 로그인이 풀렸으면 `/`로 이동하고 저장하지 않는다 (Edge Cases).

**입력 (FormData)**

| 이름 | 형식 | 비면 |
|---|---|---|
| `postId` | 숫자 글자 (수정만) | 새 글 |
| `title` | 글자 | 오류 |
| `contentHtml` | 에디터 HTML | 오류 |
| `categoryId` | 숫자 글자 | 카테고리 없음 |
| `subcategoryId` | 숫자 글자 | 소분류 없음 |
| `tags` | 글자 | 태그 없음 |
| `visibility` | `public` / `private` | `public` |

**검사 순서와 결과** (처음 걸린 하나만 돌려준다. 공용 스키마 `postInputSchema`, `src/lib/post-rules.ts`)

| # | 검사 | 결과 문구 | 근거 |
|---|---|---|---|
| 1 | `postId`가 빈 값이거나 숫자만 적힌 1~2147483647 | `잘못된 요청이에요` | FR-018 |
| 2 | 제목 앞뒤 공백 제거 후 1자 이상 | `제목을 적어 주세요` | FR-006 |
| 3 | 제목 100자 이하 | `제목은 100자까지예요` | FR-006 |
| 4 | 본문 HTML 200,000자 이하 | `글이 너무 길어요` | FR-006 |
| 5 | `categoryId`·`subcategoryId`가 빈 값이거나 숫자만 적힌 1~2147483647 (`0`, `abc`, `1.5`, `99999999999` 포함) | `잘못된 요청이에요` | FR-018, Edge Cases |
| 6 | 태그 칸 300자 이하 | `태그는 모두 합쳐 300자까지예요` | FR-006 |
| 7 | `visibility`가 `public`/`private` | `잘못된 요청이에요` | Edge Cases |
| 8 | 정화 뒤 본문 글자가 있음 | `본문을 적어 주세요` | FR-008 |
| 9 | (트랜잭션 안) 소분류가 다른 대분류 소속이 아님 | `잘못된 요청이에요` | FR-032, R15 |
| 10 | (트랜잭션 안, 수정) 내 블로그의 글 | 예외 → 오류 화면 `잠깐 문제가 생겼어요` / [다시 시도] [광장으로 돌아가기] (`src/app/error.tsx`) | FR-017 |

**실패 반환**: `{ error: 문구, values: { title, categoryId, subcategoryId, tags } }` — 화면은 이 값으로 칸을 되돌리고, 본문과 공개 설정은 화면 상태로 남는다 (FR-009).

**성공 시 처리** (하나의 트랜잭션, constitution V)

1. `lockUser(tx, 나)`.
2. 대분류·소분류 확인 (`FOR KEY SHARE`, 규칙은 [research.md R15](../research.md#r15-대분류소분류-선택과-서버-확인)): 남의·없는 대분류 → 둘 다 NULL, 없는 소분류 → 대분류만.
3. 본문 첨부 행을 키 순서로 `FOR UPDATE` → 붙일 수 있는 첨부만 남겨 정화 ([data-model.md §3.3](../data-model.md#33-글에-붙일-수-있는-첨부-저장할-때)). 정리 작업은 이 잠금을 기다리지 않고 건너뛴다 (R6, [attachments-http.md §3](attachments-http.md)).
4. 새 글: `posts` INSERT. 공개이고 본문 글자 100자 이상이면 `grantReward(tx, 나, "post", 글ID)`(하루 3번, GAME-05), 받았으면 `growForPost(tx, 나)`(TOWN-09, town 함수 호출 유지).
   수정: `posts` UPDATE (`blog_id` = 내 블로그 조건). 보상·발행 안내 없음, `created_at` 그대로 (FR-015).
5. 첨부: 남은 키에 `post_id` = 이 글, 이 글에 붙어 있다가 빠진 첨부는 `post_id` = NULL (트리거가 `detached_at`).
6. 태그: `parseTags`(쉼표·`#`·줄바꿈으로 나눔, 앞뒤 공백 제거, 가운데 공백 한 칸, 소문자, 빈 것·20자 초과 버림, 중복 제거, 앞 10개) → 없는 태그 만들기 → 빠진 연결 삭제·새 연결 추가 (FR-041·042).

**성공 반환**: `revalidatePath("/", "layout")` 후

- 새 글 → `redirect("/@{주소}/{글ID}?new=1")` (보상 여부는 주소에 담지 않는다, R13)
- 수정 → `redirect("/@{주소}/{글ID}")`

**브라우저 사전 검사** (R11): 본문 HTML의 UTF-8 크기가 900,000바이트를 넘으면 요청을 보내지 않고 같은 `postInputSchema`로 검사 1~7을 돌려 첫 문구를 같은 자리에 보인다.

## 3. `deletePost(postId: number)` — 글 삭제

`src/app/write/actions.ts`. 글 상세의 [삭제] → `confirm("이 글을 삭제할까요? 댓글과 공감도 함께 지워져요.")` → 확인이면 부른다 (`src/components/blog/delete-post-button.tsx`).

| 항목 | 내용 |
|---|---|
| 권한 | `requireMember()` + `blog_id` = 내 블로그 조건으로만 지운다 |
| 입력 | `parseId(postId)`가 `null`이면 아무것도 하지 않고 끝 (FR-017, US2-6) |
| 처리 | `DELETE FROM posts WHERE id = ? AND blog_id = 내 블로그`. 태그 연결·댓글·답글·공감·조회 기록 CASCADE, 첨부 `post_id` NULL + `detached_at`(트리거). 보상 회수 없음 (FR-016, D6) |
| 성공 | `revalidatePath("/", "layout")` → `redirect("/@{내 주소}")`, 안내 없음 |
| 남의 글 | 0행 삭제, 같은 이동 (아무것도 지워지지 않음) |

## 4. `classifyPastedAttachments(keys, postId?)` — 붙여 넣은 첨부 판정 (새)

`src/app/write/actions.ts`. 에디터가 붙여 넣는 HTML에 `/files/키`가 있을 때 부른다 (R8).

| 항목 | 내용 |
|---|---|
| 권한 | `requireMember()` |
| 입력 | `keys`: 1~50개, 각 `^[a-f0-9]{32}$` / `postId`: 수정 화면이면 글 번호(`parseId`), 새 글이면 없음 |
| 잘못된 입력 | `{ error: "잘못된 요청이에요" }` (에디터는 그 붙여 넣기를 넣지 않는다) |
| 반환 | `{ items: { key, action: "keep" \| "reupload" \| "drop" }[] }` |
| 판정 | **keep**: 내 것 & (`post_id` NULL 또는 = `postId`) & 프로필 사진 아님 / **reupload**: 내 것 & (다른 글에 붙음 또는 프로필 사진) / **drop**: 남의 것·없는 키 |

판정 규칙은 순수 함수 `pasteAction(...)`(`src/lib/attachments.ts`)이고 `scripts/test-post.ts`가 시험한다.

## 5. `reuploadAttachment(key, postId?)` — 다시 올리기 (새)

`src/app/write/actions.ts`. `reupload` 판정을 받은 키마다 하나씩 부른다. 에디터는 `올리는 중... (i/n)`을 보이고 [🖼 사진]·[📎 파일]·발행 버튼을 막는다 (FR-049).

| 항목 | 내용 |
|---|---|
| 권한 | `requireMember()` + 서버에서 다시 판정해 `reupload`일 때만 (내 것이 아니면 거부) |
| 처리 | 새 무작위 키(16바이트 → 32자 16진수) → `copyAttachment(원래 키, 새 키)`(`src/server/storage.ts`, 복사본 수정 시각 = 지금) → `attachments` INSERT (`user_id` = 나, 같은 `kind`·`name`·`mime`·`size`, `post_id` NULL). INSERT가 실패하면 복사본은 정리 작업의 "주인 없는 파일"이 지운다 |
| 성공 | `{ ok: { key, url: "/files/새키", kind, name, size } }` → 에디터가 원래 주소를 새 주소로 바꿔 넣는다 |
| 실패 | `{ error: "올리지 못했어요. 다시 시도해 주세요" }` → 그 사진·파일은 넣지 않고 `파일 이름: 올리지 못했어요. 다시 시도해 주세요` (FR-047, FR-050) |
| 원래 첨부 | 바뀌지 않는다 (원래 글에 그대로 붙어 있음, SC-013) |

## 6. 임시 글 (브라우저 저장 계약)

`src/lib/draft.ts`. 서버로 보내지 않는다. 모든 접근은 `try/catch` (저장을 쓸 수 없으면 조용히 건너뜀, FR-064).

| 항목 | 내용 |
|---|---|
| 키 | `blogville:draft:{userId}` (회원당 1개) |
| 값 | `{ title: string, categoryId: number \| null, subcategoryId: number \| null, visibility: "public" \| "private", contentHtml: string, tags: string, savedAt: string(ISO) }` |
| 쓰기 | 새 글 화면에서 마지막 변경 2초 뒤. 제목(앞뒤 공백 제외)과 본문 글자가 모두 비면 쓰지 않음 (FR-060). 발행 버튼을 누르면 남은 예약을 바로 쓰고 멈춤 |
| 묻기 | 새 글 화면을 열 때 값이 있으면 1번 `작성 중이던 글이 있어요. 불러올까요?` → 확인: 채움(지워진 카테고리는 비움) / 취소: 빈 화면 (FR-062) |
| 지우기 | ① 발행 성공 후 글 상세의 `PublishNotice` (주인일 때) ② 로그아웃 버튼 (`SignOutButton`에 `userId`가 있을 때, 로그아웃 요청을 보내기 전 브라우저에서. auth 단계 1 뒤에는 `<form action={signOut}>`의 `onSubmit`) (FR-063, 기본값, R14) |
| 안 하는 곳 | 수정 화면 (FR-064) |
