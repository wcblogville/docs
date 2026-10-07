# Contract: 알림 발생 조건 (2차, D11)

**Feature**: `004-social` | 관련: [research.md](../research.md) R15, game spec `specs/005-game/spec.md` FR-042~046

social은 알림이 생기는 **조건**만 갖는다. 알림 표, 알림함(🔔), 문구(`❤️ {닉네임}님이 「{글 제목}」에 공감했어요` 등), 읽음 처리는 game(GAME-08)이 만든다. 이 문서는 social의 Server Action이 game의 기록 도우미를 언제, 무엇을 넘겨 부르는지 정한다. 도우미 이름과 정확한 타입은 game 소유이며, 지금 game plan은 `notifyActivity(tx, { recipientId, actorId, kind, postId })`(`src/server/notifications.ts`, `kind` = `like` / `comment` / `reply`)로 정했다(`specs/005-game/contracts/notifications.md` 6.1). 아래는 social이 넘기는 값이다.

## 1. 선행 조건

- game 단계 6(알림 표 + 레벨업 알림)이 merge되어 있어야 한다 (plan 의존성 D-2).
- 도우미는 `Tx`(`src/server/points.ts`)를 받아 같은 트랜잭션 안에서 기록한다.

## 2. social이 넘기는 값

| 값 (`notifyActivity` 인자) | 공감 | 댓글 | 답글 |
|---|---|---|---|
| 받는 회원 (`recipientId`) | 글 주인 (`blogs.owner_id`) | 글 주인 | 원댓글 작성자 (`comments.author_id`) |
| 종류 (`kind`) | `like` | `comment` | `reply` |
| 행동한 회원 (`actorId`) | 나 | 나 | 나 |
| 관련 글 (`postId`) | 글 ID | 글 ID | 원댓글의 글 ID |

- game의 도우미도 `recipientId === actorId`면 기록하지 않고 DB CHECK(`notifications_not_self_check`)로도 막는다. social은 아래 3절의 조건으로 먼저 거른다.

## 3. 부르는 조건

| Server Action | 부르는 때 | 부르지 않는 때 |
|---|---|---|
| `toggleLike` | 공감 행이 **새로 들어갔고** 글 주인 ≠ 나 (보상 여부·하루 상한과 상관없음) | 취소, 동시 요청으로 새 행 없음, 내 글 (FR-033) |
| `addComment` | 저장 성공 AND 글 주인 ≠ 나 | 내 글에 쓴 댓글, 거부된 요청 (FR-055) |
| `addReply` | 저장 성공 AND 원댓글 작성자 ≠ 나 | 내 댓글에 단 답글, 거부된 요청 (FR-056). 내 댓글이 남의 글에 있어도 글 주인에게는 남기지 않는다 |

- 알림 기록이 실패하면 공감·댓글·답글 저장과 보상도 함께 취소된다 (같은 트랜잭션).
- 공감 → 취소 → 공감을 반복하면 공감 알림도 반복된다 (FR-033 "새로 저장되면"; game Assumptions "공감 취소 때 알림을 지우지 않는다", "묶어 보여 주지 않는다").

## 4. 알림을 눌렀을 때 갈 곳 (social이 제공하는 앵커)

| 종류 | 주소 |
|---|---|
| 공감 | `/@{slug}/{postId}` |
| 댓글·답글 | `/@{slug}/{postId}#comments` (`<section id="comments">`, 줄마다 `id="comment-{id}"` / `id="reply-{id}"`) |

## 5. 삭제와의 관계

| 일 | 알림 |
|---|---|
| 글 삭제 | game의 `notifications.post_id` FK `CASCADE`로 함께 삭제 (game data-model 2.4에 있음) |
| 댓글·답글 삭제 표시 | 알림은 그대로 둔다 (spec에 규칙 없음, 알림은 글 단위) |
| 행동한 회원·받는 회원 탈퇴 | game FR-046 (FK `CASCADE`) |
