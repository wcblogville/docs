# Implementation Plan: 교류 (SOCIAL) — 댓글·답글·공감·이웃·마을 소식

**Branch**: `004-social` | **Date**: 2026-10-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/004-social/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

마을 소식(SOC-05)·댓글(SOC-01)·공감(SOC-03)·이웃(SOC-04)은 이미 동작한다. 이 plan은 2026-10-07 결정과 clarify 결정(D2·D8·D9·D11·D15)에 맞게 다음을 고친다.

1. **답글을 `replies` 표로 분리**(SOC-02, ERD 3.8): 지금은 `comments.parent_id`(자기 참조)다. 새 표를 만들고 기존 답글 행을 옮긴 뒤 `parent_id`를 지운다. 답글의 답글은 구조상 만들 수 없게 된다.
2. **답글 거부 규칙(D9)**: 없는 댓글·다른 글의 댓글·삭제된 댓글을 대상으로 한 답글은 저장하지 않고 `답글을 달 댓글이 없어요` / `삭제된 댓글에는 답글을 달 수 없어요`를 보여 주며, 입력한 내용은 입력칸에 남긴다. 원댓글 행을 `FOR SHARE`로 잠가 "삭제와 동시에 답글"도 막는다.
3. **삭제 권한(D8)**: 댓글·답글은 작성자, 그 글이 올라간 블로그의 주인, 관리자가 지울 수 있다. 권한은 `UPDATE ... WHERE` 한 문장의 조건으로 서버에서 판단하고, 화면의 [삭제]는 서버가 계산한 `canDelete`로만 보인다.
4. **삭제 표시 = 내용 비우기**: 삭제하면 `deleted_at`을 기록하고 `content`를 빈 글자로 바꾼다. CHECK로 "삭제된 행은 내용이 비어 있다"를 DB가 지킨다(SC-006, constitution VII).
5. **탈퇴 대비 구조(D2, 요청: auth AUTH-06)**: `comments.author_id`를 NULL 허용 + `ON DELETE SET NULL` + CHECK(작성자가 없으면 반드시 삭제 표시)로 바꾸고, auth의 탈퇴 트랜잭션이 부를 도우미를 새 파일 `src/server/social.ts`에 둔다. 답글(`replies.author_id`)은 `CASCADE`다.
6. **공감 동시성(FR-026)**: 첫 공감 insert에 `onConflictDoNothing()`을 붙여 동시에 두 번 눌러도 오류 화면 없이 1개만 남긴다.
7. **입력 검증 한국어화(FR-003, NF-19)**: `z.coerce.number()`가 `abc`·`0`·`2.5`에 영어 문구를 내는 문제를 `parseId()`로 바꾸고, 줄바꿈(CRLF)을 정규화한 뒤 1~1000자를 센다.
8. **댓글 수 = 댓글 + 답글(FR-016)**: 계산식을 `src/server/social.ts` 한 곳에 두고 글 상세·글 카드·관리자 통계가 같은 식을 쓴다.
9. **이웃 새 글 순서(D15)**: `follows.is_favorite`(요청: town, TOWN-08)를 추가하고, 이웃 새 글은 "오늘 포함 최근 7일(한국 시간) 안에 쓴 즐겨찾은 이웃 글"을 맨 위에 둔다.
10. **알림 발생(D11, 2단계)**: game이 알림 표와 기록 도우미를 만든 뒤(단계 6), 공감·댓글·답글 트랜잭션 안에서 그 도우미를 부른다(단계 7).

새 패키지는 없다. 기술 선택의 근거는 [research.md](research.md), 표 구조는 [data-model.md](data-model.md), 바깥 인터페이스는 [contracts/](contracts/), 검증 순서는 [quickstart.md](quickstart.md)에 있다.

## Technical Context

**Language/Version**: TypeScript ^5 (`strict: true`, 경로 별칭 `@/*` → `src/*`), Node.js 20.9 이상 (README). React 19.2.8, Next.js 16.3.8 App Router

**Primary Dependencies**: `next` 16.3.8 (Server Component·Server Action, `revalidatePath`), `react` 19.2.8 (`useActionState`, `useOptimistic`, `useTransition`), `drizzle-orm` ^0.45.3 / `drizzle-kit` ^0.31.11, `pg` ^8.23.1, `better-auth` ^1.7.7 (세션과 `users.role`), `zod` ^4.6.5, `tailwindcss` ^4. 새 의존성 없음

**Storage**: PostgreSQL (README 설치 안내 17). 변경하는 표: `comments`, `replies`(새), `follows`(`is_favorite`). 그대로 쓰는 표: `post_likes`. 읽는 표: `posts`, `blogs`, `profiles`, `items`, `users`. 보상은 `grantReward()`로 `point_ledger`(game)에 쓴다. 2단계에서 game의 알림 표에 쓴다

**Testing**: `npm test`(테스트 프레임워크 없이 `tsx scripts/test-*.ts`) + 새 `npm run test:social`(`scripts/test-social.ts`). E2E는 `@playwright/test`의 `chromium`을 쓰는 Node 스크립트: 새 `e2e/comments.mjs`, 고칠 `e2e/social.mjs`·`e2e/params.mjs`, 새 `e2e/feed-load.mjs`(SC-001), 회귀 `e2e/blog.mjs`·`e2e/nonfunctional.mjs`. CI가 없어 PR 작성자가 직접 돌린다 (constitution 품질 관문)

**Target Platform**: 웹 브라우저(PC와 375px 휴대폰), 서버는 Node.js(`next start`). 배포 환경은 미정(NF-08)

**Project Type**: 웹 애플리케이션, Next.js 단일 프로젝트 (화면·Server Action·DB 쿼리가 한 저장소)

**Performance Goals**: SC-001 — 공개 글 1,000개에서 마을 소식 첫 화면이 캐시 없는 첫 방문 기준 1초 안 (NF-07, 배포 환경). 이웃 새 글도 같은 목록 규칙(한 페이지 8개)

**Constraints**: 권한·입력 검증은 서버(constitution IV), 보상과 저장은 한 트랜잭션·원장(V), 숫자 ID는 `parseId()`(1~2147483647), 하루 기준은 한국 시간(`todayKST()`, `startOfTodayKST`), 오류 문구는 spec 문구 그대로(한국어), 375px 가로 스크롤 0·누르는 영역 44×44px(VI), 스키마는 `src/db/schema.ts` → `npm run db:generate` → `npm run db:migrate`. 이 체크아웃에는 `node_modules`가 없어 Next.js 16·Drizzle·React 19 세부 동작은 구현 전에 설치된 패키지 문서로 확인한다 (research.md R19)

**Scale/Scope**: 3명 팀 프로젝트. 회원 수백 명·공개 글 1,000개 수준을 목표로 잡는다(SC-001 기준). User Story 5개, FR 56개. 코드 변경은 서버 2파일·화면 3파일·스키마·마이그레이션 4개·테스트 4파일 정도

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### 설계 전 평가

| 원칙 | 판정 | 근거 |
|---|---|---|
| I. 글쓰기가 먼저 | 통과 | 댓글·공감·이웃은 읽기·교류를 돕는 기능이다. 보상 값(댓글 ✨5·🪙5 하루 10번, 공감 ✨2·🪙2 하루 20번)은 game의 `REWARD_RULES`를 그대로 쓰고 새 보상을 만들지 않는다. 이웃 추가에는 보상이 없다(FR-041) |
| II. 요구사항 ID 유지 | 통과 | 모든 변경 단위·검증 항목이 SOC-01~05와 FR·SC 번호, clarify 결정 D2·D8·D9·D11·D15를 가리킨다. 마이그레이션 SQL 맨 위 주석에도 요구사항 ID를 단다 (코드 저장소 관례) |
| III. 확인할 수 있는 수용 기준 | 통과 | spec의 수용 시나리오와 SC를 quickstart의 검증 항목에 1:1로 연결했다. 화면 문구는 spec 문구를 그대로 쓴다. spec의 "최근 7일(한국 시간)" 경계는 research R11에서 정했다 |
| IV. 권한과 검증은 서버에서 (NON-NEGOTIABLE) | 통과 (지금 코드의 빈틈은 이 plan에서 고친다) | 모든 쓰기는 Server Action 안에서 `requireMember()`·`parseId()`·글 공개 여부·삭제 권한을 다시 확인한다. 지금 코드의 빈틈: (1) `addComment`의 `z.coerce.number()`가 `abc`·`0`에 영어 문구 (2) `toggleFollow` 인자 타입 미확인 (3) 답글 대상이 잘못되면 일반 댓글로 저장 (4) 삭제된 댓글에도 답글 저장 — 모두 R6·R10·R5에서 고친다. 쿼리는 Drizzle 값 바인딩만 쓴다 |
| V. 원장 무결성 (NON-NEGOTIABLE) | 통과 | 댓글·답글 저장과 댓글 보상, 공감 저장과 공감 보상은 한 트랜잭션 안에서 `lockUser()` → `grantReward()`로 한다. "한 글에 공감 한 번"은 `post_likes` PK, "같은 이웃 한 번"은 `follows` PK, "자기 자신 이웃 불가"는 CHECK, "답글의 답글 없음"은 표 구조, "삭제된 행은 내용 없음"·"작성자가 없으면 삭제 표시"는 CHECK가 막는다. 구조 변경은 마이그레이션 파일로만 한다. "같은 사람·같은 글 공감 보상 1번"은 지금도 ERD 3.6 설계(회원 잠금 + 원장 확인)로 지키지만, 원칙 V는 "DB 제약조건으로도"를 요구하므로 game에 `point_ledger` 부분 UNIQUE를 **필요한 요청**으로 보낸다(의존성 D-7, game이 출석에 둔 `point_ledger_attendance_uq`와 같은 방식). game이 받지 않으면 Complexity Tracking에 이유를 적는다 |
| VI. 모바일 | 통과 (화면 보완 포함) | 375px 가로 스크롤 0을 `e2e/comments.mjs`·`e2e/nonfunctional.mjs`로 잰다. 지금 [답글]·[삭제]는 `text-xs` 글자 버튼이고, [댓글 등록]·[답글 등록]과 마을 소식 탭은 `btn py-1.5 text-sm`(높이 약 32px, 계산값)이라 44×44px보다 작다 → social 소유 화면은 직접 키우고(R17), post 소유 `feed-view.tsx`·`pagination.tsx`는 요청한다(D-10). 버튼 글자는 `whitespace-nowrap` |
| VII. 단순하게, 최소 정보 | 통과 | 새 표는 ERD가 이미 정한 `replies` 하나다. 삭제한 댓글·답글의 내용을 DB에서도 지우고, 화면으로 넘기는 데이터에서 작성자 회원 ID(`authorId`)를 뺀다(R13). 공감한 사람 목록·댓글 수정 같은 범위 밖 기능은 넣지 않는다 |

**보안·무결성 기준**: Server Action은 Next.js가 Origin을 확인해 다른 사이트 요청을 막는다(이 기능은 Route Handler를 새로 만들지 않는다). 로그인 쿠키·세션은 auth 영역 그대로다.

### 설계 후 재평가 (Phase 1 뒤)

| 원칙 | 판정 | 설계에서 확인한 것 |
|---|---|---|
| I | 통과 | 보상 규칙 변경 없음. 답글 보상은 사유 `comment`를 그대로 써서 "댓글과 합쳐 하루 10번"(FR-023)이 원장 집계로 저절로 지켜진다 |
| II | 통과 | data-model·contracts·quickstart의 모든 항목에 FR/SC 번호가 있다 |
| III | 통과 | contracts에 입력·결과·문구를 표로 적었고, quickstart가 각 수용 시나리오를 검증 명령에 연결한다 |
| IV | 통과 | contracts의 모든 Server Action이 "로그인 → ID 형식 → 내용 → 글 공개 여부 → 대상 행 상태" 순서로 서버에서 판단한다. 삭제 권한은 SQL 조건 하나로 판단해 화면 값(`canDelete`)과 상관없이 거부된다 |
| V | 통과 (D-7 조건부) | 잠금 순서(행 잠금 → 회원 잠금)가 `savePost`(회원 잠금 → 글 UPDATE)와 교착하지 않도록 글 행은 `FOR KEY SHARE`, 원댓글 행은 `FOR SHARE`로 정했다(R5). 2단계 알림도 같은 트랜잭션 안에서 기록한다. 공감 보상 1번의 DB 제약은 game의 D-7 수용에 달려 있다 (game data-model에는 아직 없다) |
| VI | 통과 (D-10 조건부) | 답글 들여쓰기(`ml-10`)·긴 내용 `break-words`는 유지, 댓글 영역의 버튼·링크 누르는 영역 44px(R17), 이웃 버튼 `min-h-11`(blog 소유 파일의 이웃 버튼 부분만). 마을 소식 탭·인기 태그·페이지 번호는 post 소유라 D-10으로 요청 |
| VII | 통과 | 탈퇴 자리는 작성자 정보 없이 문구만 남긴다. 새 의존성·새 화면 없음 |

위반이 없으므로 Complexity Tracking에는 조건(D-7, D-10)만 적어 둔다.

## 변경 단위 요약 (현재 코드 → spec 목표)

구현 순서는 `plan-context.md` 5.1의 단계 5(1차)와 단계 7(2차, 알림)이다. 마이그레이션 번호는 박지 않고 내용으로 적는다 (먼저 merge된 브랜치가 있으면 최신 `main`에서 `npm run db:generate`를 다시 돌린다).

| # | 단위 | 종류 | 현재 코드 | 목표 | 근거 | 선행 |
|---|---|---|---|---|---|---|
| U1 | `replies` 만들기, `comments.content` CHECK 임시 해제, `comments.author_id` NULL 허용 + `ON DELETE SET NULL` | 마이그레이션 (생성) | 답글이 `comments.parent_id`, 작성자 FK `CASCADE` | data-model "마이그레이션 순서" 1 | SOC-02, ERD 3.8·7장 2, D2 | auth 단계 1 |
| U2 | 답글 이전(id 유지), 이전한 행 삭제, 삭제된 댓글 내용 비우기 | 마이그레이션 (직접 쓴 SQL) | - | data-model "백필" | SOC-02, SC-006 | U1 |
| U3 | `comments.parent_id`·`comments_parent_fk` 삭제, `comments` CHECK 2개(내용, 작성자) | 마이그레이션 (생성) | 자기 참조 FK | 답글의 답글 구조상 불가 | FR-018 | U2 |
| U4 | `follows.is_favorite` (기본 false) | 마이그레이션 (생성) | 없음 | 즐겨찾기 여부 저장 | D15, 요청: town TOWN-08 | - |
| U5 | `addComment` 고치기 + `addReply` 새로 | Server Action | `parentId`가 잘못되면 일반 댓글로 저장, 삭제된 댓글에도 답글, `z.coerce` 영어 문구, 오류 때 입력칸 비워짐 | 거부 문구 2개, 입력 유지, `parseId`, 줄바꿈 정규화 | FR-006~010, FR-018~023, D9, SC-010 | U3 |
| U6 | `deleteComment` 고치기 + `deleteReply` 새로 | Server Action | 작성자만, 내용 그대로 남김 | 작성자·블로그 주인·관리자, 내용 비우기 | FR-013~015, FR-024, D8, SC-005·006 | U3 |
| U7 | `toggleLike` 고치기 | Server Action | 동시 첫 공감에 PK 위반 → 오류 화면, 글 삭제 경합 때 FK 오류 가능 | `onConflictDoNothing`, 글 `FOR KEY SHARE` | FR-025~031, SC-003·004 | - |
| U8 | `toggleFollow` 고치기 (+ 공통 모듈 추가: `src/server/db-errors.ts`에 `foreignKeyViolation()`) | Server Action | 인자 타입 미확인, 대상 확인과 insert가 두 문장 | 문자열 검사, `INSERT ... SELECT` 한 문장, 그 순간 대상 회원이 지워진 23503은 무시 | FR-035~041, SC-005·007 | - |
| U9 | `src/server/social.ts` 새로: 댓글 목록(`getCommentThread`), 댓글 수 식, 관리자용 합계, 탈퇴 도우미 | 서버 함수 | `getComments`(`src/server/blog.ts`)가 작성자 inner join, 삭제 내용 조회, 작성자 ID를 화면에 넘김 | 탈퇴 자리 표시, 서버가 `canReply`·`canDelete` 계산 | FR-011~016, FR-019, D2 | U3 |
| U10 | `listColumns.commentCount` 식 교체, `listFeed` 이웃 정렬, `paged`·`baseList` 선택 정렬 인자(공통 모듈 추가, `listFeed`는 `paged`를 거쳐 `baseList`를 부르므로 둘 다) | 서버 함수 (`src/server/blog.ts`) | 댓글 수는 `comments`만, 이웃 새 글은 최신순만 | 댓글+답글, 즐겨찾은 이웃 최근 7일 우선 | FR-016, FR-032, FR-042, D15 | U3, U4 |
| U11 | 댓글 영역 화면 (`comment-section.tsx`) | 화면 | 한 목록을 `parentId`로 나눔, [삭제]는 작성자만, 작은 버튼(등록 버튼도 약 32px) | 댓글·답글 분리 데이터, 권한별 버튼, 탈퇴 자리, 앵커 `#comments`, 버튼·링크 44px | FR-011~014, FR-019~022, FR-004·005 | U5, U6, U9 |
| U12 | 공감 버튼 화면 (`like-button.tsx`) | 화면 | 방문자 안내가 `title`에만 있어 안 보임(`.btn:disabled`는 `pointer-events: none`) | 버튼 아래 보이는 안내 | FR-030 | - |
| U13 | 글 상세 페이지의 댓글·공감 영역 연결 (`src/app/blog/[slug]/[postId]/page.tsx`, post 소유 파일의 social 영역) | 화면 | `getComments` 결과를 직접 매핑 | `getCommentThread` 결과를 그대로 넘김 | FR-012, FR-014 | U9 |
| U14 | 검증: `scripts/test-social.ts`(새)·`package.json` `test:social`(새), `e2e/comments.mjs`(새), `e2e/social.mjs`·`e2e/params.mjs`(고침), `e2e/feed-load.mjs`(새) | 검증 | 답글 e2e 없음, 이웃 e2e에 온보딩 전 회원 | quickstart 항목 전부 | 모든 SC | U1~U13 |
| U15 | 문서: `docs/02-erd.md`(comments·replies·follows 부분), `README.md` 스크립트 표, `CLAUDE.md`는 바꿀 규칙 없음 | 문서 | ERD 3.8 "deleted_at만 기록", 7장 2·6 ⏳ | 이 plan 구조로 갱신 | 3.1 규칙 5 | U1~U4 |
| U16 | (2차) 공감·댓글·답글 알림 발생 | Server Action | 알림 없음 | game 도우미를 같은 트랜잭션에서 호출 | FR-033, FR-055, FR-056, D11 | game 단계 6 |

## 다른 spec과의 의존성

| ID | 상대 spec | 방향 | 내용 |
|---|---|---|---|
| D-1 | auth (단계 1) | social이 기다림 | 가입 통합으로 `e2e/helpers.mjs`의 `loginDev`가 온보딩 없이 동작해야 social의 모든 e2e를 돌릴 수 있다. `requireMember()`의 의미(온보딩 이동 제거)는 auth가 정한다. 그 뒤 `e2e/social.mjs`의 "온보딩 전 회원" 부분을 social이 고친다 |
| D-2 | game (단계 6) | social이 기다림 | 알림 표 `notifications`와 기록 도우미(game plan: `notifyActivity(tx, { recipientId, actorId, kind, postId })`, `src/server/notifications.ts`)가 있어야 U16을 할 수 있다. social이 넘기는 값: 받는 회원, 종류(`like`·`comment`·`reply`), 행동한 회원, 관련 글 ([contracts/notification-triggers.md](contracts/notification-triggers.md)) |
| D-3 | auth (단계 9, 탈퇴) | auth가 기다림 | 탈퇴 트랜잭션은 회원 행을 지우기 전에 social의 `prepareCommentsForWithdrawal(tx, userId)`를 부른다. `comments.author_id`의 `SET NULL`과 CHECK가 U1·U3에 있어야 한다 |
| D-4 | auth (관리자 화면) | 요청 | `src/app/admin/page.tsx`의 댓글 통계를 social의 `countLiveComments()`(댓글+답글, 삭제 제외)로 바꾼다 (FR-016). U3 뒤에는 `comments`만 세면 답글이 빠지므로 같은 PR에서 auth 리뷰어 승인을 받아 함께 바꾸는 것을 권장한다 |
| D-5 | post (공통 목록 함수) | 공통 모듈 추가 | `src/server/blog.ts`의 `paged`·`baseList`에 선택 정렬 인자 `orderFirst`를 더한다(없으면 지금 정렬, post와 합의). `listFeed`는 `paged`만 부르므로 `paged`가 받아 `baseList`로 넘긴다. `listColumns.commentCount`의 식은 social이 정의를 갖는다(plan-context 5.2). post도 같은 함수에 소분류 JOIN을, blog는 `listFeed`에 `search` 선택 인자를 더하므로(blog plan B9) 나중에 merge하는 쪽이 최신 `main`에 맞춘다 |
| D-6 | post (D5 조회수) | 함께 확인 | Server Action의 `revalidatePath("/", "layout")`가 글 상세를 다시 그리면 지금은 `incrementViewCount()`가 또 불린다(공감·댓글마다 조회수 +1). post의 "하루 1번" 작업이 들어가면 해소된다. social은 이 동작을 바꾸지 않는다 |
| D-7 | game (원장) | 요청 (constitution V 때문에 필요) | `point_ledger`에 부분 UNIQUE (`user_id`, `ref_id`) WHERE `reason = 'like_received'`를 더해 "같은 사람·같은 글 공감 보상 1번"을 DB로도 막는다. 앱 코드는 회원 잠금 + 원장 확인으로 이미 지키지만, 원칙 V는 이런 규칙을 DB 제약으로도 막으라고 한다. game이 출석에 둔 `point_ledger_attendance_uq`와 같은 방식이며 game data-model에는 아직 없다. game이 받지 않으면 social plan의 Complexity Tracking에 이유를 적는다 |
| D-8 | town (TOWN-08) | town이 기다림 | `follows.is_favorite`(U4)가 있어야 town의 "내 이웃 목록"·즐겨찾기(최대 10명)가 동작한다. town은 이 컬럼을 쓰기만 하고 구조는 바꾸지 않는다. town이 늦어도 이웃 새 글은 모두 `false`로 최신순이 된다 |
| D-9 | blog (블로그 홈) | 참조 | 이웃 버튼은 `src/components/blog/blog-header.tsx`(blog 소유) 정보 줄 오른쪽에 있다. blog가 머리 부분을 바꿀 때 이웃 버튼 자리를 유지한다 |
| D-10 | post (`src/components/blog/feed-view.tsx`, `src/components/pagination.tsx`) | 요청 | 마을 소식·이웃 새 글의 탭 [🏘 마을 전체] [💛 이웃 새 글](FR-049, 지금 `btn py-1.5 text-sm`, 약 32px), 인기 태그 칩(`py-1 text-sm`), 페이지 번호 링크(`py-1`)의 누르는 영역을 44×44px로 키운다 (constitution VI). 파일 소유가 post라 social은 고치지 않고 요청한다. 검증은 social의 `e2e/social.mjs` 375px 항목에 넣는다 |

## 남은 문제 (팀 확인 권장)

spec이 정하지 않아 plan에서 정한 값과, 구현 전에 확인해야 할 것이다. `NEEDS CLARIFICATION`은 아니다(모두 기본값을 정해 두었다).

| # | 내용 | 정한 기본값 | 근거 |
|---|---|---|---|
| Q1 | 이웃 새 글 "최근 7일(한국 시간)"의 경계 | 오늘을 포함한 한국 날짜 7일 (`getBlogVisitDays`와 같은 기준) | research R11 |
| Q2 | 삭제한 댓글·답글의 내용을 DB에서도 비우기 | 비운다 + CHECK. ERD 3.8 "`deleted_at`만 기록"을 고친다 | research R3 |
| Q3 | 탈퇴 자리에 작성 시각도 보일지 | 보이지 않는다 (spec은 닉네임·캐릭터만 정함) | research R13 |
| Q4 | 공감 → 취소 → 공감 반복 때 공감 알림도 반복되는 것 | spec 문장("새로 저장되면") 그대로 반복 | research R15 |
| Q5 | 댓글·답글을 삭제 표시해도 그 댓글로 생긴 알림을 남길지 | 남긴다 | contracts/notification-triggers.md 5절 |
| Q6 | 일반 댓글 거부 때도 입력 내용을 남길지 (spec은 답글 대상 오류만 정함) | 모든 거부에서 남긴다 | research R7 |
| Q7 | SC-001은 배포 환경 기준인데 배포 환경이 미정(NF-08) | 로컬 프로덕션 빌드에서 `e2e/feed-load.mjs`로 측정 | quickstart 5절 |
| Q8 | 다른 spec 요청 D-7(game 부분 UNIQUE), D-10(post 44px), D-4(auth 관리자 통계)의 수용 | 받지 않으면 Complexity Tracking에 적는다 | 위 의존성 표 |
| Q9 | 공감·댓글 뒤 다시 그릴 때 조회수가 또 오르는 것 | social은 바꾸지 않고 post D5 작업으로 해소 | 의존성 D-6 |
| Q10 | 라이브러리 동작 (Drizzle 행 잠금·`onConflictDoNothing().returning()`·`orderBy` 두 번·`generate --custom`, React 19 폼 리셋, Next.js 16 동적 렌더링, 마이그레이션 트랜잭션 범위) | 구현 전 설치된 패키지 문서로 확인 | research R19 |

## Project Structure

### Documentation (this feature)

```text
specs/004-social/
├── spec.md                       # 기능 명세 (고치지 않음)
├── checklists/
│   └── requirements.md           # spec 품질 점검
├── plan.md                       # 이 파일
├── research.md                   # Phase 0: 기술 선택과 근거
├── data-model.md                 # Phase 1: 표 현재/목표, 마이그레이션 순서, 백필
├── quickstart.md                 # Phase 1: 검증 순서와 기대 결과
├── contracts/
│   ├── comments.md               # 댓글·답글 Server Action, 댓글 영역 데이터, 탈퇴 도우미
│   ├── likes.md                  # 공감 Server Action, 공감 버튼
│   ├── follows-feed.md           # 이웃 Server Action, 마을 소식·이웃 새 글 화면 라우트
│   └── notification-triggers.md  # (2차) game 알림 도우미 호출 조건
└── tasks.md                      # Phase 2 (/speckit-tasks가 만든다, 이 명령은 만들지 않음)
```

### Source Code (repository root)

코드 저장소(`blogville`) 기준. `★`는 새 파일, `✎`는 고치는 파일, `·`는 읽기만 하는 참고 파일이다.

```text
src/
├── db/
│   └── schema.ts                         ✎ comments(parent_id 삭제, author_id NULL·SET NULL, CHECK 2개), replies ★블록, follows.is_favorite
├── lib/
│   ├── social.ts                         ★ 댓글 내용 정규화·검증 문구, 삭제 권한 판단(순수 함수), 즐겨찾기 우선 기간(7일)과 시작 날짜 계산(listFeed 쿼리가 이 값을 쓴다)
│   ├── ids.ts                            · parseId
│   └── game.ts                           · REWARD_RULES, todayKST, previousDay
├── server/
│   ├── social.ts                         ★ getCommentThread, liveCommentCountSql, countLiveComments, prepareCommentsForWithdrawal
│   ├── blog.ts                           ✎ getComments 삭제(→ social.ts), listColumns.commentCount 식, listFeed 이웃 정렬, paged·baseList 선택 정렬 인자
│   ├── dal.ts                            · getViewer, requireMember (auth 소유)
│   ├── points.ts                         · lockUser, grantReward (game 소유)
│   └── db-errors.ts                      ✎ 공통 모듈 추가: foreignKeyViolation()(23503) 한 함수. 기존 uniqueViolation은 그대로 (이유: toggleFollow가 그 순간 지워진 회원을 이웃 추가할 때 500 대신 무시, SC-007)
├── app/
│   ├── blog/
│   │   ├── actions.ts                    ✎ toggleLike, addComment, addReply★, deleteComment, deleteReply★, toggleFollow (recordBlogVisit은 blog 소유, 그대로)
│   │   ├── [slug]/page.tsx               · 블로그 홈 (이웃 버튼 상태 isFollowing)
│   │   └── [slug]/[postId]/page.tsx      ✎ 댓글·공감 영역만: getCommentThread 연결 (post 소유 파일)
│   ├── feed/page.tsx                     · 마을 소식 (변경 없음, 검증 대상)
│   ├── feed/following/page.tsx           · 이웃 새 글 (listFeed 정렬 변경을 그대로 받음)
│   └── admin/page.tsx                    · 댓글 통계 (요청 D-4, auth 소유)
└── components/
    ├── blog/
    │   ├── comment-section.tsx           ✎ 댓글·답글 분리 렌더, ReplyForm, 권한별 버튼, 탈퇴 자리, 앵커, 버튼·링크 44px(등록 버튼 포함)
    │   ├── like-button.tsx               ✎ 방문자 안내 문구 표시
    │   ├── blog-header.tsx               ✎ 이웃 버튼 부분만 (누르는 영역 min-h-11, blog 소유 파일)
    │   ├── feed-view.tsx                 · 탭·인기 태그 (post 소유, 44px는 요청 D-10)
    │   └── post-card.tsx                 · ♥ N, 💬 N (post 소유, 값만 바뀜)
    └── pagination.tsx                    · 페이지 번호 (post 소유, 44px는 요청 D-10)
drizzle/
├── NNNN_<replies 만들기>.sql             ★ npm run db:generate
├── NNNN_<답글 이전>.sql                  ★ npx drizzle-kit generate --custom (직접 쓴 SQL)
├── NNNN_<comments 정리>.sql              ★ npm run db:generate
├── NNNN_<follows 즐겨찾기>.sql           ★ npm run db:generate
└── meta/                                 ✎ 스냅숏·journal (생성됨)
scripts/
└── test-social.ts                        ★ src/lib/social.ts 단위 테스트
e2e/
├── comments.mjs                          ★ 댓글·답글·삭제 권한·보상 상한·삭제 내용 노출·입력 유지
├── social.mjs                            ✎ 이웃(+동시 요청, 방문자), 공감(동시 첫 공감, 보상 1번, 조작), 이웃 새 글 순서, DB 직접 위반 거부
├── params.mjs                            ✎ 답글 대상 ID·답글 삭제 ID 범위 밖·형식 오류
├── feed-load.mjs                         ★ 공개 글 1,000개 준비 후 /feed 응답 시간 (SC-001)
└── helpers.mjs                           · loginDev (auth 소유)
package.json                              ✎ test:social 추가, test 체인 끝에 붙임
docs/02-erd.md                            ✎ comments·replies·follows 부분 (1장, 3.8, 3.14, 3.16, 7장, 부록)
README.md                                 ✎ 스크립트 표에 e2e 두 줄 (comments, feed-load)
```

**Structure Decision**: 기존 Next.js 단일 프로젝트 구조를 그대로 쓴다. 규칙은 `src/lib/social.ts`(DB 없는 순수 함수, 화면·서버 공용·단위 테스트 대상), DB 쿼리는 `src/server/social.ts`(`import "server-only"`), Server Action은 기존 `src/app/blog/actions.ts`에 둔다(social이 5개 중 4개 함수의 소유자이고, 파일 쪼개기는 하지 않는다). `src/server/blog.ts`·글 상세 페이지·블로그 머리는 다른 spec 소유라 plan-context 5.2의 함수·영역 단위 소유 규칙 안에서만 고친다. 공통 파일 `src/server/db-errors.ts`에는 함수 하나만 더하고 기존 함수는 바꾸지 않는다(이유는 위 표와 research R10).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

위반 없음. 단, 다른 spec에 보낸 요청 D-7(game: 공감 보상 부분 UNIQUE, 원칙 V)이나 D-10(post: 탭·태그·페이지 번호 44px, 원칙 VI)을 상대가 받지 않으면 그 예외와 이유를 여기에 적는다.
