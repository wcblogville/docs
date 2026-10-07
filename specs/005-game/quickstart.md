# Quickstart: 캐릭터 / 성장 (GAME) 검증

**Feature**: `005-game` | **Date**: 2026-10-07 | **Plan**: [plan.md](plan.md)

이 문서는 GAME이 끝까지 동작하는지 확인하는 **실행 순서와 기대 결과**다. 구현 코드는 적지 않는다. 명령은 코드 저장소 루트에서 실행한다.
화면·Action 약속은 [contracts/](contracts/), 표·제약은 [data-model.md](data-model.md)를 본다.

## 1. 준비

### 1.1 환경

- Node.js 20.9 이상, PostgreSQL 17(README 안내, 15 이상이면 된다), 코드 저장소 `.env.local`에 `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `ADMIN_USERNAME`, `ADMIN_PASSWORD`가 있어야 한다 (값은 여기 적지 않는다).
- 선행 작업: auth 단계 1(가입 통합, `e2e/helpers.mjs`의 `loginDev` 수정)이 merge되어 있어야 E2E 로그인 도우미가 동작한다 ([plan.md](plan.md) 의존성).
- 새 패키지는 없다.

### 1.2 DB와 개발 서버

```bash
npm run db:migrate      # 출석 일차·보상표, 알림 마이그레이션 적용
npm run db:seed         # 아이템·동물 카탈로그 (출석 보상표는 마이그레이션이 넣는다)
npm run db:reset        # 로컬 회원·글 비우기 (attendance_rewards는 남는다)
npm run admin:create    # 관리자 계정 다시 만들기
npm run dev             # http://localhost:3000
```

### 1.3 마이그레이션 전후 확인 (psql 또는 `npm run db:studio`)

| 언제 | 확인 | 기대 |
|---|---|---|
| 적용 전 | `point_ledger`에서 `reason = 'attendance'`를 (`user_id`, `ref_id`)로 묶어 2줄 이상인 묶음 | 0개 (있으면 `point_ledger_attendance_uq` 생성이 실패한다 — 중복 원인을 먼저 본다) |
| 적용 전 | 같은 방법으로 `reason = 'like_received'` | 0개 (있으면 `point_ledger_like_received_uq` 생성이 실패한다) |
| 적용 후 | `attendance_rewards` 전체 | 7행, 경험치 10·10·15·15·20·20·30, 코인 10·20·30·40·50·70·100 (FR-020) |
| 적용 후 | 옛 출석 행의 `cycle_day` | 적용 전 `streak`이 1·7·8·13·14였던 행이 1·7·1·6·7 (FR-031) |
| 적용 후 | `attendances.checked_at` | 옛 행은 그날 출석 원장 시각(없으면 그날 한국 0시), `session_id`는 NULL |
| 적용 후 | `attendances`에 `streak` 컬럼 | 없음 |
| 적용 후 | `notifications`에 레벨업 모양이 아닌 행 넣기(`kind = 'level_up'`인데 `actor_id` 있음), 자기 자신 알림(`actor_id = user_id`) | 둘 다 CHECK 위반으로 거부 (constitution V) |
| 적용 후 | 같은 회원·같은 레벨 `level_up` 두 번 넣기 | 두 번째는 고유 인덱스 위반 (FR-037) |
| 적용 후 | 같은 글 주인·같은 `ref_id`(`글ID:공감한회원ID`)로 `like_received` 두 번 넣기 | 두 번째는 고유 인덱스 위반 (FR-017, FR-047) |

## 2. 정적 검사와 단위 시험

```bash
npx tsc --noEmit
npx eslint
npm test                # test:game → test:ids → test:sanitize → test:notifications (test:notifications는 새로 추가해 체인 끝에 붙인다)
```

| 시험 | 확인 | 기대 | 연결 |
|---|---|---|---|
| `npm run test:game` (고침) | `nextCycleDay`: 기록 없음 → 1, 어제 3 → 4, 어제 7 → 1, 그저께 → 1, 10/1~10/11 예시(10/10 빠짐), 9/30 → 10/1, 12/31 → 1/1 | 1~7, 1, 2, (없음), 1 등 FR-022 표와 100% 일치 | US3-1~5·11, SC-005 |
| 〃 | `levelsGained`: 99→100, 290→300, 90→340, 485,090→485,100, 600,000→600,010 | `[2]`, `[3]`, `[2,3]`, `[99]`, `[]` | US2-8·10, US5-1·6 |
| 〃 | `todayKST`: UTC 14:59와 15:01 (한국 23:59, 0:01) | 서로 다른 날짜 | US3-13, US2-11 |
| 〃 | 레벨 공식·MAX (지금 시험 유지) | Lv.2 = 100, 485,100 → 99, `MAX` | FR-008·009·011 |
| `npm run test:notifications` (새로) | `unreadBadge` 0·3·9·10·25 | 없음·`3`·`9`·`9+`·`9+` | US6-1, FR-042 |
| 〃 | `notificationTime` 30초·59분·60분·23시간 59분·24시간 | `방금`·`59분 전`·`1시간 전`·`23시간 전`·`YYYY.MM.DD` | FR-043 |
| 〃 | `levelUpTitle` 3·99, `notificationText` 4종, `notificationHref` 4종 | FR-038·045 문구와 주소 그대로 | FR-038·045 |

## 3. E2E

개발 서버를 띄운 상태에서 1.2 준비 뒤 실행한다. 첫 인자는 스크린샷 폴더. 새 시나리오는 실행마다 새 회원을 만들고, `pg`로 날짜·경험치를 준비하고, `❌`가 있으면 `exit(1)`한다.

```bash
node e2e/auth.mjs shots          # (auth 소유) 가입·로그인 — 가입 지급 확인 포함
node e2e/blog.mjs shots          # 기존: 글쓰기·공감·댓글 보상
node e2e/attendance.mjs shots    # 새로: 자동 출석·일차·동시 요청·실패 롤백·출석 화면
node e2e/rewards.mjs shots       # 새로: 가입 지급·보상 규칙·하루 상한·동시 요청·내역 화면
node e2e/notifications.mjs shots # 새로: 레벨업 팝업·알림함(1단계)·조작 요청
node e2e/decisions.mjs shots     # 고침: GAME-04 블록 삭제, GAME-07 옛 보너스 줄 확인 방식
node e2e/game.mjs shots          # 고침: 출석 버튼 부분 삭제 (상점·꾸미기는 shop)
node e2e/write-count.mjs shots   # 기존: 글쓰기 보상 안내 = 서버 판단
node e2e/mobile.mjs shots        # 기존(town): 375px 헤더
node e2e/nonfunctional.mjs shots # 기존: 375px 가로 스크롤 — 대상 주소에 /notifications 추가
```

코드 저장소 `README.md` 스크립트 표에 새 E2E 세 줄을 더하고, `e2e/game.mjs` 줄의 설명을 바꾼다.

## 4. 시나리오와 기대 결과

### US1 가입하면서 캐릭터를 고르고 받기 (auth 구현, game 확인)

| # | 실행 | 기대 | 검증 |
|---|---|---|---|
| US1-1 | 새 아이디로 여자 주민을 골라 가입 | 보유 캐릭터는 여자 주민 1개, 여자 주민·초원 장착 | `e2e/rewards.mjs` (DB `user_items` 2행), `e2e/decisions.mjs` GAME-01 |
| US1-2 | 첫 화면 [회원가입] 탭 | 캐릭터 카드 2개 2열, 남자 주민이 골라져 있음 | `e2e/auth.mjs`(auth) |
| US1-3 | 가입 직후 헤더·내역 | 내역에 `🎉 가입 축하` `🪙 +100` 1줄. 헤더는 1일차 출석이 더해진 `🪙 110` (spec 문구와 다름, open item) | `e2e/rewards.mjs` |
| US1-4 | 가입 요청의 캐릭터 ID를 기본 캐릭터가 아닌 아이템으로 바꿔 다시 보냄 | 가입 거부, 그 아이디의 `users`·`profiles`·`blogs`·`user_items`·`point_ledger` 0행 | `e2e/auth.mjs`(auth), `e2e/rewards.mjs` |
| US1-5 | 기존 회원에게 `char_cat`을 넣고 꾸미기 | 계속 장착 가능, 코인 변화 없음 | `e2e/decisions.mjs` (SHOP-04 블록, 지금 있음) |
| US1-6 | 상점 | 남자 주민·여자 주민 없음 | shop E2E |
| SC-001 | 가입한 회원 전부 | 고른 캐릭터·초원 장착, 가입 축하 +100 | `e2e/rewards.mjs` |

### US2 활동 보상과 레벨

| # | 실행 | 기대 | 검증 |
|---|---|---|---|
| US2-1~3 | 100자 이상 공개 새 글 4개, 99자 글, 100자 글 | 보상 3번(✨30·🪙30), 4번째·99자 보상 없음, 100자 보상 있음, 헤더 숫자 바로 바뀜 | `e2e/rewards.mjs`, `e2e/write-count.mjs` |
| US2-4 | 글쓰기 화면에서 글자 수·공개 여부 바꾸기 | 4가지 안내 문구 | `e2e/write-count.mjs` (지금 있음) |
| US2-5 | 비공개 글을 공개로 수정 | 보상 없음 | `e2e/rewards.mjs` |
| US2-6 | 남의 글 댓글 / 내 글 댓글 | ✨5·🪙5 / 없음 (답글은 social 단계 5 뒤) | `e2e/blog.mjs`, `e2e/rewards.mjs` |
| US2-7 | 다른 회원이 공감 → 취소 → 다시 공감 | 글 주인 `like_received` 1줄, 취소 뒤에도 그대로 | `e2e/rewards.mjs` |
| US2-8 | 경험치 99 회원이 +1 이상 | Lv.2 | `npm run test:game`, `e2e/notifications.mjs` |
| US2-9 | 경험치 150으로 맞추고 꾸미기 | `Lv.2`, `50 / 200 EXP`, 초록 막대 | `e2e/rewards.mjs` |
| US2-10 | 경험치 60만 | 헤더 `Lv.99`, 꾸미기 `MAX` | `e2e/decisions.mjs` (지금 있음) |
| US2-11 | 어제 한국 23:59 시각의 `post` 3줄을 넣고 오늘 새 글 | 보상 받음 | `e2e/rewards.mjs` |
| US2-12, SC-006 | 댓글 Server Action 요청을 12번 동시에 다시 보냄 | 오늘 `comment` 원장 10줄 이하, 댓글은 모두 저장 | `e2e/rewards.mjs` |
| US2-13, SC-008 | 375px 화면 | 헤더에 `Lv.N`과 `🪙 N` 둘 다 보임 | `e2e/rewards.mjs`, `e2e/mobile.mjs` |
| US2-14, SC-011 | 보상받은 글·댓글 삭제 | 경험치·코인·레벨 그대로, 내역의 `✏️ 글 작성`·`💬 댓글 작성` 줄 그대로 | `e2e/rewards.mjs` |
| SC-002, SC-007 | 보상·구매 뒤 다음 화면 | 헤더·내역 위 요약·꾸미기 값 = 원장 합계 (DB `SUM`) | `e2e/rewards.mjs`, `e2e/decisions.mjs` |

### US3 자동 출석

준비: 새 회원으로 로그인(로그인 렌더에서 오늘 출석이 이미 생긴다) → 각 단계 앞에서 그 회원의 오늘 출석 행과 오늘 `attendance` 원장 줄을 지우고, 필요한 어제·그저께 출석을 `pg`로 넣는다.

| # | 실행 | 기대 | 검증 |
|---|---|---|---|
| US3-1 | 출석 기록 없이 `/town` 열기 | 버튼 없이 1일차 출석 1행, 원장 `📮 출석` ✨10·🪙10 1줄 | `e2e/attendance.mjs` |
| US3-2 | 어제 3일차 → 오늘 열기 | 4일차, ✨15·🪙40 | `e2e/attendance.mjs` |
| US3-3 | 어제 7일차 → 오늘 열기 | 1일차 | `e2e/attendance.mjs` |
| US3-4 | 그저께 5일차 → 오늘 열기 | 1일차 | `e2e/attendance.mjs` |
| US3-5, US3-11, SC-005 | 10/1~10/11 예시, 달·해 바뀜 | FR-022 표와 일치 | `npm run test:game` |
| US3-6, SC-003 | 같은 회원 탭 10개로 동시에 `/town` | 오늘 출석 1행, 오늘 `attendance` 원장 1줄, 탭마다 헤더 코인 같음 | `e2e/attendance.mjs` |
| US3-7 | 오늘 `attendance` 원장 줄만 미리 넣고(출석 행 없음) `/town` 열기 → 출석 트랜잭션이 고유 인덱스에 걸려 실패 | 오늘 출석 0행(롤백), 원장 미리 넣은 1줄뿐, `/town`은 HTTP 200으로 열림, `/attendance`는 오류 화면 | `e2e/attendance.mjs` |
| US3-8 | `/attendance` | 버튼 없음, `🎁 출석 완료! N일차`, `오늘 N일차 출석 완료`, 7칸 보상표(오늘 칸 강조), 달력 | `e2e/attendance.mjs` + 스크린샷 |
| US3-9 | 광장 게시판 출석 도장 | 오늘 일차 표시 (문구는 town과 결정 뒤 확정, open item) | town E2E |
| US3-10 | 이번 달의 오늘 이전 날짜에 출석 k개(최대 3개, 매달 1일이면 0개)를 넣고 `/attendance` | `이번 달 {k+1}일 출석 · 현재 N일차`, 🌟 k+1칸, 오늘 칸 노란 테두리 | `e2e/attendance.mjs` + 스크린샷 |
| US3-12 | 방문자로 `/attendance` | `/`로 이동 | `e2e/attendance.mjs` |
| US3-13 | 한국 23:59 / 0:01 | 다른 날 | `npm run test:game` |
| SC-004 | 오늘 출석을 지우고 `/town`을 한 번만 열기 | 새로고침 없이 첫 화면 헤더 코인 = 원장 합계(출석 보상 포함) | `e2e/attendance.mjs` |
| FR-021 (Action 경로) | 오늘 출석을 지운 뒤, 이미 열린 화면에서 Server Action(예: 공감)만 보냄 | 그 요청에서 오늘 출석 1행·원장 1줄이 생기고 응답 뒤 헤더 코인 = 원장 합계 (research R1) | `e2e/attendance.mjs` |
| FR-025, Edge | 출석 뒤 그 세션 행을 DB에서 지움(로그아웃과 같음) | 출석 행 남고 `session_id`만 NULL. 출석한 세션이 있을 때 `session_id` = 그 세션 | `e2e/attendance.mjs` |
| FR-030 | 어제 6일차 → 오늘 7일차 | 원장에 `attendance_streak` 새 줄 없음, 7일차 ✨30·🪙100만 | `e2e/attendance.mjs` |
| Edge 0시 넘김 | Playwright `page.clock`으로 브라우저 시계를 다음 날로 옮기고 화면 이동 | 새로고침 요청(RSC)이 한 번 나감. 서버 날짜가 같으면 출석 변화 없음 | `e2e/attendance.mjs` (선택 확인) |
| SC-008 | 375px `/attendance` | 가로 스크롤 없음, 7칸 한 줄 | `e2e/attendance.mjs`, `e2e/nonfunctional.mjs` |

### US4 경험치·코인 내역

| # | 실행 | 기대 | 검증 |
|---|---|---|---|
| US4-1, SC-010 | 헤더 `🪙 N` 클릭 | `/wallet` 3초 안 | `e2e/decisions.mjs`, `e2e/rewards.mjs`(시간 측정) |
| US4-2 | 위 요약 | 코인 = 헤더 | `e2e/decisions.mjs` |
| US4-3 | 바닷가 구매 뒤 | `🏪 아이템 구매 · 바닷가`, 빨간 `🪙 −120` | `e2e/decisions.mjs` |
| US4-4 | 원장 21줄 이상 | 20줄, `기록 N개`, 2페이지 | `e2e/rewards.mjs` |
| US4-5 | 그 회원 원장을 모두 지우고 열기 | `아직 기록이 없어요` | `e2e/rewards.mjs` |
| US4-6 | 다른 회원 기록이 있음 | 내 기록만 | `e2e/rewards.mjs` |
| US4-7 | 방문자로 `/wallet` | `/`로 이동 | `e2e/rewards.mjs` |
| US4-8 | 옛 `attendance_streak` 줄을 넣고 열기 | `🔥 연속 출석 보너스` | `e2e/decisions.mjs` (고침) |

### US5 레벨업 팝업

| # | 실행 | 기대 | 검증 |
|---|---|---|---|
| US5-1 | 경험치 290, 오늘 출석 없음, 어제 출석 없음 → `/attendance` | 그 화면에서 `Lv.3이 되었어요!` 팝업 1번, 벚꽃길 보임(Lv.3 판매 아이템이 3개를 넘으면 `외 N개`), 판다(캐릭터) 안 보임 | `e2e/notifications.mjs` + 스크린샷 |
| US5-2 | 글 주인 경험치 98, 다른 회원이 공감 → 글 주인이 아무 화면 | 팝업 `Lv.2이 되었어요!` 1번 | `e2e/notifications.mjs` |
| US5-3, SC-009 | [확인] → 새로고침 / [상점 가기] | 팝업 다시 안 뜸, `/notifications`에 `🎉 Lv.3이 되었어요!` 남음 / `/shop`으로 이동 | `e2e/notifications.mjs` |
| US5-4 | 레벨이 안 오르는 보상(경험치 10에서 댓글) | 팝업 없음, `level_up` 알림 0행 | `e2e/notifications.mjs` |
| US5-5 | 안 읽은 `level_up` Lv.3·Lv.4를 넣고 화면 열기 → [확인] | `Lv.4이 되었어요!`만, 둘 다 읽음 | `e2e/notifications.mjs` |
| US5-6 | 경험치 485,090 + 1일차 출석 | `최고 레벨 Lv.99가 되었어요!` | `e2e/notifications.mjs` |
| FR-037 동시 | 레벨 경계 직전에서 보상 요청 여러 개를 동시에 | 그 레벨 `level_up` 1행 | `e2e/notifications.mjs` |
| Esc | 팝업에서 Esc | [확인]과 같음 (open item 확정 뒤) | `e2e/notifications.mjs` |

### US6 알림함

| # | 실행 | 기대 | 검증 |
|---|---|---|---|
| US6-1 | 안 읽은 알림 3개 / 10개 | 🔔 옆 `3` / `9+` | `e2e/notifications.mjs` |
| US6-2 | [모두 읽음] | 노란 배경·빨간 숫자 사라짐 | `e2e/notifications.mjs` |
| US6-3 | 레벨업 알림 누름 | `/shop`, 그 알림 읽음 | `e2e/notifications.mjs` |
| US6-4 | 다른 회원 알림이 있음 / 그 ID로 `openNotification` 요청 본문을 바꿔 보냄 | 내 알림만 / 남의 알림 `read_at` 그대로, 500 없음 | `e2e/notifications.mjs` |
| US6-5 | 알림 0개 | `아직 알림이 없어요` | `e2e/notifications.mjs` |
| US6-6 | 방문자 헤더 | 🔔 없음 | `e2e/notifications.mjs` |
| 표시 미리 확인 | 다른 회원을 행동한 회원으로 `like`·`comment`·`reply` 행을 DB에 넣고 목록 | FR-045 문구 3가지, 누르면 글 / 글의 `#comments` | `e2e/notifications.mjs` |
| US6-7~10, SC-012 (2단계) | 다른 회원이 공감·댓글·답글 / 내가 내 글에 | 🔔 +1, 문구, 이동 / 알림 없음 | social 단계 7 E2E (예: `e2e/replies.mjs`, social 소유) |
| FR-046 | (로컬 DB) 행동한 회원 행 삭제 | 그 회원이 남긴 알림 0행 | `e2e/notifications.mjs` (DB 확인), 탈퇴 화면은 auth 단계 9 |
| 조작 | `/notifications?page=99999999999999999999`, `dismissLevelUp`에 `level=100` | HTTP 200 / 변화 없음 | `e2e/notifications.mjs`, `e2e/params.mjs`에 한 줄 |
| SC-008 | 375px `/notifications`, 헤더 🔔 | 가로 스크롤 없음, 레벨·코인·🔔 보임 | `e2e/notifications.mjs`, `e2e/nonfunctional.mjs` |

## 5. 화면 확인 (스크린샷)

| 파일 예 | 화면 |
|---|---|
| `attendance-day4.png`, `attendance-375.png` | 출석 체크 (PC, 375px) |
| `levelup-lv3.png`, `levelup-lv99.png` | 레벨업 팝업 |
| `notifications-list.png`, `notifications-empty.png` | 알림함 |
| `header-bell-375.png` | 375px 헤더 (레벨·코인·🔔) |
| `wallet.png` | 내역 |

## 6. 끝났다고 볼 조건

- 2장 명령이 모두 통과하고 3장의 새 E2E 세 개가 `❌` 없이 끝난다.
- 4장 표의 game 검증 줄이 모두 확인되고, open item(US1-3 문구, 광장 출석 도장 문구, Esc)이 정해진 대로 반영되어 있다.
- 결과를 PR 설명에 적는다 (CI가 없으므로 작성자가 직접 실행, constitution 품질 관문).
