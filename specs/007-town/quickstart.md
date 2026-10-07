# Quickstart: 광장 (TOWN) 검증

**Feature**: `007-town` | **Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Contracts**: [contracts/](contracts/)

이 기능이 끝까지 동작하는지 확인하는 실행 순서다. 명령은 모두 코드 저장소 루트에서 실행한다. 구현 코드는 적지 않는다 — 무엇을 돌리고 무엇이 나와야 하는지만 적는다.

## 0. 전제

| 전제 | 왜 |
|---|---|
| auth 단계 1 (가입 통합, 온보딩 없음, `e2e/helpers.mjs`의 `loginDev`가 가입 폼에서 바로 광장으로) | 모든 e2e의 로그인 도우미 |
| auth `signUp`이 쿠키 `bv_welcome`을 심음 | 환영 문구 (US1-2·3) |
| social `follows.is_favorite` (U4) | 즐겨찾기 (US4) |
| shop 성장 아이템 (`items.type = 'growth'`, `growth_value`, `user_items.quantity`, `consumeGrowthItem`, 상점에서 사기) | 성장 아이템 사용 (US6-8·9) |
| blog `blogs.roof_color` (B-M2) | 지붕 색 (US10) |
| blog 도감·전시 (단계 12) | US7 확인 (blog 검증 문서 참고) |
| game 자동 출석 (`viewer.attendance`) | 게시판 출석 표시 (US3-6). 없으면 지금 쿼리로 확인 |
| 아이소메트릭(US9, SC-012)은 town 단계 13 뒤 | 그 전에는 2D로 같은 시나리오를 돌린다 |

`.env.local`에는 `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `ADMIN_USERNAME`, `ADMIN_PASSWORD`가 있어야 한다 (값은 문서에 적지 않는다).

## 1. 준비

```bash
npm run db:migrate          # user_animals UNIQUE(user_id, id) 포함 (+ 다른 spec 마이그레이션)
npm run db:seed             # 동물 4종(SPECIES), 성장 아이템 3종(shop ITEMS)
npm run dev                 # http://localhost:3000 (다른 터미널에서 계속 띄워 둠)
npm run db:reset            # 로컬 DB에서만: 회원·글 비움 (카탈로그는 남음)
npm run admin:create        # 관리자 계정·/@notice 다시 만들기
```

적용 확인 (psql 등): `\d user_animals`에 `user_animals_user_id_id_uq` UNIQUE (`user_id`, `id`)가 보인다.

## 2. 정적 검사와 단위 테스트

```bash
npx tsc --noEmit
npx eslint
npm test                    # test:game → test:ids → test:sanitize → … → test:town (test:town은 새로 추가)
npm run test:town           # (새로 추가하는 스크립트) 따로 돌릴 때
```

`npm run test:town`(`scripts/test-town.ts`, 새)의 기대 결과 — 모든 줄 `✅`, 종료 코드 0:

| 시험 | 기대 | spec |
|---|---|---|
| `houseStage` Lv.1·9·10·29·30·99 | 1·1·2·2·3·3 | FR-059, SC-014 |
| `houseRoof` | 코드값 → 정한 색, `null` → 배경 강조색, 배경이 바뀌어도 고른 색 그대로 | FR-040, US10-2·4 |
| `houseLabel` | 12자 이하 그대로 + `의 집`, 13자·20자·이모지는 12자 + `…의 집` | FR-024 |
| `sampleDistinct` | 100개 → 10개(겹침 없음, 모두 후보 안), 7개 → 7개, 0개 → 빈 배열 | FR-029, US5-1·4 |
| `canFavorite` | 9 → 가능, 10 → 불가 | FR-033 |
| `layout()` 배치 | 3단계 집 10채 + 3단계 내 집: 그림 사각형이 서로·건물과 겹치지 않고 입구가 모두 광장 안 | FR-060, US11-6 |
| 투영 (단계 13) | `toLogical(toScreen(p)) ≈ p`, 화면 대각선 속도 = 직선 속도 | FR-010, FR-038 |
| 부화 비중 1,000번 | 병아리·토끼·아기 돼지·송아지 비율이 40·30·20·10%에서 ±5%p 안 | FR-045, SC-009 |

## 3. E2E 실행

개발 서버가 떠 있는 상태에서 (첫 인자는 스크린샷 폴더):

```bash
node e2e/auth.mjs   e2e-shots/auth        # (auth 소유) 로그인 → /town
node e2e/blog.mjs   e2e-shots/blog        # (다른 시나리오의 전제)
node e2e/town.mjs   e2e-shots/town        # 새: 광장·환영·이동·입구·헤더·집 단계·지붕 색·FPS
node e2e/favorites.mjs e2e-shots/fav      # 새: 즐겨찾기·광장 집·패널·방문자 인기 블로그
node e2e/farm.mjs   e2e-shots/farm        # 고침: 성장 아이템, 꽉 참 문구, 레벨 알·동시 요청·조작 요청 추가
node e2e/mobile.mjs e2e-shots/mobile      # 고침: 휴대폰도 광장, 두 줄 헤더
node e2e/decisions.mjs e2e-shots/decide   # 고침: 이웃집 규칙(즐겨찾기), 조이스틱
node e2e/social.mjs e2e-shots/social      # (social 소유) 이웃 추가·취소 회귀 확인. 이웃 취소 → 즐겨찾기 해제는 e2e/favorites.mjs 4-8이 확인
node e2e/nonfunctional.mjs e2e-shots/nf   # 375px 가로 스크롤 (/town, /farm 포함)
```

새·고친 시나리오는 모두 `✅`/`❌` 줄을 출력하고 `❌`가 하나라도 있으면 종료 코드 1이다 (`e2e/decisions.mjs`·`e2e/mobile.mjs`처럼 지금 종료 코드가 없는 파일은 고칠 때 `exit(1)`을 더한다). 콘솔 오류 0개도 확인 항목이다.

## 4. 시나리오와 기대 결과

캔버스 안은 [contracts/town-screen.md](contracts/town-screen.md) 6절의 숨은 집 목록(`[data-town-houses]`)과 개발 모드 훅(`window.__blogvilleTown`)으로 확인한다.

### US1 광장에서 시작 (P1)

| # | 실행 | 기대 결과 | 검증 | SC |
|---|---|---|---|---|
| 1-1 | 아이디로 로그인 | 주소 `/town` | `e2e/town.mjs`, `e2e/auth.mjs` | SC-001 |
| 1-2 | 새 아이디로 가입 | 광장 가운데 위에 환영 문구(닉네임 굵게, `🪙 100`, 내 집·게시판·동물 농장 안내) | `e2e/town.mjs` | SC-002 |
| 1-3 | 새로고침 10번, `/feed` → 뒤로 가기 10번, `/town` 주소 다시 열기 | 문구 0번. 쿠키 `bv_welcome` 없음 | `e2e/town.mjs` | SC-002 |
| 1-4 | 문구의 [✕] | 문구가 닫히고 주소 그대로 | `e2e/town.mjs` | |
| 1-5 | 로그인 없이 `/town` | 캔버스, 훅 `player().name = "구경하는 중"`, 방향키로 위치가 바뀜 | `e2e/town.mjs` | SC-001 |
| 1-6 | 광장의 헤더 | 이동 메뉴 링크 0개, 나가기 버튼 0개 | `e2e/town.mjs`, `e2e/decisions.mjs` | |
| 1-7 | `/feed`·`/attendance`·`/shop`·`/farm`에서 [← 광장으로 나가기] | `/town` | `e2e/decisions.mjs` | |
| 1-8 | 375px 광장 밖 화면 | 나가기 버튼 글자 `← 나가기` | `e2e/mobile.mjs` | SC-004 |

### US2 이동 (P1)

| # | 실행 | 기대 결과 | 검증 |
|---|---|---|---|
| 2-1 | →를 1초, →+↓를 1초 | 둘 다 화면에서 약 230px(±15%) 이동, 대각선 거리 ≈ 직선 거리 | `e2e/town.mjs` (훅 `player().screen`) |
| 2-2 | 빈 땅 클릭 / 분수 너머 클릭 | 그 지점 근처에서 멈춤 / 분수에 막혀 멈춤 | `e2e/town.mjs` |
| 2-3 | 클릭으로 걷는 중 ← | 왼쪽으로 바뀜 | `e2e/town.mjs` |
| 2-4 | 가장자리로 `teleport` 뒤 바깥쪽 키 2초 | 논리 좌표가 0~1800 × 0~1400 안 | `e2e/town.mjs` |
| 2-5 | 375px 터치(`hasTouch`) 광장 | 왼쪽 아래 조이스틱 그림(스크린샷), 끌면 위치가 바뀜, 안내 `조이스틱이나 탭으로 이동 · 건물을 탭해서 들어가기` | `e2e/mobile.mjs`, `e2e/decisions.mjs`(태블릿) |
| 2-6 | 데스크톱 광장 | 안내 `방향키·WASD 또는 클릭으로 이동 · 건물 앞에서 Space로 들어가기`, 조이스틱 없음 | `e2e/town.mjs` |
| 2-7 | 건물 앞·뒤에 `teleport`해 스크린샷 | 앞에서는 캐릭터가 위, 뒤에서는 가려짐 | 스크린샷 확인 (손) |
| 2-8 | 광장에서 ↓·Space | `window.scrollY` = 0 그대로 | `e2e/town.mjs` |

### US3 건물에 들어가기 (P1)

| # | 실행 | 기대 결과 | 검증 |
|---|---|---|---|
| 3-1 | 입구마다 `teleport` | 훅 `prompt()` = `{아이콘} {이름} · Space 들어가기` (예: `📮 출석 체크 (오늘 완료) · Space 들어가기`) | `e2e/town.mjs` |
| 3-2 | 그 상태에서 Space / Enter | `/feed`, `/attendance`, `/shop`, `/farm`, `/@{내 주소}`, `/@{이웃 주소}` | `e2e/town.mjs` |
| 3-3 | 입구에서 200px 떨어져 Space | `prompt()` = `null`, 주소 그대로 | `e2e/town.mjs` |
| 3-4 | 멀리서 상점 클릭 → 도착 뒤 다시 클릭 | 문 앞까지 걸어가 `prompt()`가 상점 → `/shop` | `e2e/town.mjs` |
| 3-5 | 게시판 왼쪽 절반 / 오른쪽 절반 클릭(가까이에서) | `/feed` / `/attendance` | `e2e/town.mjs` |
| 3-6 | 오늘 출석된 회원 | 게시판 부제 `마을 소식 · 출석 체크 (오늘 완료 ✅)`, 안내 `출석 체크 (오늘 완료)` | `e2e/town.mjs` (훅 `entrances()`·스크린샷) |
| 3-7 | 방문자: 상점·출석 도장·농장 앞 → Space | `… · Space 로그인하고 이용하기` → `/` | `e2e/town.mjs` |
| 3-8 | 방문자: 마을 소식·이웃집 | 로그인 없이 `/feed`, `/@{주소}` | `e2e/town.mjs` |
| 3-9 | 회원: 내 집 | `/@{내 주소}` | `e2e/town.mjs` |

### US4 즐겨찾는 이웃 (P2) — `e2e/favorites.mjs`

| # | 실행 | 기대 결과 | SC |
|---|---|---|---|
| 4-1 | 이웃 3명 둔 회원이 내 블로그 홈 | `🏘 내 이웃` 목록 3줄, 줄마다 ☆ | |
| 4-2 | 다른 회원·방문자가 그 블로그 홈 | 내 이웃 목록 없음 (HTML에도 없음) | |
| 4-3 | ☆ → ⭐ → ☆ | DB `is_favorite` true → false | |
| 4-4 | 10명 즐겨찾기 뒤 11번째 ☆ | `즐겨찾기할 이웃은 최대 10명이에요`, DB 10행 | SC-006 |
| 4-5 | 9명에서 두 페이지가 동시에 서로 다른 이웃 ☆ | DB `is_favorite` 10행 이하 (5번 반복) | SC-006 |
| 4-6 | 10곳 즐겨찾기(그중 공개 글 없는 블로그 2곳) → `/town` | `[data-town-houses]`에 10곳 모두, 글 없는 블로그도 | SC-007 |
| 4-7 | 즐겨찾기 안 한 이웃 | 집 목록에 없음, 🏘 패널에는 링크 있음 | SC-007 |
| 4-8 | 즐겨찾은 이웃을 [✓ 이웃]으로 취소 → `/town` | 그 집 없음, DB 행 없음 | |
| 4-9 | 즐겨찾기 0명 회원 `/town` | 집 목록 0채, 패널에 `마음에 드는 블로그를 즐겨찾기하면 광장에 집이 생겨요` | |
| 4-10 | 이웃 0명 회원의 내 블로그 홈 | `아직 이웃이 없어요. 마을 소식에서 마음에 드는 블로그를 이웃으로 추가해 보세요.` | |
| 4-11 | `created_at`을 DB로 맞춘 글(즐겨찾은 이웃 3일 전·10일 전, 일반 이웃 어제) → `/feed/following` | 3일 전 → 어제 → 10일 전 순서 (social 구현) | |
| 4-12 | 이웃집에 들어가기 | 그 블로그 홈 | |
| 조작 | `toggleFavorite` 요청을 잡아 이웃 아닌 회원 ID·자기 ID·`"x".repeat(100)`으로 다시 보냄 | 거부, DB 변화 없음, 500 없음 | |

### US5 방문자 인기 블로그 (P2) — `e2e/favorites.mjs`

DB로 시험 블로그를 만든다: 공개 글이 있는 블로그 105곳(이웃 수를 서로 다르게 넣어 1~105위가 정해지게), 비공개 글만 있는 블로그 1곳, 글 없는 블로그 1곳. 끝나면 시험 회원을 지운다.

| # | 실행 | 기대 결과 | SC |
|---|---|---|---|
| 5-1·2 | 방문자로 `/town` 20번 | 매번 집 10채, 적어도 두 번은 조합이 다름 | |
| 5-3 | 방문자가 집에 들어가기 | 로그인 없이 블로그 홈 | |
| 5-5 | 위 20번 동안 | 비공개 글만·글 없는 블로그 0번 | SC-013 |
| 5-6 | 위 20번 동안 | 101~105위 블로그 0번 | SC-013 |
| 5-4 | `npm run db:reset && npm run admin:create` 직후(다른 시나리오보다 먼저) 공개 글 블로그 3곳만 만든 상태 (`admin:create`가 만드는 `/@notice`는 글이 없어 후보가 아니다) | 집 3채, 나머지 자리 빈칸. 공개 글 블로그가 이미 10곳 이상인 DB에서는 이 줄을 건너뛰고 단위 테스트 `sampleDistinct`(7개 → 7개)로 대신한다 | |

### US6 동물 농장 (P2) — `e2e/farm.mjs`

| # | 실행 | 기대 결과 | SC |
|---|---|---|---|
| 6-1·3~7 | (지금 `e2e/farm.mjs`에 있는 시나리오) 첫 알·알 사기(🪙 −100)·부화·돌보기 3가지(성장 25, 경험치 +2씩)·`완료`·공개 글 성장 +10 | 지금과 같음 | SC-008 |
| 6-2 | (새로 추가) DB 원장으로 경험치를 넣어 Lv.5로 만든 뒤 `/farm` | [🎁 레벨 5 보상 알 (무료)] 보임 → 누르면 알 1개 | |
| 동시 무료 알 | (새로 추가) 두 페이지가 같은 [🎁 농장 첫 알 (무료)]를 동시에 누름 | 알 1개만, 다른 쪽 `이미 받은 알이에요` | SC-008 |
| 동시 돌보기 | (새로 추가) 두 페이지가 같은 동물에 [🥕 밥 주기]를 동시에 누름 | `animal_cares` 1행, `farm_care` 원장 1줄, 다른 쪽 `오늘은 이미 밥 주기를 했어요` | SC-008 |
| 조작 (알) | (새로 추가) `claimEgg`를 잡아 아직 안 된 레벨(예: Lv.1 회원이 `level = 5`)로, `buyEgg`를 코인 50인 회원으로, `hatchEgg`·`careAnimal`을 남의 알·동물 ID로 다시 보냄 | `아직 받을 수 없는 알이에요` / `코인이 50개 부족해요` / `부화시킬 수 있는 알이 아니에요` / `돌볼 수 있는 동물이 아니에요`, DB 그대로 | |
| 6-8 | DB로 성장 아이템 수량을 넣고(또는 상점에서 사고) 동물에게 [동물 먹이 ×2] 두 번 | 성장 +20씩, 수량 2 → 0, 같은 날 두 번 됨, 원장 행 수 변화 없음 | |
| 6-9 | 수량 0에서 요청을 조작해 보냄 | `가지고 있는 성장 아이템이 없어요`, 성장 그대로 | |
| 조작 (아이템) | `applyGrowthItem`을 잡아 남의 동물 ID, 성장 아이템이 아닌 아이템 ID(배경), `0`·`2147483648`·`"1e3"`으로 다시 보냄 | `돌볼 수 있는 동물이 아니에요` / `가지고 있는 성장 아이템이 없어요` / `잘못된 요청이에요`, 수량·성장 그대로, 500 없음 | |
| 알 | 알 ID로 성장 아이템 요청 | `돌볼 수 있는 동물이 아니에요`, 수량 그대로 | |
| 넘침 | 남은 성장이 10인 동물에 촉진제(+100) | `다 자랐어요`, `growth = grow_exp`, 다른 동물 성장 그대로, `farm_grown` 1줄 | SC-008 |
| 동시 | 수량 1로 두 페이지가 동시에 사용 | 한쪽만 성공, 수량 0 (음수 없음) | |
| 6-10 | 다 자람 | 다 키운 동물 카드, `키우는 중` 하나 줄어듦 | SC-011 |
| 6-11 | 5마리 | 알 버튼 막힘 + `자리가 꽉 찼어요. 다 키운 뒤에 받을 수 있어요`. 요청을 조작해 보내도 같은 문구 | |
| 원장 | 끝에 | 헤더 코인 = 원장 합계 | SC-008 |
| 6-12 | [← 광장으로 나가기] | `/town` | |
| 375px | (`e2e/mobile.mjs`) 375px에서 `/farm`의 모든 버튼 크기, 가로 스크롤 | 버튼 44×44px 이상(돌보기·🌱 포함), 가로 스크롤 0 | SC-004 |

### US7 도감·전시 (P2)

blog 검증(`specs/002-blog/quickstart.md`)이 확인한다. town이 볼 것: `e2e/farm.mjs`에서 동물을 다 키운 직후 `/@{내 주소}`에 그 카드가 보인다(SC-011), `user_animals_user_id_id_uq`가 있어 blog의 복합 FK 마이그레이션이 적용된다.

### US8 헤더 (P2) — `e2e/town.mjs`, `e2e/mobile.mjs`

| # | 실행 | 기대 결과 | SC |
|---|---|---|---|
| 8-1 | 회원으로 아무 화면 | 왼쪽 로고, 오른쪽 상태창(프로필·닉네임·블로그 제목), `Lv.N`, `🪙 N`, 로그아웃 | |
| 8-2 | 로고 | `/town` | |
| 8-3 | `/settings/blog`에서 블로그 이름 저장 | 같은 화면 헤더 상태창에 새 이름 (새로고침 없이) | SC-010 |
| 8-3b | 꾸미기(`/closet`)에서 다른 캐릭터 장착 / 내 정보에서 닉네임 저장(blog 닉네임 칸이 생긴 뒤) | 같은 화면 헤더 상태창에 새 캐릭터 얼굴 / 새 닉네임 (새로고침 없이). 프로필 사진은 올리는 화면이 없어 DB로 넣은 8-6으로만 확인 | SC-010 |
| 8-4 | 375px | 두 줄 헤더, 블로그 제목 숨김, 프로필·닉네임·`Lv`·`🪙`·로그아웃 보임, 가로 스크롤 0, 누르는 것 44×44px 이상 | SC-004 |
| 8-4b | 640px·768px, 닉네임 20자·블로그 제목 40자인 관리자 회원으로 `/feed` | 가로 스크롤 0, 상태창 글자는 `…`로 줄어듦 (넘치면 두 줄 기준을 768px로, research R-11) | SC-004 |
| 8-5 | 방문자 | 상태창 없음, [시작하기] | |
| 8-6 | 프로필 사진 없는 회원 / DB로 `photo_key`를 넣은 회원 | 캐릭터 얼굴 / `/files/{key}` 그림 | |
| 8-7 | 상태창 누르기 | `/@{내 주소}` | |

### US9 아이소메트릭 (P2, 단계 13 뒤)

| # | 실행 | 기대 결과 | SC |
|---|---|---|---|
| 9-1 | `/town` 스크린샷 | 바닥이 마름모 타일, 캐릭터가 타일 위 | |
| 9-2 | US2·US3 표 전체를 다시 실행 (`e2e/town.mjs`, `e2e/mobile.mjs`) | 모두 `✅` | SC-012 |
| 9-3 | 코드·화면 검색 | 2D 광장을 고르는 설정·주소 없음 | |
| FPS | `e2e/town.mjs` 5초 이동 중 훅 `fps()` | 평균 55 이상, 최저 50 이상 (데스크톱 Chromium 참고값) | SC-005 |

### US10 지붕 색 (P3) — `e2e/town.mjs`

| # | 실행 | 기대 결과 |
|---|---|---|
| 10-1 | `/closet`의 `🏠 지붕 색`에서 파랑 → `/town` | 내 집 `data-roof` = 파랑 색 |
| 10-2 | 꾸미기에서 배경을 바꿈 → `/town` | 지붕 색 그대로 |
| 10-3 | 나를 즐겨찾기한 회원의 `/town` | 내 집 `data-roof` = 파랑 색 |
| 10-4 | [배경 색 따라가기] | `data-roof` = 장착 배경 강조색, DB `roof_color` NULL |
| 10-5 | `setRoofColor` 요청을 잡아 `"pink"`, `"#000000"`, `123`으로 다시 보냄 | `고를 수 없는 색이에요`, DB 그대로, 500 없음 |

### US11 집 단계 (P3) — `e2e/town.mjs` + `npm run test:town`

| # | 실행 | 기대 결과 | SC |
|---|---|---|---|
| 11-1~3 | DB 원장으로 Lv.1·9·10·29·30 회원을 만들고 한 회원이 모두 즐겨찾기 → `/town` | `data-stage` 1·1·2·2·3, 스크린샷에서 2단계에 창문, 3단계에 다락방 창 | SC-014 |
| 11-4 | Lv.9 회원에게 경험치를 더해 Lv.10 → 본인·즐겨찾은 회원·방문자(나올 때까지 최대 30번) 광장 | 모두 `data-stage = 2` | SC-014 |
| 11-5 | 각 단계 집 앞 `teleport` → Space | 그 블로그 홈 | |
| 11-6 | 3단계 집 10채 | 단위 테스트 겹침 없음 + 스크린샷(나무가 집·문 앞을 막지 않음, 각 집 `teleport` 뒤 `prompt()`가 그 집) | |
| 11-7 | `/shop`, `/closet` | 증축 메뉴 없음 | |

## 5. 손으로 확인

| 항목 | 방법 | 기대 | SC |
|---|---|---|---|
| 실제 휴대폰 | iOS Safari, Android Chrome에서 `/town` (375px 근처) | 조이스틱·탭으로 마을 소식·출석·상점·동물 농장·내 집에 처음 온 사람이 1분 안에 걸어 들어감 (팀원 아닌 사람 1~2명) | SC-003 |
| 실제 휴대폰 FPS | 원격 개발자 도구 성능 탭에서 10초 이동 | 약 60fps, 50 아래로 떨어지지 않음 | SC-005 |
| 화면 회전 | 휴대폰을 돌림 | 광장이 새 크기로 다시 맞춰짐, 조이스틱이 왼쪽 아래 | Edge Cases |
| 가림 | 큰 건물 옆·뒤에서 스크린샷 | 앞·뒤 판정이 자연스러움 | US2-7 |

## 6. 실패하면 볼 곳

- 환영 문구가 다시 보임: 응답 `Set-Cookie`의 `bv_welcome` Path·Max-Age, 브라우저에서 쿠키가 지워졌는지 ([contracts/town-screen.md](contracts/town-screen.md) 3절).
- 즐겨찾기가 10을 넘음: `toggleFavorite`가 `lockUser` 안에서 세는지 ([contracts/favorites.md](contracts/favorites.md) 2절).
- 집 단계가 다름: 그 회원의 `SELECT SUM(exp_delta) FROM point_ledger WHERE user_id = …`와 `levelFromExp` 결과.
- 375px 가로 스크롤: `e2e/nonfunctional.mjs` 결과의 넘친 px, 헤더 두 줄 배치 ([contracts/header-roof.md](contracts/header-roof.md) 1.1).
