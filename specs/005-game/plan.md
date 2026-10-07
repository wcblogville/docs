# Implementation Plan: 캐릭터 / 성장 (GAME)

**Branch**: `005-game` | **Date**: 2026-10-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/005-game/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

GAME은 캐릭터 지급, 경험치·코인 원장, 레벨, 출석, 경험치·코인 내역, 레벨업 팝업, 알림함을 다룬다. 지금 코드(`main` `feb4c05`)에는 원장·레벨 공식·하루 상한·내역 화면이 이미 있고(GAME-02·03·05·07), 2026-10-07 결정 중 **출석 자동화·1~7일차 주기(GAME-04)**, **레벨업 팝업(GAME-06)**, **알림함(GAME-08)**이 없다. 가입 때 캐릭터·코인 지급(GAME-01)은 auth의 가입 통합 작업 안에서 구현된다.

접근 방식:

- **자동 출석**: 버튼 Server Action(`attend`)을 없애고, 요청마다 한 번 도는 `getViewer()`(`src/server/dal.ts`)가 오늘 출석이 없을 때 `ensureTodayAttendance()`를 부른다. 회원 잠금 트랜잭션에서 마지막 출석으로 일차(1~7)를 정하고, 새 표 `attendance_rewards`의 값으로 원장에 기록한다. 헤더와 페이지가 지갑을 읽기 전에 끝나므로 첫 화면에 바로 반영된다 (research R1).
- **출석 데이터**: `attendances.streak` → `cycle_day`, `session_id`, `checked_at` (ERD 3.12), 기존 행은 `((streak − 1) % 7) + 1`로 이전. 보상 1회는 출석 PK와 원장 부분 고유 인덱스로 DB가 지킨다.
- **레벨업**: 원장 기록을 새 함수 `addLedgerEntry()`로 모아, 같은 트랜잭션에서 경험치 전후 레벨을 비교해 `notifications`에 레벨마다 한 줄(고유 인덱스로 한 번)을 넣는다. 헤더의 서버 컴포넌트가 안 본 레벨업을 읽어 네이티브 `<dialog>` 팝업으로 보여 준다.
- **알림함**: 새 표 `notifications`(종류 4가지를 처음부터), 헤더 🔔 + 안 읽은 수, 새 화면 `/notifications`, 읽음 처리 Server Action 3개. 공감·댓글·답글 알림(2단계)은 social이 `notifyActivity()`를 불러 붙인다.

## Technical Context

**Language/Version**: TypeScript ^5 (`strict: true`, 경로 별칭 `@/*` → `src/*`), Node.js 20.9 이상

**Primary Dependencies**: Next.js 16.3.8 (App Router, Server Component, Server Action), React 19.2.8, Drizzle ORM ^0.45.3 / drizzle-kit ^0.31.11, Better Auth ^1.7.7 (`getSession()`의 세션 ID를 출석에 씀), zod ^4.6.5 (Action 입력 검증), Tailwind CSS ^4 (`phone:` 변형). 새 의존성 없음

**Storage**: PostgreSQL (README 안내 17, 15 이상), `pg` ^8.23.1 (연결 풀 기본 최대 10개, `src/db/index.ts`). 담당 표: `attendances`(변경), `attendance_rewards`(새로), `point_ledger`(부분 고유 인덱스 2개 추가: 출석 보상 한 번, 공감 보상 한 번), `notifications`(새로), 열거형 `notification_kind`(새로)

**Testing**: `npm test` = `tsx` 단위 시험 스크립트(`scripts/test-game.ts` 고침, `scripts/test-notifications.ts` 새로), E2E = Playwright `chromium`을 쓰는 Node 스크립트(`e2e/attendance.mjs`, `e2e/rewards.mjs`, `e2e/notifications.mjs` 새로, `e2e/game.mjs`·`e2e/decisions.mjs` 고침). CI 없음, PR 작성자가 직접 실행

**Target Platform**: 웹 브라우저 (PC, 375px 휴대폰). 서버는 Node.js에서 `next start`

**Project Type**: 웹 애플리케이션 (한 Next.js 프로젝트에 화면·서버가 함께)

**Performance Goals**: 글 목록 1초(NF-07)를 유지하도록 헤더 쿼리는 하나만 늘린다 (안 읽은 알림 수 + 안 본 레벨업을 한 쿼리, research R15). 출석 쓰기는 회원당 하루 한 번, 그 밖의 요청은 기존 프로필 쿼리에 JOIN 하나. 헤더 `🪙`에서 내역까지 3초(SC-010)

**Constraints**: 하루 기준은 한국 시간(`todayKST()`, `startOfTodayKST`). 잔액·레벨 컬럼 금지, 원장 합계로 계산(NF-16). 보상·출석·알림은 `lockUser()` 트랜잭션 안. Server Component는 쿠키를 쓸 수 없다. 루트 레이아웃은 화면 이동만으로 다시 그려지지 않는다. 화면 문구는 spec 그대로. `node_modules`가 없어 라이브러리 세부 동작은 구현 전 확인 (research R24)

**Scale/Scope**: 3명 팀 학습 프로젝트, 회원 규모는 정해진 바 없음(수백~수천 명으로 가정, 추정). 화면: 고침 1(`/attendance`), 새로 1(`/notifications`), 헤더 요소 3(🔔, 레벨업 팝업, 날짜 감시), 확인만 3(`/wallet`, `/closet`, `/shop`). 마이그레이션 2개

Technical Context에서 모르던 것(자동 출석을 일으킬 곳, 보상표를 둘 곳, 마이그레이션 순서, 레벨업 감지 위치, 팝업 그리기, 알림함 화면 형태)은 [research.md](research.md) R1~R15에서 모두 정했다. 남은 것은 spec 사이의 문구 충돌과 팀 확인 사항이다 (아래 "남은 문제").

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` v1.0.0 기준.

| 원칙 | 설계 전 | 근거 | 설계 후 재평가 |
|---|---|---|---|
| I. 글쓰기가 먼저 | 통과 | 출석·레벨·알림은 매일 들어와 글을 쓰게 돕는 보상이다. 출석이 버튼 없이 되어 글쓰기 흐름을 막지 않는다. 팝업은 레벨이 오를 때만, 공감·댓글 알림은 팝업 없이 숫자만 (FR-045). 코인은 활동으로만 얻는다 | 통과. 팝업은 [확인] 한 번으로 닫히고 다시 뜨지 않는다 |
| II. 요구사항 ID 유지 | 통과 | GAME-01~08 ID를 그대로 쓰고 FR마다 원본 ID를 단다. GAME-09는 보류로 남기고 지우지 않는다. 원본 GAME-04 본문의 `id` PK 문단은 "2026-10-07 설계 변경" 표가 우선한다 (research R4) | 통과. 마이그레이션 주석·코드 주석에 GAME-04·06·08을 단다 |
| III. 확인할 수 있는 수용 기준 | 통과 (팀 확인 필요: 남은 문제 1·2·4·5·8) | 모든 수용 시나리오와 SC를 [quickstart.md](quickstart.md)의 명령·기대 결과에, 모든 FR을 아래 "요구사항 대응" 표에 연결했다. spec끼리 맞지 않는 문구(US1-3 `🪙 100`, 출석 도장 문구)와 spec에 없는 문구(알림함 제목, 🔔 접근 이름, Esc)는 지어내지 않고 남은 문제로 올렸다 | 통과. 문구는 spec 표 그대로 ([contracts/](contracts/)) |
| IV. 권한과 검증은 서버에서 (NON-NEGOTIABLE) | 통과 | 출석은 서버의 `getViewer()` 안에서만 일어난다 (화면 렌더, 그리고 `requireMember()`·`getViewer()`로 시작하는 Server Action과 `POST /api/uploads`). 출석만 따로 요청하는 Action·주소는 없고, 어느 경로든 같은 잠금·PK·고유 인덱스를 거친다 (research R1). 일차·보상·레벨업은 서버가 계산한다. 알림 Action은 `requireMember()` + `user_id = 나` 조건, 입력은 zod·`parseId()`·레벨 2~99로 검증. Server Action이라 Origin 확인. 쿼리는 Drizzle 값 바인딩 | 통과. 남의 알림 ID·범위 밖 값 조작 시험을 E2E에 넣었다 |
| V. 원장 무결성 (NON-NEGOTIABLE) | 통과 | 잔액·레벨 컬럼 없음. 출석 기록 + 보상 + 레벨업 알림을 한 트랜잭션(`lockUser`)으로. "하루 한 번 출석"은 PK, "출석 보상 한 번"은 원장 부분 고유 인덱스, "같은 사람·같은 글 공감 보상 한 번"(FR-047, 지금은 앱 코드로만 막음)은 원장 부분 고유 인덱스를 새로 더해 DB로도 막는다 (research R25), "레벨업 알림 한 번"은 부분 고유 인덱스, 자기 알림 금지·알림 모양은 CHECK. 하루 상한(횟수)은 제약으로 표현할 수 없어 지금처럼 `lockUser` 잠금으로 지킨다. 구조 변경은 마이그레이션 파일로만 | 통과. 회수 없음(`exp_delta ≥ 0`)이 "레벨업 한 번" 규칙의 전제임을 data-model에 적었다 |
| VI. 모바일에서도 | 통과 | 출석 7칸 표·달력·알림함·팝업을 375px에서 가로 스크롤 없이, 버튼 44px 이상. 헤더 🔔은 town의 헤더 개편과 함께 375px 확인 | 통과 (헤더 공간은 town과 함께 `e2e/mobile.mjs`·`e2e/nonfunctional.mjs`로 확인) |
| VII. 단순하게, 최소 정보 | 통과 | 새 의존성 없음. 알림은 ID만 저장하고 닉네임·제목은 JOIN(복사 없음). 표 하나에 종류 열거형. 기존 패턴(잠금, `ON CONFLICT DO NOTHING`, `Pagination`, `revalidatePath`) 재사용. 새 개인정보 없음 | 통과. 클라이언트 컴포넌트는 팝업과 날짜 감시 2개뿐 |

**결과**: 위반 없음. Complexity Tracking은 비워 둔다.

## Project Structure

### Documentation (this feature)

```text
specs/005-game/
├── spec.md                  # 기능 명세 (고치지 않음)
├── checklists/
│   └── requirements.md      # spec 품질 점검
├── plan.md                  # 이 파일
├── research.md              # Phase 0: 결정 R1~R25
├── data-model.md            # Phase 1: 표 현재/목표, 상태 전이, 마이그레이션
├── quickstart.md            # Phase 1: 검증 순서와 기대 결과
└── contracts/               # Phase 1: 바깥 인터페이스
    ├── attendance.md        # 자동 출석, /attendance, 광장에 넘기는 값
    ├── notifications.md     # 헤더 🔔·레벨업 팝업, /notifications, Server Action 3개, notifyActivity
    └── rewards-ledger.md    # points.ts·game.ts 모듈 약속, /wallet, 가입 지급
```

`tasks.md`는 `/speckit-tasks`가 만든다.

### Source Code (repository root)

코드 저장소 기준. `(소유: X)`는 다른 spec이 소유한 파일이라 "추가"만 하거나 그 spec에 요청하는 곳이다 (plan-context 5.2).

```text
src/
├── app/
│   ├── attendance/
│   │   ├── page.tsx                  # 고침: 버튼 없이 결과·7칸 보상표·달력 (FR-026·027)
│   │   ├── actions.ts                # 삭제: attend() 버튼 Action
│   │   └── attend-button.tsx         # 삭제
│   ├── notifications/                # 새로 (GAME-08)
│   │   ├── page.tsx                  # 알림함 목록, [모두 읽음]
│   │   └── actions.ts                # openNotification, markAllNotificationsRead, dismissLevelUp
│   ├── wallet/page.tsx               # 확인만 (GAME-07 이미 구현)
│   ├── closet/page.tsx               # 확인만 (소유: shop, 레벨 막대 FR-011)
│   └── shop/page.tsx                 # 확인만 (소유: shop, Lv·코인 FR-013)
├── components/
│   ├── site-header.tsx               # 추가 (소유: town): 🔔, LevelUpPopup, AttendanceDayWatcher
│   └── game/                         # 새로
│       ├── notification-bell.tsx     # 🔔 + 안 읽은 수 배지
│       ├── level-up-popup.tsx        # 서버: 안 본 레벨업·아이템 조회
│       ├── level-up-dialog.tsx       # 클라이언트: <dialog> showModal, 버튼 폼
│       └── attendance-day-watcher.tsx # 클라이언트: 날짜가 바뀌면 router.refresh()
├── db/
│   └── schema.ts                     # attendanceRewards, attendances 변경, pointLedger 인덱스, notificationKind, notifications
├── lib/
│   ├── game.ts                       # nextCycleDay, levelsGained 추가 / currentStreak, ATTENDANCE_STREAK_BONUS_EVERY, REWARD_RULES의 출석 2개 삭제
│   └── notifications.ts              # 새로: 배지, 시간 표시, 문구, 팝업 제목, 이동 주소
└── server/
    ├── points.ts                     # addLedgerEntry 추가, grantReward가 사용
    ├── attendance.ts                 # 새로: ensureTodayAttendance, 이번 달 출석, 보상표
    ├── notifications.ts              # 새로: getHeaderNotifications, getPendingLevelUp, listNotifications, notifyActivity
    ├── dal.ts                        # 추가 (소유: auth): getViewer에 오늘 출석 JOIN + ensureTodayAttendance 호출, attendance 필드
    ├── farm.ts                       # 요청 (소유: town): addGrowth의 farm_grown 원장 INSERT → addLedgerEntry (한 줄이지만 기존 함수 동작을 바꾸므로 town 승인)
    └── town.ts                       # 요청 (소유: town): hasAttendedToday → 일차 값 사용
drizzle/
├── NNNN_<이름>.sql                   # 새로: 출석 일차·보상표 (생성 후 순서 손질, data-model 5.1)
├── NNNN_<이름>.sql                   # 새로: 알림 (data-model 5.2)
└── meta/                             # 스냅숏 (생성)
scripts/
├── test-game.ts                      # 고침: currentStreak 시험 → nextCycleDay, levelsGained, todayKST 경계
└── test-notifications.ts             # 새로
e2e/
├── attendance.mjs                    # 새로
├── rewards.mjs                       # 새로
├── notifications.mjs                 # 새로
├── game.mjs                          # 고침: 출석 버튼 부분 삭제
├── decisions.mjs                     # 고침: GAME-04 블록 삭제, GAME-07 옛 보너스 줄
├── params.mjs                        # 추가: /notifications?page= 한 줄
└── nonfunctional.mjs                 # 추가: 375px 확인 대상 pages 배열에 /notifications 한 줄
docs/02-erd.md                        # 출석·보상표·원장 인덱스·알림 절 (data-model 5.4)
package.json                          # test:notifications 추가, test 체인 끝에
CLAUDE.md, README.md                  # 출석·레벨업·호출 순서 규칙, E2E 표
```

**Structure Decision**: 한 Next.js 프로젝트의 기존 구조(`src/app` 화면·Action, `src/server` 서버 전용 쿼리, `src/lib` 순수 규칙, `src/components` 화면 조각)를 그대로 쓴다. game 전용 화면 조각은 새 폴더 `src/components/game/`에, 서버 처리는 영역 파일 `src/server/attendance.ts`·`notifications.ts`(둘 다 새로)에 둔다. 다른 spec 소유 파일 중 `dal.ts`(auth)·`site-header.tsx`(town)는 plan-context 5.2가 허용한 "추가"로, `farm.ts`·`town.ts`(town)는 기존 동작이 바뀌므로 "요청"으로 다룬다.

## 변경 단위 요약 (현재 코드 → spec 목표)

| # | 종류 | 대상 | 지금 | 목표 | 요구사항 | 소유 / 선행 |
|---|---|---|---|---|---|---|
| M1 | 마이그레이션 | 출석 일차·보상표 + 원장 고유 인덱스 | `attendances(streak)`, 보상 고정 ✨10·🪙20 + 7일 🪙50, 공감 보상 한 번은 앱 코드로만 | `attendance_rewards` 7행, `cycle_day`·`session_id`·`checked_at`, `streak` 삭제, 이전 `((streak−1)%7)+1`, `point_ledger_attendance_uq`, `point_ledger_like_received_uq` | FR-017·020·023·025·031·047 | game |
| M2 | 마이그레이션 | 알림 | 없음 | `notification_kind`, `notifications` + CHECK 3 + 인덱스 4 | FR-037·042~046 | game |
| S1 | 순수 규칙 | `src/lib/game.ts` | `currentStreak`, `ATTENDANCE_STREAK_BONUS_EVERY`, `REWARD_RULES.attendance*` | `nextCycleDay`, `levelsGained`, 출석 규칙 삭제 | FR-022·030·037 | game |
| S2 | 순수 규칙 | `src/lib/notifications.ts` (새로) | 없음 | 배지·시간·문구·제목·주소 | FR-038·042·043·045 | game |
| S3 | 서버 함수 | `src/server/points.ts` | `grantReward`가 원장 직접 INSERT | `addLedgerEntry`(레벨업 알림 포함)를 `grantReward`가 사용 | FR-007·012·037·047 | game |
| S4 | 서버 함수 | `src/server/attendance.ts` (새로) | `attend()` 버튼 Action | `ensureTodayAttendance`, 이번 달 출석, 보상표 | FR-021~025 | game |
| S5 | 서버 함수 | `src/server/notifications.ts` (새로) | 없음 | 헤더 집계, 안 본 레벨업·아이템, 목록, `notifyActivity` | FR-038~046 | game |
| S6 | 서버 함수 | `src/server/dal.ts` `getViewer` | 세션·프로필만 | + 오늘 출석 JOIN, 없으면 `ensureTodayAttendance`, `attendance` 필드 | FR-021, SC-004 | auth 소유 → 공통 모듈 추가 (auth 합의) |
| S7 | 서버 함수 | `src/server/farm.ts` `addGrowth` | `farm_grown` 직접 INSERT | `addLedgerEntry` | FR-037 | town 소유 → 요청 (기존 함수 동작이 바뀜. town이 바꾸거나 game이 바꾸는 것을 town이 승인) |
| A1 | Server Action | `src/app/notifications/actions.ts` (새로) | 없음 | `openNotification`, `markAllNotificationsRead`, `dismissLevelUp` | FR-040·044 | game |
| A2 | Server Action | `src/app/attendance/actions.ts` | `attend()` | 삭제 | FR-021 | game |
| U1 | 화면 | `/attendance` | 버튼, 연속 일수, 끊김 문구, 고정 보상 안내 | 결과, 7칸 보상표, 달력(오늘 노란 테두리), 방문자 → `/` | FR-026·027·029 | game |
| U2 | 화면 | `/notifications` (새로) | 없음 | 알림함 20개씩, 노란 배경, 시간, [모두 읽음], 빈 문구 | FR-042~045 | game |
| U3 | 화면 조각 | 헤더 | `Lv.N`, `🪙 N` | + 🔔 배지, 레벨업 팝업(`<dialog>`), 날짜 감시 | FR-038~042 | town 소유 헤더에 추가 |
| U4 | 화면 | 광장 게시판 출석 도장 | `(오늘 완료 ✅)` / `(보상 받기 🎁)` | 오늘 일차 표시 (문구 확정 필요) | FR-028 | town (요청) |
| U5 | 화면 | `/wallet`, `/closet`, `/shop`, 글쓰기 안내 | 구현됨 | 유지 확인 | FR-011·013·019·032~036 | game·shop·post |
| U6 | 화면·Action | 회원가입 캐릭터·가입 지급 | 온보딩(`src/app/onboarding/*`)에서 지급 | 가입 트랜잭션에서 지급, 캐릭터 카드 2개 | FR-001~003 | auth 단계 1 |
| V1 | 단위 시험 | `scripts/test-game.ts`, `scripts/test-notifications.ts`, `package.json` | `currentStreak` 시험 | 일차·레벨 목록·0시 경계, 알림 순수 함수 | SC-005 등 | game |
| V2 | E2E | `e2e/attendance.mjs`, `e2e/rewards.mjs`, `e2e/notifications.mjs` (새로) | `e2e/game.mjs` 버튼 동시 클릭 | 자동 출석·동시 10탭·롤백·보상 규칙·팝업·알림함·조작 | SC-001~012 | game |
| V3 | E2E | `e2e/game.mjs`, `e2e/decisions.mjs`, `e2e/params.mjs` | 버튼·`streak` INSERT·보너스 문구 | 출석 부분 삭제·교체, 페이지 인자 한 줄 | - | game (상점 부분은 shop) |
| D1 | 문서 | `docs/02-erd.md`, `CLAUDE.md`, `README.md` | 출석 ⏳, 알림 없음 | data-model 5.4, 규칙(R17), E2E 표 | - | game |

### 요구사항 대응 (spec FR → 변경 단위·산출물)

"유지"는 지금 코드가 이미 spec대로라 바꾸지 않고 quickstart로 확인만 하는 것이다.

| FR | 대응 | 검증 |
|---|---|---|
| FR-001·002·003 | U6 (auth 단계 1이 구현), [contracts/rewards-ledger.md](contracts/rewards-ledger.md) 1장·3.3 (가입 트랜잭션에서 `grantReward("signup")`, 기본 캐릭터 서버 확인) | quickstart US1-1~4 |
| FR-004 | 유지: `src/app/shop/page.tsx`가 `is_starter` 아이템을 판매 목록에서 빼고 `buyItem`(`src/app/shop/actions.ts`)이 거부한다. 판매 여부 컬럼은 shop (`items.is_on_sale`) | quickstart US1-6 (shop E2E) |
| FR-005 | 유지: 기존 `user_items` 행을 바꾸는 이전 없음 (data-model 1장 `user_items` 참조) | quickstart US1-5 |
| FR-006 | 유지: 광장·헤더·미니룸이 `profiles.character_item_id`를 그린다 (town·blog·shop 화면) | rewards-ledger 3.2 |
| FR-007·012 | S3 (`addLedgerEntry`가 원장 한 줄), 기존 `getWallet()` 합계 | quickstart SC-002 |
| FR-008·009 | 유지: `src/lib/game.ts` `expForLevel`·`levelFromExp`·`MAX_LEVEL` (S1이 바꾸지 않음) | `npm run test:game` 기존 시험 |
| FR-010 | U3 (town 헤더 개편 때 `Lv.N` 유지, 의존성) | quickstart US2-13 |
| FR-011·019 | U5 유지 (`src/app/closet/page.tsx`, `src/components/editor/post-form.tsx`) | quickstart US2-4·9 |
| FR-013 | U5 유지 + 보상·구매 Action의 `revalidatePath("/", "layout")` (rewards-ledger 1장) | quickstart SC-007 |
| FR-014 | 유지: `buyItem`·`buyEgg`가 `lockUser` 안에서 잔액 확인 (shop·town) | quickstart US4-3 |
| FR-015·016·017·018 | S1 (`REWARD_RULES`에서 출석 2개만 뺌), S3 (`grantReward` 상한 그대로), M1 (`point_ledger_like_received_uq`), 회수 코드 없음 | quickstart US2-1~7·11·12·14 |
| FR-020·031 | M1 (보상표 7행, `cycle_day` 이전) | quickstart 1.3 |
| FR-021·023·024·025 | S4·S6·A2 (자동 출석, 잠금·PK·고유 인덱스, 한 트랜잭션, `session_id`·`checked_at`·`ref_id` = 날짜) | quickstart US3-1·6·7, FR-025 줄 |
| FR-022·030 | S1 (`nextCycleDay`, 보너스 규칙 삭제) | quickstart US3-2~5·11, FR-030 줄 |
| FR-026·027·029 | U1 | quickstart US3-8·10·12 |
| FR-028 | U4 (town 요청, 문구는 남은 문제 2) | quickstart US3-9 |
| FR-032~036 | U5 유지 (`/wallet`) | quickstart US4 |
| FR-037 | M2, S1 (`levelsGained`), S3, S4, S7 | quickstart US5-4, FR-037 동시 |
| FR-038~041 | U3, S5, A1 (`dismissLevelUp`) | quickstart US5 |
| FR-042~044 | U2, U3, A1 | quickstart US6-1~6 |
| FR-045 | M2 (열거형 4종), S2 (문구), S5 (`notifyActivity`), social 단계 7 | quickstart US6 표시 미리 확인, US6-7~10 |
| FR-046 | M2 (`user_id`·`actor_id` CASCADE), auth 단계 9 | quickstart FR-046 줄 |
| FR-047 | M1·M2 고유 인덱스, S3·S4 잠금 | data-model 4장 |
| FR-048 | contracts의 문구 표 (spec 문구 그대로), spec에 없는 문구는 남은 문제 4 | contracts |

### game 안의 구현 순서 (plan-context 5.1 단계 6)

1. **출석**: M1 → S1(일차) → S3(`addLedgerEntry`, 레벨업 감지는 아직 빈 동작이어도 됨) → S4 → S6 → U1 → A2 삭제 → V1 일부 → `e2e/attendance.mjs`, V3
2. **레벨업·알림함 1단계**: M2 → S1(`levelsGained`) → S2 → S3 레벨업 알림 → S7 → S5 → A1 → U2 → U3 → `e2e/notifications.mjs`, `e2e/rewards.mjs`
3. **알림함 2단계**: game은 2단계 문구·링크를 1단계에서 이미 만든다. social이 단계 7에서 `notifyActivity()` 호출을 넣는다.

## 의존성

### 이 plan이 먼저 필요로 하는 일 (선행)

| spec | 필요한 일 | 왜 |
|---|---|---|
| auth (단계 1) | 온보딩 없애기, 가입 트랜잭션(캐릭터 1종 + 초원 + "일상" + `grantReward("signup")`), 가입 폼 캐릭터 카드, `requireMember()` 의미 정리, `e2e/helpers.mjs`의 `loginDev` 수정 | US1(FR-001~003)의 구현 자체, 모든 E2E 로그인, "프로필이 있는 회원"에서만 출석 |
| auth | `src/server/dal.ts` `getViewer`에 출석 필드·호출 추가 합의 (공통 모듈 추가) | 자동 출석 위치 (research R1) |
| auth (단계 8, 병행 가능) | 로그인 유지 2시간/7일 — 세션 행 `id`를 그대로 둘 것 | `attendances.session_id` FK |

### 함께 맞출 일 (다른 spec 소유 파일, 요청)

| spec | 요청 | 관련 |
|---|---|---|
| town | 헤더(`src/components/site-header.tsx`)에 🔔·`LevelUpPopup`·`AttendanceDayWatcher` 끼우기, 유저 상태창 개편 때 375px 공간과 `Lv.N`·`🪙 N` 유지 | FR-010·038·042, SC-008 |
| town | `TownData.attendedToday` → `attendanceDay`, 게시판·입구 문구 (GAME FR-028 vs TOWN FR-023 결정 뒤) | FR-028 |
| town | `src/server/farm.ts`의 `farm_grown` INSERT를 `addLedgerEntry`로 (town 소유 함수의 동작이 바뀌므로 요청. 빠지면 다 키움으로 오른 레벨에 알림이 생기지 않아 FR-037을 못 지킨다), 성장 아이템으로 다 키우는 Action도 `revalidatePath("/", "layout")` | FR-037·038 |
| shop | `items.is_on_sale`(shop plan `specs/006-shop/data-model.md`가 정한 이름, 기본 true, 캐릭터는 모두 false로 이전) — 팝업 아이템 조건 `is_on_sale = true AND type <> 'character'`에 쓴다. shop 마이그레이션 전에는 `type <> 'character' AND is_starter = false` | FR-039 |
| shop | `e2e/game.mjs`의 상점·꾸미기 부분 (game은 출석 부분만 지운다) | - |
| blog (예약어 값 주인) | 새 최상위 주소 `/notifications`가 생긴다. blog FR-009 예약어 16개는 지금 화면 주소 이름(`attendance`, `wallet`, `tags` …)을 담고 있으므로 `notifications`를 더할지 정한다 (블로그 주소가 `/@{주소}`라 실제 주소 충돌은 없다. 더하면 spec FR-009 목록 수정) | 남은 문제 8 |
| social (단계 5) | 답글 보상도 `grantReward(…, "comment")` (댓글과 하루 10 합산), 남의 글일 때만 | FR-015·017, US2-6 |
| social (단계 7) | 공감·댓글·답글 트랜잭션 안에서 `notifyActivity()` 호출, 댓글 영역에 `id="comments"` (social plan `specs/004-social/contracts/notification-triggers.md`와 같은 조건) | FR-045, US6-7~10 |
| social (알림) | `toggleLike`의 공감 보상은 지금처럼 `lockUser(글 주인)` 안에서 `ref_id` 확인 뒤 `grantReward`. M1의 `point_ledger_like_received_uq`는 그 확인이 틀렸을 때만 걸리는 안전망이다 (걸리면 공감 트랜잭션 전체가 취소된다) | FR-017·047 |

### 이 plan에 기대는 다른 spec (후행)

| spec | 기대는 것 |
|---|---|
| social (단계 7) | `notifications` 표·`notification_kind`·`notifyActivity()` (M2, S5) |
| auth (단계 9, 탈퇴) | `notifications.user_id`·`actor_id` CASCADE로 알림 삭제 (FR-046), `attendances` CASCADE |
| town | `viewer.attendance.cycleDay` (광장 출석 도장), 레벨업 팝업이 집 단계 변화 안내를 대신함 (TOWN Assumptions) |
| auth | 관리자 통계 `오늘 출석` (`attendances` 오늘 행 수, 구조 변화 없음) |

## 남은 문제

1. **US1-3·SC-001 "가입 직후 헤더 `🪙 100`" ↔ FR-021 자동 출석**: 가입 직후 첫 화면에서 1일차 출석(🪙 10)이 붙어 헤더는 🪙 110이다. 이 plan은 FR-021을 따르고 가입 코인은 원장 `🎉 가입 축하 +100`으로 확인한다. spec 문구(그리고 auth spec US1 Independent Test) 수정이 필요하다 (research R18).
2. **광장 출석 도장 문구**: GAME FR-028 `오늘 N일차 ✅` ↔ TOWN FR-023 `마을 소식 · 출석 체크 (오늘 완료 ✅)` / `출석 체크 (오늘 완료)`. 제안: `(오늘 N일차 ✅)` (research R20).
3. **반복 공감 알림 (확인 권장)**: SOC FR-033 "공감이 새로 저장되면"과 GAME Assumptions "공감 취소 시 알림을 지우지 않는다"를 그대로 따르면 공감을 취소했다 다시 누를 때마다 알림이 또 생긴다. 이 plan과 social plan(`specs/004-social/contracts/notification-triggers.md` 3장) 모두 spec 문구대로 설계했다. 한 번으로 바꾸려면 spec을 고치고 부분 고유 인덱스 하나를 더하면 된다 (research R21).
4. **spec에 없는 화면 문구·동작**: 알림함 제목(`🔔 알림함`으로 둠), 🔔의 접근 이름(`알림`으로 둠), 팝업 Esc 동작([확인]과 같게 둠). FR-048 "spec 문구 그대로"와 맞추려면 spec에 적어야 한다. (출석 화면 위 보상 안내 줄 `하루 한 번 ✨ 10 · 🪙 20, …`을 없애는 것은 원본 GAME-04 변경 결정 "보상 안내를 7칸 표(오늘 일차 강조)로"를 따른 것이라 여기서 묻지 않는다.)
5. **팝업 아이템 범위**: 레벨업이 쌓였을 때 가장 높은 레벨의 아이템만이 아니라 건너뛴 레벨의 아이템까지 보여 준다 (research R12). spec "이 레벨부터"를 이렇게 읽어도 되는지 확인.
6. **원본 문서 정리**: `docs/01-requirements.md` GAME-04 "데이터 변경" 문단(`id` PK + UNIQUE, `ref_id` = 출석 ID)은 2026-10-07 결정 표와 ERD 3.12에 밀린 옛 설계다. 원본에서 고칠 곳으로 기록 필요.
7. **라이브러리 동작 확인**: `node_modules`가 없어 Better Auth 세션 ID, Next.js 16의 렌더 중 쓰기·`router.refresh()`·prefetch·Server Action `redirect()`의 `#comments` 해시, drizzle-kit 마이그레이션 트랜잭션·`streak` → `cycle_day` 이름 바꾸기 질문은 구현 전에 확인한다 (research R24).
8. **예약어 `notifications`**: 새 화면 주소 `/notifications`를 blog FR-009 예약어(가입 아이디·블로그 주소 금지 목록)에 더할지 blog와 정한다. 더하지 않아도 `/@notifications`와 `/notifications`는 다른 주소라 동작에는 문제가 없다.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

위반 없음.
