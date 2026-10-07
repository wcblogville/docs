# Contract: 자동 출석과 출석 체크 화면 (GAME-04)

**Feature**: `005-game` | 관련: FR-020~031, SC-003·004·005·008 | 설계 근거: [research.md](../research.md) R1~R7, R16, R20

출석에는 **출석만 요청하는 Server Action·주소가 없다**. 로그인한 회원의 요청을 서버가 처리하며 `getViewer()`를 부를 때 일어난다. 아래는 그 서버 내부 계약(다른 spec이 기대는 값), 화면 라우트, 광장에 넘기는 값이다.

## 1. 자동 출석 (`getViewer()`에 붙은 처리)

| 항목 | 내용 |
|---|---|
| 일어나는 곳 | `getViewer()`(`src/server/dal.ts`, React `cache`로 요청당 한 번)를 부르는 모든 서버 처리: ① 화면 렌더 — 루트 레이아웃 헤더(`SiteHeader`)가 모든 화면에서 부른다, ② 회원 Server Action — 모두 `requireMember()`(→ `getViewer()`)로 시작하고 `recordBlogVisit`은 `getViewer()`를 직접 부른다, ③ `POST /api/uploads` Route Handler(`getViewer()`로 회원 확인). ②·③에서 실제로 출석이 생기는 것은 보통 0시 전에 연 화면에서 0시 뒤에 보낸 요청이나, 화면 렌더의 출석이 실패한 뒤의 요청이다 (research R1, R16) |
| 조건 | 로그인 세션이 있고, 프로필이 있고, 오늘(`todayKST()`) 출석 행이 없음 |
| 입력 | `userId` = 세션의 회원 ID, `sessionId` = 세션 행 ID (Better Auth `getSession()` 결과, research R24), `today` = 서버의 한국 날짜 |
| 처리 | `ensureTodayAttendance(userId, sessionId)` (새 파일 `src/server/attendance.ts`) 한 트랜잭션: 회원 잠금 → 마지막 출석으로 일차 계산 → 출석 INSERT(이미 있으면 끝) → 보상표 값으로 원장 1줄(`attendance`, `ref_id` = 날짜) → 레벨이 오르면 레벨업 알림 |
| 결과 | `viewer.attendance = { date: "YYYY-MM-DD", cycleDay: 1~7 }`. 실패하면 `null` (서버 로그만, 화면은 그대로 그린다) |
| 같은 날 다시 | 출석 행이 있으면 JOIN으로 읽기만 한다. 쓰기·보상 없음 |
| 동시 요청 | 탭·기기 N개가 동시에 와도 출석 1행, 원장 1줄 (잠금 + PK + `point_ledger_attendance_uq`) |
| 일어나지 않는 곳 | 방문자, 그리고 `getViewer()`를 부르지 않는 요청: Route Handler `/api/auth/[...all]`(Better Auth), `/files/[key]`(첨부 내려받기), 로그인 전 Action `signUp`·`signIn` |
| 트랜잭션 규칙 | 안의 모든 조회·기록은 같은 `tx`로 한다. 연결 풀(기본 최대 10)이 잠금 대기 트랜잭션으로 다 찼을 때 `db`로 새 연결을 기다리면 멈춘다 (research R7) |
| 화면을 연 채 0시를 넘김 | 헤더의 `AttendanceDayWatcher`(새로, `src/components/game/attendance-day-watcher.tsx`)가 주소 이동·탭 다시 보기 때 날짜가 바뀌었으면 `router.refresh()` → 위 처리 + 헤더 갱신 |
| 호출 규칙 | `getViewer()`·`requireMember()`는 `db.transaction()` 밖에서 먼저 부른다 (research R17) |

**없어지는 인터페이스**: Server Action `attend()`(`src/app/attendance/actions.ts`)와 `AttendButton`(`attend-button.tsx`)을 지운다. 출석만 따로 요청하는 경로가 없어진다. 출석은 위 "일어나는 곳"의 요청에서 서버가 정한 일차·보상으로만 생긴다 (FR-021, FR-047).

**다른 spec이 읽는 값**

| 값 | 읽는 spec | 쓰임 |
|---|---|---|
| `viewer.attendance.cycleDay` | town | 광장 게시판 출석 도장 문구 (3장) |
| `attendances` 오늘 행 수 | auth | 관리자 통계 `오늘 출석` (지금 쿼리 그대로, `src/app/admin/page.tsx`) |
| `sessions.id` FK | auth | 세션을 지우면 `attendances.session_id`만 NULL. 출석은 남는다 |

## 2. 화면 `/attendance` — 출석 체크

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/attendance/page.tsx` (Server Component) |
| 접근 | 회원: `requireMember()`. 방문자: `/`(로그인·회원가입 첫 화면)로 redirect (FR-029, US3-12) |
| 메타 제목 | `출석 체크` (지금 그대로) |
| 헤더 | `← 광장으로 나가기` (모바일 `← 나가기`, `ExitButton` 그대로) |
| 데이터 | `viewer.attendance`(없으면 `ensureTodayAttendance()` 한 번 더, 실패하면 오류 화면), 이번 달 출석 날짜(한국 시간 이번 달 1일 이후), `attendance_rewards` 7행 |

**보여 주는 것** (문구는 spec 그대로)

| 영역 | 내용 | 근거 |
|---|---|---|
| 제목 | `📮 출석 체크` | FR-026 |
| 결과 카드 | 🎁 그림, `🎁 출석 완료! N일차`, `오늘 N일차 출석 완료` | FR-026, US3-8 |
| 7칸 보상표 | 1~7일차 칸마다 `N일차` · `✨ {경험치}` · `🪙 {코인}`, 오늘 일차 칸 강조(노란 테두리·굵게). 375px에서도 7칸 한 줄 | FR-020, FR-026 |
| 달력 위 줄 | `이번 달 N일 출석 · 현재 N일차` | FR-027 |
| 달력 | 이번 달(한국 시간)만, 일~토 7칸 정사각형. 출석한 날 🌟 + 초록 칸, 오늘은 노란 테두리(출석한 칸 위에도) | FR-027, US3-10 |

**보이지 않게 되는 것** (지금 코드에 있음): [📮 출석하고 보상 받기], `편지 여는 중...`, `출석 완료! N일 연속`, `(7일 연속 보너스 포함!)`, `✅ 오늘은 이미 출석했어요. 내일 또 만나요!`, `현재 연속 N일`, `연속 출석이 끊겼어요 (…)`, `아직 출석 기록이 없어요`, 위 안내 `하루 한 번 ✨ 10 · 🪙 20, 7일 연속마다 🪙 50 보너스` (spec Assumptions, FR-030).

## 3. 광장 출석 도장에 넘기는 값 (town 소유 화면)

| 항목 | 지금 | 목표 |
|---|---|---|
| 데이터 | `TownData.attendedToday: boolean` (`src/components/town/types.ts`), `hasAttendedToday()` (`src/server/town.ts`) | `TownData.attendanceDay: number \| null` = `viewer.attendance?.cycleDay ?? null` (town이 바꾼다) |
| 회원 문구 | 게시판 부제 `마을 소식 · 출석 체크 (오늘 완료 ✅)` / `(보상 받기 🎁)`, 입구 `출석 체크 (오늘 완료)` (`src/components/town/scene.ts`, `town-menu.tsx`) | GAME FR-028: 입구에 `오늘 N일차 ✅`. TOWN FR-023과 문구가 달라 팀 결정 필요 (research R20, open item) |
| 방문자 | 입구 안내 `Space 로그인하고 이용하기` → 첫 화면 | 그대로 (FR-029, TOWN FR-022) |
| 입구 이동 | `/attendance` | 그대로 |

## 4. 오류·조작

| 상황 | 결과 |
|---|---|
| 방문자가 `/attendance` 주소로 들어옴 | `/`로 redirect |
| 옛 `attend` Server Action ID로 요청을 보냄 | 그 Action이 없다 (Next.js가 찾을 수 없는 Action으로 처리). 출석·원장 변화 없음 |
| 출석 트랜잭션 실패 | 출석·보상 둘 다 없음. 일반 화면은 그대로 열리고 다음 화면에서 다시 시도. 출석 화면은 기존 오류 화면 |
| 같은 날 10개 탭 동시 첫 화면 | 출석 1행, 원장 `attendance` 1줄 (SC-003) |
