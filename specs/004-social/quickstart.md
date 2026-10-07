# Quickstart: 교류 (SOCIAL) 검증

**Feature**: `004-social` | **Plan**: [plan.md](plan.md) | **Contracts**: [contracts/](contracts/)

구현이 끝난 뒤 이 기능이 처음부터 끝까지 동작하는지 확인하는 순서다. 명령은 코드 저장소(`blogville`) 루트에서 실행한다. 비밀값(DB 접속 정보, 관리자 비밀번호)은 `.env.local`에만 있고 여기에 적지 않는다. 결과(✅/❌ 줄, 스크린샷)는 PR에 붙인다 (constitution 품질 관문).

## 0. 준비

| 항목 | 내용 |
|---|---|
| 선행 작업 | auth 단계 1(가입 통합, `e2e/helpers.mjs`의 `loginDev`가 온보딩 없이 동작) merge. 2차 검증(7절)은 game 단계 6 merge |
| 환경 변수 (`.env.local`, 이름만) | `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `ADMIN_USERNAME`, `ADMIN_PASSWORD` |
| DB | 로컬 PostgreSQL (README 안내 17). `npm run db:reset`은 로컬 DB에서만 동작 |
| 패키지 | README 안내대로 설치된 상태 (`node_modules`) |
| 스크린샷 폴더 | 예: `shots/` (각 e2e의 첫 인자) |

## 1. 기존 데이터 이전 확인 (U1~U3, 한 번만)

`db:reset`보다 **먼저** 한다. 지금 `main`의 DB에 답글이 있는 상태에서 새 마이그레이션이 답글을 잃지 않는지 본다.

1. `main` 체크아웃 상태에서 개발 서버(`npm run dev`)를 띄우고 `node e2e/blog.mjs shots`를 돌린 뒤, 브라우저에서 `tester2`로 그 글의 댓글에 답글 2개를 달고 그중 하나를 지운다. 원댓글 하나도 지운다.
2. `DATABASE_URL`의 DB에 `psql`(PostgreSQL과 함께 설치되는 클라이언트, 저장소 npm 스크립트가 아님)이나 `npm run db:studio`로 접속해 이전 전 값을 적는다. 접속 정보는 `.env.local`에서 읽고 여기 적지 않는다.
   - `SELECT count(*) FROM comments WHERE parent_id IS NOT NULL;` → A
   - `SELECT count(*) FROM comments WHERE parent_id IS NULL;` → B
3. 이 기능 브랜치로 바꾸고 `npm run db:migrate`.
4. 이전 후 기대 결과:

| 확인 쿼리 (요지) | 기대 |
|---|---|
| `replies` 행 수 | A (data-model 6.2의 "다른 글 부모"가 0일 때) |
| `comments` 행 수 | B |
| `comments`에 `parent_id` 컬럼 | 없음 (`information_schema.columns`) |
| `comments`·`replies`에서 `deleted_at IS NOT NULL AND content <> ''` | 0 |
| 옮긴 답글의 `id` | 이전 전 그 답글의 댓글 `id`와 같다 |
| `point_ledger` 행 수 | 이전 전과 같다 |

5. 개발 서버에서 그 글을 열어 답글이 원댓글 아래 들여쓰기로, 지운 답글·원댓글이 `삭제된 댓글이에요`로 보이는지 스크린샷(`shots/00-migrated.png`).

## 2. 정적 검사·단위 테스트

```bash
npx tsc --noEmit
npx eslint
npm test
```

기대: 오류 0. `npm test` 끝의 `test:social`(새)이 모두 ✅.

| `scripts/test-social.ts`(새) 항목 | 기대 | 연결 |
|---|---|---|
| 내용 정규화: `"  안녕 \r\n반가워  "` | `"안녕 \n반가워"` | FR-007, research R6 |
| 공백만 / 빈 값 | `댓글을 적어 주세요` | FR-008 |
| 정확히 1000자 / 1001자 | 통과 / `댓글은 1000자까지예요` | FR-008 |
| CRLF 줄바꿈 10개 포함 1000자(LF 기준) | 통과 | research R6 |
| `canDeleteComment` 작성자 / 블로그 주인 / 관리자 / 남 / 방문자 | true / true / true / false / false | FR-013, D8 |
| `favoriteWindowStart("2026-10-07")` (즐겨찾기 우선 시작 날짜, `listFeed`가 쓰는 함수) | `2026-10-01` (오늘 포함 7일) | FR-042, research R11 |
| `favoriteWindowStart("2026-03-03")` (월 경계) | `2026-02-25` | FR-042 |

## 3. DB 준비와 개발 서버

```bash
npm run db:migrate && npm run db:seed && npm run db:reset && npm run admin:create
npm run dev        # 다른 터미널에서 띄워 둔다 (http://localhost:3000)
```

- 초기화 직후 브라우저로 `/feed`를 열어 `아직 마을에 글이 없어요. 첫 글의 주인공이 되어 보세요! ✏️`와 `아직 태그가 없어요`를 확인한다 (US1-4, 스크린샷 `shots/01-feed-empty.png`). 관리자 블로그 `/@notice`에는 글이 없으므로 빈 마을이다.
- 이어서 관리자로 태그 달린 **비공개** 글 하나만 써 두고 `/feed`를 다시 열어도 같은 두 문구가 보이는지 확인한 뒤 그 글을 지운다 (US1-4 "비공개 글만 있는 경우 포함", FR-050·051).

## 4. E2E

순서대로 돌린다. 새로 만들거나 고치는 `comments.mjs`·`social.mjs`·`params.mjs`는 ❌가 하나라도 있으면 종료 코드 1이다. `blog.mjs`는 지금처럼 결과만 출력하므로(종료 코드 없음) 출력 줄을 눈으로 확인한다.

```bash
node e2e/blog.mjs shots       # tester1 글, tester2 공감·댓글 (다른 시나리오의 전제, 회귀)
node e2e/comments.mjs shots   # 새: SOC-01·SOC-02
node e2e/social.mjs shots     # 고침: SOC-03·SOC-04·SOC-05
node e2e/params.mjs shots     # 고침: 범위 밖·형식 오류 값
```

### 4.1 `e2e/comments.mjs` (새, 실행마다 새 회원 A·B·C + 관리자)

준비: A가 공개 글 2개(P1, P2)와 비공개 글 1개(Q)를 쓴다(DB로 넣어도 된다). B·C는 다른 회원. 관리자는 `.env.local`의 `ADMIN_*`로 로그인.

| 확인 | 기대 결과 | 연결 |
|---|---|---|
| B가 P1에 댓글 등록 | 새로고침(페이지 이동) 없이 목록 맨 아래에 보임, 입력칸 빔, `💬 댓글 N` +1, B 코인 +5 | US2-1, FR-009·010, SC-008 |
| 처리 중 버튼 | `등록 중...`, 비활성 | FR-009 |
| A가 자기 글 P1에 댓글 | 등록, A 원장 `comment` 행 늘지 않음 | US2-2 |
| A가 자기 비공개 글 Q에 댓글 | 등록됨(내 글은 볼 수 있는 글), 보상 없음 | FR-006 |
| 댓글 입력 중 다른 탭에서 로그아웃(또는 세션 쿠키 삭제) → [댓글 등록] / 같은 상태로 공감·[답글 등록]·[삭제] | 첫 화면 `/`로 이동, `comments`·`replies`·`post_likes` 행 수 그대로 | FR-002, Edge Cases |
| 공백만 등록 | `댓글을 적어 주세요`, 행 수 그대로, 입력칸 내용 유지 | US2-4 |
| 1001자 요청(입력칸 `maxLength`를 지우고 보냄) | `댓글은 1000자까지예요`, 저장 안 됨 | US2-5 |
| 폼의 `postId`를 Q(남의 비공개 글)·없는 글로 바꿔 등록 | `글을 찾을 수 없어요`, 저장 안 됨 | US2-6, SC-005 |
| 방문자로 P1 | `로그인하면 댓글을 남길 수 있어요`(링크 `/`), [답글]·[삭제] 없음 | US2-8, FR-012 |
| B가 자기 댓글 삭제 → 확인 창 [취소] | 아무 변화 없음 | US2-10 |
| B가 자기 댓글 삭제 → [확인] | 같은 자리에 B 캐릭터·닉네임·시각 + `삭제된 댓글이에요`, `💬 댓글 N` −1, B 원장 그대로, DB `content = ''` | US2-9, FR-014·015 |
| 다른 브라우저로 그 글 HTML·RSC 응답 본문 검색 | 지운 댓글 원래 내용 0건, 응답에 작성자 회원 ID 없음 | US2-12, SC-006 |
| 블로그 주인 A가 C의 댓글 삭제 | 삭제 자리 남음, `💬` −1, C 원장 그대로 | US2-13, D8 |
| 관리자가 남의 글의 남의 댓글 삭제 | 삭제 자리 남음 | US2-14, D8 |
| C로 A의 글을 볼 때 [삭제] | C의 댓글에만 있음 | US2-15 |
| C가 B의 댓글 삭제 요청을 조작(액션 재전송) / 이미 삭제된 댓글 / ID `99999999999` | DB 변화 0, HTTP 200, 문구·오류 화면 없음 | US2-11, SC-005 |
| B가 P1의 댓글 [답글] | 버튼 줄 아래·기존 답글 위에 2줄 입력칸 `답글을 남겨 주세요`, 버튼 [답글 취소] | US5-1 |
| [답글 취소] | 입력칸 닫힘, 다시 열면 비어 있음 | US5-2 |
| 답글 2개 등록 | 원댓글 아래 왼쪽 선 들여쓰기, 오래된 순, 입력칸 닫히고 [답글] | US5-3, FR-022 |
| 답글 줄 | [답글] 없음, [삭제]는 답글 작성자·A·관리자에게만 | US5-4 |
| 원댓글 삭제 뒤 | 원댓글 자리 + 답글 2개 그대로, `💬 댓글 2`, [답글] 없음 | US5-5·7 |
| 답글 입력칸을 연 채 다른 브라우저에서 원댓글 삭제 → [답글 등록] | `삭제된 댓글에는 답글을 달 수 없어요`, 입력 내용 그대로, 저장 0 | US5-10, SC-010 |
| `commentId`를 없는 ID·P2의 댓글 ID로 조작 | `답글을 달 댓글이 없어요`, 내용 유지, `comments`·`replies` 행 수 그대로 | US5-11, SC-010 |
| `commentId`를 `99999999999`·`abc` | `잘못된 요청이에요` | US5-9 |
| 답글 공백 / 1001자 / 그사이 글 삭제 | `댓글을 적어 주세요` / `댓글은 1000자까지예요` / `글을 찾을 수 없어요` | US5-8 |
| A가 B의 답글 삭제 | 답글 자리 `삭제된 댓글이에요`, `💬` −1 | US5-12, FR-024 |
| 새 회원 D가 하루에 남의 글에 댓글 6 + 답글 5 | 모두 등록, D 원장 `comment` 10행(✨50·🪙50) | US5-6, SC-004 |
| 새 회원 E가 하루에 남의 글에 댓글 11개 | 11개 등록, 원장 10행, 11번째 안내 없음 | US2-3, SC-004 |
| A가 자기 글에 단 답글 | 보상 없음 | US5-6 |
| 내용 `<b>굵게</b> https://example.com` + 줄바꿈 | 글자 그대로, 링크 아님, 줄바꿈 보임 | FR-007 |
| 글 P2 삭제 | P2의 `comments`·`replies` 행 0 | FR-017 |
| 키보드만(Tab → 입력 → Tab → Enter)으로 댓글·답글 등록 | 등록됨, 선택된 곳이 보임 (스크린샷) | FR-005 |
| 375px에서 P1 (답글 포함) | `scrollWidth <= clientWidth`, 버튼 글자 한 줄, [답글]·[답글 취소]·[삭제]·[댓글 등록]·[답글 등록]과 닉네임·`로그인` 링크의 누르는 영역 ≥ 44×44px (`getBoundingClientRect`) | FR-004, SC-002, constitution VI |

### 4.2 `e2e/social.mjs` (고침)

| 확인 | 기대 결과 | 연결 |
|---|---|---|
| 방문자로 `/feed` | 목록 보임, [🏘 마을 전체]만, [💛 이웃 새 글] 없음, 탭 제목 `마을 소식 \| Blogville` | US1-1, FR-046·049 |
| 새 회원 F의 공개 글 9개 + 비공개 1개(DB로 최신 시각) 뒤 `/feed` (방문자·F·관리자 각각) | 1페이지 8개 모두 공개, 최신순, 9번째는 2페이지 첫째, 비공개 배지 없음 | US1-2·6, FR-047 |
| 카드 작성자 줄 / 나머지 클릭 | 블로그 홈 / 글 상세 | US1-3, FR-048 |
| `/feed?page=999` | 빈 목록 문구, 페이지 번호 있음·현재 표시 없음 | Edge Cases, FR-051 |
| 이웃 글이 있는 회원의 `/feed/following?page=999` | 이웃 새 글 빈 화면 두 줄 문구 | Edge Cases, FR-043 |
| 태그 3개(달린 횟수 다르게, 같은 횟수 2개)를 DB로 준비 | `#태그 N`이 횟수 많은 순, 같으면 이름순, 최대 30개, 누르면 `/tags/...` | US1-5, FR-050 |
| [+ 이웃 추가] / [✓ 이웃] (기존) | `이웃 N` +1 / −1, 확인 창 없음 | US4-1·2 |
| 이웃 새 글 | 이웃 B의 공개 글(이웃 추가 전 글 포함)만, 비공개·이웃 아닌 글 0 | US4-3, SC-009 |
| 이웃 취소 뒤 이웃 새 글 | B 글 빠짐 | US4-4 |
| 이웃 0명 회원의 이웃 새 글 | 두 줄 문구, "마을 소식" 링크 `/feed` | US4-5, FR-043 |
| 내 블로그 홈 | 이웃 버튼 대신 [✏️ 글쓰기] [🎨 꾸미기] [⚙️ 관리], 자기 ID로 조작한 요청 저장 0 | US4-6, SC-005 |
| 방문자로 `/feed/following` | `/`로 이동 | US4-7, FR-044 |
| 없는 회원 ID / 블로그 없는 `users` 행(DB로 직접 넣음) / 숫자·객체 인자로 조작 | 이웃 0, HTTP 200 | US4-8, FR-040 (기존 "온보딩 전 회원" 확인을 대체) |
| 같은 대상 이웃 추가 요청 2개 동시 | `follows` 1행 | US4-9, FR-039 |
| 이웃 추가·취소 전후 원장 | 변화 0 | US4-10 |
| 즐겨찾은 이웃 C의 3일 전·10일 전 글, 일반 이웃 B의 어제 글 (DB로 `created_at`·`is_favorite` 준비) | 순서 C(3일 전) → B(어제) → C(10일 전) | US4-11, D15 |
| 즐겨찾은 이웃 C의 6일 전(오늘 포함 7일째) 글 / 7일 전 글 | 6일 전은 맨 위 묶음, 7일 전은 일반 순서 | research R11 경계 |
| G가 H의 글에 공감 | `♥ 공감 N+1`, 분홍 테두리, `aria-pressed="true"`, H 코인 +2 | US3-1 |
| 다시 누름 | `♡ 공감 N`, 수 −1 | US3-2 |
| 탭 1에서 공감한 뒤, 새로고침하지 않은 탭 2의 [♡ 공감 N]을 누름 | 서버가 취소로 처리: 잠깐 ♥였다가 ♡로 돌아오고, `post_likes` 0행, 오류 화면 없음 | Edge Cases, FR-025 |
| 공감 → 취소 → 공감 3회 반복 | 공감 1개, H의 그 글 `like_received` 원장 1행 | US3-3, SC-004 |
| H가 자기 글에 공감 | 수에 들어감, 보상 없음 | US3-4 |
| 방문자 | 수 보임, 버튼 비활성, 버튼 아래 `로그인하면 공감할 수 있어요` 글자 보임 | US3-5, FR-030 |
| 두 탭에서 동시에 첫 공감, 10회 반복 | 매번 `post_likes` 1행, 오류 화면·콘솔 오류 0 | US3-6, SC-003 |
| 비공개 글·`abc`·`99999999999`로 공감 조작 / 누르는 사이 글을 비공개로 바꿈 | 저장 0, 하트·수 누르기 전으로, 오류 화면 없음 | US3-7, FR-031 |
| 카드 `♥ N` vs 상세 `공감 N`, 카드 `💬 N` vs 상세 `💬 댓글 N` | 같은 수 | US3-8, FR-016·032 |
| DB에 직접: 같은 공감 두 번 / 자기 자신 이웃 / 삭제 행 내용 넣기 | SQLSTATE 23505 / 23514 / 23514 | FR-026, FR-038, research R3 |
| 375px `/feed`, `/feed/following`, 블로그 홈 이웃 버튼 | 가로 스크롤 0, 인기 태그 칩 칸이 목록 아래, 이웃 버튼 글자 한 줄, 이웃 버튼 누르는 영역 ≥ 44×44px | US1-8, FR-053, SC-002 |
| 375px 탭 [🏘 마을 전체]·[💛 이웃 새 글], 인기 태그 칩, 페이지 번호 | 누르는 영역 ≥ 44×44px (post가 D-10을 반영한 뒤 켠다. 그 전에는 ❌ 대신 "D-10 대기"로 출력) | constitution VI, plan D-10 |
| 마을 소식에서 카드 한 번으로 글 상세 | 클릭 1번에 글 상세 (광장 입구 → 마을 소식은 town 검증) | SC-008 |

### 4.3 `e2e/params.mjs` (항목 추가)

| 확인 | 기대 |
|---|---|
| `/feed?page=abc`, `/feed?page=0`, `/feed?page=2.5`, `/feed?page=1e300`, `/feed?page=99999999999999999999` | HTTP 200, 1페이지 (US1-7, FR-052) |
| 댓글 폼 `postId` = `abc`, `0`, `2.5` | `잘못된 요청이에요` (지금은 영어 문구가 나오는 값) |
| 답글 폼 `commentId` = `99999999999`, `abc` | `잘못된 요청이에요`, 저장 0 |
| `deleteReply` 인자 `99999999999`, `"abc"` | HTTP 200 |
| 스크립트 전체 | 서버 오류(500)·영어 오류 문구 0 (SC-007) |

## 5. 성능 (SC-001)과 모바일 (SC-002)

```bash
npm run build && npm run start          # 프로덕션 빌드 (개발 서버는 끄고, 이 명령은 다른 터미널에 띄워 둔다)
node e2e/auth.mjs shots                 # 기존(auth 소유): nonfunctional.mjs가 로그인할 normal01 계정을 만든다 (3절 db:reset으로 지워졌으므로)
node e2e/feed-load.mjs shots            # 새: 공개 글 1,000개를 SQL로 넣고 /feed 측정 후 지운다
node e2e/nonfunctional.mjs shots        # 기존: 375px 가로 스크롤, /feed 응답 시간 (normal01로 로그인)
```

| 확인 | 기대 | 연결 |
|---|---|---|
| `e2e/feed-load.mjs`: 공개 글 1,000개에서 `/feed` 첫 화면(캐시 없는 새 브라우저 컨텍스트, `load`까지) | 로컬 기준 1초 안 (배포 환경 측정은 NF-08 결정 뒤) | SC-001, NF-07 |
| `e2e/feed-load.mjs`: 같은 데이터로 `/feed/following`(이웃 50명 중 즐겨찾기 10명) | 1초 안 | FR-042 (참고 값) |
| `e2e/nonfunctional.mjs` `/feed` 줄 | ✅ (문서 너비 ≤ 화면) | SC-002 |

## 6. 화면 확인 (스크린샷)

`shots/`의 다음 그림을 PR에 붙인다: 댓글·답글·삭제 자리(PC, 375px), 방문자 댓글·공감 영역, 이웃 버튼 두 상태, 마을 소식·이웃 새 글(PC, 375px), 1절 이전 결과.

## 7. 2차: 알림 발생 (game 단계 6 뒤)

`e2e/comments.mjs`·`e2e/social.mjs`의 2차 항목을 켠다 (알림 표 행을 DB로 센다. 알림함 화면은 game의 e2e가 본다).

| 확인 | 기대 | 연결 |
|---|---|---|
| B가 A의 글에 공감 | A에게 공감 알림 1행 (행동한 회원 B, 관련 글) | US3-9, FR-033 |
| A가 자기 글에 공감 | 알림 0 | FR-033 |
| B가 A의 글에 댓글 / A가 자기 글에 댓글 | A에게 댓글 알림 1행 / 0 | US2-16, FR-055 |
| B가 C의 댓글(A의 글)에 답글 | C에게 답글 알림 1행, A에게는 없음 | US5-13, FR-056 |
| C가 자기 댓글에 답글 | 알림 0 | FR-056 |
| 공감 → 취소 → 공감 | 공감 알림 2행 (취소로 지워지지 않음) | contracts/notification-triggers.md |
| 알림 기록 실패를 흉내(테스트 DB에서 알림 표 권한 제거 등, 선택) | 공감·댓글도 저장되지 않음 | 같은 트랜잭션 |
| 알림 링크 `/@{slug}/{postId}#comments` | 댓글 영역으로 스크롤 | game FR-044 |

## 8. 완료 기준

- 2절 전부 통과, 4절의 `comments.mjs`·`social.mjs`·`params.mjs` 종료 코드 0(`blog.mjs`는 출력 확인), 5절 SC-001 값 기록, 6절 스크린샷 첨부.
- `README.md` 스크립트 표에 `node e2e/comments.mjs <폴더>`, `node e2e/feed-load.mjs <폴더> [주소]` 두 줄이 있고, `e2e/social.mjs` 설명이 "이웃·공감·마을 소식"으로 바뀌어 있다.
- `docs/02-erd.md`의 comments·replies·follows 부분이 data-model 8절대로 고쳐져 있다.
