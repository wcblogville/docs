---

description: "캐릭터 / 성장 (GAME) 구현 작업 목록"
---

# Tasks: 캐릭터 / 성장 (GAME)

**Input**: Design documents from `/specs/005-game/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: plan.md Technical Context와 quickstart.md 2·3장이 단위 시험(`scripts/test-game.ts` 고침, `scripts/test-notifications.ts` 새로)과 E2E(`e2e/attendance.mjs`, `e2e/rewards.mjs`, `e2e/notifications.mjs` 새로)를 산출물로 정했으므로 시험 작업을 포함한다. 시험은 구현 전에 먼저 써서 실패를 확인한다.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- 모든 파일 경로는 **코드 저장소 `ehgo508/blogville`** 루트 기준이다 (이 문서 저장소가 아니다).
- 한 Next.js 프로젝트: 화면·Action `src/app/`, 서버 전용 쿼리 `src/server/`, 순수 규칙 `src/lib/`, 화면 조각 `src/components/`, 스키마 `src/db/schema.ts`, 마이그레이션 `drizzle/`, 단위 시험 `scripts/`, E2E `e2e/`.
- `(소유: X)`가 붙은 파일은 다른 spec 소유다. plan.md "의존성"대로 **추가**만 하거나 그 spec에 **요청**한다.
- 화면 문구는 spec·contracts 표 그대로 쓴다 (FR-048). spec에 없는 문구는 plan.md "남은 문제" 4의 임시값을 쓰고 팀 결정을 기다린다.

## 선행 조건 (이 작업 목록 밖)

- **auth 단계 1** merge: 온보딩 제거, 가입 트랜잭션(고른 캐릭터 1종 + 초원 + "일상" + `grantReward("signup")`), 가입 폼 캐릭터 카드 2개, `e2e/helpers.mjs`의 `loginDev` 수정. 이것이 없으면 US1 확인과 모든 E2E 로그인이 동작하지 않는다.
- auth와 `src/server/dal.ts` `getViewer()`에 출석 필드·호출을 더하는 것에 합의 (research R1).

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 라이브러리 동작 확인과 시험 뼈대

- [x] T001 `npm install` 뒤 research R24 표의 동작을 설치된 패키지 문서·소스로 확인하고 결과를 PR 설명 초안에 적는다: Better Auth `auth.api.getSession()`의 `session.id`, Next.js 16.3 렌더 중 DB 쓰기·`router.refresh()`의 루트 레이아웃 재렌더·`<Link>` prefetch·`revalidatePath("/", "layout")`·Server Action `redirect()`의 `#comments` 해시 스크롤·React `cache` 범위, drizzle-orm `pgEnum`·`uniqueIndex().on().where()`·대상 없는 `.onConflictDoNothing()`, drizzle-kit `migrate` 트랜잭션 여부와 `streak` → `cycle_day` 이름 바꾸기 질문 (`node_modules/better-auth`, `node_modules/next`, `node_modules/drizzle-orm`, `node_modules/drizzle-kit`)
- [x] T002 [P] 새 단위 시험 파일 뼈대 `scripts/test-notifications.ts`를 만들고(`scripts/test-game.ts`와 같은 `check()` 관례, 실패 시 `exit(1)`), `package.json`에 `"test:notifications": "tsx scripts/test-notifications.ts"`를 더해 `test` 체인 끝(`test:game → test:ids → test:sanitize → test:notifications`)에 붙인다
- [x] T003 [P] 새 E2E 파일 뼈대 3개를 만든다: `e2e/attendance.mjs`, `e2e/rewards.mjs`, `e2e/notifications.mjs` — 첫 인자 = 스크린샷 폴더, `.env.local`에서 비밀값 읽기, 실행마다 새 회원, `pg`로 날짜·경험치 준비(`e2e/decisions.mjs`의 `kstDate()` 방식), `❌`가 있으면 `exit(1)` (research R23)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 출석·보상·레벨업이 모두 기대는 스키마(M1)와 원장 기록 함수(`addLedgerEntry`)

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [x] T004 마이그레이션 적용 전 확인: 개발 DB `point_ledger`에서 `reason = 'attendance'`, `reason = 'like_received'` 각각 (`user_id`, `ref_id`)로 묶어 2줄 이상인 묶음이 0개인지 확인한다 (quickstart.md 1.3). 있으면 원인을 먼저 본다
- [x] T005 `src/db/schema.ts`를 M1 목표로 고친다 (data-model 2.1~2.3): 새 표 `attendanceRewards`(`day` integer PK, CHECK `day BETWEEN 1 AND 7`, `exp` integer NOT NULL CHECK `exp >= 0`, `coins` integer NOT NULL CHECK `coins >= 0`, CHECK `exp > 0 OR coins > 0`), `attendances`에서 `streak` 삭제 · `cycleDay`(integer NOT NULL, `check("attendances_cycle_day_check", …)` 1~7, FK → `attendance_rewards.day`) · `sessionId`(text NULL, FK → `sessions.id` `ON DELETE SET NULL`) · `checkedAt`(timestamptz NOT NULL 기본 `now()`) 추가, 인덱스 `attendances_session_idx (session_id)`, `pointLedger`에 부분 고유 인덱스 `point_ledger_attendance_uq` UNIQUE (`user_id`, `ref_id`) WHERE `reason = 'attendance'`와 `point_ledger_like_received_uq` UNIQUE (`user_id`, `ref_id`) WHERE `reason = 'like_received'` (PK (`user_id`, `date`)는 그대로)
- [x] T006 `npm run db:generate`로 `drizzle/NNNN_<이름>.sql`을 만들고(이름 바꾸기 질문엔 "새로 만들기"), data-model 5.1 순서로 손질한다: 맨 위 주석 `-- GAME-04 (2026-10-07): 자동 출석, 1~7일차 보상표. 기존 출석은 cycle_day = ((streak − 1) % 7) + 1. GAME-05·FR-047: 출석 보상·공감 보상 한 번을 원장 고유 인덱스로` → ① `attendance_rewards` 생성 + 7행 INSERT(경험치 10·10·15·15·20·20·30, 코인 10·20·30·40·50·70·100) ② `cycle_day`(NULL 허용)·`session_id`·`checked_at` 추가 ③ 이전: `cycle_day = ((streak − 1) % 7) + 1`, `checked_at` = 같은 회원·`reason = 'attendance'`·`ref_id = date::text` 원장의 가장 이른 `created_at`(없으면 `date`의 한국 0시), `session_id` NULL ④ `cycle_day` NOT NULL·CHECK·FK 두 개 ⑤ `attendances_streak_check`·`streak` 삭제 ⑥ 인덱스 3개 (`drizzle/meta/` 스냅숏 함께)
- [x] T007 `npm run db:migrate` 뒤 quickstart.md 1.3 "적용 후" 줄을 확인한다: `attendance_rewards` 7행, 옛 `streak` 1·7·8·13·14 → `cycle_day` 1·7·1·6·7, `checked_at`·`session_id`, `streak` 컬럼 없음, 같은 `like_received` `ref_id` 두 번째 INSERT가 고유 인덱스 위반 (FR-020·031·017·047)
- [x] T008 `src/server/points.ts`에 `addLedgerEntry(tx, { userId, reason, expDelta?, coinDelta?, refId? }) => { levelUps: number[] }`를 더한다 (contracts/rewards-ledger.md 1장): `lockUser`를 건 같은 `tx`로 원장 1줄 INSERT, 경험치가 늘면 같은 `tx`로 전후 누적 경험치를 읽는다. 레벨업 알림 INSERT는 US5(T050)에서 채우고 지금은 빈 `levelUps: []`를 돌려줘도 된다. `grantReward`의 원장 직접 INSERT를 `addLedgerEntry` 호출로 바꾼다 (시그니처·하루 상한 동작 그대로, FR-007·012·016·018)
- [x] T009 [P] `src/lib/game.ts`에서 출석 고정 보상을 걷어 낸다 (research R19): `REWARD_RULES`의 `attendance`·`attendance_streak` 삭제(`signup` 0/100/1, `post` 30/30/3, `comment` 5/5/10, `like_received` 2/2/20, `farm_care` 2/0/15만 남김), `RewardReason` 타입에서도 두 값 삭제, `ATTENDANCE_STREAK_BONUS_EVERY` 삭제. `ledger_reason` 열거형의 `attendance_streak` 값은 DB에 그대로 둔다 (FR-030)
- [x] T010 [P] 코드 저장소 `CLAUDE.md`에 호출 순서 규칙을 더한다: "`requireMember()`·`getViewer()`는 `db.transaction()` 밖에서, Server Action·페이지 맨 앞에서 먼저 부른다" + "경험치가 생기는 원장 기록은 `addLedgerEntry()`(또는 `grantReward()`)로만" (research R17, data-model 2.3)

**Checkpoint**: 스키마 M1 적용, `addLedgerEntry` 경유 원장 기록, 출석 고정 보상 규칙 삭제 — 이제 사용자 스토리 작업을 시작할 수 있다

---

## Phase 3: User Story 1 - 가입하면서 내 캐릭터를 고르고 받기 (Priority: P1) 🎯 MVP

**Goal**: 가입할 때 고른 기본 캐릭터 1종 + 초원 배경이 보유·장착되고 원장에 `🎉 가입 축하` +100이 한 줄 남는다. 구현 자체는 auth 단계 1이 하고, game은 `grantReward("signup")`·기본 캐릭터 확인 규칙을 제공하고 검증한다 (research R18).

**Independent Test**: 새 아이디로 가입하며 여자 주민을 고른 뒤 보유 아이템(여자 주민·초원만)·장착 상태·원장 `signup` +100 한 줄만 확인한다. 조작한 캐릭터 ID로 가입하면 회원·프로필·블로그·아이템·코인 중 아무것도 생기지 않는다.

### Tests for User Story 1 ⚠️

- [x] T011 [P] [US1] `e2e/rewards.mjs`에 가입 지급 블록을 쓴다 (quickstart US1-1~4): 여자 주민 선택 가입 → `user_items` 캐릭터 1개(여자 주민)·초원, `profiles` 장착 둘 다, 원장 `reason = 'signup'` `coin_delta = 100` 한 줄, 처음 연 가입 화면의 카드 2개·남자 주민 선택 기본값, 기본 캐릭터가 아닌 `items.id`로 가입 요청 → 회원 행 없음. 헤더 코인은 1일차 자동 출석이 붙어 `🪙 110`임을 기대값으로 둔다 (plan 남은 문제 1)
- [x] T012 [P] [US1] `e2e/rewards.mjs`에 기존 회원·상점 블록을 쓴다 (US1-5·6): `pg`로 캐릭터 여러 개(가입 것 + 산 것)를 가진 회원을 만들어 각 캐릭터를 장착할 수 있는지, 상점 판매 목록에 남자 주민·여자 주민이 없고 `buyItem`이 거부하는지 확인한다 (FR-004·005)

### Implementation for User Story 1

- [x] T013 [US1] `src/server/points.ts`의 `grantReward(tx, userId, "signup")`가 ✨0 · 🪙100, 하루 1회로 동작하고 `addLedgerEntry`를 거치는지 확인한다(경험치 0이라 레벨업 없음). auth 가입 트랜잭션(`src/app/(auth)/actions.ts`, 소유: auth)이 `lockUser` 뒤 한 번 부르고 캐릭터를 `items.type = 'character' AND is_starter = true`로 서버에서 다시 확인하는지 auth PR 리뷰로 확인한다 (FR-002·003, contracts/rewards-ledger.md 3.3)
- [x] T014 [US1] 상점 판매 제외 유지를 확인한다: `src/app/shop/page.tsx`가 `is_starter` 아이템을 목록에서 빼고 `src/app/shop/actions.ts` `buyItem`이 거부하는지 (소유: shop, FR-004). 기존 `user_items` 행을 바꾸는 이전이 없는지 확인한다 (FR-005)
- [x] T015 [US1] 장착 캐릭터가 광장·헤더·블로그 미니룸에 그려지는지(`profiles.character_item_id`) 확인한다 (소유: town·blog·shop, FR-006)

**Checkpoint**: 새로 가입한 회원이 고른 캐릭터·초원·가입 축하 코인을 갖는다 (SC-001, 헤더 값은 남은 문제 1 결정에 따름)

---

## Phase 4: User Story 2 - 활동하면 경험치·코인을 받고 레벨이 오른다 (Priority: P2)

**Goal**: 글·댓글·답글·공감 보상과 하루 상한, 원장 합계 레벨, 헤더 `Lv.N`·`🪙 N`과 꾸미기 진행 막대가 spec대로 동작하고, 보상 회수는 없다. 대부분 지금 구현을 유지하며 `addLedgerEntry` 경유와 공감 보상 DB 고유 인덱스를 확인한다.

**Independent Test**: 경험치·코인 0인 테스트 회원으로 100자 이상 공개 글 4개, 남의 글 댓글, 다른 회원의 공감을 만든 뒤 헤더 숫자·원장 합계를 비교한다.

### Tests for User Story 2 ⚠️

- [x] T016 [P] [US2] `scripts/test-game.ts`에 `todayKST` 0시 경계 시험(UTC 14:59 → 그날, 15:01 → 다음 날)을 더하고, 레벨 공식·`MAX_LEVEL` 기존 시험(Lv.2 = 100, 485,100 → 99, `MAX`)을 유지한다 (US2-8·10·11, FR-008·009)
- [x] T017 [P] [US2] `e2e/rewards.mjs`에 보상 규칙 블록을 쓴다 (quickstart US2-1~7·14): 100자 공개 새 글 → ✨30·🪙30, 같은 날 4개 → 보상 3번, 99자 → 없음·100자 → 있음, 수정(비공개 → 공개 포함) → 없음, 남의 글 댓글·답글 → ✨5·🪙5, 내 글 댓글 → 없음, 공감 → 취소 → 공감 → 글 주인 보상 1번, 보상받은 글·댓글 삭제 뒤 원장·레벨 그대로
- [x] T018 [P] [US2] `e2e/rewards.mjs`에 상한·동시성·레벨 블록을 쓴다 (US2-8·9·10·11·12, SC-006): 같은 보상 요청 동시 N개 → 상한 초과 0건, `pg`로 어제 날짜 보상 3줄을 넣은 뒤 오늘 글 → 보상 있음, 경험치 99 → +1 → `Lv.2`, 150 → 꾸미기 `Lv.2`·`50 / 200 EXP`, 485,100 이상 → `MAX`
- [x] T019 [P] [US2] `e2e/rewards.mjs`에 375px 헤더에서 `Lv.N`·`🪙 N`이 모두 보이는지와 보상 뒤 다음 화면에서 헤더 숫자가 바뀌어 있는지 확인을 더한다 (US2-13, FR-010·013, SC-007·008)

### Implementation for User Story 2

- [x] T020 [US2] `src/server/points.ts` `grantReward`가 `addLedgerEntry` 경유 뒤에도 하루 상한(글 3 · 댓글 10 · 공감 20)을 `lockUser` 잠금 안에서 `startOfTodayKST` 기준으로 세는지 확인하고, 회수 코드(원장 행 삭제·음수 경험치)가 어디에도 없는지 `grep`으로 확인한다 (FR-015·016·018, SC-011)
- [x] T021 [US2] 공감 보상 경로를 확인한다: `src/app/blog/actions.ts` `toggleLike`(소유: social)가 `lockUser(글 주인)` 안에서 `ref_id = "{글ID}:{공감한 회원ID}"` 원장 줄이 없을 때만 `grantReward(…, "like_received", refId)`를 부르는지. `point_ledger_like_received_uq` 위반은 트랜잭션 전체 취소라는 점을 social에 알린다 (FR-017·047, research R25)
- [x] T022 [US2] social에 요청: 답글 등록도 남의 글일 때만 `grantReward(…, "comment", id)`(댓글과 하루 10 합산), 보상을 준 Action 끝에 `revalidatePath("/", "layout")` (plan 의존성 "social (단계 5)", FR-015·017·013)
- [x] T023 [P] [US2] 표시 유지 확인: 글쓰기 아래 안내 4가지 문구(`src/components/editor/post-form.tsx`, 소유: post, FR-019), 꾸미기 `Lv.N`·`현재 / 필요 EXP`·초록 막대·`MAX`(`src/app/closet/page.tsx`, 소유: shop, FR-011), 상점 오른쪽 위 `Lv.N 🪙 N`(`src/app/shop/page.tsx`, 소유: shop, FR-013)
- [x] T024 [US2] town에 요청: 헤더(`src/components/site-header.tsx`, 소유: town) 유저 상태창 개편 때 375px에서도 `Lv.N`·`🪙 N`(천 단위 쉼표, `/wallet` 링크)을 숨기지 않기 (FR-010·013, SC-008)

**Checkpoint**: 보상·상한·레벨 표시가 원장 합계와 100% 일치한다 (SC-002·006·007·011)

---

## Phase 5: User Story 3 - 로그인만 하면 자동 출석, 1~7일차 주기 보상 (Priority: P2)

**Goal**: 버튼 없이, 로그인한 회원이 그날(한국 시간) 처음 화면을 열면 `getViewer()` 안에서 출석과 일차 보상이 한 트랜잭션으로 기록되고, `/attendance`는 결과·7칸 보상표·이번 달 달력을 보여 준다.

**Independent Test**: `pg`로 어제·그저께 출석을 넣고 오늘 행을 지운 뒤 로그인 상태로 첫 화면을 열어, 일차·원장 `attendance` 줄·출석 화면을 7일 보상표와 비교한다 (10/1~10/11 예시).

### Tests for User Story 3 ⚠️

- [x] T025 [P] [US3] `scripts/test-game.ts`의 `currentStreak` 시험을 `nextCycleDay(last, today)` 시험으로 바꾼다: 기록 없음 → 1, 어제 3 → 4, 어제 7 → 1, 그저께 → 1, 10/1~10/11(10/10 빠짐) → 1~7, 1, 2, (없음), 1, 9/30 → 10/1, 12/31 → 1/1 이어짐 (US3-1~5·11, SC-005)
- [x] T026 [P] [US3] `e2e/attendance.mjs`에 자동 출석 블록을 쓴다 (quickstart US3-1~5·11·13): 출석 기록 없는 회원 첫 화면 → `cycle_day = 1`, 원장 `attendance` ✨10·🪙10 한 줄(`ref_id` = 오늘 날짜), 어제 1~6 → +1, 어제 7 → 1, 그저께 → 1, 각 일차 보상이 `attendance_rewards`와 같음, `session_id`·`checked_at` 채워짐 (FR-025)
- [x] T027 [P] [US3] `e2e/attendance.mjs`에 동시성·실패 블록을 쓴다: 같은 회원 10개 탭 동시 첫 화면 → 출석 1행·원장 `attendance` 1줄 (SC-003, US3-6), 보상 INSERT를 실패시키는 조건(예: `pg`로 오늘 `ref_id`의 `attendance` 원장 줄을 먼저 넣어 고유 인덱스 위반)에서 출석 행도 남지 않고 일반 화면은 그대로 열림 (US3-7, FR-024), 로그아웃(세션 삭제) 뒤 출석 행 남고 `session_id` NULL
- [x] T028 [P] [US3] `e2e/attendance.mjs`에 출석 화면 블록을 쓴다 (US3-8·10·12, SC-008): [출석하기] 버튼 없음, `🎁 출석 완료! N일차`, `오늘 N일차 출석 완료`, 오늘 일차 강조 7칸 보상표, `이번 달 N일 출석 · 현재 N일차`, 달력 이번 달만·출석일 🌟 초록·오늘 노란 테두리, 방문자 → `/` redirect, 375px 가로 스크롤 없음, 옛 `attend` Action ID 요청 → 변화 없음, 스크린샷 `attendance-day4.png`·`attendance-375.png`

### Implementation for User Story 3

- [x] T029 [P] [US3] `src/lib/game.ts`에 `nextCycleDay(last: { date, cycleDay } | null, today)`를 더하고 `currentStreak`을 지운다: 없음 → 1, 어제 1~6 → 어제 + 1, 어제 7 → 1, 그저께 이전 → 1. 어제 판단은 `previousDay()`로 달·해를 넘긴다 (FR-022, data-model 3.2)
- [x] T030 [US3] 새 파일 `src/server/attendance.ts`에 `ensureTodayAttendance(userId, sessionId)`를 만든다 (contracts/attendance.md 1장): 한 `db.transaction` 안에서 `lockUser` → 같은 `tx`로 오늘 이전 마지막 출석 1건 → `nextCycleDay` → `attendances` INSERT `ON CONFLICT DO NOTHING`(이미 있으면 그 행을 읽고 끝) → `attendance_rewards`에서 그 일차 값 → `addLedgerEntry(tx, { reason: "attendance", expDelta, coinDelta, refId: 오늘 "YYYY-MM-DD" })`. `session_id`는 `(SELECT id FROM sessions WHERE id = 현재 세션)` 서브쿼리로 넣는다 (research R7·R16). 결과 `{ date, cycleDay }` (depends on T008, T029)
- [x] T031 [US3] `src/server/attendance.ts`에 화면용 조회 두 개를 더한다: 이번 달(한국 시간 1일 이후) 출석 날짜·일차 목록, `attendance_rewards` 7행 (FR-026·027)
- [x] T032 [US3] `src/server/dal.ts` `getViewer()`(소유: auth, 합의된 추가)에 오늘(`todayKST()`) 출석 행 LEFT JOIN을 기존 프로필 쿼리에 붙이고, 로그인 세션·프로필이 있고 오늘 행이 없으면 `ensureTodayAttendance(userId, session.id)`를 try/catch로 부른다. 실패하면 서버 로그만 남기고 `attendance: null`. `viewer.attendance = { date, cycleDay } | null` 필드를 더한다 (FR-021, SC-004, research R1·R16) (depends on T030)
- [x] T033 [US3] `src/app/attendance/page.tsx`를 고친다 (contracts/attendance.md 2장): 방문자 `/` redirect(`requireMember()`), `viewer.attendance`가 null이면 `ensureTodayAttendance()`를 한 번 더 부르고 실패하면 던져 `src/app/error.tsx`로, 제목 `📮 출석 체크`, `← 광장으로 나가기`(`ExitButton`), 결과 카드 `🎁 출석 완료! N일차`·`오늘 N일차 출석 완료`, 7칸 보상표(`grid-cols-7`, 칸마다 `N일차`·`✨ N`·`🪙 N`, 오늘 칸 노란 테두리·굵게), `이번 달 N일 출석 · 현재 N일차`, 달력(일~토 7칸 정사각형, 출석일 🌟 + 초록 칸, 오늘은 초록 바탕 + 🌟 + 노란 테두리). 옛 문구(`[📮 출석하고 보상 받기]`, `편지 여는 중...`, `출석 완료! N일 연속`, `(7일 연속 보너스 포함!)`, `✅ 오늘은 이미 출석했어요. 내일 또 만나요!`, `현재 연속 N일`, `연속 출석이 끊겼어요 (…)`, `아직 출석 기록이 없어요`, 위 안내 `하루 한 번 ✨ 10 · 🪙 20, …`)를 지운다 (FR-026·027·029, research R22) (depends on T031, T032)
- [x] T034 [US3] 버튼 출석을 지운다: `src/app/attendance/actions.ts`(`attend()`)와 `src/app/attendance/attend-button.tsx`(`AttendButton`) 삭제, 남은 import 정리 (FR-021, contracts/attendance.md "없어지는 인터페이스")
- [x] T035 [P] [US3] 새 클라이언트 컴포넌트 `src/components/game/attendance-day-watcher.tsx`(`AttendanceDayWatcher`, 화면 표시 없음)를 만든다: 처음 그린 한국 날짜를 기억하고 주소 이동·`visibilitychange`(탭 다시 보기) 때 날짜가 바뀌었으면 `router.refresh()` (research R2, Edge Cases 0시 경계)
- [x] T036 [US3] `src/components/site-header.tsx`(소유: town, 추가)에 회원일 때만 `AttendanceDayWatcher`를 끼운다 (depends on T035)
- [x] T037 [US3] town에 요청: `src/components/town/types.ts` `TownData.attendedToday: boolean` → `attendanceDay: number | null`(= `viewer.attendance?.cycleDay ?? null`), `src/server/town.ts` `hasAttendedToday()` 대체, 출석 도장 입구 문구 `오늘 N일차 ✅`(`src/components/town/scene.ts`, `town-menu.tsx`) — TOWN FR-023과 문구 충돌은 팀 결정 뒤 반영 (FR-028, plan 남은 문제 2, research R20)
- [x] T038 [US3] 기존 E2E에서 버튼 출석을 걷어 낸다: `e2e/game.mjs`의 출석 버튼·동시 클릭 부분 삭제(상점·꾸미기 부분은 shop 소유라 그대로), `e2e/decisions.mjs`의 GAME-04 블록(`streak` 직접 INSERT, 버튼, 끊김 문구) 삭제 (research R23)

**Checkpoint**: 로그인 상태 첫 화면에서 자동 출석·일차 보상, 출석 화면 새 모양, 하루 한 번 보장 (SC-003·004·005)

---

## Phase 6: User Story 4 - 내 경험치·코인 내역 보기 (Priority: P2)

**Goal**: 헤더 `🪙 N`·꾸미기 [경험치·코인 내역 보기]에서 `/wallet`으로 가서 본인 원장을 최신순 20개씩 본다. 지금 구현을 유지하고, 출석 변경 뒤에도 옛 `🔥 연속 출석 보너스` 줄이 보이는지 확인한다.

**Independent Test**: 출석·글·구매 기록이 있는 테스트 회원으로 `/wallet`을 열어 위 요약과 헤더 값, 각 줄 표시, `기록 N개`, 페이지 넘김을 비교한다.

### Tests for User Story 4 ⚠️

- [x] T039 [P] [US4] `e2e/rewards.mjs`에 내역 블록을 쓴다 (quickstart US4-1~7, SC-002·010): 헤더 `🪙 N` 클릭 → `/wallet`, 위 요약 코인 = 헤더 코인, 구매 줄 `🏪 아이템 구매 · {아이템 이름}`·빨간 `🪙 −가격`, 21개 이상 → 최신순 20개·`기록 N개`·2페이지, 빈 회원 → `아직 기록이 없어요`, 다른 회원 기록 안 보임, 방문자 → `/`, 줄의 `📮 출석` 이름·`YYYY. MM. DD. HH:MM`·0 생략, 맨 아래 `코인은 상점에서 쓸 수 있어요.`, 스크린샷 `wallet.png`
- [x] T040 [P] [US4] `e2e/decisions.mjs`의 GAME-07 "연속 출석 보너스" 확인을 `pg`로 `reason = 'attendance_streak'` 옛 원장 줄을 직접 넣어 `/wallet`에 `🔥 연속 출석 보너스`가 보이는지 보는 방식으로 바꾼다 (US4-8, FR-030·035)

### Implementation for User Story 4

- [x] T041 [US4] `src/app/wallet/page.tsx`와 `src/server/points.ts` `listLedger`·`getWallet`이 contracts/rewards-ledger.md 3.1과 같은지 확인한다: 사유 이름 `🎉 가입 축하`, `📮 출석`, `🔥 연속 출석 보너스`(옛 기록), `✏️ 글 작성`, `💬 댓글 작성`, `♥ 공감 받음`, `🏪 아이템 구매` (+ town 사유 그대로), T009로 `RewardReason`에서 `attendance_streak`를 뺀 뒤에도 사유 이름 매핑이 `ledger_reason` 열거형 전체를 다루는지(타입 오류 없이) 고친다 (FR-032~036)
- [x] T042 [US4] 꾸미기 레벨 막대 아래 `경험치·코인 내역 보기` 링크(`src/app/closet/page.tsx`, 소유: shop)와 헤더 `🪙 N`(title `코인`) → `/wallet` 링크가 유지되는지 확인한다 (FR-032)

**Checkpoint**: 내역 화면 합계 = 헤더 = 원장 합계 (SC-002), 헤더에서 한 번에 내역 (SC-010)

---

## Phase 7: User Story 5 - 레벨이 오르면 팝업으로 축하받기 (Priority: P3)

**Goal**: 원장 기록으로 레벨이 오르면 같은 트랜잭션에서 레벨마다 `level_up` 알림을 정확히 한 번 저장하고, 안 본 레벨업이 있으면 헤더에서 가장 높은 레벨 하나를 `<dialog>` 팝업으로 보여 준다. 알림 표(M2)도 이 단계에서 만든다 (US6이 함께 쓴다).

**Independent Test**: 경험치 290인 테스트 회원이 ✨10 이상을 받게 한 뒤 `Lv.3이 되었어요!` 팝업이 한 번 뜨고, [확인] 뒤 새로고침해도 다시 뜨지 않으며 `notifications`에 기록이 남는지 본다.

### Tests for User Story 5 ⚠️

- [x] T043 [P] [US5] `scripts/test-game.ts`에 `levelsGained(beforeExp, afterExp)` 시험을 더한다: 99→100 `[2]`, 290→300 `[3]`, 90→340 `[2, 3]`, 485,090→485,100 `[99]`, 600,000→600,010 `[]` (US2-8·10, US5-1·6)
- [x] T044 [P] [US5] `scripts/test-notifications.ts`에 `levelUpTitle` 시험을 더한다: 3 → `Lv.3이 되었어요!`, 99 → `최고 레벨 Lv.99가 되었어요!` (FR-038)
- [x] T045 [P] [US5] `e2e/notifications.mjs`에 팝업 블록을 쓴다 (quickstart US5-1~6, SC-009): 경험치 290 + 글 보상 → 그 화면에서 `Lv.3이 되었어요!` 한 번, Lv.3 판매 아이템(예: 벚꽃길) 보임·캐릭터 안 보임, 다른 회원 공감으로 레벨업 → 글 주인 다음 화면에서 팝업, [확인]·[상점 가기] 뒤 새로고침 → 안 뜸·알림함에 남음, 레벨 안 오른 보상 → 팝업 없음, 안 본 Lv.3·Lv.4 → `Lv.4이 되었어요!` 하나·둘 다 읽음, Lv.98 → 99 `최고 레벨 Lv.99가 되었어요!`, 같은 레벨 동시 보상 → 알림 1행, `dismissLevelUp`에 `level = 100`·`0`·문자 → 변화 없음, 스크린샷 `levelup-lv3.png`·`levelup-lv99.png`

### Implementation for User Story 5

- [x] T046 [US5] `src/db/schema.ts`에 M2를 더한다 (data-model 2.4·2.5): `notificationKind` pgEnum(`level_up`, `like`, `comment`, `reply` 네 값 모두 처음부터), 표 `notifications`(`id` integer identity PK, `user_id` text NOT NULL FK → `users.id` CASCADE, `kind` NOT NULL, `actor_id` text NULL FK → `users.id` CASCADE, `post_id` integer NULL FK → `posts.id` CASCADE, `level` integer NULL, `created_at` timestamptz NOT NULL 기본 `now()`, `read_at` timestamptz NULL), CHECK `notifications_shape_check`(`level_up`이면 `level IS NOT NULL AND actor_id IS NULL AND post_id IS NULL`, 아니면 `level IS NULL AND actor_id IS NOT NULL AND post_id IS NOT NULL`), `notifications_level_check`(`level IS NULL OR level BETWEEN 2 AND 99`), `notifications_not_self_check`(`actor_id IS NULL OR actor_id <> user_id`), 인덱스 `notifications_level_up_uq` UNIQUE (`user_id`, `level`) WHERE `kind = 'level_up'`, `notifications_user_created_idx` (`user_id`, `created_at` DESC, `id` DESC), `notifications_unread_idx` (`user_id`) WHERE `read_at IS NULL`, `notifications_post_idx` (`post_id`) WHERE `post_id IS NOT NULL`
- [x] T047 [US5] `npm run db:generate`로 `drizzle/NNNN_<이름>.sql`을 만들고(손질 없음) 맨 위 주석 `-- GAME-06·GAME-08 (2026-10-07): 레벨업·공감·댓글·답글 알림. 레벨업은 회원·레벨마다 한 번`을 단다. 데이터 이전 없음(옛 레벨업은 만들지 않는다). `npm run db:migrate` 뒤 quickstart.md 1.3의 CHECK·고유 인덱스 위반 확인 줄을 돌린다 (depends on T046)
- [x] T048 [P] [US5] `src/lib/game.ts`에 `levelsGained(beforeExp, afterExp): number[]`를 더한다: `levelFromExp` 전후 사이의 레벨 목록(L1+1 … L2), 99를 넘지 않음 (FR-037, data-model 3.3)
- [x] T049 [P] [US5] 새 파일 `src/lib/notifications.ts`에 `levelUpTitle(level)`을 만든다: `Lv.{N}이 되었어요!`, 99면 `최고 레벨 Lv.99가 되었어요!` (FR-038)
- [x] T050 [US5] `src/server/points.ts` `addLedgerEntry`에 레벨업 알림을 채운다: `expDelta > 0`이면 같은 `tx`로 기록 전후 누적 경험치 → `levelsGained` → 레벨마다 `notifications`(`kind = 'level_up'`, `level`) INSERT `ON CONFLICT DO NOTHING`, `{ levelUps }` 반환 (FR-037·047, research R8) (depends on T047, T048)
- [x] T051 [US5] town에 요청: `src/server/farm.ts` `addGrowth`(소유: town)의 `farm_grown` 원장 직접 INSERT를 `addLedgerEntry`로 바꾸고, 성장 아이템으로 다 키우는 Action 끝에 `revalidatePath("/", "layout")` (FR-037·038, plan S7) (depends on T050)
- [x] T052 [US5] 새 파일 `src/server/notifications.ts`에 `getHeaderNotifications(userId)`(한 쿼리 → `{ unread: 0~10, pendingLevel, pendingMinLevel }`, research R15)와 `getPendingLevelUp(userId)`(가장 높은 안 읽은 레벨 L2, 안 읽은 범위 L1~L2의 판매 아이템: `is_on_sale = true AND type <> 'character'`, shop 마이그레이션 전에는 `type <> 'character' AND is_starter = false`, 최대 3개 + 나머지 수)를 만든다. 모든 쿼리에 `user_id = 나` (FR-038·039·041, research R12) (depends on T047)
- [x] T053 [US5] 새 파일 `src/app/notifications/actions.ts`에 `dismissLevelUp(formData)`를 만든다 (contracts/notifications.md 4.3): 첫 줄 `requireMember()`(트랜잭션 밖), zod로 `level` 정수 2~99·`go` `shop`|`stay`, `user_id = 나 AND kind = 'level_up' AND read_at IS NULL AND level <= 입력`인 행 `read_at = now()`, `revalidatePath("/", "layout")`, `go = shop`이면 `/shop` redirect. 형식 오류면 아무것도 바꾸지 않음 (FR-040·041)
- [x] T054 [P] [US5] 새 클라이언트 컴포넌트 `src/components/game/level-up-dialog.tsx`(`LevelUpDialog`)를 만든다: 네이티브 `<dialog>` ref `showModal()`, 뒤 화면 어둡게, 맨 위 🎉, 큰 글씨 제목(`levelUpTitle`), 아이템이 있으면 `이제 이런 친구를 데려올 수 있어요` + 그림·이름 최대 3개(+ `외 N개`), 없으면 이 줄 없음, [상점 가기]·[확인] 두 버튼 모두 `dismissLevelUp` 폼(`go` = `shop`/`stay`, 높이 44px 이상), Esc(`cancel` 이벤트)는 [확인]과 같게 (FR-038~040, plan 남은 문제 4, research R11)
- [x] T055 [US5] 새 서버 컴포넌트 `src/components/game/level-up-popup.tsx`(`LevelUpPopup`)를 만든다: `getPendingLevelUp`으로 안 본 레벨업이 있을 때만 `LevelUpDialog`를 그린다 (depends on T052, T053, T054)
- [x] T056 [US5] `src/components/site-header.tsx`(소유: town, 추가)에 회원일 때만 `LevelUpPopup`을 끼운다 (FR-038) (depends on T055)

**Checkpoint**: 레벨업 한 번에 알림 1행·팝업 1번, [확인] 뒤 다시 뜨지 않음 (SC-009)

---

## Phase 8: User Story 6 - 알림함(🔔)에서 소식 모아 보기 (Priority: P3)

**Goal**: 헤더 🔔 + 안 읽은 수 배지, `/notifications` 알림함(최신순 20개, 노란 배경, 시간 표시, [모두 읽음]), 알림을 누르면 관련 화면 이동 + 읽음. 1단계는 레벨업, 2단계 공감·댓글·답글은 game이 문구·링크·`notifyActivity()`를 1단계 때 만들고 social이 단계 7에서 호출을 붙인다.

**Independent Test**: 1단계 — 레벨업 알림 2개가 쌓인 테스트 회원으로 🔔 숫자·목록·[모두 읽음]을 확인한다. 2단계 — 다른 회원의 공감·댓글·답글과 자기 활동 뒤 🔔 숫자·목록을 비교한다.

### Tests for User Story 6 ⚠️

- [x] T057 [P] [US6] `scripts/test-notifications.ts`에 시험을 더한다 (quickstart 2장): `unreadBadge` 0·3·9·10·25 → 없음·`3`·`9`·`9+`·`9+`, `notificationTime` 30초·59분·60분·23시간 59분·24시간 → `방금`·`59분 전`·`1시간 전`·`23시간 전`·`YYYY.MM.DD`(한국 시간), `notificationText` 4종(FR-045 문구 그대로), `notificationHref` 4종(`/shop`, `/@{주소}/{글ID}`, `/@{주소}/{글ID}#comments` ×2) (FR-042·043·045)
- [x] T058 [P] [US6] `e2e/notifications.mjs`에 알림함 1단계 블록을 쓴다 (quickstart US6-1~6): 안 읽은 3개 → 🔔 옆 빨간 `3`, 10개 이상 → `9+`, [모두 읽음] → 숫자 사라짐·노란 배경 없음, 레벨업 알림 클릭 → `/shop`·읽음, 다른 회원 알림 안 보임, 빈 목록 `아직 알림이 없어요`, 방문자 헤더에 🔔 없음·`/notifications` → `/`, 스크린샷 `notifications-list.png`·`notifications-empty.png`·`header-bell-375.png`
- [x] T059 [P] [US6] `e2e/notifications.mjs`에 조작 블록을 쓴다 (contracts/notifications.md 7장): 다른 회원 알림 ID로 `openNotification` → `read_at` 그대로·`/notifications` redirect, ID `2147483648`·`1e3`·문자열 → 변화 없음·500 없음, 방문자 Action 호출 → `/` redirect, `/notifications?page=99999999999999999999` → 1페이지 HTTP 200
- [x] T060 [P] [US6] `e2e/notifications.mjs`에 2단계 블록을 쓴다 (US6-7~10, SC-012) — social 단계 7 merge 뒤 켜도록 분리: 다른 회원 공감 → 🔔 +1·`❤️ {닉네임}님이 「{글 제목}」에 공감했어요`·팝업 없음, 댓글 알림 클릭 → 그 글 `#comments`·읽음, 답글 → 원댓글 작성자에게 `💬 {닉네임}님이 내 댓글에 답글을 달았어요`, 자기 글 공감·댓글·자기 댓글 답글 → 알림 없음, 행동한 회원 탈퇴 → 그 알림 삭제 (FR-046)
- [x] T061 [P] [US6] `e2e/params.mjs`에 `/notifications?page=` 한 줄, `e2e/nonfunctional.mjs`의 375px 확인 대상 pages 배열에 `/notifications` 한 줄을 더한다 (SC-008)

### Implementation for User Story 6

- [x] T062 [P] [US6] `src/lib/notifications.ts`에 순수 함수를 더한다: `unreadBadge(n)`(0 → null, 1~9 → 숫자, 10 이상 → `9+`), `notificationTime(createdAt, now)`(1분 미만 `방금`, 60분 미만 `N분 전`, 24시간 미만 `N시간 전`, 그 밖 `YYYY.MM.DD` 한국 시간), `notificationText(row)`(`🎉 Lv.{level}이 되었어요!` / `❤️ {닉네임}님이 「{글 제목}」에 공감했어요` / `💬 {닉네임}님이 「{글 제목}」에 댓글을 달았어요` / `💬 {닉네임}님이 내 댓글에 답글을 달았어요`), `notificationHref(row)`(`level_up` → `/shop`, `like` → `/@{주소}/{글ID}`, `comment`·`reply` → `/@{주소}/{글ID}#comments`) (FR-042·043·045, research R14)
- [x] T063 [US6] `src/server/notifications.ts`에 `listNotifications(userId, page)`를 더한다: `user_id = 나`, `created_at` DESC·`id` DESC 20개, 닉네임(`profiles.nickname` ← `actor_id`)·글 제목(`posts.title`)·블로그 주소(`blogs.slug`)를 JOIN으로 읽음(저장하지 않음), `{ rows, total, page, pageCount }` (FR-043, data-model 2.4) (depends on T047)
- [x] T064 [US6] `src/server/notifications.ts`에 `notifyActivity(tx, { recipientId, actorId, kind, postId })`를 더한다: `kind`는 `like`·`comment`·`reply`만(`level_up` 거부), `recipientId === actorId`면 아무것도 넣지 않음, 같은 `tx`로 INSERT, 넣었는지 여부 반환 (FR-045, contracts/notifications.md 6.1, research R21) (depends on T047)
- [x] T065 [US6] `src/app/notifications/actions.ts`에 `openNotification(formData)`(zod `id` 정수 1~2147483647 = `parseId` 규칙, `id = 입력 AND user_id = 나`인 행만 `read_at = COALESCE(read_at, now())`, `notificationHref`로 redirect, 실패하면 아무것도 바꾸지 않고 `/notifications` redirect)와 `markAllNotificationsRead()`(`user_id = 나 AND read_at IS NULL` → `read_at = now()`)를 더한다. 둘 다 첫 줄 `requireMember()`, 성공 시 `revalidatePath("/", "layout")` (FR-044, contracts/notifications.md 4.1·4.2) (depends on T053, T062)
- [x] T066 [US6] 새 화면 `src/app/notifications/page.tsx`를 만든다 (contracts/notifications.md 3장): `requireMember()`(방문자 `/`), `?page=N` `parsePage()`, 제목 `🔔 알림함`(메타 제목 `알림함`, plan 남은 문제 4 임시값), `← 광장으로 나가기`, 안 읽은 알림이 있을 때 목록 위 [모두 읽음], 줄마다 문구·시간, 줄 전체가 `openNotification` 버튼(높이 44px 이상), 안 읽은 줄 노란 배경, 빈 목록 `아직 알림이 없어요`, `Pagination` 그대로 (FR-042~044) (depends on T063, T065)
- [x] T067 [P] [US6] 새 컴포넌트 `src/components/game/notification-bell.tsx`(`NotificationBell`)를 만든다: `/notifications` 링크, 접근 이름 `알림`(plan 남은 문제 4 임시값), `unreadBadge` 빨간 숫자 배지(글자 하나 폭) (FR-042, research R22) (depends on T062)
- [x] T068 [US6] `src/components/site-header.tsx`(소유: town, 추가)에 회원일 때만 `NotificationBell`을 끼우고, `getHeaderNotifications` 한 쿼리 결과를 🔔 배지와 `LevelUpPopup`이 함께 쓰게 한다(헤더 쿼리 하나만 늘림, NF-07). 방문자에겐 🔔·팝업·날짜 감시 모두 없음. 375px 공간은 town과 `e2e/mobile.mjs`로 확인 (FR-042, US6-6, research R15) (depends on T056, T067)
- [x] T069 [US6] social에 요청(단계 7): 공감이 새로 저장될 때(`toggleLike`, 글 주인에게), 남의 글 댓글(`addComment`, 글 주인에게), 답글 등록(원댓글 작성자에게) 트랜잭션 안에서 `notifyActivity()` 호출, 댓글 영역 `src/components/blog/comment-section.tsx`에 `id="comments"` (`specs/004-social/contracts/notification-triggers.md`와 같은 조건). 반복 공감 알림은 spec 문구대로 두고 plan 남은 문제 3으로 확인 (FR-045, research R21)
- [x] T070 [US6] auth에 확인 요청(단계 9 탈퇴): `notifications.user_id`·`actor_id` CASCADE로 받은 알림·남긴 알림이 함께 지워지는지, `attendances` CASCADE (FR-046)
- [ ] T071 [US6] blog와 정한다: 새 최상위 주소 `/notifications`를 blog FR-009 예약어 목록에 더할지 (plan 남은 문제 8). 더하기로 하면 blog spec·코드 예약어 목록 수정 요청

**Checkpoint**: 1단계 알림함 완성, 2단계는 social의 `notifyActivity()` 호출만 남음 (SC-012는 social 단계 7 뒤)

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: 문서, 전체 검증, 남은 결정 반영

- [x] T072 [P] `docs/02-erd.md`를 data-model 5.4 표대로 고친다: 머리말 v1.5·알림 표, 1장 관계도(`notifications` 엔터티, 받은 알림·행동한 회원·관련 글 관계, 출석·보상표 ⏳ 제거), 2장 보상 그룹, 3.3 "⏳ 자동 출석" ⏳ 제거(auth에 알리고), 3.6 공감 보상 고유 인덱스 한 줄, 3.12 출석, 새 3.x 알림, 3.14 삭제 규칙, 3.15 `notification_kind`, 3.16 인덱스, 3.17 정규화, 7장 완료 표시·알림 행, 부록 NULL 허용
- [x] T073 [P] 코드 저장소 `README.md` 스크립트·E2E 표에 `e2e/attendance.mjs`, `e2e/rewards.mjs`, `e2e/notifications.mjs` 세 줄과 `test:notifications`를 더하고 `e2e/game.mjs` 설명을 바꾼다. `CLAUDE.md`에 자동 출석·레벨업(`addLedgerEntry`) 규칙을 정리한다 (T010과 합침)
- [ ] T074 plan 남은 문제 결정을 반영한다: 1(US1-3·SC-001 `🪙 100` ↔ 자동 출석 `🪙 110`), 2(광장 출석 도장 문구), 4(알림함 제목·🔔 접근 이름·Esc), 5(팝업 아이템 범위) — 정해진 값으로 코드·E2E 기대값을 맞추고, spec 수정이 필요하면 문서 저장소에 요청한다. 6(원본 `docs/01-requirements.md` GAME-04 "데이터 변경" 문단)은 원본 수정 목록에 기록
- [x] T075 `npx tsc --noEmit`, `npx eslint`, `npm test`(test:game → test:ids → test:sanitize → test:notifications)를 통과시킨다
- [ ] T076 quickstart.md 3장 E2E 전체(`node e2e/auth.mjs shots` … `node e2e/nonfunctional.mjs shots`)를 `❌` 없이 돌리고, 4장 표의 game 검증 줄과 5장 스크린샷을 확인한 결과를 PR 설명에 적는다 (CI 없음, constitution 품질 관문)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 의존 없음. 단, E2E 실행은 auth 단계 1 merge가 선행 (위 "선행 조건")
- **Foundational (Phase 2)**: Setup 뒤. 모든 사용자 스토리를 막는다 (M1 스키마, `addLedgerEntry`, 출석 고정 보상 삭제)
- **User Stories (Phase 3+)**: 모두 Foundational 뒤
  - US1 (P1): auth 단계 1에 기댄다. game 쪽은 확인·E2E 위주라 가장 먼저 끝낼 수 있다
  - US2 (P2): Foundational만 있으면 된다. social·post·shop·town 확인·요청 포함
  - US3 (P2): Foundational(T005~T009) 뒤. auth의 `dal.ts` 추가 합의 필요
  - US4 (P2): T009(`RewardReason` 변경) 뒤. US3과 독립이지만 `📮 출석` 줄 확인은 US3 뒤가 자연스럽다
  - US5 (P3): Foundational 뒤. M2 마이그레이션(T046·T047)을 이 단계에서 만든다
  - US6 (P3): **US5의 T047(M2 마이그레이션)·T053(`actions.ts` 파일)·T056(헤더 팝업 자리)에 기댄다**. 2단계(T060·T069)는 social 단계 7에 기댄다
- **Polish (Phase 9)**: 원하는 스토리가 끝난 뒤

### User Story Dependencies

- **User Story 1 (P1)**: Foundational 뒤 시작, 다른 스토리와 무관 (auth 단계 1 선행)
- **User Story 2 (P2)**: Foundational 뒤 시작, 다른 스토리와 무관
- **User Story 3 (P2)**: Foundational 뒤 시작, 다른 스토리와 무관 (레벨업 알림은 US5가 붙기 전까지 빈 동작)
- **User Story 4 (P2)**: Foundational 뒤 시작, 독립 검증 가능
- **User Story 5 (P3)**: Foundational 뒤 시작. US2·US3의 보상 경로가 `addLedgerEntry`를 거치므로 그 보상으로 레벨업이 생긴다
- **User Story 6 (P3)**: US5의 M2·Action 파일·헤더 자리 뒤 (plan "game 안의 구현 순서" 2단계가 M2 → … → U2 → U3 차례)

### plan 구현 순서와의 대응

1. 출석: T005~T008 → T029 → T030~T032 → T033 → T034 → T025 → T026~T028, T038 (Phase 2 + US3)
2. 레벨업·알림함 1단계: T046·T047 → T048 → T049·T062 → T050 → T051 → T052·T063·T064 → T053·T065 → T066 → T054~T056·T067·T068 → E2E (US5 + US6)
3. 알림함 2단계: T069 (social 단계 7) → T060

### Within Each User Story

- 시험(단위·E2E 블록)을 먼저 쓰고 실패를 확인한 뒤 구현
- 스키마 → 마이그레이션 → 순수 규칙(`src/lib/`) → 서버 함수(`src/server/`) → Server Action → 화면·헤더 연결
- 다른 spec 소유 파일은 "추가"는 이 작업에서, "요청"은 해당 spec 담당과 합의 뒤

### Parallel Opportunities

- Setup: T002, T003 병렬
- Foundational: T009, T010은 T005~T008과 병렬 (파일이 다름)
- 사용자 스토리: Foundational 뒤 US1·US2·US3·US4·US5는 서로 다른 파일 위주라 병렬 가능. 단 `e2e/rewards.mjs`(US1·US2·US4), `e2e/notifications.mjs`(US5·US6), `scripts/test-game.ts`(US2·US3·US5), `src/components/site-header.tsx`(US3·US5·US6), `src/lib/game.ts`(US3·US5)는 같은 파일이라 블록 단위로 차례로 합친다
- 스토리 안의 [P] 시험 작업과 [P] 순수 함수·클라이언트 컴포넌트 작업은 병렬

---

## Parallel Example: User Story 3

```bash
# User Story 3 시험을 함께 쓴다:
Task: "nextCycleDay 시험 in scripts/test-game.ts"
Task: "자동 출석 블록 in e2e/attendance.mjs"

# 서로 다른 파일의 구현을 함께 시작한다:
Task: "nextCycleDay 추가 in src/lib/game.ts"
Task: "AttendanceDayWatcher in src/components/game/attendance-day-watcher.tsx"
```

## Parallel Example: User Story 5

```bash
Task: "levelsGained 시험 in scripts/test-game.ts"
Task: "levelUpTitle 시험 in scripts/test-notifications.ts"
Task: "levelsGained in src/lib/game.ts"
Task: "levelUpTitle in src/lib/notifications.ts"
Task: "LevelUpDialog in src/components/game/level-up-dialog.tsx"
```

## Parallel Example: User Story 6

```bash
Task: "unreadBadge·notificationTime·notificationText·notificationHref 시험 in scripts/test-notifications.ts"
Task: "알림함 1단계 블록 in e2e/notifications.mjs"
Task: "params·nonfunctional 한 줄 in e2e/params.mjs, e2e/nonfunctional.mjs"
Task: "NotificationBell in src/components/game/notification-bell.tsx"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: Setup
2. Phase 2: Foundational (CRITICAL - 모든 스토리를 막는다)
3. Phase 3: User Story 1 (auth 단계 1 결과 확인)
4. **STOP and VALIDATE**: 가입 지급만 따로 확인
5. 준비되면 PR

### Incremental Delivery

1. Setup + Foundational → 기반 준비
2. US1 → 확인 → PR (MVP)
3. US2 + US4 (지금 구현 유지 확인 위주) → 확인
4. US3 (자동 출석) → 확인 → PR (plan 구현 순서 1)
5. US5 + US6 1단계 (레벨업 팝업·알림함) → 확인 → PR (plan 구현 순서 2)
6. US6 2단계: social 단계 7 merge 뒤 T060 켜기 (plan 구현 순서 3)

### Parallel Team Strategy

3명 팀 기준:

1. 함께 Setup + Foundational
2. 그다음:
   - 개발자 A: US3 (출석, `dal.ts` 합의 포함)
   - 개발자 B: US5 → US6 (알림)
   - 개발자 C: US1·US2·US4 확인·E2E와 다른 spec 요청 정리
3. 같은 파일(`site-header.tsx`, `e2e/rewards.mjs`, `scripts/test-game.ts`)은 차례로 합친다

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- 다른 spec 소유 파일(`dal.ts`, `site-header.tsx`, `farm.ts`, `town.ts`, `(auth)/actions.ts`, `blog/actions.ts`, shop 화면)은 plan.md "의존성" 표의 방식(추가/요청)을 지킨다
- 모든 보상·출석·알림 쓰기는 `lockUser()` 트랜잭션 안, 같은 `tx`로 (research R7)
- 잔액·레벨 컬럼을 만들지 않는다 (NF-16)
- Verify tests fail before implementing
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
