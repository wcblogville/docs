# Data Model: 캐릭터 / 성장 (GAME)

**Feature**: `005-game` | **Date**: 2026-10-07 | **Plan**: [plan.md](plan.md) | **Research**: [research.md](research.md)

이 문서는 GAME이 쓰는 엔터티를 **지금 DB(`src/db/schema.ts`, 마이그레이션 `0000`~`0006`)**와 **목표(spec + `docs/02-erd.md` v1.4)**로 비교한다.
표기: game 담당 테이블은 **변경**, 다른 spec 담당 테이블은 **참조**. 코드 위치는 코드 저장소 기준 경로다.

## 1. 테이블 한눈에

| 테이블 / 열거형 | 구분 | 내용 |
|---|---|---|
| `attendances` | 변경 | `streak` 삭제, `cycle_day`(FK → `attendance_rewards`)·`session_id`(FK → `sessions`, SET NULL)·`checked_at` 추가, 인덱스 (`session_id`). 기존 행 이전 `cycle_day = ((streak − 1) % 7) + 1` (ERD 3.12, 7장 4) |
| `attendance_rewards` | 변경 (새 테이블) | 1~7일차 보상 7행 (D10) |
| `point_ledger` | 변경 | 컬럼 변경 없음. 부분 고유 인덱스 `point_ledger_attendance_uq`(출석 보상 한 번), `point_ledger_like_received_uq`(같은 사람·같은 글 공감 보상 한 번) 추가 |
| `notifications` | 변경 (새 테이블) | 레벨업·공감·댓글·답글 알림 (FR-037~046). ERD에 없던 표 |
| `notification_kind` (열거형) | 변경 (새 열거형) | `level_up`, `like`, `comment`, `reply` |
| `ledger_reason` (열거형) | 변경 없음 | `attendance_streak`은 지난 기록용으로 남긴다 (FR-030) |
| `users` | 참조 | 알림 받는 회원·행동한 회원, 출석·원장의 회원 (FK CASCADE) |
| `sessions` | 참조 | `attendances.session_id`가 가리킨다. 구조 변경 요청 없음 (auth의 로그인 유지 작업은 `id` PK를 바꾸지 않는다) |
| `profiles` | 참조 | 알림 문구의 `{닉네임}`, 헤더 캐릭터. 가입 때 생성은 auth |
| `items` | 참조 | 가입 캐릭터 확인(`type`, `is_starter`), 레벨업 팝업 아이템(`required_level`, `type`). 판매 중 여부 `is_on_sale`은 shop이 추가한다 (`specs/006-shop/data-model.md`, SHOP-01) |
| `user_items` | 참조 | 가입 지급(auth 구현), 기존 회원 캐릭터 유지(FR-005, 데이터 변경 없음) |
| `posts` | 참조 | `notifications.post_id`가 가리킨다 (FK CASCADE), 알림 문구의 `{글 제목}` |
| `blogs` | 참조 | 알림 링크의 블로그 주소(`slug`) |
| `comments`, `replies` | 참조 | 2단계 댓글·답글 알림의 발생 조건은 social (단계 7) |
| `user_animals`, `animal_species` | 참조 | 동물 다 키움 보상(`farm_grown`)의 경험치 → 레벨업 감지 |

## 2. 엔터티

### 2.1 출석 기록 `attendances` — 변경

| 컬럼 | 지금 | 목표 | 규칙 |
|---|---|---|---|
| `user_id` | text PK·FK → `users` CASCADE | 그대로 | |
| `date` | date PK | 그대로 | 한국 날짜 `todayKST()` |
| `streak` | integer NOT NULL, CHECK ≥ 1 | **삭제** | |
| `cycle_day` | 없음 | integer NOT NULL, CHECK 1~7, FK → `attendance_rewards.day` | `nextCycleDay()`로 정함 (FR-022) |
| `session_id` | 없음 | text NULL, FK → `sessions.id` `ON DELETE SET NULL` | 출석이 일어난 세션 (FR-025) |
| `checked_at` | 없음 | timestamptz NOT NULL 기본 `now()` | 출석 시각 |

- 키: PK (`user_id`, `date`) 그대로 → 하루 한 번 (FR-023, NF-05). 식별 관계 (ERD 3.7).
- 인덱스: 새 `attendances_session_idx (session_id)` — 세션 삭제 때 SET NULL 대상 찾기 (research R4).
- 삭제: 회원 삭제 → CASCADE. 세션 삭제(로그아웃·만료) → `session_id`만 NULL, 출석은 남는다 (Edge Cases "로그인 상태가 끝남").
- Drizzle 이름: `attendances` 블록에 `cycleDay`, `sessionId`, `checkedAt`. `check("attendances_cycle_day_check", …)`, `foreignKey` 또는 `.references()`로 두 FK.

### 2.2 일차별 출석 보상표 `attendance_rewards` — 변경 (새 테이블)

| 컬럼 | 타입 | 규칙 |
|---|---|---|
| `day` | integer PK | CHECK `day BETWEEN 1 AND 7` |
| `exp` | integer NOT NULL | CHECK `exp >= 0` |
| `coins` | integer NOT NULL | CHECK `coins >= 0` |

- 추가 CHECK `exp > 0 OR coins > 0`: 원장 `point_ledger_nonzero_check`와 맞춘다 (research R5).
- 데이터(마이그레이션이 넣음, D10 / FR-020):

  | day | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
  |---|---|---|---|---|---|---|---|
  | exp | 10 | 10 | 15 | 15 | 20 | 20 | 30 |
  | coins | 10 | 20 | 30 | 40 | 50 | 70 | 100 |

- 회원을 가리키지 않는 카탈로그 표라 `npm run db:reset`(TRUNCATE users … CASCADE)에 지워지지 않는다.
- 타입: ERD는 `SMALLINT`로 적지만 ERD 부록 규칙대로 실제 DB는 `integer`를 쓴다 (`items.required_level`과 같음).

### 2.3 원장 `point_ledger` — 변경 (인덱스만)

| 항목 | 지금 | 목표 |
|---|---|---|
| 컬럼 | `id`, `user_id`, `reason`, `exp_delta`(≥ 0), `coin_delta`, `ref_id`, `created_at` | 그대로 |
| CHECK | `point_ledger_exp_check`, `point_ledger_nonzero_check` | 그대로 |
| 인덱스 | (`user_id`, `reason`, `created_at`), (`user_id`, `created_at` DESC) | + **`point_ledger_attendance_uq`** UNIQUE (`user_id`, `ref_id`) WHERE `reason = 'attendance'` + **`point_ledger_like_received_uq`** UNIQUE (`user_id`, `ref_id`) WHERE `reason = 'like_received'` (research R25) |

- 출석 보상 줄: `reason = 'attendance'`, `exp_delta`·`coin_delta` = 그날 일차 보상표 값, `ref_id` = 출석 날짜 `YYYY-MM-DD` (FR-025, ERD 3.7).
- 공감 보상 줄: `reason = 'like_received'`, `user_id` = 글 주인, `ref_id` = `"{글ID}:{공감한 회원ID}"` (지금 코드 그대로). 고유 인덱스가 쌍마다 한 줄을 DB로 지킨다 (FR-017, FR-047).
- **경험치가 생기는 기록**(출석, `grantReward`의 모든 사유, `farm_grown`)은 `addLedgerEntry()`(research R8)로만 넣는다. 이 함수가 같은 `tx`로 경험치 전후 레벨을 비교해 레벨업 알림을 같은 트랜잭션에 넣는다. 경험치 0인 `purchase`·`egg_purchase`는 지금처럼 직접 INSERT해도 된다 (레벨이 바뀌지 않는다).
- 보상 회수 없음: 글·댓글·답글 삭제, 공감 취소 때 원장 행을 지우거나 음수 행을 넣지 않는다 (FR-016, D6). `exp_delta ≥ 0` CHECK가 "누적 경험치는 줄지 않는다"를 DB로 지킨다.
- `id` 타입(ERD 부록 BIGINT vs 지금 integer)은 바꾸지 않는다 (plan-context 3.6).
- 사유별 규칙:

  | `reason` | 경험치 / 코인 | 하루 상한 | 넣는 곳 |
  |---|---|---|---|
  | `signup` | 0 / 100 | 1 (가입 때 한 번) | auth 가입 트랜잭션 → `grantReward` |
  | `attendance` | 보상표 | 1 (출석 PK + 부분 UK) | `ensureTodayAttendance` |
  | `attendance_streak` | (옛 기록만) | 새로 넣지 않음 | 없음 (FR-030) |
  | `post` | 30 / 30 | 3 | post `savePost` → `grantReward` |
  | `comment` | 5 / 5 | 10 (댓글 + 답글 합산) | social → `grantReward` |
  | `like_received` | 2 / 2 | 20 | social `toggleLike` → `grantReward` (`ref_id = 글ID:공감한회원ID`로 쌍마다 한 번, `point_ledger_like_received_uq`) |
  | `purchase` | 0 / −가격 | 없음 | shop `buyItem` |
  | `farm_care`, `farm_grown`, `egg_purchase` | town 규칙 | town 규칙 | town (`farm_grown`은 `addLedgerEntry`로 바뀜) |

### 2.4 알림 `notifications` — 변경 (새 테이블)

| 컬럼 | 타입 | 규칙 |
|---|---|---|
| `id` | integer identity PK | 엔터티 → 자기 번호 (ERD 3.7) |
| `user_id` | text NOT NULL, FK → `users.id` `ON DELETE CASCADE` | 받는 회원 |
| `kind` | `notification_kind` NOT NULL | `level_up` / `like` / `comment` / `reply` |
| `actor_id` | text NULL, FK → `users.id` `ON DELETE CASCADE` | 행동한 회원. 레벨업은 NULL |
| `post_id` | integer NULL, FK → `posts.id` `ON DELETE CASCADE` | 관련 글. 레벨업은 NULL |
| `level` | integer NULL | 레벨업일 때 오른 레벨 |
| `created_at` | timestamptz NOT NULL 기본 `now()` | |
| `read_at` | timestamptz NULL | 읽은(봤음) 시각. NULL = 안 읽음 |

**CHECK**

- `notifications_shape_check`: (`kind = 'level_up'` AND `level IS NOT NULL` AND `actor_id IS NULL` AND `post_id IS NULL`) OR (`kind <> 'level_up'` AND `level IS NULL` AND `actor_id IS NOT NULL` AND `post_id IS NOT NULL`)
- `notifications_level_check`: `level IS NULL OR level BETWEEN 2 AND 99` (레벨업은 Lv.2부터, 최고 99 — FR-009)
- `notifications_not_self_check`: `actor_id IS NULL OR actor_id <> user_id` (자기 활동 알림 금지, FR-045)

**인덱스**

| 이름 | 정의 | 쓰이는 곳 |
|---|---|---|
| `notifications_level_up_uq` | UNIQUE (`user_id`, `level`) WHERE `kind = 'level_up'` | 레벨업 알림 한 번 (FR-037, FR-047) |
| `notifications_user_created_idx` | (`user_id`, `created_at` DESC, `id` DESC) | 알림함 최신순 20개씩 (FR-043) |
| `notifications_unread_idx` | (`user_id`) WHERE `read_at IS NULL` | 헤더 안 읽은 수, 안 본 레벨업 (FR-042, FR-038) |
| `notifications_post_idx` | (`post_id`) WHERE `post_id IS NOT NULL` | 글 삭제 때 CASCADE 대상 찾기 |

**관계와 삭제**

| 지워지는 것 | 알림 |
|---|---|
| 받는 회원(탈퇴) | 그 회원이 받은 알림 모두 삭제 (FR-046) |
| 행동한 회원(탈퇴) | 그 회원이 남긴 공감·댓글·답글 알림 삭제 → 다른 회원 알림함에 닉네임이 남지 않음 (FR-046) |
| 글 | 그 글을 가리키는 알림 삭제 (spec에 없음, research R9) |
| 공감 취소, 댓글·답글 삭제 | 알림은 그대로 (Assumptions "공감 취소 시 알림을 지우지 않는다") |

**보여 줄 때 읽는 값** (저장하지 않음): 행동한 회원 닉네임 `profiles.nickname`(`actor_id`), 글 제목 `posts.title`, 블로그 주소 `blogs.slug`(`posts.blog_id`). 문구는 FR-045 표 ([contracts/notifications.md](contracts/notifications.md)).

**접근 규칙**: 모든 조회·변경은 `user_id = 지금 로그인한 회원` 조건을 붙인다 (FR-043 "본인의 알림만", US6-4).

### 2.5 열거형 `notification_kind` — 변경 (새 열거형)

| 값 | 단계 | 만드는 곳 |
|---|---|---|
| `level_up` | 1 | game `addLedgerEntry` (레벨이 오른 원장 기록과 같은 트랜잭션) |
| `like` | 2 | social `toggleLike` → `notifyActivity` (공감이 새로 저장될 때, 글 주인에게) |
| `comment` | 2 | social `addComment` → `notifyActivity` (남의 글 댓글, 글 주인에게) |
| `reply` | 2 | social 답글 등록 → `notifyActivity` (원댓글 작성자에게) |

네 값을 처음 마이그레이션에서 모두 만든다. 2단계에서 열거형을 바꾸지 않는다 (research R9).

### 2.6 참조 엔터티에서 읽는 값

| 엔터티 | 읽는 값 | 쓰는 곳 |
|---|---|---|
| `items` | `type`, `is_starter`, `required_level`, `price`, `name`, `asset_key`, (shop이 더할) `is_on_sale` | 가입 캐릭터 확인(auth 구현), 레벨업 팝업 아이템 (`is_on_sale = true AND type <> 'character'`, shop 마이그레이션 전에는 `type <> 'character' AND is_starter = false`) |
| `profiles` | `nickname`, `character_item_id` | 알림 문구, 헤더·광장 캐릭터 (FR-006) |
| `sessions` | `id` | 출석 세션 |
| `posts`·`blogs` | `title`, `blog_id` → `slug` | 알림 문구·링크 |

## 3. 상태 전이

### 3.1 하루 출석 (회원 1명, 한국 날짜 1일)

```text
[오늘 출석 없음] --(로그인한 채 그날 첫 화면 렌더: getViewer → ensureTodayAttendance)-->
    트랜잭션 성공 → [오늘 출석 있음 (cycle_day = N, 보상 원장 1줄)]
    트랜잭션 실패 → [오늘 출석 없음] (기록·보상 둘 다 없음, 다음 화면에서 다시 시도)
[오늘 출석 있음] --(같은 날 다른 화면·탭·기기)--> [오늘 출석 있음] (아무것도 바뀌지 않음)
[오늘 출석 있음] --(한국 시간 0시)--> 다음 날의 [오늘 출석 없음]
세션 삭제(로그아웃·만료) → session_id만 NULL, 상태는 그대로
```

### 3.2 출석 일차 (`nextCycleDay`, FR-022)

| 마지막 출석 (오늘 이전 최신 1건) | 오늘 `cycle_day` |
|---|---|
| 없음 | 1 |
| 어제, 1~6일차 | 어제 + 1 |
| 어제, 7일차 | 1 |
| 그저께 이전 | 1 |

예 (SC-005): 10/1~10/7 → 1~7, 10/8 → 1, 10/9 → 2, 10/10 없음, 10/11 → 1. 9/30 → 10/1, 12/31 → 1/1도 이어진다 (`previousDay()`가 달·해를 넘는다).

### 3.3 레벨 (원장 합계에서 계산, 저장하지 않음)

- 레벨 n 조건: 누적 경험치 ≥ `50 × n × (n − 1)` (`expForLevel`), 최고 99 (`MAX_LEVEL`). 485,100 이상이면 99에서 멈춘다 (FR-008, FR-009).
- 경험치는 줄지 않으므로 레벨은 오르기만 한다.
- 원장 기록 한 줄로 레벨이 L1 → L2(L2 > L1)가 되면 L1+1 … L2 각각 `level_up` 알림 한 줄 (99를 넘지 않음).

### 3.4 알림

```text
(생성) --> [안 읽음: read_at NULL]  → 헤더 🔔 숫자에 포함, 목록에서 노란 배경
[안 읽음] --(알림 누름: openNotification)--> [읽음]
[안 읽음] --([모두 읽음]: markAllNotificationsRead)--> [읽음]
[안 읽은 level_up] --(팝업 [확인]·[상점 가기]·Esc: dismissLevelUp(L))--> 레벨 ≤ L인 안 읽은 level_up 모두 [읽음]
[읽음] --> [읽음] (되돌리기 없음)
받는 회원·행동한 회원·관련 글 삭제 → 행 삭제
```

- 팝업은 안 읽은 `level_up`이 있을 때 가장 높은 레벨 하나만 보여 준다 (FR-041).
- 레벨업 알림을 알림함에서 먼저 읽으면 팝업도 다시 뜨지 않는다 (같은 `read_at`).

## 4. 검증 규칙 (모두 서버)

| 규칙 | 앱 코드 | DB |
|---|---|---|
| 하루 한 번 출석 (FR-023) | `lockUser` + `ON CONFLICT DO NOTHING` | PK (`user_id`, `date`) |
| 출석 보상 한 번 (SC-003) | 출석 INSERT가 성공한 트랜잭션에서만 원장 기록 | `point_ledger_attendance_uq` |
| 출석 기록과 보상 함께 (FR-024) | 한 트랜잭션 | FK `cycle_day` → 보상표 |
| 일차 1~7 | `nextCycleDay()` | CHECK + FK |
| 같은 사람·같은 글 공감 보상 한 번 (FR-017, FR-047) | social `toggleLike`: `lockUser(글 주인)` 안에서 같은 `ref_id` 원장 줄이 없을 때만 `grantReward` (지금 그대로) | `point_ledger_like_received_uq` (새로) |
| 활동 보상 하루 상한 (FR-016, FR-018) | `grantReward`가 잠금 안에서 오늘 같은 사유 수를 셈 | (상한 수는 CHECK로 표현 불가, 잠금으로 지킴 — 지금과 같음) |
| 경험치 줄지 않음 (FR-016) | 회수 코드 없음 | `exp_delta ≥ 0` |
| 코인 잔액 음수 금지 (FR-014) | 구매 때 잠금 안 잔액 확인 (shop) | (합계라 CHECK 불가, 잠금으로 지킴 — 지금과 같음) |
| 레벨업 알림 한 번 (FR-037) | `levelsGained` + `ON CONFLICT DO NOTHING` | `notifications_level_up_uq` |
| 자기 활동 알림 금지 (FR-045) | `notifyActivity`가 같은 회원이면 넣지 않음 | `notifications_not_self_check` |
| 종류별 알림 모양 | `notifyActivity` 입력 타입 | `notifications_shape_check` |
| 본인 알림만 (US6-4) | 모든 쿼리에 `user_id = 나` | - |
| 가입 캐릭터는 기본 캐릭터만 (FR-003) | auth 가입 트랜잭션: `type = 'character' AND is_starter = true` | `profiles` 장착 복합 FK(보유한 것만) |
| 알림 ID·레벨·페이지 입력 | zod + `parseId()`(1~2147483647), 레벨 2~99 정수 | - |

## 5. 마이그레이션 순서와 데이터 이전

마이그레이션 번호는 구현할 때 정한다 (plan-context 3.6). 먼저 merge된 브랜치가 있으면 최신 `main`에서 `npm run db:generate`를 다시 돌린다.

### 5.1 마이그레이션: 출석 일차·보상표 + 원장 고유 인덱스 (GAME-04, GAME-05)

`src/db/schema.ts` 수정 → `npm run db:generate` → 생성 SQL을 아래 순서로 손질 (research R6). `db:generate`가 `streak` → `cycle_day`를 이름 바꾸기인지 물으면 **새로 만들기**를 고른다 (research R6 주의). 맨 위 주석: `-- GAME-04 (2026-10-07): 자동 출석, 1~7일차 보상표. 기존 출석은 cycle_day = ((streak − 1) % 7) + 1. GAME-05·FR-047: 출석 보상·공감 보상 한 번을 원장 고유 인덱스로`.

1. `attendance_rewards` 생성 + CHECK, 7행 INSERT (위 2.2 값)
2. `attendances`에 `cycle_day`(NULL 허용), `session_id`, `checked_at`(기본 `now()`) 추가
3. 데이터 이전
   - `cycle_day = ((streak − 1) % 7) + 1`
   - `checked_at` = 같은 회원·`reason = 'attendance'`·`ref_id = date::text`인 원장 기록의 가장 이른 `created_at`, 없으면 `date`의 한국 0시
   - `session_id`는 NULL (옛 출석은 버튼으로 했다)
4. `cycle_day` NOT NULL, `attendances_cycle_day_check`, FK `cycle_day` → `attendance_rewards.day`, FK `session_id` → `sessions.id` `ON DELETE SET NULL`
5. `attendances_streak_check` 삭제, `streak` 삭제
6. 인덱스 `attendances_session_idx`, `point_ledger_attendance_uq`, `point_ledger_like_received_uq`

이전 전 확인: 원장의 출석 보상(`attendance`)과 공감 보상(`like_received`)에 (`user_id`, `ref_id`) 중복이 없어야 6이 성공한다 ([quickstart.md](quickstart.md) 1.3).

### 5.2 마이그레이션: 알림 (GAME-06, GAME-08)

`src/db/schema.ts`에 `notificationKind`, `notifications` 추가 → `npm run db:generate` (손질 없음). 맨 위 주석: `-- GAME-06·GAME-08 (2026-10-07): 레벨업·공감·댓글·답글 알림. 레벨업은 회원·레벨마다 한 번`.

- 데이터 이전 없음. 기존 회원의 지난 레벨업은 만들지 않는다 → 배포 직후 옛 레벨 팝업이 뜨지 않는다 (research R8).

### 5.3 순서와 선행

| 순서 | 마이그레이션 | 선행 |
|---|---|---|
| 1 | 5.1 출석 | 없음 (`sessions`, `point_ledger`는 이미 있다). 코드(자동 출석)는 auth 단계 1 뒤 |
| 2 | 5.2 알림 | 없음 (`users`, `posts`는 이미 있다). 5.1과 독립이지만 같은 단계 6에서 차례로 |

- 다른 spec의 마이그레이션(replies, subcategories 등)과 테이블이 겹치지 않는다. `posts` 구조를 바꾸는 post의 마이그레이션은 `notifications.post_id` FK에 영향이 없다 (PK `posts.id` 그대로).

### 5.4 함께 고칠 문서 (`docs/02-erd.md`, game 담당 부분)

| 위치 | 고칠 것 |
|---|---|
| 머리말 | 버전 1.5, 알림 표 추가 |
| 1장 관계도 | `users ||..o{ notifications : "받은 알림"`, `users |o..o{ notifications : "행동한 회원"`, `posts |o..o{ notifications : "관련 글"`, `notifications` 엔터티 블록. 출석·보상표 ⏳ 표시 제거 |
| 2장 그룹 | 보상 그룹에 `notifications` (GAME-06, GAME-08) |
| 3.3 세션 | 마지막 줄 "⏳ 자동 출석"의 ⏳ 제거. `sessions` 절은 auth 담당이므로 auth에 알리고 고친다 (한 줄) |
| 3.6 동시성 | 공감 보상 한 번이 원장 부분 고유 인덱스로도 막힌다는 한 줄 (R25) |
| 3.12 출석 | ⏳ 제거, `session_id` 인덱스, 원장 부분 고유 인덱스, 보상표 값과 "숫자 변경은 데이터 마이그레이션" |
| 새 3.x 알림 | 이 문서 2.4의 결정 (한 표 + 종류, CHECK, 레벨업 한 번, CASCADE, 문구는 JOIN) |
| 3.14 삭제 규칙 | 회원 → 받은·남긴 알림 삭제, 글 → 알림 삭제 |
| 3.15 열거형 | `notification_kind` |
| 3.16 인덱스 | `attendances (session_id)`, `point_ledger_attendance_uq`, `point_ledger_like_received_uq`, 알림 인덱스 4개 |
| 3.17 정규화 | 알림 문구·닉네임·제목을 저장하지 않음 (3NF) |
| 7장 | 4번 완료 표시, 7번 행의 "자동 출석" 완료 표시, 알림 행 추가 |
| 부록 NULL 허용 | `notifications.actor_id`, `post_id`, `level`, `read_at` |
