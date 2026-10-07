# Contract: 댓글·답글 (SOC-01, SOC-02)

**Feature**: `004-social` | 관련: [data-model.md](../data-model.md) 2.1·2.2·3·4, [research.md](../research.md) R1~R8·R12·R13

이 기능은 Route Handler를 만들지 않는다. 바깥 인터페이스는 Server Action 4개, 글 상세 화면의 댓글 영역, 다른 spec이 부르는 서버 함수 3개다.

## 1. 화면 라우트

| 주소 | 실제 페이지 | 접근 | 이 영역이 보여 주는 것 |
|---|---|---|---|
| `/@{slug}/{postId}` | `src/app/blog/[slug]/[postId]/page.tsx` (post 소유, 댓글·공감 영역은 social) | 누구나. 비공개 글은 주인만(아니면 404, post 규칙) | `💬 댓글 N`, 댓글 목록(오래된 순), 각 댓글 아래 답글(오래된 순), 회원에게 입력칸, 방문자에게 로그인 안내 |

- 댓글 영역 `<section id="comments" aria-label="댓글">`. 댓글 줄 `id="comment-{id}"`, 답글 줄 `id="reply-{id}"`. game 알림을 누르면 `/@{slug}/{postId}#comments`로 온다 ([notification-triggers.md](notification-triggers.md)).
- 리다이렉트: 이 영역 자체에는 없다. 쓰기 요청 때 로그인이 풀려 있으면 Server Action이 `/`로 보낸다 (FR-002).

## 2. Server Action

모두 `src/app/blog/actions.ts`(`"use server"`). 공통 규칙:

- 첫 줄에서 `requireMember()`(`src/server/dal.ts`). 로그인이 없으면 아무것도 저장하지 않고 `/`로 이동 (FR-002).
- 숫자 ID는 `parseId()`. 내용 규칙은 새 파일 `src/lib/social.ts`.
- 저장과 보상은 `db.transaction` 하나. 성공하면 `revalidatePath("/", "layout")`.
- 예상하지 못한 DB 오류만 예외로 던진다(→ `src/app/error.tsx`). 아래 표의 실패는 모두 문구 또는 무반응으로 끝나고 서버 오류 화면이 나오지 않는다 (FR-003, SC-007).

### 2.1 `addComment(prev: CommentState, formData: FormData): Promise<CommentState>`

`useActionState`로 쓴다.

**입력 (FormData)**

| 이름 | 형식 | 규칙 |
|---|---|---|
| `postId` | 문자열 | `parseId` 통과 (1~2147483647, 숫자만) |
| `content` | 문자열 | 줄바꿈 정규화 + 앞뒤 공백 제거 뒤 1~1000자 |

**결과 타입**: `type CommentState = { ok?: number; error?: string; content?: string }`

| 판정 순서 | 조건 | 결과 | 사용자 문구 (버튼 왼쪽, 작은 빨간 글씨) | 저장 |
|---|---|---|---|---|
| 1 | 로그인 없음 | `/`로 이동 | - | 안 함 |
| 2 | `postId`가 범위 밖·형식 오류 (`99999999999`, `abc`, `0`, `2.5`, `1e3`) | `{ error, content }` | `잘못된 요청이에요` | 안 함 |
| 3 | 내용이 비었거나 공백만 | `{ error, content }` | `댓글을 적어 주세요` | 안 함 |
| 4 | 내용이 1000자 초과 | `{ error, content }` | `댓글은 1000자까지예요` | 안 함 |
| 5 | 글이 없음·지워짐·남의 비공개 글 | `{ error, content }` | `글을 찾을 수 없어요` | 안 함 |
| - | 성공 | `{ ok: Date.now() }` | (없음) | 댓글 1행 + (남의 글이면) 보상 |

- 권한: 회원. 볼 수 있는 글(공개 글 또는 내 글)에만 (FR-006).
- 보상: 글 주인 ≠ 나이면 같은 트랜잭션에서 `lockUser(나)` → `grantReward(tx, 나, "comment", 댓글ID)`. 하루 10번(답글과 합침) 넘으면 저장만 하고 안내 없음 (FR-010, SC-004). 보상 기록이 실패하면 댓글도 저장되지 않는다.
- `content`(실패 때): 정규화 전 받은 그대로 돌려주어 입력칸에 남긴다 (research R7).
- 2차: 글 주인 ≠ 나이면 댓글 알림 기록 (FR-055).

### 2.2 `addReply(prev: CommentState, formData: FormData): Promise<CommentState>` (새)

**입력 (FormData)**

| 이름 | 형식 | 규칙 |
|---|---|---|
| `postId` | 문자열 | `parseId` |
| `commentId` | 문자열 | `parseId` (원댓글 ID) |
| `content` | 문자열 | 댓글과 같음 |

| 판정 순서 | 조건 | 사용자 문구 (답글 입력칸 버튼 왼쪽) | 저장 |
|---|---|---|---|
| 1 | 로그인 없음 | `/`로 이동 | 안 함 |
| 2 | `postId` 또는 `commentId`가 범위 밖·형식 오류 | `잘못된 요청이에요` | 안 함 |
| 3 | 내용 빔 / 1000자 초과 | `댓글을 적어 주세요` / `댓글은 1000자까지예요` | 안 함 |
| 4 | 글이 없음·지워짐·남의 비공개 글 | `글을 찾을 수 없어요` | 안 함 |
| 5 | 원댓글이 없음, 또는 다른 글의 댓글 | `답글을 달 댓글이 없어요` | 안 함 (일반 댓글로도 저장하지 않음) |
| 6 | 원댓글이 삭제 표시(작성자·주인·관리자 삭제, 탈퇴 자리 포함) | `삭제된 댓글에는 답글을 달 수 없어요` | 안 함 |
| - | 성공 | `{ ok }` | 답글 1행 + (남의 글이면) 보상 |

- 모든 실패에서 `content`를 돌려주어 입력칸에 그대로 남긴다 (FR-018, SC-010).
- 원댓글 확인은 트랜잭션 안에서 원댓글 행을 `FOR SHARE`로 잠근 뒤 한다 (research R5). 그래서 삭제와 동시에 온 답글도 6으로 거부되거나 삭제보다 먼저 저장된다.
- 보상: 글 주인 ≠ 나이면 같은 트랜잭션에서 `lockUser(나)` → `grantReward(tx, 나, "comment", "reply:{답글ID}")`. 사유가 댓글과 같은 `comment`라 하루 10번은 댓글과 합쳐 센다. 내 댓글에 단 답글도 남의 글이면 보상 대상, 내 글에 단 답글은 보상 없음 (FR-023). 보상 기록이 실패하면 답글도 저장되지 않는다.
- 2차: 원댓글 작성자 ≠ 나이면 답글 알림 (받는 회원 = 원댓글 작성자) (FR-056).

### 2.3 `deleteComment(commentId: unknown): Promise<void>`

| 조건 | 결과 |
|---|---|
| 로그인 없음 | `/`로 이동 |
| `commentId`가 `parseId` 실패 | 아무것도 바뀌지 않음, 문구 없음 |
| 삭제 안 된 댓글이고, 나 = 작성자 또는 그 글의 블로그 주인 또는 관리자 | `deleted_at = now()`, `content = ''`. 답글·보상은 그대로 (FR-015) |
| 그 밖(남의 댓글에 권한 없음, 이미 삭제됨, 없는 ID) | 아무것도 바뀌지 않음, 문구 없음 (FR-013, SC-005) |

- 판단은 `UPDATE ... WHERE` 한 문장 (research R8). 화면의 확인 창 `댓글을 삭제할까요?`는 화면에서만 띄운다.
- 관리자 = `viewer.user.role === "admin"`.

### 2.4 `deleteReply(replyId: unknown): Promise<void>` (새)

`deleteComment`와 같다. 블로그 주인 판단은 `replies → comments → posts → blogs`. 확인 창 문구도 `댓글을 삭제할까요?` (FR-024).

## 3. 댓글 영역 데이터 (서버 → 화면)

새 파일 `src/server/social.ts`의 `getCommentThread(postId: number, viewer: { userId: string; isAdmin: boolean } | null)`가 만들고, 글 상세 페이지가 `CommentSection`(`src/components/blog/comment-section.tsx`, 클라이언트 컴포넌트)에 넘긴다.

```ts
type CommentAuthor = { nickname: string; characterAsset: string; blogSlug: string };

type ReplyView = {
  id: number;
  createdAtText: string;       // formatDateTime, 한국 시간 24시간제 (FR-011)
  deleted: boolean;
  content: string;             // 삭제면 "" (DB에도 "")
  author: CommentAuthor;       // 답글은 탈퇴하면 행이 없어지므로 늘 있다
  canDelete: boolean;          // 삭제 안 된 줄이고 canDeleteComment(작성자·블로그 주인·관리자)가 참일 때만
};

type CommentView = {
  id: number;
  createdAtText: string;
  deleted: boolean;
  content: string;
  author: CommentAuthor | null; // null = 탈퇴 자리
  canReply: boolean;            // 회원이고 삭제 안 된 댓글
  canDelete: boolean;           // 삭제 안 된 줄이고 canDeleteComment가 참일 때만
  replies: ReplyView[];         // 오래된 순 (created_at, id)
};

// comments는 오래된 순 (created_at, id) — FR-011 "같으면 먼저 등록된 순"
type CommentThread = { count: number; comments: CommentView[] }; // count = 삭제 안 된 댓글 + 삭제 안 된 답글
```

`CommentSection` props: `{ postId: number; thread: CommentThread; isMember: boolean }`

**표시 규칙**

| 상황 | 보이는 것 | FR |
|---|---|---|
| 제목 | `💬 댓글 {count}` (0이어도 보임, 빈 목록 문구 없음) | FR-012, FR-016 |
| 보이는 댓글·답글 | 캐릭터 · 닉네임(그 사람 블로그 홈 링크) · 작성 시각 / 내용(줄바꿈 유지, HTML·주소 글자 그대로) / 버튼 | FR-007, FR-011 |
| 삭제 자리 | 캐릭터 · 닉네임 · 작성 시각 / `삭제된 댓글이에요`(흐린 기울임꼴) / 버튼 없음 | FR-014, FR-024 |
| 탈퇴 자리 (`author: null`) | `삭제된 댓글이에요`만 (캐릭터·닉네임·시각 없음) | auth FR-052 |
| [답글] | `canReply`인 댓글에만. 누르면 버튼 줄 아래, 기존 답글보다 위에 2줄 입력칸 `답글을 남겨 주세요`, 버튼은 [답글 취소]. 원댓글마다 따로 열리고 자동 커서 없음 | FR-019, FR-020 |
| [답글 취소] | 입력칸 닫힘, 쓰던 내용 사라짐 | FR-020 |
| [답글 등록] / [댓글 등록] | 처리 중 `등록 중...` + 비활성. 성공하면 입력칸 비움(답글은 닫히고 버튼이 [답글]로) | FR-009, FR-020 |
| [삭제] | `canDelete`인 줄에만. 확인 창 `댓글을 삭제할까요?` → [취소]면 아무 일 없음 | FR-013, FR-024 |
| 답글 줄 | 왼쪽 세로선 들여쓰기, [답글] 버튼 없음 | FR-022 |
| 방문자 | 입력칸 대신 `로그인하면 댓글을 남길 수 있어요`(`로그인`은 `/` 링크), [답글]·[삭제] 없음 | FR-012 |
| 입력칸 | 댓글 3줄 `따뜻한 댓글을 남겨 주세요 💬`, `maxLength={1000}` | FR-006 |
| 누르는 영역 | [답글]·[답글 취소]·[삭제]·[댓글 등록]·[답글 등록] 버튼과 닉네임·`로그인` 링크 최소 44×44px, 버튼 글자 한 줄, 키보드 선택 표시 | FR-004, FR-005, constitution VI |

화면으로 넘기는 데이터에 작성자 회원 ID와 삭제된 내용이 없다 (SC-006, research R13).

## 4. 다른 spec이 부르는 서버 함수 (새 파일 `src/server/social.ts`)

| 함수 | 부르는 곳 | 계약 |
|---|---|---|
| `liveCommentCountSql(postId: AnyPgColumn \| SQL): SQL<number>` | post: `src/server/blog.ts`의 `listColumns.commentCount` | 그 글의 삭제 안 된 댓글 수 + 삭제 안 된 답글 수 (FR-016, FR-032 카드 `💬 N`) |
| `countLiveComments(): Promise<number>` | auth: `src/app/admin/page.tsx` 통계 카드 `댓글` (요청 D-4) | 마을 전체 삭제 안 된 댓글 + 답글 수 |
| `prepareCommentsForWithdrawal(tx: Tx, userId: string): Promise<void>` | auth: 탈퇴 트랜잭션, 회원 행을 지우기 **전** (D-3) | (1) 그 회원 댓글 행 `FOR UPDATE` (2) 다른 회원 답글(삭제 표시된 답글 포함)이 하나도 없는 그 회원 댓글 삭제(아래 그 회원 답글 `CASCADE`) (3) 남은 그 회원 댓글을 `deleted_at = COALESCE(deleted_at, now())`, `content = ''`. 이후 auth가 회원을 지우면 `replies`는 `CASCADE`, 남은 댓글 `author_id`는 `SET NULL`. 부르지 않으면 회원 삭제가 `comments_author_check`로 실패한다 |

`Tx`는 `src/server/points.ts`의 트랜잭션 타입이다.
