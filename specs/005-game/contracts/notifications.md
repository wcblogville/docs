# Contract: 레벨업 팝업과 알림함 (GAME-06, GAME-08)

**Feature**: `005-game` | 관련: FR-037~046, SC-009·012 | 설계 근거: [research.md](../research.md) R8~R15, R21 | 데이터: [data-model.md](../data-model.md) 2.4

## 1. 헤더 요소 (회원만, `src/components/site-header.tsx`에 끼움 — town 소유 파일에 추가)

| 요소 | 내용 | 근거 |
|---|---|---|
| 🔔 | `/notifications`로 가는 링크. 접근 이름 `알림` (spec에 없음 → plan 남은 문제 4). 안 읽은 알림이 있으면 옆에 빨간 숫자 배지: 1~9는 숫자, 10개 이상은 `9+`, 0이면 배지 없음 | FR-042, US6-1 |
| 레벨업 팝업 | 안 읽은 `level_up` 알림이 있으면 화면 가운데 팝업 (2장) | FR-038 |
| 날짜 감시 | `AttendanceDayWatcher` (화면 표시 없음, [attendance.md](attendance.md) 1장) | FR-021 |
| 방문자 | 🔔·팝업·날짜 감시 모두 없음 | FR-042, US6-6 |

데이터: `getHeaderNotifications(userId)` 한 쿼리 → `{ unread: 0~10, pendingLevel: number | null, pendingMinLevel: number | null }`.

## 2. 레벨업 팝업

| 항목 | 내용 |
|---|---|
| 컴포넌트 | 서버 `LevelUpPopup` → 클라이언트 `LevelUpDialog` (새 폴더 `src/components/game/`), 네이티브 `<dialog>` `showModal()` |
| 뜨는 때 | 안 읽은 `level_up`이 있는 회원의 화면을 서버가 그릴 때. 본인 활동 보상은 그 Server Action의 `revalidatePath("/", "layout")` 뒤 그 화면에서, 남의 공감은 다음에 여는 화면에서 |
| 보여 주는 레벨 | 안 읽은 `level_up` 중 가장 높은 레벨 L2 하나 (FR-041) |
| 모양 | 뒤 화면 어둡게, 맨 위 🎉, 큰 글씨 `Lv.{L2}이 되었어요!` (L2 = 99면 `최고 레벨 Lv.99가 되었어요!`) |
| 아이템 줄 | 안 읽은 레벨 범위(L1~L2)에 `required_level`이 있는 판매 중 아이템(캐릭터 제외: `is_on_sale = true AND type <> 'character'`, shop 마이그레이션 전에는 `type <> 'character' AND is_starter = false`)이 있으면 `이제 이런 친구를 데려올 수 있어요` + 그림·이름 최대 3개, 더 있으면 `외 N개`. 없으면 이 줄 없음 (FR-039, research R12) |
| 버튼 | [상점 가기] · [확인] — 둘 다 `dismissLevelUp` 폼 (4.3). 높이 44px 이상 |
| Esc | [확인]과 같다 (research R11, spec에 없음 → open item) |
| 다시 뜨지 않음 | 버튼을 누르면 L2 이하 안 읽은 레벨업이 모두 읽음 → 새로고침해도 뜨지 않음. 알림함에는 남는다 (FR-040, SC-009) |

## 3. 화면 `/notifications` — 알림함

| 항목 | 내용 |
|---|---|
| 파일 | 새로: `src/app/notifications/page.tsx` (Server Component), `actions.ts` |
| 접근 | 회원 `requireMember()`. 방문자는 `/`로 redirect |
| 주소 인자 | `?page=N` — `parsePage()`로 1~2147483647, 아니면 1페이지 (500 없음, #21) |
| 제목 | `🔔 알림함` (메타 제목 `알림함`) — spec에 화면 제목 문구가 없어 용어를 그대로 씀 (open item) |
| 헤더 | `← 광장으로 나가기` |
| 목록 | 본인(`user_id = 나`) 알림만 `created_at` 최신순(같으면 `id` 큰 순) 20개. 안 읽은 줄은 노란 배경. 줄마다 문구(5장)와 시간 표시. 줄 전체가 `openNotification` 버튼 (FR-043, FR-044) |
| 시간 | 1분 미만 `방금`, 60분 미만 `N분 전`, 24시간 미만 `N시간 전`, 그 밖 `YYYY.MM.DD`(한국 시간) (FR-043) |
| [모두 읽음] | 목록 위. 안 읽은 알림이 있을 때 보인다 → `markAllNotificationsRead` (FR-044) |
| 빈 목록 | `아직 알림이 없어요` (FR-043, US6-5) |
| 페이지 번호 | `Pagination` 그대로 (내역 화면과 같은 규칙) |

## 4. Server Actions (새 파일 `src/app/notifications/actions.ts`)

모든 Action은 첫 줄에서 `requireMember()`를 부른다(트랜잭션 밖). Next.js Server Action이므로 다른 사이트에서 보낸 요청은 Origin 확인으로 거부된다. 성공하면 `revalidatePath("/", "layout")`로 헤더 🔔·팝업을 갱신한다.

### 4.1 `openNotification(formData)`

| 항목 | 내용 |
|---|---|
| 입력 | `FormData { id }` — zod: 정수 1~2147483647 (`parseId` 규칙) |
| 권한 | `notifications.id = id AND user_id = 나`인 행만 |
| 처리 | `read_at = COALESCE(read_at, now())`, 그 행의 종류·글을 읽어 이동할 곳 계산 |
| 성공 | redirect: `level_up` → `/shop`, `like` → `/@{주소}/{글ID}`, `comment`·`reply` → `/@{주소}/{글ID}#comments` (FR-044, FR-045). 해시 위치로 스크롤되는지는 구현 전에 확인 (research R24) |
| 실패 | 형식 오류·없는 ID·남의 알림 → 아무것도 바꾸지 않고 `/notifications`로 redirect. 문구 없음, 500 없음 |

### 4.2 `markAllNotificationsRead()`

| 항목 | 내용 |
|---|---|
| 입력 | 없음 |
| 권한 | `user_id = 나`인 안 읽은 알림만 |
| 처리 | `read_at = now()` |
| 성공 | 같은 화면, 노란 배경과 🔔 숫자 사라짐 (US6-2) |

### 4.3 `dismissLevelUp(formData)`

| 항목 | 내용 |
|---|---|
| 입력 | `FormData { level, go }` — `level`: 정수 2~99, `go`: `shop` 또는 `stay` |
| 권한 | `user_id = 나`, `kind = 'level_up'`, `read_at IS NULL`, `level <= 입력 level`인 행만 |
| 처리 | `read_at = now()` (보여 준 레벨과 그보다 낮은 안 본 레벨업 모두, FR-041) |
| 성공 | `go = shop` → `/shop`으로 redirect, `stay` → 같은 화면(팝업 닫힘) |
| 실패 | 형식 오류 → 아무것도 바꾸지 않음. 문구 없음 |

## 5. 알림 문구 (FR-045, 그대로)

| `kind` | 문구 | 누르면 | 팝업 | 단계 |
|---|---|---|---|---|
| `level_up` | `🎉 Lv.{level}이 되었어요!` | `/shop` | 있음 (2장) | 1 |
| `like` | `❤️ {행동한 회원 닉네임}님이 「{글 제목}」에 공감했어요` | 그 글 | 없음 | 2 |
| `comment` | `💬 {행동한 회원 닉네임}님이 「{글 제목}」에 댓글을 달았어요` | 그 글의 댓글 (`#comments`) | 없음 | 2 |
| `reply` | `💬 {행동한 회원 닉네임}님이 내 댓글에 답글을 달았어요` | 그 글의 댓글 (`#comments`) | 없음 | 2 |

- 닉네임·제목·주소는 보여 줄 때 `profiles`·`posts`·`blogs`에서 읽는다 (바뀐 이름이 바로 반영).
- `#comments` 위치는 social이 댓글 영역(`src/components/blog/comment-section.tsx`)에 `id="comments"`를 두어 맞춘다 (의존성).

## 6. 서버 함수 계약 (다른 spec이 부름, 새 파일 `src/server/notifications.ts`)

### 6.1 `notifyActivity(tx, { recipientId, actorId, kind, postId })` — social 단계 7

| 항목 | 내용 |
|---|---|
| 부르는 곳 | social의 공감·댓글·답글 트랜잭션 안 (공감·댓글이 롤백되면 알림도 없음) |
| `kind` | `like` / `comment` / `reply` (`level_up`은 받지 않는다) |
| 받는 회원 | `like`·`comment`: 글 주인, `reply`: 원댓글 작성자 (FR-045) |
| 자기 활동 | `recipientId === actorId`면 아무것도 넣지 않는다 (FR-045, US6-10). DB CHECK로도 막힌다 |
| 결과 | 없음 (넣었는지 여부만 돌려줄 수 있다) |

### 6.2 레벨업 알림 — game 내부

`addLedgerEntry()`(`src/server/points.ts`)만 만든다. 다른 spec은 원장 기록을 `grantReward()`나 `addLedgerEntry()`로 하면 레벨업 알림이 자동으로 생긴다 ([rewards-ledger.md](rewards-ledger.md)).

## 7. 오류·조작

| 상황 | 결과 |
|---|---|
| 다른 회원 알림 ID로 `openNotification` 요청 | 그 알림 `read_at` 그대로, `/notifications`로 redirect |
| 범위 밖 ID(`2147483648`, `1e3`, 문자열) | 아무것도 바꾸지 않음, 500 없음 |
| `dismissLevelUp`에 `level = 100`·`0`·문자 | 아무것도 바꾸지 않음 |
| 방문자가 Action 호출 | `requireMember()`가 `/`로 redirect, 변화 없음 |
| `/notifications?page=99999999999999999999` | 1페이지, HTTP 200 |
