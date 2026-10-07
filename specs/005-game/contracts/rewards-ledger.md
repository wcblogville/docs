# Contract: 보상·원장 모듈과 경험치·코인 화면 (GAME-01·02·03·05·07)

**Feature**: `005-game` | 관련: FR-001~019, FR-032~036, SC-001·002·006·007·008·010·011 | 설계 근거: [research.md](../research.md) R7, R8, R18, R19

game이 소유한 `src/server/points.ts`, `src/lib/game.ts`는 다른 spec(auth, post, social, shop, town)이 부르는 **서버 내부 인터페이스**다. 이 문서는 그 약속과, 원장을 보여 주는 화면(`/wallet`, 헤더·꾸미기·상점의 숫자)을 정한다. HTTP Route Handler는 없다.

## 1. 서버 모듈 `src/server/points.ts`

| 함수 | 시그니처 (목표) | 지금 | 약속 |
|---|---|---|---|
| `lockUser` | `(tx, userId) => Promise<void>` | 있음 | 그대로. 같은 회원의 보상·구매·출석을 한 줄로 세운다 (`pg_advisory_xact_lock(hashtext(userId))`) |
| `getWallet` | `(userId, executor?) => { coins, exp, level, current, needed, ratio, isMax }` | 있음 | 그대로. 헤더·내역·꾸미기·상점이 같은 함수를 쓴다 (SC-002) |
| `grantReward` | `(tx, userId, reason, refId?) => { granted: false } \| { granted: true, exp, coins }` | 있음 | 시그니처 그대로. 원장 INSERT를 `addLedgerEntry`로 바꿔 레벨업 알림이 함께 생긴다. `reason`에서 `attendance`·`attendance_streak`가 빠진다 |
| `addLedgerEntry` | `(tx, { userId, reason, expDelta?, coinDelta?, refId? }) => { levelUps: number[] }` | **새로** | 원장 1줄 + (경험치가 늘어 레벨이 오르면) 오른 레벨마다 `level_up` 알림. 반드시 `lockUser`를 건 트랜잭션 안에서 부르고, 누적 경험치 조회도 같은 `tx`로 한다 (research R7·R8) |
| `listLedger` | `(userId, page) => { rows, total, page, pageCount }` | 있음 | 그대로 (최신순, 같으면 ID 큰 순, 20개, 구매 줄은 아이템 이름) |
| `startOfTodayKST` | SQL 조각 | 있음 | 그대로 (하루 상한의 "오늘") |

**부르는 쪽 규칙**

| 부르는 spec | 쓰는 곳 | 약속 |
|---|---|---|
| auth | 가입 트랜잭션 (`src/app/(auth)/actions.ts`) | `lockUser` 뒤 `grantReward(tx, userId, "signup")` 한 번. 캐릭터는 `items.type = 'character' AND is_starter = true`인지 서버에서 다시 확인하고 아니면 전체 롤백 (FR-002, FR-003) |
| post | `savePost` (`src/app/write/actions.ts`) | 새 공개 글 + `content_text` 100자 이상일 때만 `grantReward(…, "post", 글ID)` (지금 그대로, FR-015, FR-017) |
| social | 댓글·답글·공감 | 남의 글 댓글·답글에 `grantReward(…, "comment", id)` (댓글·답글 합쳐 하루 10), 공감 보상은 `ref_id = 글ID:공감한회원ID`로 쌍마다 한 번 (지금 그대로, FR-015, FR-017). DB에도 `point_ledger_like_received_uq`가 있어 같은 쌍의 두 번째 줄은 고유 인덱스 위반으로 트랜잭션이 취소된다 (research R25) |
| shop | `buyItem` | 경험치 0인 `purchase` 줄은 지금처럼 직접 넣어도 된다 (레벨이 바뀌지 않는다) |
| town | `addGrowth`의 `farm_grown` (`src/server/farm.ts`, town 소유) | 원장 INSERT를 `addLedgerEntry`로 바꾼다 (town에 요청. 한 줄이지만 다 키움으로 레벨이 오를 수 있어 빠지면 그 레벨업 알림이 생기지 않는다) |
| town | `buyEgg`의 `egg_purchase` (`src/app/farm/actions.ts`) | 경험치 0이라 지금처럼 직접 넣어도 된다 |
| 모두 | 보상을 준 Server Action 끝 | `revalidatePath("/", "layout")` (헤더 숫자·레벨업 팝업, FR-013, SC-007) |
| 모두 | 회수 | 글·댓글·답글 삭제, 공감 취소 때 원장을 지우거나 음수 경험치를 넣지 않는다 (FR-016, SC-011) |

## 2. 규칙 숫자 `src/lib/game.ts`

| 이름 | 목표 값 | 바뀌는 것 |
|---|---|---|
| `expForLevel`, `levelFromExp`, `levelProgress`, `MAX_LEVEL = 99` | 그대로 (FR-008, FR-009, FR-011) | 없음 |
| `REWARD_RULES` | `signup` 0/100/1, `post` 30/30/3, `comment` 5/5/10, `like_received` 2/2/20, `farm_care` 2/0/15 | `attendance`, `attendance_streak` 삭제 (출석은 보상표, FR-030) |
| `POST_REWARD_MIN_LENGTH` | 100 | 없음 |
| `ATTENDANCE_STREAK_BONUS_EVERY`, `currentStreak` | - | 삭제 |
| `nextCycleDay(last, today)` | 새로 | 출석 일차 (FR-022) |
| `levelsGained(beforeExp, afterExp)` | 새로 | 오른 레벨 목록, 99까지 (FR-037) |
| `todayKST`, `previousDay` | 그대로 | 없음 |

## 3. 화면·표시 계약

### 3.1 `/wallet` — 경험치·코인 내역 (지금 구현 유지)

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/wallet/page.tsx` |
| 접근 | 회원 `requireMember()`, 방문자 → `/` (FR-036, US4-7) |
| 들어오는 길 | 헤더 `🪙 N`(title `코인`) 링크, 꾸미기 `경험치·코인 내역 보기` (FR-032) |
| 주소 인자 | `?page=N` (`parsePage`) |
| 위 요약 | 레벨 `Lv.N`, 누적 경험치 `✨ N`, 코인 `🪙 N` — `getWallet()`이라 헤더와 같다 (FR-033, SC-002) |
| 목록 | 본인 원장만, 최신순 20개, `기록 N개`, 한 줄 = 사유 이름, `YYYY. MM. DD. HH:MM`, `✨ +N`(하늘색), `🪙 +N`(초록)/`🪙 −N`(빨강), 0은 생략, 구매 줄은 `🏪 아이템 구매 · {아이템 이름}` (FR-034, FR-035) |
| 사유 이름 | `🎉 가입 축하`, `📮 출석`, `🔥 연속 출석 보너스`(옛 기록), `✏️ 글 작성`, `💬 댓글 작성`, `♥ 공감 받음`, `🏪 아이템 구매` (+ town 사유 `🥕 동물 돌보기`, `🏅 동물 다 키움`, `🥚 알 구매`는 지금 그대로) |
| 빈 목록 | `아직 기록이 없어요` |
| 맨 아래 | `코인은 상점에서 쓸 수 있어요.` (`상점` 링크) |

### 3.2 다른 화면의 숫자 (각 화면 소유 spec이 유지)

| 위치 | 표시 | 소유 | 근거 |
|---|---|---|---|
| 헤더 | `Lv.N`, `🪙 N`(천 단위 쉼표, `/wallet` 링크). 375px에서도 숨기지 않음 | town (TOWN-10 상태창 개편 때 유지) | FR-010, FR-013, SC-008 |
| 꾸미기 `/closet` | `Lv.N`, `현재 / 필요 EXP`(예: `50 / 200 EXP`), 초록 막대, Lv.99는 가득 찬 막대 + `MAX`, `경험치·코인 내역 보기` | shop (지금 구현 유지) | FR-011, FR-032 |
| 상점 `/shop` | 오른쪽 위 `Lv.N 🪙 N` | shop | FR-013 |
| 글쓰기 아래 | `N자 · 저장하면 ✨ 경험치 30 · 🪙 30 보상 (하루 3번까지)` / `비공개 글은 보상이 없어요` / `100자 이상 쓰면 보상을 받아요` / (수정 중) `글을 고치고 있어요` — 숫자는 `REWARD_RULES.post` | post (지금 구현 유지, `src/components/editor/post-form.tsx`) | FR-019 |
| 광장·헤더·미니룸 캐릭터 | 장착 캐릭터 | town·blog·shop | FR-006 |

### 3.3 가입 지급 (auth가 구현하는 화면·Action)

| 항목 | 약속 |
|---|---|
| 화면 | 회원가입 탭에 기본 캐릭터 카드 2개(남자 주민·여자 주민, 2열, 그림·이름·한 줄 설명), 처음엔 남자 주민 (FR-001) |
| 성공 | 고른 캐릭터 1종 + 초원 보유·장착, 원장 `signup` +100 한 줄, 기본 카테고리·프로필·블로그 (한 트랜잭션, FR-002) |
| 조작 | 기본 캐릭터가 아닌 아이템 ID → 가입 거부, 아무것도 생기지 않음 (FR-003, US1-4). 문구는 auth spec |
| 첫 화면 | 가입 직후 광장 렌더에서 1일차 자동 출석이 함께 일어나 헤더는 🪙 110 (research R18, open item: US1-3·SC-001 문구) |
