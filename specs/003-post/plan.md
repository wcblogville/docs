# Implementation Plan: 글 (POST) — 글쓰기·공개 범위·분류·목록·조회수·첨부·임시 저장

**Branch**: `003-post` | **Date**: 2026-10-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/003-post/spec.md`

**Note**: 기준 코드는 코드 저장소 `main` `feb4c05`(2026-10-07 결정 미반영). 코드 위치는 코드 저장소 기준 상대 경로, 문서 위치는 문서 저장소 기준 상대 경로다.
테이블 담당·공통 모듈 소유·구현 순서는 7개 spec 공통 맥락 문서의 약속을 따른다. 결정 근거는 [research.md](research.md)의 R 번호로 가리킨다.

## Summary

글 영역(POST-01~09)을 2026-10-07 결정에 맞춘다. 글쓰기·목록·태그·정화·첨부 올리기는 이미 동작하므로 **바뀐 결정과 빈 곳만** 옮긴다:

1. **카테고리 2단계**: `posts.subcategory_id` + 복합 FK + CHECK를 더하고, 글쓰기를 [대분류 ▼] [소분류 ▼] 두 칸으로, 배지를 `대분류 › 소분류`로 바꾼다.
   대분류 삭제·회원 삭제 때 CHECK가 깨지지 않게 `posts` 트리거로 소분류를 함께 비운다 (R4, ERD 3.18 보완).
2. **첨부는 글에 붙는다**: `attachments.post_id`·`detached_at`을 더하고 `savePost` 트랜잭션에서 "내가 올렸고 아직 안 붙었거나 이 글에 붙은 첨부"만 붙인다.
   붙여 넣은 내 다른 글의 첨부는 서버에서 복사해 새 첨부로 다시 올린다. `/files/키`는 붙은 글의 공개 범위·올린 사람을 확인하고, 하루 지난 붙지 않은 첨부는 `npm run posts:cleanup`이 지운다 (R6~R10).
3. **조회수 하루 1번**: 서버 렌더의 `incrementViewCount`를 없애고, 화면이 열린 뒤 부르는 Server Action `recordPostView`가 `post_views` 표(복합 PK)와 BLOG-06의 `bv_visitor` 쿠키로 센다 (R1~R3).
4. **발행 안내 한 번·임시 저장**: 안내는 주인에게 10분 안에, 보상 여부는 원장으로 판단하고 주소의 `?new`를 지운다. 새 글은 `localStorage`에 2초마다 임시 저장한다 (R13, R14).
5. **한국어 오류 문구·요청 크기**: 조작된 형식 값도 `잘못된 요청이에요`, 아주 큰 본문은 브라우저가 같은 스키마로 먼저 막는다 (R11, R12).
6. **누르는 영역 44×44px**: post 소유 화면의 작은 버튼·링크(페이지 번호, 에디터 도구, 공개 토글, 상세 배지·`#태그`·[수정]·[삭제], 마을 소식 탭·인기 태그, 카드 작성자 줄)를 키운다 (constitution VI, blog·social 요청, R21).

새 화면·새 npm 의존성은 없다. 새 테이블 1개(`post_views`), 바뀌는 테이블 2개(`posts`, `attachments`), 마이그레이션 4개다.

## Technical Context

**Language/Version**: TypeScript ^5 (`strict: true`, 별칭 `@/*` → `src/*`), Node.js 20.9 이상 (README)

**Primary Dependencies**: `next` 16.3.8 (App Router, Server Component, Server Action, Route Handler), `react`/`react-dom` 19.2.8 (`useActionState`, `useTransition`),
`drizzle-orm` ^0.45.3 / `drizzle-kit` ^0.31.11, `better-auth` ^1.7.7 (세션 → `src/server/dal.ts`), `zod` ^4.6.5, `@tiptap/*` ^3.31.4 (에디터, `Image` 확장 + 자체 `FileCard`),
`sanitize-html` ^2.18.0, `tailwindcss` ^4. 새 의존성 없음.

**Storage**: PostgreSQL (README 안내 17, 이 기능은 15 이상 필요: `ON DELETE SET NULL (컬럼)`) + `pg` ^8.23.1, Drizzle 스키마 `src/db/schema.ts`, 마이그레이션 `drizzle/`(다음 번호 0007부터, plan에 번호를 박지 않음).
첨부 파일 내용은 서버 디스크 `UPLOAD_DIR`(기본 `storage/uploads`, `src/server/storage.ts`). 배포 저장소는 미정(NF-08) — 이 기능은 `storage.ts`의 함수만 늘린다.
브라우저: `localStorage`(임시 글), 쿠키 `bv_visitor`(조회·방문 공용).

**Testing**: `npm test` = `tsx`로 돌리는 `scripts/test-*.ts`(테스트 프레임워크 없음, `✅`/`❌` 출력 + 실패 시 `exit(1)`) + 새 `test:post`.
E2E는 `@playwright/test` ^1.63.0의 `chromium` API를 쓰는 Node 스크립트 `e2e/*.mjs`(개발 서버 `http://localhost:3000` 대상, pg로 준비·확인). CI 없음 → PR 작성자가 직접 실행.

**Target Platform**: 웹 브라우저(PC, 375px 휴대폰), Node 서버(`next start`). 배포 서비스 미정(NF-08).

**Project Type**: 단일 Next.js 풀스택 웹 앱 (화면·Server Action·Route Handler가 한 저장소).

**Performance Goals**: 글 목록 화면 1초 안 (SC-002, NF-07, 글 1,000개, 배포 환경 첫 방문). 목록 쿼리는 `subcategories` LEFT JOIN(PK) 하나만 늘어난다.
글 저장은 트랜잭션 1번, 조회 기록은 INSERT 1번 + 필요할 때 UPDATE 1번, 첨부 내려주기는 행 조회 1번 + 세션 조회.

**Constraints**:

- Server Action 요청 본문 기본 상한 1MB (확인 필요, R11) vs 본문 200,000자 → 900,000바이트 넘는 본문은 브라우저가 같은 스키마로 검사.
- 30MB 업로드는 Route Handler(`POST /api/uploads`)만. Server Action은 출처 확인이 내장, Route Handler는 직접 확인.
- 모든 화면·오류 문구는 spec 문구 그대로의 한국어 (NF-19). 비밀값은 환경 변수 이름만.
- 이 체크아웃에 `node_modules`가 없어 Next.js 16·Drizzle 세부 API는 구현 전에 설치된 패키지 문서로 확인한다 (research의 "추측" 표시).
- 다른 spec 소유 파일은 "추가만" (공통 모듈 소유 규칙).

**Scale/Scope**: 3명 팀 프로젝트. 화면 라우트 수정 3곳(`/write`, `/write/[postId]`, `/@주소/글ID`) + 목록 함수, Server Action 새 3개·변경 1개(`savePost`, `deletePost`는 그대로), Route Handler 변경 1개, 스크립트 2개, e2e 새 6개.
데이터: 글 1,000개 이상(성능 기준), 첨부는 글당 제한 없음, `post_views`는 (그날 읽은 브라우저 × 글) 행을 이틀 보관.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

기준: `.specify/memory/constitution.md` v1.0.0.

| 원칙 | 설계 전 | 근거 | 설계 후 재평가 |
|---|---|---|---|
| I. 글쓰기가 먼저 | 통과 | 바꾸는 것 모두가 글쓰기·읽기 경험이다 (임시 저장, 카테고리, 첨부 보호, 조회수). 게임 요소는 기존 글쓰기 보상(GAME-05) 호출만 유지하고 새로 더하지 않는다 | 통과 |
| II. 요구사항 ID 유지 | 통과 | 모든 변경 단위·검증이 POST-01~09, GAME-05, BLOG-05·06, NF-03·06·12·19 ID와 FR·SC 번호를 가리킨다. 코드 주석·마이그레이션 주석에도 ID를 단다 | 통과 |
| III. 확인할 수 있는 수용 기준 | 통과 | spec의 수용 시나리오·SC를 quickstart의 e2e 단계와 잇고, FR·User Story마다 대응 위치를 아래 "요구사항 대응표"에 적었다. 문구는 spec 그대로다. spec이 정하지 않은 동작(지워진 소분류, 남의 첨부를 붙여 넣는 순간)과 spec에 없는 화면 문구 하나(소분류 칸 첫 항목 `소분류 없음`)는 research에 결정과 이유를 적고 남은 문제로 올렸다 | 통과 (남은 문제 3·4·8은 팀 확인) |
| IV. 권한과 검증은 서버에서 (NON-NEGOTIABLE) | 통과 | `savePost`가 모든 입력을 공용 스키마로 다시 검증, 대분류·소분류·첨부 주인을 서버에서 확인. `/files/키`는 서버가 붙은 글의 공개 범위·올린 사람을 확인. `recordPostView`는 주인·비공개를 서버에서 확인. 붙여 넣기 판정·복사도 서버. 숨김 버튼은 편의일 뿐. 정화는 저장할 때 허용 목록. 쿼리는 Drizzle 바인딩만 | 통과. 브라우저 사전 검사(R11)는 편의이고 서버 검사가 그대로 남는다 |
| V. 원장·트랜잭션·DB 제약 (NON-NEGOTIABLE) | 통과 | 잔액 컬럼 없음, 글 저장 + 보상 + 태그 + 첨부 붙이기가 `lockUser` 트랜잭션 하나. "하루 1번 조회"는 `post_views` 복합 PK, "한 첨부 한 글"은 `post_id` 컬럼 하나, "소분류는 그 대분류 소속"은 복합 FK, "소분류만 있는 글 금지"는 CHECK. 구조 변경은 모두 마이그레이션, 데이터 이전도 마이그레이션 SQL. 보상 회수 없음(D6) | 통과. 트리거 2개는 제약을 모든 삭제 경로에서 지키기 위한 것 (R4, R7). 글 저장과 정리 작업은 첨부 행 잠금으로 줄 서되, 저장은 키 순서로 잠그고 정리 작업은 `SKIP LOCKED`라 서로 기다리다 교착하지 않는다 (R6, R10) |
| VI. 모바일에서도 | ❌ 지금 코드 위반 발견 | 새 화면 없음. 글쓰기 선택 칸이 둘로 늘어 375px에서 줄바꿈되어야 한다. 그런데 post 소유 화면의 누르는 영역이 44×44px보다 작다 (코드 확인): 페이지 번호 `min-w-9 py-1`(`src/components/pagination.tsx`), 에디터 도구·[🖼 사진]·[📎 파일] `px-2.5 py-1 text-sm`(`rich-editor.tsx`), 공개 토글 `px-3 py-1.5`(`post-form.tsx`), 상세의 카테고리 배지 `py-0.5`·`#태그` `py-1`·[수정]·[삭제] 글자 버튼, 마을 소식 탭 `btn py-1.5`·인기 태그 칩 `py-1`(`feed-view.tsx`), 카드 작성자 줄(`post-card.tsx`). blog(페이지 번호)·social(D-10: 탭·인기 태그·페이지 번호)도 post에 요청했다 | 통과 — 변경 18이 모두 `min-h-11`(+ 필요하면 `min-w-11`)로 키우고 글자는 `whitespace-nowrap` (R21). `e2e/post-lists.mjs`가 375px에서 가로 스크롤 0과 누르는 영역 크기를 잰다 |
| VII. 단순하게, 최소한의 정보 | 통과 | 조회 기록은 무작위 UUID와 날짜만(IP·회원 ID 없음) 이틀 보관, 임시 글은 브라우저에만, 새 의존성·새 화면 없음, 기존 쿠키 재사용 | 통과. `detached_at` 컬럼 하나와 트리거 2개는 spec 문장("떨어진 지 하루", "대분류 삭제 시 글은 남음")을 지키는 최소 추가 |
| 보안·무결성 기준 | 통과 | 다른 사이트 요청: Server Action 내장 확인 + 업로드 Route Handler의 Origin 확인(그대로). 쿠키 `HttpOnly`·`SameSite=Lax`·배포 `Secure`. 비공개 첨부는 `private, no-cache` | 통과 |
| 품질 관문 | 통과 | quickstart에 `tsc`·`eslint`·`npm test`·e2e 순서. CI 없으므로 PR에 결과 기록 | 통과 |

**판정**: 설계가 원칙을 어기는 곳은 없다. 지금 코드의 VI 위반(누르는 영역)은 설계 범위에 넣어(변경 18) 해소한다 → Phase 0 진행. Phase 1 설계 뒤 다시 보아도 새 위반 없음.

## Project Structure

### Documentation (this feature)

```text
specs/003-post/
├── spec.md                      # 기능 명세 (clarify 반영, 이 plan에서 고치지 않음)
├── plan.md                      # 이 파일 (/speckit-plan)
├── research.md                  # Phase 0: 결정 R1~R21
├── data-model.md                # Phase 1: 테이블 변경·참조, 상태 전이, 마이그레이션·이전
├── quickstart.md                # Phase 1: 검증 명령·시나리오·SC 연결
├── contracts/                   # Phase 1
│   ├── write-actions.md         # /write 화면, savePost·deletePost·붙여 넣기 Server Action, 임시 글 저장 계약
│   ├── post-pages.md            # 글 상세·발행 안내·조회수(recordPostView)·목록 규칙
│   └── attachments-http.md      # POST /api/uploads, GET /files/{key}, posts:cleanup, storage.ts
├── checklists/
│   └── requirements.md          # spec 품질 점검 (기존)
└── tasks.md                     # Phase 2 (/speckit-tasks, 아직 없음)
```

### Source Code (repository root)

코드 저장소에서 이 기능이 만지는 파일(수정)과 새로 만들 파일(새)만 적었다. 괄호는 다른 spec 소유 파일에 "추가만" 하는 곳이다.

```text
src/
├── app/
│   ├── write/
│   │   ├── actions.ts                 # 수정: savePost(소분류·첨부 붙이기·한국어 오류·?new=1). deletePost는 그대로
│   │   │                              #       새 classifyPastedAttachments, reuploadAttachment
│   │   ├── page.tsx                   # 수정: 카테고리 트리, 임시 저장 주인(userId)
│   │   └── [postId]/page.tsx          # 수정: 카테고리 트리, 저장된 소분류
│   ├── blog/[slug]/[postId]/
│   │   ├── page.tsx                   # 수정: incrementViewCount 제거, ViewCount·PublishNotice, 배지 `대분류 › 소분류`(`?category=…&sub=…`), 배지·`#태그`·[수정] 44px
│   │   └── actions.ts                 # 새: recordPostView
│   ├── files/[key]/route.ts           # 수정: 붙은 글 공개 범위·올린 사람 확인, private no-cache + ETag
│   ├── api/uploads/route.ts           # 변경 없음 (계약 재확인)
│   └── tags/[name]/page.tsx           # 변경 없음 (계약 재확인)
├── components/
│   ├── editor/
│   │   ├── post-form.tsx              # 수정: 대분류·소분류 두 칸, 임시 저장, 큰 본문 사전 검사, ko-KR 숫자, 공개 토글 44px
│   │   ├── rich-editor.tsx            # 수정: 붙여 넣은 /files 첨부 처리, 도구 버튼 44px
│   │   └── use-attachment-upload.ts   # 수정: 다시 올리기 흐름 (진행 표시·막기 공유)
│   ├── blog/
│   │   ├── post-card.tsx              # 수정: 배지 `대분류 › 소분류`, 작성자 줄 누르는 영역 44px
│   │   ├── feed-view.tsx              # 수정: 마을 소식 탭·인기 태그 칩 누르는 영역 44px (social D-10 요청)
│   │   ├── delete-post-button.tsx     # 수정: [삭제] 누르는 영역 44px (동작 그대로)
│   │   ├── view-count.tsx             # 새: 👀 N + recordPostView
│   │   └── publish-notice.tsx         # 새: 발행 안내 한 번, ?new 지우기, 임시 글 지우기
│   ├── pagination.tsx                 # 수정: 페이지 번호 누르는 영역 44px (blog·social 요청)
│   ├── sign-out-button.tsx            # (auth 소유, 추가만) 선택 prop userId → 로그아웃 전 임시 글 지우기
│   └── site-header.tsx                # (town 소유, 추가만) SignOutButton에 userId 넘기기
├── db/schema.ts                       # 수정(post 담당 블록): posts.subcategoryId·복합 FK·CHECK, attachments.postId·detachedAt·인덱스, postViews
├── lib/
│   ├── post-rules.ts                  # 새: postInputSchema(한국어 문구·순서), parseTags, 본문 크기 판단
│   ├── draft.ts                       # 새: 임시 글 키·읽기·쓰기·지우기 (try/catch)
│   └── attachments.ts                 # 수정(추가): attachmentAccess, pasteAction 순수 규칙
└── server/
    ├── blog.ts                        # 수정(post 소유 함수만): listColumns·baseList·getPost에 소분류, listBlogPosts subcategoryId, incrementViewCount 삭제
    ├── posts.ts                       # 새: getCategoryOptions, resolveCategory, linkAttachments, getPublishNotice, recordView
    ├── attachments.ts                 # 새: findReadableAttachment, 붙여 넣기 판정·복사, cleanupPostData
    ├── visitor.ts                     # 새: bv_visitor 읽기(readVisitorId)·만들기(ensureVisitorId)
    ├── storage.ts                     # 수정(추가): copyAttachment, deleteAttachment, listStoredFiles
    └── sanitize.ts                    # 변경 없음 (known을 좁혀 넘김)
drizzle/
├── <번호>_attachment_post.sql          # 새: post_id·FK·인덱스·detached_at + 트리거 (생성 후 트리거 직접 씀)
├── <번호>_attachment_backfill.sql      # 새: 기존 본문으로 post_id 채우기 (직접 쓴 데이터 SQL)
├── <번호>_post_views.sql               # 새: post_views (생성)
├── <번호>_post_subcategory.sql         # 새: subcategory_id·복합 FK SET NULL(subcategory_id)·CHECK + 트리거 (생성 후 손질)
└── meta/                              # 생성
scripts/
├── test-post.ts                       # 새: 스키마·태그·첨부 권한·붙여 넣기 판정
└── cleanup-posts.ts                   # 새: 정리 작업 CLI
e2e/
├── post-write.mjs                     # 새
├── post-lists.mjs                     # 새
├── post-categories.mjs                # 새
├── post-views.mjs                     # 새
├── attachment-links.mjs               # 새
├── post-drafts.mjs                    # 새
├── blog.mjs, params.mjs               # 수정: 카테고리 칸 이름 `대분류`
├── write-count.mjs                    # 수정: 보상 판단을 안내 문구로(33행), 글 ID를 주소 경로에서 읽기(34행, `?new`가 지워지므로)
├── farm.mjs                           # (town 파일, 이 변경으로 깨지는 줄만) 보상 판단을 안내 문구로(94행)
└── nonfunctional.mjs                  # (공통 모듈 추가) NF-07 측정 경로에 `/tags/{태그}` 한 줄 (SC-002)
docs/02-erd.md                         # 수정: posts·attachments·post_views 부분 (data-model §8)
docs/01-requirements.md                # 수정: POST-01~09 구현 방식·상태 갱신 (변경 이력 한 줄)
package.json                           # 수정: test:post(test 체인 끝), posts:cleanup
README.md, CLAUDE.md                   # 수정: 스크립트 표, 첨부·조회수 규칙 한 줄씩
```

**Structure Decision**: 기존 단일 Next.js 앱 구조를 그대로 쓴다. 화면·Server Action·Route Handler는 `src/app`, DB 쿼리와 서버 전용 처리는 `src/server`(`import "server-only"`),
화면·서버가 같이 쓰는 순수 규칙은 `src/lib`(단위 테스트 대상), 화면 조각은 `src/components`. 새 쿼리는 post 영역 파일(`src/server/posts.ts`, `src/server/attachments.ts`)에 두고
`src/server/blog.ts`는 post 소유 함수만 고친다. 테스트 폴더는 따로 없고 `scripts/test-*.ts`와 `e2e/*.mjs`를 쓴다 (저장소 관례).

## 변경 단위 (현재 코드 → spec 목표)

| # | 단위 | 종류 | 현재 코드 | 목표 | 주요 파일 | spec | 선행 |
|---|---|---|---|---|---|---|---|
| 1 | 첨부 ↔ 글 연결 | 마이그레이션 | `attachments`에 글 정보 없음 | `post_id`(FK SET NULL)·인덱스·`detached_at`·트리거 | `src/db/schema.ts`, `drizzle/` | FR-047, FR-059 | 없음 |
| 2 | 첨부 데이터 이전 | 마이그레이션(데이터) | - | 기존 본문 `/files/키`로 `post_id` 채우기 | `drizzle/` | ERD 7장 3 | 1 |
| 3 | 조회 기록 표 | 마이그레이션 | 없음 | `post_views` 복합 PK | `src/db/schema.ts`, `drizzle/` | FR-046 | 없음 |
| 4 | 소분류 컬럼 | 마이그레이션 | 없음 | `subcategory_id` + 복합 FK `SET NULL (subcategory_id)` + CHECK + 트리거 | `src/db/schema.ts`, `drizzle/` | FR-030~033 | blog `subcategories` |
| 5 | 글 저장 | 서버 함수 | 1단계 카테고리, 남의·다른 글 첨부도 남음, 형식 오류가 영어, `?new=reward` | 공용 스키마(한국어), 소분류 확인, 첨부 붙이기·떼기·빼기, `?new=1` | `src/app/write/actions.ts`, `src/lib/post-rules.ts`, `src/server/posts.ts` | FR-006·009·018·032·047·054, Edge | 1, 4 |
| 6 | 글 삭제 | 서버 함수 (코드 변경 없음) | 행 삭제 | 그대로. FK `SET NULL` + 트리거가 첨부를 떼고 떨어진 시각을 기록 | `src/app/write/actions.ts` | FR-016, FR-059 | 1 |
| 7 | 붙여 넣기 다시 올리기 | 서버 함수 + 화면 | `/files` 든 HTML을 그대로 붙임 | 판정 → 서버 복사 → 새 키로 넣기, 남의 첨부는 넣지 않음 | `src/app/write/actions.ts`, `src/server/attachments.ts`, `src/server/storage.ts`, `rich-editor.tsx`, `use-attachment-upload.ts` | FR-047, US8-10, US9-9, SC-013 | 1 |
| 8 | 첨부 내려주기 권한 | Route Handler | 누구나, `public` 1년 캐시 | 공개 글 누구나·비공개 주인·안 붙음 올린 사람, `private, no-cache` + ETag | `src/app/files/[key]/route.ts`, `src/lib/attachments.ts`, `src/server/attachments.ts` | FR-029, FR-059, SC-004 | 1 (+ auth `photo_key`) |
| 9 | 정리 작업 | 스크립트 | 없음 | `npm run posts:cleanup` (첨부·주인 없는 파일·오래된 조회 기록) | `scripts/cleanup-posts.ts`, `src/server/attachments.ts`, `src/server/storage.ts`, `package.json` | FR-059, SC-011 | 1, 2, 3 |
| 10 | 조회수 하루 1번 | 서버 함수 + 화면 | 렌더마다 +1 (공감·댓글 뒤에도) | `recordPostView` + `ViewCount`, 쿠키 `bv_visitor` 공용 | `src/app/blog/[slug]/[postId]/{page,actions}.ts(x)`, `src/components/blog/view-count.tsx`, `src/server/visitor.ts`, `src/server/blog.ts` | FR-046, US7, SC-012 | 3 |
| 11 | 발행 안내 한 번 | 화면 | `?new=`만 보고 누구에게나 | 주인 + 10분 안 + 원장 판단 + 주소 정리 | `src/app/blog/[slug]/[postId]/page.tsx`, `src/components/blog/publish-notice.tsx`, `src/server/posts.ts` | FR-013 | 5 |
| 12 | 글쓰기 두 칸 | 화면 | `카테고리` 한 칸 | [대분류 ▼] [소분류 ▼], 수정 화면은 저장값 | `post-form.tsx`, `src/app/write/page.tsx`, `src/app/write/[postId]/page.tsx`, `src/server/posts.ts` | FR-030, FR-031, US5 | 4 |
| 13 | 배지·목록 함수 | 서버 함수 + 화면 | 대분류 이름만 | `대분류 › 소분류`, 상세 배지 링크 `?category=대분류&sub=소분류`(blog R-13), `listBlogPosts`에 `subcategoryId` | `src/server/blog.ts`, `post-card.tsx`, 상세 `page.tsx` | FR-033, Assumptions | 4 |
| 14 | 임시 저장 | 화면 | 없음 | `localStorage` 2초 저장·불러오기·발행 후·로그아웃 때 지우기 | `post-form.tsx`, `src/lib/draft.ts`, `publish-notice.tsx`, (`sign-out-button.tsx`, `site-header.tsx`) | FR-060~064, US10 | 11 |
| 15 | 큰 본문·숫자 표시 | 화면 | 1MB 넘으면 오류 화면, 글자 수가 브라우저 언어를 따름 | 900,000바이트 넘으면 브라우저 검사, `ko-KR` 고정 | `post-form.tsx`, `src/lib/post-rules.ts` | FR-006, FR-010, Assumptions | 5 |
| 16 | 검증 | 테스트 | `test:sanitize`, `attachments.mjs`, `write-count.mjs` 등 | `test:post` + 새 e2e 6개 + 기존 e2e 4개 고침 + `nonfunctional.mjs` 경로 한 줄 | `scripts/test-post.ts`, `e2e/*.mjs`, `package.json` | SC-001~013 | auth 단계 1 |
| 17 | 문서 | 문서 | ERD ⏳ 표시 | ERD·요구사항 문서·README·CLAUDE.md 갱신 | `docs/02-erd.md`, `docs/01-requirements.md`, `README.md`, `CLAUDE.md` | 테이블 담당 규칙 5, constitution II | 1~15, 18 |
| 18 | 누르는 영역 44×44px | 화면 | 페이지 번호·에디터 도구·공개 토글·상세 배지·`#태그`·[수정]·[삭제]·마을 소식 탭·인기 태그 칩·카드 작성자 줄이 약 26~36px | 모두 `min-h-11`(+ 필요하면 `min-w-11`), 글자 `whitespace-nowrap`. 동작·문구는 그대로 | `src/components/pagination.tsx`, `src/components/editor/{rich-editor,post-form}.tsx`, 상세 `page.tsx`, `src/components/blog/{delete-post-button,feed-view,post-card}.tsx` | FR-066, constitution VI, blog FR-059·social D-10 요청 | 없음 |

1~3, 5(소분류 제외)~11, 14~15, 18은 blog를 기다리지 않는다. 4·12·13과 5의 소분류 부분만 blog 단계 2 뒤에 한다.

## 요구사항 대응표 (FR·User Story)

spec의 모든 FR이 어디서 다뤄지는지 적었다. "유지"는 지금 코드가 이미 spec대로 동작해 바꾸지 않는 것이고(코드 확인 위치), 회귀 검증으로 지킨다.

| FR | 이 plan에서 | 근거·위치 | 검증 |
|---|---|---|---|
| FR-001 | 유지. 진입 버튼은 blog 소유 화면에 있다 (`src/components/blog/blog-header.tsx`의 `✏️ 글쓰기`, `src/app/blog/[slug]/page.tsx`의 `첫 글 쓰기`). 온보딩 제거는 auth 단계 1 | 의존성 | `e2e/post-write.mjs` |
| FR-002~005 | 유지 (`src/components/editor/rich-editor.tsx`: StarterKit H2·H3, 링크 창 `링크 주소 (비우면 링크 해제)`, 본문 안내 문구). 도구 버튼 크기만 변경 18 | [write-actions §1](contracts/write-actions.md) | `e2e/post-write.mjs`(US1-3·9), `e2e/blog.mjs` |
| FR-006, FR-008, FR-009 | 변경 5·15 (공용 스키마, 검사 순서) | [write-actions §2](contracts/write-actions.md), R11·R12 | `test:post`, `e2e/post-write.mjs` |
| FR-007, FR-011 | 유지 (`src/server/sanitize.ts`, `src/lib/text-length.ts`). 저장 때 `known`만 좁힘 | R17 | `test:sanitize`, `e2e/write-count.mjs` |
| FR-010 | 유지 + 숫자 `ko-KR` 고정 (변경 15) | R14 | `e2e/write-count.mjs` |
| FR-012, FR-027 | 유지 (`grantReward`가 글 저장과 같은 트랜잭션, 수정에는 보상 없음) | [write-actions §2](contracts/write-actions.md) | `e2e/post-write.mjs`, `e2e/write-count.mjs` |
| FR-013 | 변경 11 | R13, [post-pages §1](contracts/post-pages.md) | `e2e/post-write.mjs` |
| FR-014, FR-019, FR-022 | 유지 (`post-form.tsx`의 `저장하는 중...`·공개 토글 기본 `public`, `requireMember()`) | [write-actions §1](contracts/write-actions.md) | `e2e/post-write.mjs` |
| FR-015, FR-020 | 유지 + 대분류·소분류 두 칸 (변경 12) | R15 | `e2e/post-write.mjs`, `e2e/post-categories.mjs` |
| FR-016 | 변경 6 (코드 그대로, FK + 트리거) | R7, data-model §6 | `e2e/post-write.mjs`, `e2e/attachment-links.mjs` |
| FR-017, FR-018 | 유지 + 형식 오류 한국어 (변경 5) | R12 | `e2e/post-write.mjs`, `e2e/params.mjs` |
| FR-021 | `👀 N`은 변경 10, 나머지 유지. [수정]·[삭제] 크기 변경 18 | R2 | `e2e/post-views.mjs` |
| FR-023, FR-028 | 유지 (상세 `page.tsx`의 `notFound()`, `generateMetadata`) | [post-pages §1](contracts/post-pages.md) | `e2e/post-write.mjs` |
| FR-024, FR-025 | 유지 (`src/server/blog.ts`의 목록·이전/다음 함수, `post-card.tsx` 배지). `글 N`·`전체 글 (N)`(`getBlogBySlug`)은 blog, 광장 순서는 town 화면 | [post-pages §3](contracts/post-pages.md) | `e2e/post-lists.mjs` |
| FR-026 | social 함수(`src/app/blog/actions.ts`) 유지 | 의존성 | `e2e/post-write.mjs`(US3-6) |
| FR-029 | 변경 8 | R9 | `e2e/attachment-links.mjs` |
| FR-030~032 | 변경 4·5·12 | R15, data-model §2 | `e2e/post-categories.mjs` |
| FR-033 | 변경 13 | R16 | `e2e/post-categories.mjs` |
| FR-034~040 | 유지 (목록 함수, `src/components/pagination.tsx`의 `Pagination`·`parsePage`, `post-card.tsx`) + 카드 `subcategoryName`(변경 13), 크기(변경 18). 이웃 새 글 순서(FR-035, D15)·댓글 수(FR-036)는 social | [post-pages §3](contracts/post-pages.md) | `e2e/post-lists.mjs` |
| FR-041~045 | 유지 (`parseTags`는 `src/lib/post-rules.ts`로 옮김, `getAllTags`, `src/app/tags/[name]/page.tsx`) + 인기 태그 칩 크기(변경 18) | R20 | `test:post`, `e2e/post-lists.mjs` |
| FR-046 | 변경 3·10 | R1~R3 | `e2e/post-views.mjs` |
| FR-047 | 변경 1·5·7 | R6·R8, [write-actions §4·§5](contracts/write-actions.md) | `e2e/attachment-links.mjs` |
| FR-048~053, FR-055~058 | 유지 (`POST /api/uploads`, `use-attachment-upload.ts`, `src/lib/attachments.ts`, `/files` 머리글). 바뀌는 것은 `/files`의 캐시 머리글(변경 8)과 다시 올리기의 같은 표시·막기(변경 7) | [attachments-http](contracts/attachments-http.md) | `e2e/attachments.mjs`, `e2e/attachment-links.mjs` |
| FR-054 | 유지 (붙여 넣기 필터, 정화) + 저장 때 `known` 좁힘 | R6·R17 | `test:sanitize`, `e2e/attachment-links.mjs` |
| FR-059 | 변경 1·8·9 | R7·R9·R10·R19 | `e2e/attachment-links.mjs` |
| FR-060~064 | 변경 14 | R14 | `e2e/post-drafts.mjs` |
| FR-065 | 변경 5 (형식 오류도 한국어). 새 문구는 `소분류 없음` 하나 (남은 문제 8) | R12 | `test:post`, `e2e/post-write.mjs` |
| FR-066 | 검증 + 변경 18 | R21 | `e2e/post-lists.mjs`, `e2e/attachments.mjs` |

| User Story | 변경 단위 | 검증 |
|---|---|---|
| US1 글쓰기·발행 | 5·11·15 (나머지 유지) | `e2e/post-write.mjs`, `e2e/write-count.mjs` |
| US2 수정·삭제 | 5·6·12 | `e2e/post-write.mjs` |
| US3 공개·비공개 | 8 (나머지 유지) | `e2e/post-write.mjs`, `e2e/post-lists.mjs`, `e2e/attachment-links.mjs` |
| US4 목록 | 13·18 (나머지 유지, US4-10은 social) | `e2e/post-lists.mjs` |
| US5 대분류·소분류 | 4·5·12·13 | `e2e/post-categories.mjs` |
| US6 태그 | 유지 (`parseTags` 옮김) | `test:post`, `e2e/post-lists.mjs` |
| US7 조회수 | 3·10 | `e2e/post-views.mjs` |
| US8·US9 사진·파일 | 1·2·5·7·8·9 (올리기 자체는 유지) | `e2e/attachments.mjs`(기존: US8-1·3~5·7·9, US9-1·3~5·7·8), `test:sanitize`(US9-6 카드 이름·크기), `e2e/attachment-links.mjs`(US3-8, US8-2·6·8·10, US9-2·6·9) |
| US10 임시 저장 | 14 | `e2e/post-drafts.mjs` |

## 다른 spec과의 의존성

**post가 기다리는 것**

| spec | 무엇 | 막는 범위 |
|---|---|---|
| auth (단계 1) | 온보딩 제거·가입 통합, `e2e/helpers.mjs`의 `loginDev`, `requireMember()` 의미 정리 | 모든 e2e 실행 (코드 작업은 막지 않음) |
| auth (단계 1) | `SignOutButton`을 Server Action 폼(`<form action={signOut}>`)으로 바꿈 (auth contracts `auth-entry.md` 5절) | 변경 14의 로그아웃 때 임시 글 지우기: 그 폼의 브라우저 `onSubmit`에서 지운다 (R14) |
| blog (단계 2) | `subcategories` 표 + UK (`category_id`, `id`) | 변경 단위 4·12·13, `e2e/post-categories.mjs` |
| blog (단계 2·4) | 블로그 홈 주소 `?sub=소분류ID` 규칙 (blog research R-13) | 상세 배지 링크(변경 13). 이미 정해졌으므로 post는 `?category=대분류ID&sub=소분류ID`로 연결한다. 단계 4 전에는 블로그 홈이 `sub`를 몰라 대분류로 거르고, 단계 4 뒤에는 소분류로 거른다 (R16) |
| auth | `profiles.photo_key` (ERD 7장 3) | 프로필 사진 예외 조건만 (R19). 없이 먼저 내보내고 컬럼이 들어오면 켠다 |
| social (단계 5) | `replies` 분리 뒤 `listColumns.commentCount` 정의 (social이 고침), 이웃 새 글의 즐겨찾기 7일 우선(D15, `follows.is_favorite`) | 카드 댓글 수·US4-10 확인만 (post 코드는 막지 않음) |

**post를 기다리는 것·post에 온 요청**

| spec | 무엇 | post의 처리 |
|---|---|---|
| blog (단계 4) | `listBlogPosts`의 `subcategoryId` 인자, 카드의 `subcategoryName` (블로그 홈 트리·거르기). blog plan B8도 같은 인자를 "공통 모듈 추가"로 적었다 | post가 변경 13에서 넣고 blog는 쓰기만 한다. 먼저 들어간 쪽을 다른 쪽이 그대로 쓴다 |
| blog | `src/components/pagination.tsx` 페이지 번호 44×44px (blog FR-059) | 변경 18 |
| blog | `/files/[key]`가 프로필 사진 첨부는 누구에게나 내려준다 | 변경 8 (R19) |
| social (D-5) | `src/server/blog.ts`의 `paged`·`baseList`에 선택 정렬 인자 `orderFirst` 추가 (post 소유 함수에 "추가만") | 동의. 없으면 지금 정렬. 소분류 JOIN(변경 13)과 같은 함수를 고치므로 나중에 merge하는 쪽이 최신 `main`에 맞춘다 |
| social (D-10) | `feed-view.tsx` 탭·인기 태그 칩, `pagination.tsx` 페이지 번호 44×44px | 변경 18 |
| blog (B9) | `listFeed`에 선택 인자 `search` 추가 (post 소유 공통 부분에 "추가만") | 동의. 없으면 지금 동작 |
| auth (9) | 정리 작업이 DB 행 없는 저장소 파일(탈퇴 CASCADE)도 지운다 | 변경 9 (R10의 2번) |

**다른 spec 소유 파일에 하는 일 (추가만)**

- 공통 모듈 추가: `src/components/sign-out-button.tsx`(auth) — 선택 prop `userId`, 로그아웃 요청을 보내기 전(폼 `onSubmit`, 브라우저)에 그 회원의 임시 글 지우기. 버튼이 브라우저 컴포넌트(`"use client"`)로 남아야 한다 (auth와 합의).
- 공통 모듈 추가: `src/components/site-header.tsx`(town) — `SignOutButton`에 `viewer.userId` 넘기기.
- 공통 모듈 추가: `package.json` `scripts` — `test:post`(체인 끝), `posts:cleanup`.
- 공통 모듈 추가: `e2e/nonfunctional.mjs` — NF-07 측정 경로에 `/tags/{태그}` (SC-002의 태그별 글 목록).
- `e2e/farm.mjs`(town 영역) — 발행 보상 판단을 주소 `new=reward` 대신 안내 문구로 (이 변경으로 깨지는 줄, 공통 맥락 6.3 규칙대로 같은 작업에서 고침).
- 새 모듈 `src/server/visitor.ts` — blog의 `recordBlogVisit`가 원하면 이 도우미로 바꾼다 (비차단 요청).
- 대분류 삭제: blog research R-11은 `deleteCategory`가 같은 트랜잭션에서 글의 두 칸을 먼저 비운 뒤 대분류를 지우기로 했다. post의 트리거 `posts_clear_subcategory`와 함께 써도 문제없다 (이미 NULL이면 트리거가 할 일이 없다). 트리거는 앱 처리로 막을 수 없는 경로 — 회원 삭제(탈퇴·관리자 회원 삭제)의 `blogs → categories` CASCADE, `deleteCategory`가 글을 비운 뒤 커밋 전에 같은 대분류로 저장된 글 — 를 위한 것이다 (R4). 그래서 auth의 `adminDeletePost`·탈퇴 코드는 바꾸지 않아도 된다.

**구현 순서상 위치**: 공통 맥락 5.1의 **단계 3** (선행: 단계 1 auth 가입 변경, 단계 2 blog 소분류). 단계 4(blog 블로그 홈 트리)가 이 plan의 13번을 쓴다.

## 남은 문제

| # | 내용 | 영향 | 제안 |
|---|---|---|---|
| 1 | 정리 작업을 배포 환경에서 하루 1번 돌리는 방법과 첨부 저장소 (NF-08 미정) | FR-059·SC-011은 스크립트로 확인 가능, 운영 자동화는 미정 | 배포 서비스 결정 때 cron → `npm run posts:cleanup` 또는 같은 함수를 부르는 Route Handler |
| 2 | ERD 3.18 설계(복합 FK + CHECK)가 대분류·회원 삭제 때 오류를 낸다 (R4). 트리거 2개와 `attachments.detached_at`을 더한다. blog research R-11은 같은 문제를 `deleteCategory` 앱 처리로 풀고 트리거를 "규칙이 숨는다"며 고르지 않았다 — 둘은 함께 쓸 수 있지만 회원 삭제 CASCADE 경로는 트리거만 막는다 | 결정된 ERD를 고친다, blog와 방식 맞추기 | 팀 확인 후 `docs/02-erd.md` 갱신 (post 담당 부분). 트리거 이름·이유를 ERD 3.14·3.18과 마이그레이션 주석에 적어 "숨는" 문제를 줄인다 |
| 3 | 지워진 소분류 번호로 저장하면 대분류만 저장 (spec에 직접 규정 없음, R15) | 낮음 | 팀 확인 |
| 4 | 남의 첨부를 붙여 넣으면 에디터에 바로 넣지 않는다 (spec은 저장 때 빼기만, R8) | 낮음 | 팀 확인 |
| 5 | 프로필 사진 올리기 화면을 맡은 spec이 없다 (공통 맥락 5.3). post는 예외 조건만 준비 | 프로필 사진 첨부 정리·공개 | 팀이 담당 spec을 정한다 |
| 6 | 라이브러리 동작 확인 (node_modules 없음): Next.js의 Server Action 순차 처리·본문 상한 1MB·쿠키 변경 뒤 재렌더·`history.replaceState`, React 19 `<form action>`과 `onSubmit`을 함께 쓸 때의 순서, Drizzle `onDelete` 컬럼 목록 미지원·`generate --custom`·`select().for("key share")`, PostgreSQL FK 트리거 실행 순서 | 결정 R3·R5·R11·R13·R14·R15·R4의 전제 | 구현 첫 작업으로 설치된 문서·SQL 재현으로 확인. 다르면 research의 대안으로 바꾼다 |
| 7 | SC-001(처음 쓰는 회원 3분)은 자동 측정 불가 | 검증 방법 | 수동 측정 (quickstart §5) |
| 8 | 소분류 칸 첫 항목 문구 `소분류 없음`은 spec에 없다 (spec은 대분류의 `카테고리 없음`만 정함) | 낮음 (FR-065 "spec에 적은 문구") | 팀 확인 후 spec FR-030에 한 줄 더하거나 다른 문구로 |

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Constitution Check 위반이 없어 정당화할 항목이 없다. 참고로, 트리거 2개(`posts_clear_subcategory`, `attachments_track_detached`)는 원칙 위반이 아니라
원칙 V(규칙을 DB로도 지킨다)를 모든 삭제 경로에서 지키기 위한 선택이며 이유와 대안은 [research.md](research.md) R4·R7에 있다.
