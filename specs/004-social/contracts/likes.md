# Contract: 공감 (SOC-03)

**Feature**: `004-social` | 관련: [data-model.md](../data-model.md) 2.3, [research.md](../research.md) R5·R9·R14·R16

## 1. 화면

| 주소 | 위치 | 접근 | 보이는 것 |
|---|---|---|---|
| `/@{slug}/{postId}` | 본문·태그 아래, 댓글 위 가운데 (`src/components/blog/like-button.tsx`) | 누구나 (비공개 글은 주인만, post 규칙) | 회원: `♡ 공감 N`(안 누름) / `♥ 공감 N`(누름, 분홍 테두리·글자), `aria-pressed`로 눌림 상태 전달. 방문자: 비활성(흐리게) 버튼 + 버튼 아래 `로그인하면 공감할 수 있어요` |
| 글 카드 (마을 소식·이웃 새 글·태그별·블로그 홈) | `src/components/blog/post-card.tsx` (post 소유) | 누구나 | `♥ N` (0도 보임). 글 상세의 N과 같은 수 (FR-032) |

N = 그 글의 `post_likes` 행 수(글 주인 자신의 공감 포함). 누가 공감했는지는 보여 주지 않는다 (FR-033).

**`LikeButton` props**: `{ postId: number; count: number; liked: boolean; canLike: boolean }` (지금과 같음). `canLike = false`이면 방문자 안내 글자를 버튼 아래에 그리고 `aria-describedby`로 잇는다.

**화면 동작** (FR-028)

1. 누르면 바로 하트와 수를 바꿔 보여 준다 (`useOptimistic`). 처리 중 버튼 비활성.
2. 서버가 저장하면 `revalidatePath`로 다시 그려진 실제 수·상태로 바뀐다.
3. 서버가 아무것도 하지 않으면(글 삭제·비공개·조작) 문구 없이 누르기 전 값으로 돌아간다 (FR-031).

## 2. Server Action: `toggleLike(postId: unknown): Promise<void>`

`src/app/blog/actions.ts`. 공감인지 취소인지는 서버가 그 순간 내 공감 유무로 정한다 (FR-025).

| 판정 순서 | 조건 | 결과 | 문구 |
|---|---|---|---|
| 1 | 로그인 없음 | `/`로 이동 | - |
| 2 | `postId`가 `parseId` 실패 (`abc`, `99999999999`) | 아무것도 안 함 | 없음 |
| 3 | 글이 없음·지워짐·남의 비공개 글 (트랜잭션 안 `FOR KEY SHARE`로 확인) | 아무것도 안 함 | 없음 |
| 4 | 내 공감이 있음 | 내 공감 행 삭제 (보상 회수 없음) | 없음 |
| 5 | 내 공감이 없음 | `INSERT ... ON CONFLICT DO NOTHING` | 없음 |
| 5-a | 다른 요청이 먼저 넣어 새 행이 없음 | 끝 (공감 1개 유지, 오류 화면 없음) | 없음 |
| 5-b | 새 행이 들어갔고 글 주인 ≠ 나 | `lockUser(글 주인)` → 원장에 `like_received` + `ref_id = "{글ID}:{나}"`가 없을 때만 `grantReward(tx, 글 주인, "like_received", ref_id)` (하루 20번 상한은 `grantReward`가 확인) | 없음 |
| 5-c | 글 주인 = 나 | 공감만 저장, 보상 없음 | 없음 |

- 2·3은 `revalidatePath`를 부르지 않는다. 4·5는 끝에 `revalidatePath("/", "layout")`.
- 공감 저장·취소와 보상은 한 트랜잭션이다 (FR-029).
- 보장: 한 회원·한 글 공감 최대 1개(PK, SC-003), 같은 사람·같은 글로 글 주인이 받는 보상 최대 1번(SC-004).
- 2차: 5-b(새 행이 들어갔고 글 주인 ≠ 나)이면 보상을 받았는지와 상관없이 공감 알림을 같은 트랜잭션에서 남긴다 (FR-033). 5-a·5-c·취소(4)는 알림이 없고, 취소해도 이미 남은 알림은 지우지 않는다.

## 3. 저장 단계 규칙 (DB에 직접 넣어도 거부)

| 위반 | DB 응답 |
|---|---|
| 같은 (`post_id`, `user_id`) 두 번째 행 | PK 위반 (SQLSTATE 23505) |
| 없는 글·회원 | FK 위반 (23503) |
