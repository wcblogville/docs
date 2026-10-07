# Research: 캐릭터 / 성장 (GAME)

**Feature**: `005-game` | **Date**: 2026-10-07 | **Plan**: [plan.md](plan.md)

`plan.md` Technical Context에서 모르는 것과 이 기능의 기술 선택을 정리했다. 근거는 코드 저장소 `main` `feb4c05`에서 읽은 코드와 `docs/02-erd.md` v1.4다.
코드 체크아웃에 `node_modules`가 없어 Next.js 16 · Better Auth 1.7 · Drizzle 0.45의 세부 동작은 기존 코드의 사용례와 일반 동작으로 판단했다. 그런 항목은 **"구현 전에 설치된 패키지 문서로 확인"**이라고 적고 R24에 모았다.

## 결정 목록

| # | 주제 | 결정 요약 |
|---|---|---|
| R1 | 자동 출석을 일으키는 곳 | `getViewer()` 안에서 오늘 출석이 없으면 `ensureTodayAttendance()` |
| R2 | 탭을 열어 둔 채 0시를 넘김 | 헤더의 `AttendanceDayWatcher`가 날짜가 바뀌면 `router.refresh()` |
| R3 | 출석 일차 계산 | 순수 함수 `nextCycleDay()` (`src/lib/game.ts`) |
| R4 | 출석 표 키 | 복합 PK (`user_id`, `date`) 유지, 원장 `ref_id` = 날짜 |
| R5 | 일차별 보상표 | `attendance_rewards` 표, 7행은 마이그레이션이 넣는다 |
| R6 | 출석 마이그레이션 | 생성 SQL을 손으로 순서 조정 (NULL 허용 추가 → 이전 → NOT NULL) |
| R7 | 출석 보상 지급 | 출석 전용 트랜잭션 + 원장 부분 고유 인덱스 |
| R8 | 레벨업 감지 | `addLedgerEntry()`가 원장 기록과 같은 트랜잭션에서 레벨 전후 비교 |
| R9 | 알림 표 | `notifications` 1개 + 열거형 4종을 처음부터 |
| R10 | 읽음·봤음 | `read_at` 한 컬럼 |
| R11 | 레벨업 팝업 | 네이티브 `<dialog>` `showModal()` |
| R12 | 팝업 아이템 | 안 본 레벨 범위의 판매 중 아이템(캐릭터 제외) 3개 + `외 N개` |
| R13 | 알림함 화면 | `/notifications` 페이지, 폼 Server Action |
| R14 | 알림 문구·시간·배지 | 순수 함수 `src/lib/notifications.ts` |
| R15 | 헤더 쿼리 비용 | 알림 수·안 본 레벨업을 한 쿼리로 |
| R16 | 출석 실패 처리 | 화면은 열리고 다음 화면에서 다시 시도 |
| R17 | 교착 방지 | `getViewer()`는 트랜잭션 밖에서 먼저 부른다 |
| R18 | 가입 지급과 첫 화면 코인 | auth가 구현, 가입 날도 1일차 출석 (spec 충돌, open item) |
| R19 | 7일 연속 보너스 중단 | 규칙에서 빼고 열거형·내역 이름은 남긴다 |
| R20 | 광장 출석 도장 문구 | 일차 값은 game이 주고 문구는 town과 결정 (spec 충돌) |
| R21 | 공감·댓글·답글 알림 연결 | `notifyActivity()`를 social이 부른다 |
| R22 | 375px 배치 | 7칸 표·달력·🔔·팝업 |
| R23 | 테스트 | 순수 함수 단위 시험 + 새 E2E 3개 + 깨지는 E2E 고치기 |
| R24 | 구현 전에 확인할 라이브러리 동작 | 목록 |
| R25 | 공감 보상 "같은 사람·같은 글 한 번"을 DB로도 | 원장 부분 고유 인덱스 `point_ledger_like_received_uq` |

---

## R1. 자동 출석을 일으키는 곳

**Decision**: `getViewer()`(`src/server/dal.ts`, React `cache`로 요청당 한 번)가 세션과 프로필을 읽을 때 **오늘 출석 행을 LEFT JOIN으로 함께 읽고**, 프로필이 있는데 오늘 출석이 없으면 새 함수 `ensureTodayAttendance(userId, sessionId)`(`src/server/attendance.ts`)를 부른다. 결과를 `viewer.attendance = { date, cycleDay } | null`로 돌려준다. `dal.ts`는 auth 소유이므로 "공통 모듈 추가: 필드 하나 + 호출 한 줄"로 넣고 auth와 합의한다.

**Rationale**:
- 루트 레이아웃의 `SiteHeader`(`src/components/site-header.tsx`)와 회원 화면(`requireMember()` → `getViewer()`)이 모두 `getViewer()`를 거친다. `cache`라서 헤더와 페이지가 **같은 promise**를 기다리므로, 둘 다 `getWallet()`을 부르기 전에 출석이 끝나 있다. 그래서 그 화면이 다 보일 때 헤더 코인에 이미 반영되고(SC-004), 헤더·내역·꾸미기 값이 같다(SC-002).
- 출석만 따로 요청하는 Action·주소가 없다. 출석은 서버의 `getViewer()` 안에서만 일어나고, 일차·보상은 서버가 정한다 (constitution IV, FR-047).
- **`getViewer()`를 부르는 곳은 화면 렌더만이 아니다** (코드 확인): 회원 Server Action은 모두 `requireMember()`로 시작하고(`src/app/blog/actions.ts`, `src/app/write/actions.ts`, `src/app/farm/actions.ts`, `src/app/shop/actions.ts`, `src/app/closet/actions.ts`, `src/app/settings/blog/actions.ts`), `recordBlogVisit`과 `POST /api/uploads`(`src/app/api/uploads/route.ts`)도 `getViewer()`를 부른다. 그래서 0시를 넘긴 화면에서 Server Action이나 첨부 올리기를 하면 그 요청에서 출석이 일어난다. 같은 잠금·PK·고유 인덱스를 거치므로 하루 한 번은 그대로이고, Server Action은 끝에 `revalidatePath("/", "layout")`로 헤더를 다시 그린다. 일부러 막지 않는다 (막으려면 "렌더에서만" 표시를 넘겨야 해서 복잡해진다).
- 이미 출석한 날은 기존 프로필 쿼리에 JOIN 하나가 늘 뿐 추가 왕복이 없다. 쓰기는 회원당 하루 한 번이다 (원본 GAME-04 "매 요청마다 DB에 쓰지 않도록").
- `next.config.ts`에는 `rewrites()`만 있고 `cacheComponents` 같은 설정이 없다. 루트 레이아웃의 `SiteHeader`가 `getViewer()` → `getSession()`에서 `headers()`를 읽어 모든 화면이 동적 렌더다. 빌드 때 출석이 실행될 일은 없다.

**Alternatives considered**:
- `proxy.ts`(예전 middleware): 정적 파일·prefetch를 포함한 모든 요청에서 돌고, 세션 확인과 DB 접근을 proxy에 넣어야 한다. 지금 코드에 proxy가 없고(plan-context 1.1) 렌더와 다른 단계라 헤더와 같은 값을 보장하기 어렵다.
- 헤더의 클라이언트 컴포넌트가 화면을 그린 뒤 Server Action으로 출석: 첫 화면은 출석 전 코인으로 그려져 SC-004를 못 지키고, JS가 늦으면 출석이 늦는다.
- `SiteHeader`에서만 호출: 레이아웃과 페이지가 동시에 렌더되므로 페이지의 `getWallet()`(내역·꾸미기·상점)이 출석 전 값을 읽을 수 있다 (SC-002 어긋남).
- 원본 GAME-04 구현 제안의 "오늘 출석함" 쿠키: Server Component는 쿠키를 쓸 수 없다(쿠키 설정은 Server Action·Route Handler·proxy에서만, 구현 전에 설치된 Next.js 16 문서로 확인). JOIN으로 읽기 비용이 거의 없어 쿠키가 필요 없다.
- 로그인할 때만 출석(`signIn`): [로그인 상태 유지] 세션은 7일 살아 있어 다음 날 출석이 안 된다.

## R2. 탭을 열어 둔 채 0시를 넘긴 경우

**Decision**: 헤더(회원일 때)에 클라이언트 컴포넌트 `AttendanceDayWatcher`(`src/components/game/attendance-day-watcher.tsx`)를 둔다. 서버가 그린 날짜 `day`(= `viewer.attendance.date`, 없으면 `todayKST()`)를 받고, **주소가 바뀌거나(`usePathname`) 탭이 다시 보일 때(`visibilitychange`)** 브라우저에서 계산한 `todayKST()`가 `day`와 다르면 `router.refresh()`를 한 번 부른다. 같은 (`day`, 브라우저 날짜) 조합에서는 다시 부르지 않는다.

**Rationale**:
- 루트 레이아웃은 화면 이동만으로 다시 그려지지 않는다(코드 저장소 `CLAUDE.md`). 0시 뒤 다른 화면으로 이동하면 그 페이지의 `getViewer()`가 출석은 기록하지만 헤더 코인·팝업은 예전 그대로다. `router.refresh()`는 레이아웃까지 서버에서 다시 그려 출석·헤더·팝업이 함께 맞춰진다 (Edge Cases "0시 이후 처음 화면을 새로 열 때", SC-007).
- 날짜는 서버(`todayKST()`)가 정한다. 브라우저 시계가 틀려도 출석이 잘못 기록되지 않고, 최악의 경우 불필요한 refresh 한 번이다.

**Alternatives considered**: 일정 간격 polling(쓸데없는 요청), 0시 타이머(절전·시계 차이로 틀어짐), 아무것도 안 함(헤더 숫자가 다음 새로고침까지 틀림).

**확인**: `router.refresh()`가 루트 레이아웃 Server Component를 다시 그리는지 구현 전에 설치된 Next.js 16 문서로 확인.

## R3. 출석 일차 계산

**Decision**: 순수 함수 `nextCycleDay(last: { date: string; cycleDay: number } | null, today: string): number`를 `src/lib/game.ts`에 둔다. `last`가 없음 → 1, `last.date === previousDay(today)`이고 1~6 → +1, 7 → 1, 그 밖 → 1. `previousDay()`는 지금 함수를 그대로 쓴다. `currentStreak()`과 `ATTENDANCE_STREAK_BONUS_EVERY`는 지운다.

**Rationale**: FR-022 표를 그대로 옮긴 것이고, DB 없이 SC-005 예시(10/1~10/11)·7일차 초기화·하루 빠짐·9/30→10/1·12/31→1/1을 `scripts/test-game.ts`에서 시험할 수 있다(6.1 관례). 서버는 잠금 뒤 "오늘 이전 마지막 출석 1건"을 읽어 이 함수에 넘긴다.

**Alternatives considered**: SQL `CASE`로 계산(단위 시험이 어렵다), 매번 지난 출석을 거슬러 세기(ERD 3.17이 `cycle_day`를 저장하는 이유와 반대).

## R4. 출석 표의 키와 원장 연결

**Decision**: ERD 3.12대로 **복합 PK (`user_id`, `date`)를 유지**하고 `streak` → `cycle_day`, `session_id`(FK → `sessions.id` `ON DELETE SET NULL`), `checked_at`을 둔다. 원장 출석 보상의 `ref_id`는 지금처럼 날짜 문자열(`YYYY-MM-DD`)이다. `session_id`에는 인덱스를 둔다.

**Rationale**:
- `docs/01-requirements.md` "2026-10-07 설계 변경" 표의 "엔티티는 `id` PK, 잇는 표·하루 기록은 복합 PK … 본문의 '기본 키 (A, B)' 설명은 그대로 맞다"가 GAME-04 본문의 2026-10-06 변경 결정(`id` PK + UNIQUE, `ref_id` = 출석 ID)보다 우선한다. ERD 3.7도 "원장이 출석을 가리킬 때는 날짜"라고 정했다.
- 지금 `attend()`(`src/app/attendance/actions.ts`)가 이미 `grantReward(tx, userId, "attendance", today)`로 `ref_id` = 날짜를 넣는다. 기존 원장과 그대로 이어진다 (FR-025 "어느 출석에서 나왔는지").
- `sessions` 행이 지워질 때(로그아웃·만료) `SET NULL`이 `attendances`에서 그 세션을 찾아야 한다. PostgreSQL은 참조하는 쪽 컬럼에 인덱스를 자동으로 만들지 않으므로, 회원 수 × 날짜로 늘어나는 표를 매번 훑지 않게 인덱스를 둔다.

**Alternatives considered**: `id` PK + UNIQUE (`user_id`, `date`) (원본 GAME-04 1.10 문단. UNIQUE를 또 걸어야 하고 ERD 6장 결정과 다르다), `session_id` 인덱스 없음 (지금은 행이 적지만 계속 늘어난다).

## R5. 일차별 보상표를 두는 곳

**Decision**: 새 표 `attendance_rewards`(`day` PK, `exp`, `coins`)를 만들고 **같은 마이그레이션에서 D10 7행을 넣는다** (경험치 10·10·15·15·20·20·30, 코인 10·20·30·40·50·70·100). 숫자를 바꿀 때는 새 데이터 마이그레이션(UPDATE)으로 한다. 출석 화면의 7칸 보상표도 이 표를 읽는다. CHECK: `day` 1~7, `exp ≥ 0`, `coins ≥ 0`, `exp > 0 OR coins > 0`.

**Rationale**:
- 기존 출석 행의 `cycle_day`에 FK를 걸려면 7행이 먼저 있어야 한다. `npm run db:seed`는 `db:migrate` 뒤에 따로 돌리므로 시드로는 순서를 맞출 수 없다.
- 마이그레이션으로 넣으면 팀원 DB·배포 DB에 같은 값이 들어간다 (ERD 4장, constitution V). `npm run db:reset`은 `TRUNCATE users, tags … CASCADE`라 이 표를 지우지 않는다.
- 마지막 CHECK는 원장의 `point_ledger_nonzero_check`(`exp_delta <> 0 OR coin_delta <> 0`) 위반으로 출석 트랜잭션이 실패하는 일을 표 단계에서 막는다.

**Alternatives considered**: `scripts/seed.ts` 블록(FK 이전 순서 문제, 시드를 안 돌린 DB에서는 출석이 실패), `src/lib/game.ts` 상수(ERD 3.12가 "숫자를 바꿀 때 표만 고친다"로 결정, 지난 출석의 일차와 숫자가 따로 놀게 된다).

## R6. 출석 마이그레이션 작성 방법

**Decision**: `src/db/schema.ts`를 최종 모양으로 고치고 `npm run db:generate`로 SQL을 만든 뒤, **생성된 SQL을 손으로 순서 조정**한다.

1. `attendance_rewards` 생성과 7행 INSERT
2. `attendances`에 `cycle_day`(NULL 허용으로), `session_id`, `checked_at`(기본값 `now()`) 추가
3. 데이터 이전: `cycle_day = ((streak − 1) % 7) + 1`, `checked_at` = 그날 출석 원장 기록 시각(없으면 그 날짜의 한국 0시)
4. `cycle_day` NOT NULL, CHECK 1~7, FK → `attendance_rewards.day`, FK `session_id` → `sessions.id` SET NULL
5. `attendances_streak_check`와 `streak` 삭제
6. 인덱스 `attendances (session_id)`, 원장 부분 고유 인덱스 `point_ledger_attendance_uq`(R7)·`point_ledger_like_received_uq`(R25)

파일 맨 위에 요구사항 ID를 단 한국어 주석을 붙인다(`drizzle/0006_blog_visits.sql` 관례).

**Rationale**: drizzle-kit이 만드는 `ADD COLUMN … NOT NULL`은 행이 있는 표에서 실패한다. 한 파일 안에서 순서를 바꾸면 중간 상태가 없고 `drizzle/meta/` 스냅숏은 최종 스키마와 같아 다음 `db:generate`와 충돌하지 않는다. `checked_at`을 원장 시각으로 채우면 옛 출석도 실제 시각에 가깝게 남는다.

**주의**: 같은 표에서 `streak`이 사라지고 `cycle_day`가 생기므로 `npm run db:generate`가 "이름을 바꾼 것인지, 새로 만든 것인지"를 대화형으로 물을 수 있다. **새로 만들기**를 고른다. 이름 바꾸기를 고르면 `RENAME COLUMN`이 생겨 옛 연속 일수(8, 14 …)가 그대로 `cycle_day`가 되고 CHECK 1~7에서 실패한다.

**Alternatives considered**: 마이그레이션 3개(NULL 허용 추가 → `--custom` 데이터 이전 → NOT NULL)로 나누기(`schema.ts` 중간 상태를 두 번 만들어야 해서 리뷰가 더 어렵다), 기존 출석 기록 지우기(FR-031 위반).

**확인**: `drizzle-kit migrate`가 대기 중인 마이그레이션을 트랜잭션으로 적용하는지 구현 전에 설치된 drizzle-kit 문서로 확인. 트랜잭션이 아니어도 한 파일이 실패하면 그 뒤는 적용되지 않으므로 순서 조정으로 충분하다.

## R7. 출석 보상 지급과 "하루 한 번"

**Decision**: `ensureTodayAttendance()` 트랜잭션:

1. `lockUser(tx, userId)`
2. 오늘 이전 마지막 출석 1건(`date < today` 최신순 1건) → `nextCycleDay()`
3. `INSERT attendances (user_id, date, cycle_day, session_id, checked_at) … ON CONFLICT DO NOTHING RETURNING`
4. 들어가지 않았으면(이미 출석) 오늘 행을 읽어 돌려주고 끝
5. 들어갔으면 `attendance_rewards`에서 그 일차 보상을 읽어 `addLedgerEntry(tx, { reason: "attendance", expDelta, coinDelta, refId: today })` (레벨업 알림 포함, R8)

`grantReward()`는 쓰지 않는다. DB에는 출석 PK에 더해 원장 부분 고유 인덱스 `point_ledger_attendance_uq (user_id, ref_id) WHERE reason = 'attendance'`를 둔다.

**Rationale**:
- 출석 보상은 일차마다 다르므로 `REWARD_RULES` 고정값과 하루 상한 세기(`grantReward`)에 맞지 않는다. 하루 한 번은 출석 PK가, 보상 한 번은 같은 트랜잭션과 부분 고유 인덱스가 지킨다 (constitution V "DB 제약조건으로도", FR-023, SC-003).
- 보상 기록이 실패하면 트랜잭션 전체가 롤백되어 출석 행도 남지 않는다 (FR-024).
- 지금 `attend()`도 잠금 + PK + `ON CONFLICT DO NOTHING` 구조라 같은 패턴이다.
- **트랜잭션 안의 모든 조회·기록은 `tx`로 한다** (보상표 읽기, 마지막 출석, `addLedgerEntry`의 누적 경험치). `pg` 풀은 기본 최대 10개(`src/db/index.ts`에 `max` 설정 없음)라서, 10개 탭(SC-003)이 동시에 오면 10개 연결이 모두 `lockUser` 대기 트랜잭션에 묶일 수 있다. 이때 잠금을 쥔 트랜잭션이 `db`로 새 연결을 기다리면 아무도 끝나지 않는다.

**기존 데이터**: 옛 `attend()`가 `ref_id` = 날짜로 하루 한 번만 기록했으므로 중복은 없을 것으로 본다(추측). 마이그레이션 전에 중복 확인 쿼리를 돌린다 ([quickstart.md](quickstart.md) 1.3). `ref_id`가 NULL인 옛 행은 고유 인덱스에 걸리지 않는다.

**Alternatives considered**: `grantReward()`에 경험치·코인 인자를 더하기(상한 규칙과 섞인다), 원장만 쓰고 출석 표는 그대로(FR-025 일차·세션·시각 기록이 없다), 원장 부분 고유 인덱스 생략(앱 코드가 틀리면 DB가 못 막는다).

## R8. 레벨업 감지와 "정확히 한 번"

**Decision**: `src/server/points.ts`에 `addLedgerEntry(tx, entry)`(새로)를 만든다. 경험치가 0보다 크면 INSERT 전에 같은 `tx`로 그 회원의 누적 경험치(before, `getWallet(userId, tx)`처럼)를 구하고, INSERT 뒤 순수 함수 `levelsGained(before, before + exp)`(`src/lib/game.ts`, 오른 레벨 목록, 최고 99)마다 `notifications`에 `kind = 'level_up'`, `level = N` 행을 `ON CONFLICT DO NOTHING`으로 넣는다. `grantReward()`는 원장 INSERT를 이 함수로 바꾸고, 출석(R7)과 동물 다 키움(`src/server/farm.ts`의 `farm_grown` INSERT, town 소유)도 이 함수를 쓴다. DB에는 부분 고유 인덱스 `notifications_level_up_uq (user_id, level) WHERE kind = 'level_up'`.

**Rationale**:
- 원본 GAME-06 구현 제안("`grantReward()` 안에서 보상 전후 레벨을 비교")과 FR-037 "그 보상과 함께 정확히 한 번"을 따른다. 모든 호출이 `lockUser()` 트랜잭션 안이라 동시 보상도 차례로 처리된다.
- 경험치는 줄지 않는다 (`point_ledger_exp_check`: `exp_delta ≥ 0`, D6 회수 없음). 그래서 한 레벨은 많아야 한 번 도달하고, (`user_id`, `level`) 고유 인덱스가 규칙을 그대로 DB로 지킨다 (FR-047).
- 한 번에 두 레벨 이상 오르면 레벨마다 한 줄을 넣는다. 팝업은 가장 높은 것 하나만 보여 주고 함께 봤음 처리한다 (FR-041). 동물 다 키움 보상(최대 ✨160, `scripts/seed.ts`)으로 Lv.1에서 Lv.3이 될 수 있다.
- 경험치를 주는 원장 경로는 `grantReward()`(post, comment, like_received, farm_care), 출석, `farm_grown` 셋뿐이다. 구매·알 구매는 경험치 0이다. town이 만들 성장 아이템 사용도 `addGrowth()` → `farm_grown`을 거치므로 자동으로 포함된다.
- 기존 회원은 이미 오른 레벨에 알림이 없다. 다음에 새 레벨에 오를 때부터 생기므로 백필이 필요 없고, 배포 직후 옛 레벨 팝업이 한꺼번에 뜨지도 않는다.

**Alternatives considered**:
- DB 트리거: 레벨 공식(`50 × n × (n − 1)`, 최고 99)을 SQL에 한 번 더 둬야 한다.
- 화면을 그릴 때 늦게 감지(원장 레벨 > 알림 최고 레벨이면 만들기): FR-037 "그 보상과 함께"가 아니고, 배포 때 모든 회원의 현재 레벨을 백필해야 하며 알림 시각이 보상 시각과 다르다.
- "마지막으로 본 레벨" 컬럼: 원본 GAME-06이 "필요 없다"고 결정했고 `profiles`는 auth 담당이다.
- 가장 높은 레벨 한 줄만 저장: 건너뛴 레벨 기록이 없어 알림함과 고유 인덱스 규칙이 어색해진다.

## R9. 알림 표 구조

**Decision**: 표 `notifications` 하나와 열거형 `notification_kind`(`level_up`, `like`, `comment`, `reply`)를 **처음부터 네 값으로** 만든다. 컬럼은 `id`, `user_id`(받는 회원), `kind`, `actor_id`(행동한 회원, 레벨업은 NULL), `post_id`(관련 글, 레벨업은 NULL), `level`(레벨업만), `created_at`, `read_at`. `user_id`·`actor_id`·`post_id`는 모두 `ON DELETE CASCADE`. 문구·닉네임·글 제목은 저장하지 않고 보여 줄 때 JOIN한다. CHECK로 종류별 모양(레벨업은 `level`만, 나머지는 `actor_id`·`post_id`만)과 `actor_id <> user_id`를 막는다.

**Rationale**:
- FR-045 "종류를 늘려도 기존 알림이 그대로". 2단계에서 social은 표·열거형을 바꾸지 않고 행만 넣는다 (열거형 담당은 game, plan-context 3.5).
- FR-046 탈퇴: 받는 회원·행동한 회원 FK가 CASCADE라 auth의 탈퇴 처리가 회원 행을 지우면 알림도 함께 지워진다. 앱 코드가 필요 없다.
- 닉네임·제목을 복사하지 않아 3NF를 지키고, 닉네임·제목을 바꾸면 알림에도 바로 반영된다.
- 자기 활동 알림 금지(FR-045)를 서버 코드와 DB CHECK 둘 다로 막는다 (constitution V).
- `post_id` CASCADE: 글이 지워지면 그 글을 가리키는 알림은 이동할 곳도, 보여 줄 제목도 없다. spec에 정해진 바가 없어 이렇게 정했다.
- `comment_id`는 두지 않는다. "그 글의 댓글로" 이동은 글 주소 + `#comments`로 충분하다 (constitution VII). 특정 댓글로 이동이 필요해지면 컬럼을 더한다.

**Alternatives considered**: 종류별 표 4개(알림함 목록이 UNION), JSON 내용 컬럼(CHECK·FK 불가), 2단계에서 `ALTER TYPE … ADD VALUE`(social이 game 담당 열거형을 바꿔야 한다), 닉네임·제목 스냅숏 저장(중복, 바뀐 이름이 반영 안 됨).

## R10. "읽음"과 "봤음"

**Decision**: `read_at` 한 컬럼으로 둘 다 나타낸다. 팝업의 [확인]·[상점 가기]는 보여 준 레벨 이하의 안 본 레벨업 알림을 모두 `read_at = now()`로 바꾼다. 알림함에서도 읽음으로 보인다 (노란 배경 없음, 🔔 숫자에서 빠짐). 행은 지우지 않는다.

**Rationale**: Key Entities가 "읽은(봤음) 시각" 하나로 적었고, US5-3은 "알림함에는 기록이 남아 있다"만 요구한다.

**Alternatives considered**: `seen_at`(팝업)과 `read_at`(목록) 두 컬럼(spec이 구분하지 않는다).

## R11. 레벨업 팝업을 그리는 방법

**Decision**: 서버 컴포넌트 `LevelUpPopup`(`src/components/game/level-up-popup.tsx`)을 헤더(회원)에 끼워, 안 본 레벨업이 있으면 클라이언트 컴포넌트 `LevelUpDialog`(`level-up-dialog.tsx`)를 그린다. `LevelUpDialog`는 네이티브 `<dialog>`를 마운트 때 `showModal()`로 연다 (뒤 화면은 `::backdrop`으로 어둡게). [상점 가기]·[확인]은 Server Action `dismissLevelUp` 폼이다. Esc(`cancel` 이벤트)는 [확인]과 같게 처리한다.

**Rationale**:
- 헤더 `<header>`에 `backdrop-blur`(CSS `backdrop-filter`)가 있다. `filter`·`backdrop-filter`가 있는 조상은 그 안의 `position: fixed` 요소의 기준 상자가 되므로, 헤더 안에 고정 오버레이를 그리면 화면 전체가 아니라 헤더 크기로 잘린다. `showModal()`은 top layer에 그려 이 영향을 받지 않고, 포커스 가두기와 Esc를 브라우저가 처리한다.
- 보상을 주는 Server Action은 이미 `revalidatePath("/", "layout")`을 부른다 (`src/app/write/actions.ts`, `src/app/blog/actions.ts`, `src/app/farm/actions.ts`). 레이아웃이 다시 그려지면 새 레벨업이 그 자리에서 뜬다 (FR-038 "그 화면에서 바로"). 남의 공감으로 오른 경우는 다음 렌더에서 뜬다.
- 출석으로 오른 경우는 `getViewer()`의 출석이 헤더보다 먼저 끝나므로 같은 화면에서 뜬다 (원본 GAME-06 수용 기준 "출석 화면에서 바로").

**Alternatives considered**: 고정 `<div>` + `createPortal`(포커스·Esc를 직접 처리), `src/app/layout.tsx`에 따로 배치(town 소유 파일을 하나 더 고친다), 주소 쿼리로 팝업 표시(새로고침·공유 주소에서 다시 뜬다. 지금 `?new=reward`가 같은 문제).

**열린 것**: Esc 동작은 spec에 없다. [확인]과 같게 처리하지 않으면 Esc로 닫은 뒤 새로고침할 때 다시 뜬다. open item에 적었다.

## R12. 팝업에 보여 줄 아이템

**Decision**: 안 본 레벨업 중 가장 낮은 레벨 L1부터 가장 높은 레벨 L2 사이(포함)에 `required_level`이 있는 **판매 중 아이템(캐릭터 제외)**을 필요 레벨 → 가격 → ID 순으로 앞 3개 보여 주고, 나머지가 있으면 `외 N개`를 붙인다. 없으면 `이제 이런 친구를 데려올 수 있어요` 줄을 뺀다. "판매 중" 조건은 shop plan이 정한 `items.is_on_sale`(`specs/006-shop/data-model.md`: boolean 기본 true, 캐릭터·기본 아이템은 데이터 이전으로 false)을 써서 `is_on_sale = true AND type <> 'character'`로 하고, shop 마이그레이션이 들어오기 전에는 `type <> 'character' AND is_starter = false`로 대신한다. 그림은 `ItemArt`(`src/components/item-art.tsx`, shop 소유)를 그대로 쓴다.

**Rationale**: FR-039와 Assumptions "상점에서 실제로 파는 아이템(캐릭터 제외)". 한 레벨만 오르면 원본의 `items.required_level = N`과 같다. 레벨업이 쌓였을 때 건너뛴 레벨의 아이템도 놓치지 않는다. 지금 시드에서 Lv.3은 판다(캐릭터, 제외)·벚꽃길이다 (US5-1).

**Alternatives considered**: `required_level = 가장 높은 레벨`만(Lv.3 팝업을 못 본 채 Lv.4가 되면 Lv.3 아이템이 보이지 않는다).

## R13. 알림함 화면

**Decision**: 새 화면 `/notifications`(`src/app/notifications/page.tsx`, 서버, `requireMember()`). 최신순 20개씩, 페이지 번호는 `Pagination`·`parsePage`(`src/components/pagination.tsx`)를 그대로 쓴다. 각 알림은 버튼 하나짜리 `<form>` → Server Action `openNotification`(FormData `id`), 목록 위 [모두 읽음] → `markAllNotificationsRead`. 헤더 🔔은 이 화면으로 가는 링크다. 화면 제목은 spec의 용어 그대로 `🔔 알림함`.

**Rationale**:
- FR-043 "20개씩"은 페이지 넘김이 필요하고, 같은 모양의 내역 화면(`/wallet`)이 이미 있다. JS 없이도 동작하고 375px에서 드롭다운보다 단순하다.
- 읽음 처리를 Server Action으로 하면 Next.js가 Origin을 확인한다(다른 사이트 요청 거부, 보안 기준). GET 링크로 읽음 처리하면 다른 사이트의 링크로 이동시키는 것만으로(최상위 GET에는 `SameSite=Lax` 로그인 쿠키가 실린다) 읽음이 바뀐다.
- 인자를 FormData로 받아 zod·`parseId()`로 검증하는 것은 `addComment`(`src/app/blog/actions.ts`)와 같은 관례이고, E2E에서 요청 본문을 바꿔 조작 시험을 하기 쉽다 (`e2e/social.mjs` 방식).

**Alternatives considered**: 헤더 드롭다운(헤더는 town 소유, 페이지 넘김과 375px 공간 문제), 읽음 처리용 Route Handler(Origin 확인을 직접 해야 한다).

## R14. 알림 문구·시간·배지

**Decision**: 순수 함수를 새 파일 `src/lib/notifications.ts`에 둔다.

| 함수 | 규칙 |
|---|---|
| `unreadBadge(n)` | 0 → 표시 없음, 1~9 → `N`, 10 이상 → `9+` (FR-042) |
| `notificationTime(createdAt, now)` | 1분 미만 `방금`, 60분 미만 `N분 전`, 24시간 미만 `N시간 전`, 그 밖 `YYYY.MM.DD`(한국 시간) (FR-043) |
| `notificationText(n)` | FR-045 표의 문구 4가지 |
| `levelUpTitle(level)` | `Lv.N이 되었어요!`, 99면 `최고 레벨 Lv.99가 되었어요!` (FR-038) |
| `notificationHref(n)` | 레벨업 `/shop`, 공감 `/@{주소}/{글ID}`, 댓글·답글 `/@{주소}/{글ID}#comments` (FR-045) |

`scripts/test-notifications.ts`로 경계값을 시험한다.

**Rationale**: "하루가 지나면"을 24시간 경과로 읽었다. 경계값(59초, 59분, 23시간 59분, 24시간)을 DB 없이 시험할 수 있다 (plan-context 6.1). 날짜 형식이 `src/lib/format.ts`의 `YYYY. MM. DD.`(띄어쓰기)와 달라 따로 둔다.

**Alternatives considered**: `src/lib/format.ts`에 더하기(공통 파일로 "바꿀 일 없음"으로 분류됨), 목록 화면 안에 직접 쓰기(시험할 수 없다).

## R15. 헤더 쿼리 비용 (NF-07)

**Decision**: 헤더는 회원일 때 지금의 `getWallet()`에 더해 `getHeaderNotifications(userId)` **한 쿼리**만 더 부른다. 안 읽은 알림 수는 10개까지만 세고(`9+` 판단용), 안 본 레벨업의 최고·최저 레벨을 `FILTER`로 함께 읽는다. 팝업 아이템 쿼리는 안 본 레벨업이 있을 때만 돈다. 인덱스는 목록용 (`user_id`, `created_at` DESC, `id` DESC), 안 읽은 수용 부분 인덱스 (`user_id`) WHERE `read_at IS NULL`, 글 삭제 CASCADE용 (`post_id`) WHERE `post_id IS NOT NULL`.

**Rationale**: 헤더는 모든 화면 요청에서 돈다. 글 목록 1초(NF-07)를 지키려고 쿼리 수를 하나만 늘린다. `actor_id`는 탈퇴 때만 찾으므로 인덱스를 두지 않는다 (규모가 커지면 더한다).

**Alternatives considered**: 원본 제안 인덱스 (`user_id`, `read_at`, `created_at`) 하나(안 읽은 수에는 맞지만 읽음·안 읽음을 섞은 최신순 목록에는 맞지 않는다), 헤더에서 알림 목록까지 읽기(필요 없는 행을 매번 읽는다).

## R16. 출석 처리가 실패할 때

**Decision**: `getViewer()` 안의 `ensureTodayAttendance()` 호출은 try/catch로 감싸 실패하면 서버 로그만 남기고 `attendance: null`로 화면을 그린다. 다음 화면에서 다시 시도한다. 출석 체크 화면은 `attendance`가 null이면 `ensureTodayAttendance()`를 한 번 더 부르고, 그래도 실패하면 오류를 그대로 던져 기존 오류 화면(`src/app/error.tsx`)을 보여 준다. `session_id`는 `(SELECT id FROM sessions WHERE id = 현재 세션)` 서브쿼리로 넣어, 세션이 막 지워졌으면 FK 오류 대신 NULL이 된다.

**Rationale**: 출석 기록과 보상은 한 트랜잭션이라 실패하면 둘 다 남지 않는다 (FR-024). 출석 때문에 사이트의 모든 화면이 500이 되면 안 된다. 출석 화면은 오늘 일차를 보여 줘야 하므로 조용히 넘어가지 않는다. 새 문구를 만들지 않는다 (FR-048).

**Alternatives considered**: 그대로 던지기(어느 화면이든 출석 오류로 깨진다), 실패 문구 새로 만들기(spec에 문구가 없다).

## R17. 교착을 막는 호출 순서

**Decision**: "`requireMember()`·`getViewer()`는 `db.transaction()` 밖에서, Server Action·페이지 맨 앞에서 먼저 부른다"를 코드 저장소 `CLAUDE.md` 규칙에 더한다. 지금 코드는 모두 그렇게 쓴다.

**Rationale**: `ensureTodayAttendance()`는 자기 트랜잭션(다른 연결)에서 같은 회원의 advisory lock을 잡는다. 바깥 트랜잭션이 이미 `lockUser()`를 쥔 채 안에서 처음으로 `getViewer()`를 부르면 두 연결이 서로 기다린다. `cache`라서 보통은 요청 시작 때 이미 끝나 있지만 규칙으로 남긴다.

## R18. 가입 지급(US1)과 가입 직후 코인

**Decision**: 회원가입 트랜잭션(회원·프로필·블로그·고른 캐릭터 + 초원 지급·장착·"일상"·🪙100)은 auth가 단계 1에서 구현한다 (`src/app/(auth)/actions.ts`, auth 소유). game은 그 안에서 쓰는 `grantReward(tx, userId, "signup")`(✨0 · 🪙100, 평생 1회 = 하루 1회 + 가입 1번)과 기본 캐릭터 확인 규칙(`items.type = 'character' AND is_starter = true`, 지금 온보딩 코드와 같음)을 그대로 제공하고, 수용 시나리오 검증을 auth의 가입 E2E에 맡긴다. **가입한 날도 첫 화면에서 자동 출석 1일차(✨10 · 🪙10)가 붙는다.**

**Rationale**: FR-021은 "로그인한 회원이 그날 처음 사이트의 화면을 열면"이고 가입 날을 빼지 않는다. 가입 직후 광장(`/town`)을 그릴 때 `getViewer()`가 1일차 출석을 기록하므로 헤더는 🪙 110이 된다.

**충돌**: US1-3 "헤더를 보면 `🪙 100`이 보이고"와 SC-001 "코인 100을 가진 상태"는 이 동작과 맞지 않는다. auth spec US1 Independent Test("코인 100 지급을 확인")도 같다. 이 plan은 FR-021을 따르고 가입 코인은 원장 `🎉 가입 축하` +100 한 줄로 확인한다. spec 문구 수정이 필요하다 (open item).

**Alternatives considered**: 가입한 날은 출석하지 않기(FR-021 위반, 1일차가 다음 날로 밀린다), 가입 직후 첫 화면만 출석을 미루기(규칙이 하나 더 생기고 언제 출석할지 모호해진다).

## R19. 7일 연속 보너스 중단과 옛 기록

**Decision**: `REWARD_RULES`에서 `attendance`·`attendance_streak`를 빼고 `RewardReason` 타입에서도 뺀다. `ledger_reason` 열거형의 `attendance_streak` 값은 그대로 둔다. `/wallet`의 사유 이름 `🔥 연속 출석 보너스`도 그대로 둔다.

**Rationale**: FR-030(새로 주지 않음), US4-8(예전 기록은 그대로 보임). PostgreSQL 열거형 값은 쓰는 행이 있으면 지울 수 없고, 지울 이유도 없다.

## R20. 광장 출석 도장 문구

**Decision**: game은 `viewer.attendance.cycleDay`를 제공한다. 광장 데이터 `TownData.attendedToday: boolean`(`src/components/town/types.ts`)을 `attendanceDay: number | null`로 바꾸고 `src/server/town.ts`의 `hasAttendedToday()`를 대신하는 일, 게시판 문구를 바꾸는 일은 town 소유 파일이라 town에 요청한다.

**충돌**: GAME FR-028은 출석 도장 입구에 `오늘 N일차 ✅`, TOWN FR-023은 게시판 부제 `마을 소식 · 출석 체크 (오늘 완료 ✅)`와 안내 `출석 체크 (오늘 완료)`다 (TOWN Assumptions "원본 그대로 둔다"). 원본 GAME-04 변경 결정은 "광장 게시판 출석 입구 문구도 `오늘 N일차 ✅`로"라고 적었다. 제안: 부제 `마을 소식 · 출석 체크 (오늘 N일차 ✅)`, 출석 도장 안내 `출석 체크 (오늘 N일차 ✅)`. 팀 결정 전까지 open item.

자동 출석이라 회원에게 `(보상 받기 🎁)`는 출석 처리가 실패한 화면에서만 보인다.

## R21. 공감·댓글·답글 알림 연결 (2단계)

**Decision**: game이 `notifyActivity(tx, { recipientId, actorId, kind, postId })`(`src/server/notifications.ts`)를 1단계 때 함께 만든다. `recipientId === actorId`이면 아무것도 넣지 않는다. social이 단계 7에서 같은 트랜잭션 안에서 부른다: `toggleLike`(공감 행이 새로 들어갔을 때, 글 주인에게), `addComment`(남의 글, 글 주인에게), 답글 등록(원댓글 작성자에게). 알림함의 문구·링크·읽음 처리는 game이 1단계 때 네 종류 모두 만들어 둔다.

**Rationale**: D11 "알림함은 GAME-08, social은 발생 조건". 같은 트랜잭션이라 공감·댓글이 롤백되면 알림도 남지 않는다.

**열린 것**: SOC FR-033은 "공감이 **새로 저장되면**" 알림을 남긴다. 그대로 하면 취소했다가 다시 누를 때마다 같은 사람의 공감 알림이 또 생긴다 (GAME Assumptions "공감 취소 시 알림을 지우지 않는다"). 공감 보상(같은 사람·같은 글 한 번)처럼 알림도 한 번만 남기려면 부분 고유 인덱스 (`user_id`, `actor_id`, `post_id`) WHERE `kind = 'like'`를 더하면 된다. 이 plan은 spec 문자 그대로(새로 저장될 때마다)로 설계했고, social plan(`specs/004-social/contracts/notification-triggers.md` 3장)도 "공감 → 취소 → 공감을 반복하면 공감 알림도 반복된다"로 같다. 바꾸려면 spec 수정이 먼저다 (확인 권장 항목).

## R22. 375px 배치

**Decision**:
- 7칸 보상표: 항상 7열(`grid-cols-7`), 칸마다 `N일차` / `✨ N` / `🪙 N` 세 줄, 휴대폰은 작은 글씨. 오늘 일차는 노란 테두리와 굵은 글씨로 강조.
- 달력: 지금 7열 정사각형 칸 그대로. 지금은 출석한 칸이 초록 테두리라 오늘(이미 출석) 노란 테두리가 보이지 않으므로, 오늘 칸은 초록 바탕 + 🌟 + 노란 테두리로 그린다 (FR-027).
- 헤더 🔔: 아이콘 + 빨간 숫자 배지(글자 하나 폭). 헤더 배치는 town이 유저 상태창(TOWN-10)으로 바꾸므로, 375px 공간은 town과 함께 `e2e/mobile.mjs`·`e2e/nonfunctional.mjs`로 확인한다.
- 팝업 버튼과 알림 줄은 높이 44px 이상 (constitution VI).

**Rationale**: SC-008(375px 가로 스크롤 없음), FR-010(헤더 레벨 숨기지 않음).

## R23. 테스트 전략

**Decision**:
- 단위(`npm test`): `scripts/test-game.ts`의 `currentStreak` 시험을 `nextCycleDay` 시험(SC-005 예시 전체, 7일차 → 1, 하루 빠짐 → 1, 9/30 → 10/1, 12/31 → 1/1)으로 바꾸고, `levelsGained`(99 → 100 = [2], 290 → 300 = [3], 90 → 340 = [2, 3], 485,090 → 485,100 = [99], 600,000 → 600,010 = [])와 `todayKST` 0시 경계(UTC 14:59 → 그날, 15:01 → 다음 날)를 더한다. 새 `scripts/test-notifications.ts`(R14)를 `test:notifications`로 `test` 체인 끝에 붙인다.
- E2E(새로): `e2e/attendance.mjs`, `e2e/notifications.mjs`, `e2e/rewards.mjs`. 날짜는 서버 시계를 바꿀 수 없으므로 `pg`로 어제·그저께 출석을 직접 넣고 오늘 행을 지워 준비한다 (`e2e/decisions.mjs`의 `kstDate()` 방식).
- E2E(고침): `e2e/game.mjs`의 출석 버튼 부분 삭제(상점·꾸미기 부분은 shop), `e2e/decisions.mjs`의 GAME-04 블록(`streak` 직접 INSERT, 버튼, 끊김 문구) 삭제, GAME-07의 "연속 출석 보너스" 확인은 옛 기록을 직접 넣어 확인.

**Rationale**: plan-context 6장의 관례(`check()`, 실행마다 새 회원, `.env.local`에서 비밀값 읽기, ❌면 `exit(1)`).

## R24. 구현 전에 설치된 패키지 문서로 확인할 것

| 패키지 | 확인할 동작 | 이 plan이 기대는 곳 |
|---|---|---|
| `better-auth` 1.7 | `auth.api.getSession()` 결과의 `session.id`(세션 행 ID). 지금 코드는 `session.user.id`만 쓴다 | R1, R4 (`attendances.session_id`) |
| `next` 16.3 | Server Component 렌더 중 DB 쓰기에 제약이 없는지, 쿠키 설정이 렌더에서 막히는지 | R1 |
| `next` 16.3 | `router.refresh()`가 루트 레이아웃을 다시 그리는지 | R2 |
| `next` 16.3 | `<Link>` prefetch가 동적 화면을 서버에서 렌더하는지 (렌더해도 같은 회원·같은 날 한 번이라 결과는 같다) | R1 |
| `next` 16.3 | Server Action 뒤 `revalidatePath("/", "layout")`이 레이아웃의 클라이언트 컴포넌트 props를 갱신하는지 (지금 헤더 코인이 이렇게 갱신된다) | R11 |
| `drizzle-orm` 0.45 | `pgEnum` 추가, `uniqueIndex().on().where()`(이미 `0005_animal_farm.sql`에서 씀), 대상 없는 `.onConflictDoNothing()` → `ON CONFLICT DO NOTHING`(부분 고유 인덱스 위반도 막는 PostgreSQL 동작) | R7, R8 |
| `drizzle-kit` 0.31 | `migrate`가 트랜잭션으로 적용하는지, 생성 SQL을 고쳐도 스냅숏과 어긋나지 않는지, `generate`가 `streak` 삭제 + `cycle_day` 추가를 이름 바꾸기로 묻는지(물으면 "새로 만들기") | R6 |
| `next` 16.3 | Server Action의 `redirect("/@주소/글ID#comments")`가 해시 위치로 스크롤하는지 (안 되면 댓글 영역 쪽에서 해시를 보고 스크롤) | R13, contracts/notifications 4.1 |
| `next` 16.3 | React `cache`가 Server Action·Route Handler에서도 요청 단위로 묶이는지 (묶이지 않아도 `getViewer()`를 한 번만 부르면 결과는 같다) | R1 |
| `react` 19.2 | 클라이언트 컴포넌트에서 `<dialog>` ref로 `showModal()`·`cancel` 이벤트 처리 | R11 |

## R25. 공감 보상 "같은 사람·같은 글 한 번"을 DB로도 막기

**Decision**: M1 마이그레이션에서 `point_ledger`에 부분 고유 인덱스 `point_ledger_like_received_uq` UNIQUE (`user_id`, `ref_id`) WHERE `reason = 'like_received'`를 더한다. `ref_id`는 지금 코드 그대로 `"{글ID}:{공감한 회원ID}"`다. 앱 코드는 바꾸지 않는다 (`toggleLike`가 `lockUser(글 주인)` 안에서 같은 `ref_id`를 찾아 없을 때만 `grantReward`).

**Rationale**:
- FR-047은 "같은 사람·같은 글 공감 보상 한 번"을 "레벨업 알림 한 번"과 함께 서버·동시 요청 규칙으로 적었고, constitution V는 이런 "한 번만" 규칙을 앱 코드뿐 아니라 DB 제약조건으로도 막으라고 한다. 지금은 앱 코드(`src/app/blog/actions.ts`의 `ref_id` 확인)로만 막는다.
- `point_ledger`는 game 담당 표라 game이 더한다 (plan-context 3.1). 하루 상한(20)에 걸려 원장 줄이 없었던 쌍은 나중에 한 번 받을 수 있어 FR-017과 맞는다.
- 지금 코드는 잠금 안에서 확인하므로 중복 행이 없을 것으로 본다(추측). 마이그레이션 전에 중복 확인 쿼리를 돌린다 ([quickstart.md](quickstart.md) 1.3).

**Alternatives considered**: 앱 코드만 유지(constitution V와 다름), 가입 축하 `signup` 평생 1회 인덱스도 함께 더하기(가입 트랜잭션이 회원마다 한 번만 돌아 구조로 지켜지고, `e2e/decisions.mjs`가 경험치를 맞추려고 `signup` 줄을 하나 더 넣어 개발 DB에 중복이 있을 수 있어 이번에는 더하지 않는다).
