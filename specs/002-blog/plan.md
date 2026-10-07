# Implementation Plan: 블로그 (BLOG)

**Branch**: `002-blog` | **Date**: 2026-10-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/002-blog/spec.md`

- 기준: 코드 저장소 `main` `feb4c05` (2026-10-07 결정은 아직 코드에 없음), `docs/02-erd.md` v1.4, `.specify/memory/constitution.md` v1.0.0, 공통 맥락 문서(테이블 담당·공통 모듈 소유·구현 순서).
- 코드 위치는 코드 저장소 기준 상대 경로(`src/server/blog.ts`), 문서는 문서 저장소 기준 상대 경로(`specs/002-blog/spec.md`)다.
- 설계 근거와 대안은 [research.md](research.md)의 `R-번호`로 가리킨다.

## Summary

블로그 영역(BLOG-01~07)을 2026-10-07 결정에 맞춘다. 블로그 홈·주소·방문자 수(BLOG-02·06)와 이름·소개 수정, 미니룸은 이미 동작하므로 그대로 두고 빈 곳을 채운다.

1. **주소·닉네임 변경 (BLOG-03)**: 블로그 관리에 주소 폼(`updateBlogSlug`), 내 정보에 닉네임 칸(`updateNickname`)을 더한다. 예약어 16개와 "다른 회원의 아이디와 같은 값 금지"는 auth가 만드는 공통 모듈과 **이름 단위 advisory lock**으로 가입과 같은 규칙·같은 잠금을 쓴다. 주소는 `blogs.slug` 한 곳에만 있어 UPDATE 한 번으로 사이트 전체 링크가 바뀌고, 예전 주소는 바로 404가 된다.
2. **카테고리 2단계 (BLOG-05)**: 새 표 `subcategories`(ERD 3.18)와 소분류 Server Action 4개, 관리 화면 트리, 블로그 홈 왼쪽 트리와 `?sub=` 거르기를 만든다. 카테고리 변경은 블로그 행 잠금으로 줄 세워 순서에 빈틈·겹침이 없게 한다. 글 쪽 칸(`posts.subcategory_id`)과 글쓰기 두 칸은 post가 만든다.
3. **검색 (BLOG-07)**: 새 주소 없이 블로그 홈 `?q=`를 **주인에게만** 여는 검색 모드로 한다. 공개 글(제목·본문)과 블로그(이름·닉네임)를 `ILIKE` 부분 일치(이스케이프)로 찾고, 글은 기존 `listFeed`에 선택 인자 `search`를 더해 마을 소식 카드로 보인다.
4. **주인 프로필·도감·전시 (BLOG-04)**: `blogs.showcase_animal_id` + 복합 FK(`ON DELETE SET NULL (showcase_animal_id)`)로 "내 동물만"을 DB가, "다 키운 동물만"을 서버가 지킨다. 블로그 홈에 주인 프로필(사진 또는 캐릭터 얼굴), 다 키운 동물 도감, 미니룸 옆 전시 동물을 그린다.
5. **보강**: 오류 때 입력값 유지(React 19 폼 초기화 대응), 길이를 코드 포인트로 세기(닉네임 이모지 500 방지), 누르는 영역 44px, 소개 DB CHECK, town이 요청한 `blogs.roof_color`.

새 패키지는 없다. 마이그레이션은 blog 담당 3개(소분류 표 / 블로그 작은 변경 / 전시 동물)다.

## Technical Context

**Language/Version**: TypeScript ^5 (`strict: true`, 경로 별칭 `@/*` → `src/*`), Node.js 20.9 이상

**Primary Dependencies**: Next.js 16.3.8 (App Router, Server Component, Server Action, `next.config.ts` rewrites `/@:slug`), React 19.2.8 (`useActionState`, `useTransition`), drizzle-orm ^0.45.3 / drizzle-kit ^0.31.11, better-auth ^1.7.7 (세션·username, 이 기능은 `getViewer()`로만 씀), zod ^4.6.5, Tailwind CSS 4 (`phone:` 변형, `sm` 640px·`md` 768px). 그림은 `src/lib/art/`의 코드 SVG. 새 의존성 없음.

**Storage**: PostgreSQL (README 안내 17, 최소 15 — `ON DELETE SET NULL (컬럼)`), `pg` ^8.23.1 연결 풀(`src/db/index.ts`). 프로필 사진 파일은 post 소유 첨부 저장소(디스크 `UPLOAD_DIR`, `src/server/storage.ts`)의 것을 `/files/{key}`로 읽기만 한다.

**Testing**: `npx tsc --noEmit`, `npx eslint`, `npm test`(`scripts/test-*.ts`를 `tsx`로, 자체 `expect`) + 새 `test:blog`. E2E는 `e2e/*.mjs`(`@playwright/test`의 `chromium`을 Node 스크립트에서 직접, 개발 서버 `http://localhost:3000`, `pg`로 DB 직접 준비·확인). CI 없음 → PR에 결과를 적는다 (NF-23).

**Target Platform**: 브라우저(PC, 375px 휴대폰 세로·가로) + Node.js 서버(`next start`). 배포 환경 미정(NF-08) → 성능은 로컬 프로덕션 빌드로 잰다 (R-01).

**Project Type**: 웹 서비스 — Next.js 단일 프로젝트(화면·Server Action·DB 쿼리가 한 저장소)

**Performance Goals**: 블로그 홈 1초 안(배포 환경·캐시 없는 첫 방문, 글 1,000개, SC-003·NF-07), 검색 결과 첫 페이지 1초 안(SC-011), 블로그 이름 바꾸기 1분 안(SC-006), 대분류 + 소분류 만들기 누름 4번 이하(SC-007)

**Constraints**: 권한·검증은 서버(원칙 IV), 규칙은 DB 제약으로도(V), 375px 가로 스크롤 0·누르는 영역 44×44px·버튼 글자 한 줄(VI, FR-059), 외부 그림 금지, 비밀값 문서 기재 금지, 테이블 담당 규칙(공통 맥락 3.1)과 공통 모듈 소유 규칙(5.2), 마이그레이션 번호를 plan에 고정하지 않음. 코드 체크아웃에 `node_modules`가 없어 Next.js·Drizzle·zod·React 세부 API는 구현 전에 설치된 문서로 확인한다 (R-26).

**Scale/Scope**: 3명 팀 과제 규모. 블로그당 글 1,000개, 마을 공개 글 수만 건 이하 가정(R-01). 화면 3곳(블로그 홈, 블로그 관리, 내 정보의 닉네임 칸), Server Action 12개(새 7: `updateBlogSlug`, 소분류 4개, `setShowcaseAnimal`, `updateNickname` / 바뀜 5: `updateBlogInfo`, 대분류 4개), 새 표 1개, `blogs` 컬럼 2개 + CHECK 2개.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 원칙 | 설계 전 | 근거 | 설계 후 재평가 |
|---|---|---|---|
| I. 글쓰기가 먼저 | ✅ 통과 | 블로그 홈·카테고리 2단계·검색은 글쓰기·읽기 도구다. 도감·전시는 농장 보상(다 키운 동물)을 블로그에서 보여 주는 장식이고 코인을 쓰지 않는다 (FR-029~031). | ✅ 통과. 도감은 블로그 정보 아래 작은 카드 줄로 두어 글 목록을 첫 화면 밖으로 밀어내지 않게 한다 (R-20). 검색은 글을 찾는 기능이다 |
| II. 요구사항 ID 고정 | ✅ 통과 | 모든 산출물이 BLOG-01~07, FR-001~060, SC-001~013, US 번호를 그대로 가리킨다. 마이그레이션 주석·코드 주석에도 ID를 단다 (data-model 5절) | ✅ 통과 |
| III. 확인할 수 있는 수용 기준 | ⚠️ 조건부 | 모든 수용 시나리오와 SC가 e2e 스크립트에 연결된다 ([quickstart.md](quickstart.md)). 단 spec에 없는 화면 문구 일부(검색 결과 제목, 도감 제목, 전시 버튼, 주소 폼 버튼, 닉네임 성공 문구)를 plan이 임시로 정했다 | ⚠️ 조건부 통과 — Complexity Tracking에 적고 남은 문제로 팀 확인. 문구가 정해지면 spec에 옮긴다 |
| IV. 권한과 검증은 서버에서 | ✅ 통과 | 새·바뀐 Server Action 모두 `requireMember()` + 대상 = 로그인 회원의 블로그·프로필(`owner_id`/`blog_id`/`user_id` 조건), 숫자 `parseId()`, 서버 zod. 검색은 주인 판정을 서버가 하고 공개 글 조건을 SQL에 둔다. LIKE 검색어는 바인딩 + 이스케이프. 다른 사이트 요청은 Next.js Server Action Origin 확인 (FR-060). 비밀값 없음 | ✅ 통과. 다른 블로그 ID로 이름 바꾸기가 지금은 `{ ok }`를 돌려주는 것을 `잘못된 요청이에요`로 고친다 ([contracts/blog-settings.md](contracts/blog-settings.md) 3.2) |
| V. 원장·트랜잭션·DB 제약·마이그레이션 | ⚠️ 조건부 | 코인·경험치를 건드리지 않는다. DB 제약: `blogs.slug` UNIQUE·CHECK, `owner_id` UNIQUE, `subcategories` UNIQUE 2개·CHECK·CASCADE, 전시 복합 FK, 소개 CHECK, `roof_color` CHECK. 구조 변경은 마이그레이션 파일로만. 단 DB 제약으로 표현할 수 없는 규칙 3개가 있다 (아이디↔주소·닉네임 겹침, 다 키운 동물만 전시, 순서 빈틈 없음) | ⚠️ 조건부 통과 — 셋 모두 트랜잭션 + 잠금(이름 advisory lock, 블로그 행 `FOR UPDATE`) 또는 단방향 상태로 동시 요청에도 한 번만 처리되게 했고 Complexity Tracking에 정당화를 적었다. 대분류·회원 삭제 때 `posts` CHECK와 FK 동작이 부딪히는 문제는 `deleteCategory`의 앱 처리 + post의 트리거 `posts_clear_subcategory`로 모든 경로에서 막는다 (R-11, 남은 문제 10) |
| VI. 모바일에서도 | ❌ 지금 코드 위반 발견 | 관리 화면 ▲▼(`px-1 text-xs`)·[이름 바꾸기]·[삭제]·`내 블로그로 →`(`text-sm` 글자 링크), 블로그 홈 카테고리 링크(`px-2 py-1`), 375px의 주인 버튼 3개(`btn` + `max-sm:text-sm`, 약 39px로 추정)가 44px에 못 미친다 (R-24) | ✅ 설계로 해결 — 모두 44×44px, ▲▼ 가로 배치. 페이지 번호(post 소유)는 post에 요청, 이웃 버튼은 social D-9가 같은 파일에서 키운다 (의존성) |
| VII. 단순하게, 최소 정보 | ✅ 통과 | 새 개인정보 없음. 검색어를 저장하지 않는다. 방문 기록은 IP 없이 그대로. 새 패키지·외부 검색 엔진 없음. 표 1개·컬럼 2개 | ✅ 통과 |

## Project Structure

### Documentation (this feature)

```text
specs/002-blog/
├── spec.md                         # /speckit-specify + /speckit-clarify 결과 (고치지 않음)
├── checklists/
│   └── requirements.md             # spec 품질 점검
├── plan.md                         # 이 파일 (/speckit-plan)
├── research.md                     # Phase 0: 결정·근거·대안 (R-01~R-29)
├── data-model.md                   # Phase 1: 테이블 현재/목표, 상태 전이, 마이그레이션 순서
├── quickstart.md                   # Phase 1: 검증 실행 순서와 기대 결과
├── contracts/
│   ├── blog-home.md                # 화면 라우트: /@주소, /@주소/글ID, /settings/blog, /settings/account(닉네임 칸), 검색 모드
│   ├── blog-settings.md            # Server Action: 이름·소개, 주소, 대분류 4개, 소분류 4개
│   └── profile-showcase.md         # Server Action: 닉네임, 전시 동물, 방문 기록(기존) + 읽기 함수
└── tasks.md                        # Phase 2 (/speckit-tasks, 아직 없음)
```

### Source Code (repository root)

코드 저장소에서 이 기능이 만지는 파일만 적었다. `[변경]` 기존 동작을 바꿈(이 spec 소유), `[새 파일]`, `[추가]` 다른 spec 소유 파일에 끼워 넣기만 함(공통 맥락 5.2), `[참조]` 읽기만 함.

```text
src/
├── db/
│   └── schema.ts                         [변경] blogs: showcase_animal_id + 복합 FK, roof_color + CHECK, description CHECK / subcategories 새 표
├── lib/
│   ├── blog.ts                           [새 파일] 순수 규칙: 주소 형식 `SLUG_RE`(정규화는 auth의 `normalizeName`을 쓴다), charCount, toLikePattern, parseSearchQuery, buildCategoryTree, 순서 맞바꾸기, ROOF_COLORS
│   ├── names.ts                          [참조] auth가 단계 1(U4)에서 새로 만듦: RESERVED_NAMES·normalizeName·isReservedName. 목록 값 16개는 blog가 정한다
│   └── art/animals.ts                    [참조] animalSvg (도감·전시 그림)
├── server/
│   ├── blog.ts                           [변경] getBlogBySlug(+photoKey, showcaseAnimalId), getCategories(소분류 트리·글 수), getGrownAnimals·searchBlogs(새)
│   │                                     [추가] listFeed의 선택 인자 search (post 소유 함수)
│   │                                     [참조] listBlogPosts의 선택 인자 subcategoryId (post 변경 13이 넣는다. 먼저 들어간 쪽을 그대로 쓴다)
│   ├── names.ts                          [참조] auth가 단계 1(U4)에서 새로 만듦: lockName·findNameConflict
│   └── dal.ts                            [참조] getViewer, requireMember
├── components/
│   ├── character.tsx                     [변경] MiniRoom에 선택 prop showcase (전시 동물)
│   └── blog/
│       ├── blog-header.tsx               [변경] 주인 프로필(사진 또는 캐릭터 얼굴 + 닉네임), 전시 동물 전달
│       ├── category-nav.tsx              [새 파일] 왼쪽 대분류·소분류 트리 (44px 링크, 선택 표시)
│       ├── blog-search.tsx               [새 파일] 주인 검색창 + 검색 결과(블로그 묶음·글 카드·빈 결과)
│       ├── animal-collection.tsx         [새 파일] 도감 카드 + 주인 전시 버튼 (클라이언트)
│       ├── post-card.tsx                 [참조] showAuthor 카드 (post 소유)
│       └── visit-count.tsx, record-visit.tsx   [참조] 방문 기록 (바뀌지 않음)
└── app/
    ├── blog/
    │   ├── [slug]/page.tsx               [변경] 트리·?sub=·검색 모드·도감·프로필, 데이터 병렬 조회
    │   └── actions.ts                    [참조] recordBlogVisit (blog 소유 함수, 바뀌지 않음)
    └── settings/
        ├── blog/
        │   ├── page.tsx                  [변경] 주소 폼, "(주소는 바꿀 수 없어요)" 삭제, 대분류·소분류 트리
        │   ├── actions.ts                [변경] FormState.values, 블로그 행 잠금·번호 다시 매기기, 이름 바꾸기 0행 → 잘못된 요청,
        │   │                                    대분류 삭제 전 글 칸 비우기 / [새 함수] updateBlogSlug, 소분류 4개, setShowcaseAnimal
        │   └── settings-forms.tsx        [변경] 입력값 유지, 44px, CategoryManager 트리 / [새 컴포넌트] BlogSlugForm, SubcategoryRow
        └── account/                      (auth 소유, auth가 단계 1(U9)에서 골격을 새로 만듦)
            ├── page.tsx                  [추가] 닉네임 칸 한 줄 (auth의 "닉네임 자리")
            ├── nickname-form.tsx         [새 파일] 닉네임 폼 (클라이언트)
            └── nickname-actions.ts       [새 파일] updateNickname
drizzle/
├── NNNN_subcategories.sql                [새 파일] db:generate (B-M1)
├── NNNN_blog_roof_description.sql        [새 파일] db:generate (B-M2)
├── NNNN_blog_showcase.sql                [새 파일] db:generate 후 SET NULL 열 목록 손질 (B-M3)
└── meta/                                 [변경] 스냅숏·_journal (생성)
scripts/
└── test-blog.ts                          [새 파일] npm run test:blog
e2e/
├── blog-home.mjs                         [새 파일] US1·US2
├── blog-address.mjs                      [새 파일] US3
├── categories.mjs                        [새 파일] US5
├── blog-search.mjs                       [새 파일] US6
├── blog-showcase.mjs                     [새 파일] US4
├── blog-scale.mjs                        [새 파일] SC-003·SC-011 (프로덕션 빌드 대상)
├── params.mjs                            [추가] ?sub=, 소분류·주소·전시 Server Action 조작 인자
└── nonfunctional.mjs                     [추가] 검색 결과 주소
docs/02-erd.md                            [변경] blogs·subcategories 부분 (data-model 6절)
package.json                              [추가] test:blog를 test 체인 끝에
README.md                                 [추가] 스크립트 표에 새 e2e 6줄
```

**Structure Decision**: 기존 Next.js 단일 프로젝트 구조를 그대로 쓴다. 화면·Server Action은 `src/app/`(블로그 홈 `blog/[slug]`, 관리 `settings/blog`, 내 정보 `settings/account`), DB 쿼리는 `src/server/blog.ts`(blog 소유 함수 + post 소유 함수에 선택 인자 추가), DB를 쓰지 않는 규칙은 새 `src/lib/blog.ts`(단위 테스트 대상), 화면 조각은 `src/components/blog/`. 새 최상위 라우트는 만들지 않는다 (검색은 블로그 홈 모드, R-15).

## 변경 단위 (현재 코드 → spec 목표)

"단계"는 공통 맥락 5.1의 구현 순서다 (2: 주소·소분류 표·검색, 4: 블로그 홈 트리, 12: 도감·전시). B2는 town 단계 11 전에 넣는다 (R-22).

| # | 단위 | 종류 | 현재 코드 | 목표 | FR / SC | 선행 | 단계 |
|---|---|---|---|---|---|---|---|
| B1 | 소분류 표 (B-M1) | 마이그레이션 | 없음 | `subcategories` (FK CASCADE, CHECK 1~20, UNIQUE(`category_id`,`name`), UNIQUE(`category_id`,`id`)) | FR-034·035·041 | 없음 | 2 |
| B2 | 블로그 작은 변경 (B-M2) | 마이그레이션 | `roof_color` 없음, 소개 CHECK 없음 | `roof_color` + 8색 CHECK (요청: town), `blogs_description_check` | FR-016, TOWN-07 | 없음 (점검 R-28) | 2 |
| B3 | 주소 변경 | 서버 + 화면 | `블로그 주소: /@{slug} (주소는 바꿀 수 없어요)` 표시만 (`src/app/settings/blog/page.tsx:55`) | `updateBlogSlug` + 주소 폼: 정규화·형식·예약어(관리자 `notice` 예외)·이름 잠금·아이디 겹침·UNIQUE | FR-009·010·013·018·021, SC-005·012 | auth 1 (예약어 모듈·이름 잠금·`username` NOT NULL) | 2 |
| B4 | 기본 정보 보강 | 서버 + 화면 | 오류 때 칸이 저장된 값으로 되돌아감, 길이 UTF-16 | `values` 돌려받아 남기기, `charCount` | FR-016·017, spec 기본값 | 없음 | 2 |
| B5 | 닉네임 변경 | 서버 + 화면 | 없음 (온보딩에서 2~12자로만 정함) | `updateNickname` + 내 정보 닉네임 칸 | FR-019·020, SC-012 | auth 1 (`nickname` CHECK 2~20, `lockName`·`findNameConflict`, `/settings/account` 골격 U9) | 2 |
| B6 | 대분류 동작 보강 | 서버 + 화면 | 잠금 없음, 삭제 뒤 빈 번호, 다른 블로그 ID 이름 바꾸기도 `{ ok }`, 추가 오류 때 칸 비워짐, 작은 버튼 | 블로그 행 `FOR UPDATE`, 삭제 뒤 다시 매김, 0행 → `잘못된 요청이에요`, `values`, 44px | FR-033·036·037·039·042, VI | 없음 | 2 |
| B7 | 소분류 관리 | 서버 + 화면 | 없음 | `addSubcategory`·`renameSubcategory`·`deleteSubcategory`·`moveSubcategory` + 관리 화면 트리 (소분류 글 수는 B8에서) | FR-034~039·042, SC-007 | B1 | 2 |
| B8 | 블로그 홈 트리·거르기·글 수 | 서버 + 화면 | 대분류를 `└ 이름 (N)`으로 평평하게, `category_id`로만 거름 | `getCategories` 트리, `category-nav.tsx`, `?sub=`, `listBlogPosts`의 `subcategoryId`(post 변경 13이 넣음, 먼저 들어간 쪽을 그대로 씀), 관리 화면 소분류 글 수, `deleteCategory`가 글 두 칸을 먼저 비움 | FR-039·040·056·057·059 | post 3 (`posts.subcategory_id` + FK + CHECK + 트리거 `posts_clear_subcategory`) | 4 |
| B9 | 검색 | 서버 + 화면 | 없음 | 블로그 홈 `?q=` 주인 전용 모드, `listFeed`에 `search`(추가), `searchBlogs`, `blog-search.tsx` | FR-050~053, SC-011·013 | 없음 (e2e는 auth 1) | 2 |
| B10 | 전시 컬럼 (B-M3) | 마이그레이션 | 없음 | `showcase_animal_id` + 복합 FK `ON DELETE SET NULL (showcase_animal_id)` (생성 SQL 손질) | FR-030·031 | town 11 (`user_animals` UNIQUE(`user_id`,`id`)) | 12 |
| B11 | 도감·전시·주인 프로필 | 서버 + 화면 | 미니룸 + 닉네임 배지만. 다 키운 동물 카드는 농장 화면에만 | `getGrownAnimals`, `setShowcaseAnimal`, `animal-collection.tsx`, MiniRoom `showcase`, 블로그 정보의 주인 프로필 | FR-028~031, US4 | B10, auth 1 (`photo_key`; 없으면 캐릭터 얼굴만) | 12 |
| B12 | 검증 | 테스트 | `test-blog` 없음, 블로그 2단계·주소·검색·도감 e2e 없음 | `test:blog`, 새 e2e 6개, `params.mjs`·`nonfunctional.mjs` 추가 | 모든 SC | 각 단위 | 각 단계 |
| B13 | 문서 | 문서 | ERD에 ⏳ | `docs/02-erd.md` 담당 부분, README 스크립트 표 | 원칙 II, 공통 맥락 3.1-5 | 각 마이그레이션 | 각 단계 |

**지금 코드가 이미 지키는 것** (바꾸지 않고 B12로 확인만): FR-004·005(회원당 1개, 만들기·지우기 기능 없음. 회원 삭제 CASCADE는 post 단계 3 뒤로는 post 트리거 `posts_clear_subcategory`에 기댄다, R-11), FR-006(`scripts/create-admin.ts`), FR-007(광장 내 집·`내 블로그로 →`), FR-008·011·012·014·015(rewrites, `parseId`, 404, 발행·삭제 뒤 이동, 탭 제목), FR-022, FR-023~027·032(미니룸·버튼·글 화면 캐릭터 얼굴), FR-043~049(방문자 수, `e2e/visits.mjs`), FR-054~058(트리 제외), FR-060. FR-001~003은 auth가 가입 트랜잭션으로 옮긴다 (R-27). 그래서 원자성 검증(US1-3, SC-002)은 auth의 검증 결과를 쓰고, blog는 기본값(US1-1·8, SC-001)만 `e2e/blog-home.mjs`로 확인한다.

## 의존성

### 이 기능보다 먼저 되어야 하는 일

| spec (단계) | 작업 | 필요한 단위 | 없을 때 |
|---|---|---|---|
| auth (1) | 가입 통합: 가입 트랜잭션이 blog 기본값으로 블로그·"일상"을 만든다 (FR-001~003, R-27). `e2e/helpers.mjs`의 `loginDev` 고침 | B12 전체 (모든 e2e 전제), US1 | e2e가 로그인에서 멈춘다 |
| auth (1, U4) | 이름 모듈 `src/lib/names.ts`(`RESERVED_NAMES`·`normalizeName`·`isReservedName`, 목록 값은 blog FR-009 16개)와 `src/server/names.ts`(`lockName`·`findNameConflict`) (R-02·R-04). 가입도 같은 잠금으로 남의 주소·닉네임과 겹치는 아이디를 막는다 | B3, B5 | 주소·닉네임 규칙이 가입과 어긋나고 동시성 보장이 없다 |
| auth (1, U2·U3) | `users.username` NOT NULL + 형식 CHECK, `profiles.nickname` CHECK 2~20, `profiles.photo_key` | B3(겹침 검사 전제), B5(13~20자 저장), B11(사진) | 13~20자 닉네임이 DB에서 거부, 사진 자리는 캐릭터 얼굴만 |
| auth (1, U9) | `/settings/account` 페이지 골격(닉네임 자리). 연동·탈퇴는 auth 단계 8·9 | B5 | 닉네임 칸을 둘 곳이 없다 |
| post (3) | `posts.subcategory_id` + 복합 FK `ON DELETE SET NULL (subcategory_id)` + CHECK + 트리거 `posts_clear_subcategory`(대분류·회원 삭제 때 CHECK 위반 방지, R-11), 글쓰기 대분류·소분류 두 칸과 서버 검사 (FR-041, US5-11), `listBlogPosts`의 `subcategoryId`(post 변경 13) | B8 | 소분류 글 수·거르기·US5-7·8·11을 확인할 수 없다. 트리거가 없으면 소분류 글이 있는 회원의 삭제가 실패할 수 있다 (FR-005) |
| post | `/files/[key]`가 `profiles.photo_key` 첨부는 누구에게나 내려준다 (contracts/blog-home.md 5절) | B11 | 비공개 보호가 프로필 사진까지 막을 수 있다 |
| post | `src/components/pagination.tsx` 페이지 번호 누르는 영역 44×44px (FR-059) | B8 (FR-059 완성) | 블로그 홈 페이지 번호만 44px 미달 |
| town (11) | `user_animals` UNIQUE(`user_id`, `id`) | B10 | 전시 복합 FK를 만들 수 없다 |

### 이 기능이 다른 spec에 주는 것

| 받는 spec | 주는 것 | 시점 |
|---|---|---|
| post | `subcategories` 표(posts 복합 FK 대상 UNIQUE(`category_id`,`id`)), `getCategories` 트리(글쓰기 선택 순서 FR-041), 대분류 삭제 때 글 두 칸을 먼저 비우는 동작 | B1 (단계 2), B8 |
| auth | 예약어 목록 값 16개, 가입 블로그 기본값(이름 `{아이디}의 블로그`, 주소 = 아이디, 소개 `''`, 배경 초원, 대분류 "일상") | 단계 1 전에 합의 |
| town | `blogs.roof_color`(+ `ROOF_COLORS` 코드값, TOWN-07), MiniRoom 전시 동물, 바뀐 블로그 이름·닉네임·주소가 광장 집에 반영되는 데이터 | B2는 town 11 전, B11 |
| shop | MiniRoom prop 방식 (가구 층과 전시 동물 자리를 겹치지 않게 협의) | B11 / shop 10 |
| social | `listFeed`에 `search` 선택 인자 (이웃 거르기·즐겨찾기 정렬과 독립), 블로그 홈 정보 줄의 이웃 버튼 자리 유지 | B9 |

### 공통 모듈 추가 (다른 spec 소유 파일, 끼워 넣기만)

- `src/server/blog.ts`의 `listBlogPosts` 선택 인자 `subcategoryId`는 post가 변경 13에서 넣는다 (post plan 의존성 표). blog가 먼저 필요하면 같은 모양으로 추가만 하고, 먼저 들어간 쪽을 다른 쪽이 그대로 쓴다
- 공통 모듈 추가: `src/server/blog.ts` — `listFeed`에 선택 인자 `search` (post 소유 공통 부분, 없으면 지금 동작 그대로. social의 `orderFirst`(social D-5)와 같은 함수를 고치므로 나중에 merge하는 쪽이 최신 `main`에 맞춘다)
- 공통 모듈 추가: `src/app/settings/account/page.tsx` — 닉네임 칸 컴포넌트 한 줄 (auth 소유, auth U9가 만든 "닉네임 자리")
- 공통 모듈 추가: `e2e/params.mjs` — `?sub=`, 소분류·주소·전시 Server Action 조작 인자
- 공통 모듈 추가: `e2e/nonfunctional.mjs` — 검색 결과 주소
- 공통 모듈 추가: `package.json` — `test:blog`를 `test` 체인 끝에
- 공통 모듈 추가: `README.md` — 스크립트 표에 새 e2e 6줄

### 다른 spec 담당 표에 대한 요청 (data-model 1절)

- auth: `profiles.nickname` CHECK 2~20, `profiles.photo_key`, `users.username` NOT NULL (이미 결정됨, 공통 맥락 3.4)
- post: `posts.subcategory_id` + 복합 FK + CHECK (이미 결정됨) + 트리거 `posts_clear_subcategory` (post가 정함, blog는 동의 — R-11). 측정 뒤 필요하면 `posts(category_id)`·`posts(subcategory_id)`·`pg_trgm` 인덱스 (R-14·R-16)
- town: `user_animals` UNIQUE(`user_id`, `id`) (이미 결정됨)

## 남은 문제 (팀 확인 필요)

1. **spec에 없는 화면 문구** (원칙 III): 검색창 안내 `마을의 글·블로그 검색`·[검색], 검색 결과 제목 `🔍 '{검색어}' 검색 결과`·`블로그`·`글 N개`, 도감 제목 `🏅 동물 도감`, 전시 [전시하기]·`전시 중`·[전시 빼기], 주소 폼 라벨 `블로그 주소`·[주소 바꾸기]·성공 `저장했어요 ✓`, 닉네임 성공 `저장했어요 ✓`, 소분류 추가 칸 안내 `새 소분류`, 화면 읽기용 이름 `새 소분류 이름`·`소분류 이름`·`검색어`. plan 임시값이며 spec에 옮길지 정해야 한다.
2. **51자 이상 검색어**: spec은 1~50자만 정했다. plan은 검색하지 않고 보통 블로그 홈을 보인다 (R-17).
3. **관리자 블로그 주소**: 관리자도 주소를 바꿀 수 있고 `notice`로만 돌아올 수 있게 예약어 예외를 둔다 (R-05). spec·auth spec에 없는 규칙.
4. **닉네임끼리 대소문자**: spec은 "서비스 전체에서 겹칠 수 없다"고만 적었다. plan은 지금 DB 규칙(글자 그대로)을 유지한다 (R-03).
5. **검색 결과 블로그 묶음**: 최대 8곳, 최근 공개 글 순, 1페이지에만 — plan 결정 (R-17).
6. **전시 동물 "미니룸 옆"**: plan은 미니룸 그림 안 캐릭터 오른쪽으로 해석했다 (R-20). TOWN-09 수용 기준 "프로필에 한 마리 전시"와 같은 뜻으로 본 spec 기본값을 따른다.
7. **US4-10 "모습은 같다"**: 주인에게만 도감 전시 버튼이 보이는 것은 FR-030 동작이라 차이로 보지 않았다.
8. **SC-005 "예전 주소를 가리키는 링크 0개"**: 사이트가 만드는 링크만 대상으로 보고, 사용자가 글 본문에 직접 적은 링크는 바꾸지 않는다 (R-06).
9. **프로필 사진 올리기 화면**: 어느 spec에도 없다 (`specs/README.md`). blog는 표시만 하고, e2e는 DB로 사진을 연결해 확인한다.
10. **대분류·회원 삭제 때 FK 동작 순서**: `posts.category_id` SET NULL이 소분류 CASCADE보다 먼저 돌아 CHECK 위반이 나는 문제는 블로그 관리 삭제뿐 아니라 회원 삭제(블로그 CASCADE) 경로에도 있다 (R-11). blog는 `deleteCategory`에서 글의 두 칸을 먼저 비우고, 모든 경로의 보장은 post의 트리거 `posts_clear_subcategory`(post research R4)가 맡는다. 실제 DB에서의 실행 순서는 구현 때 `e2e/categories.mjs`(대분류 삭제)와 "소분류 글이 있는 회원 삭제" 시나리오로 확인한다 (추측). 트리거 방식은 post 남은 문제 2로 팀 확인 중이다.
11. **설치된 패키지 문서 확인**: `node_modules`가 없어 R-26 목록(Next.js Server Action Origin 확인·`next/form`, React 폼 초기화, Drizzle `.for("update")`·FK 열 목록, zod 길이 단위, Better Auth username 소문자)은 구현 전에 확인한다.
12. **소분류 글 수 시점**: post 단계 3 전에는 관리 화면 소분류 줄에 글 수가 없다 (B7 → B8, R-14).
13. **예약어 `notifications`**: game plan(남은 문제 8)이 새 최상위 주소 `/notifications`를 예약어에 더할지 물었다. 목록 값의 주인은 blog지만 spec FR-009가 16개를 정했으므로 plan은 16개를 유지한다. 더하려면 spec FR-009를 먼저 고친다 (auth 가입 검사도 같은 목록).

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| 원칙 V "DB 제약조건으로도 막는다" — 아이디 ↔ 주소·닉네임 겹침을 DB 제약 대신 이름 단위 advisory lock + 트랜잭션 안 확인으로 막는다 | 아이디(`users`)·주소(`blogs`)·닉네임(`profiles`)이 다른 표라 UNIQUE·CHECK로 표현할 수 없다. 가입과 변경이 동시에 와도 한쪽만 성공해야 한다 (auth Edge Cases, SC-012) | 이름 등록 표(UNIQUE(name)): 한 회원이 같은 값을 세 곳에 쓰는 기본 상태를 표현하려면 소유자 묶음·참조 수 관리가 필요하다. 트리거: READ COMMITTED에서 같은 경쟁이 남는다 (R-04) |
| 원칙 V — "다 키운 동물만 전시"를 서버 확인으로 지킨다 (소유는 복합 FK) | 동물 상태는 다른 표의 값이라 CHECK로 막을 수 없다 (ERD 3.11) | 트리거: 규칙이 숨는다. 상태가 egg → growing → grown 한 방향이라 확인과 저장을 한 UPDATE 문장으로 묶으면 경쟁이 없다 (R-19) |
| 원칙 V — 카테고리 순서의 빈틈·겹침 없음을 블로그 행 잠금 + 다시 매기기로 지킨다 | UNIQUE(`blog_id`, `position`)은 맞바꾸는 중간 상태에서 위반된다 | DEFERRABLE UNIQUE: 대분류·소분류 모든 갱신에 미룬 제약 관리가 필요하다 (R-12) |
| 원칙 III "문구는 spec에 그대로" — 일부 화면 문구를 plan이 임시로 정했다 | 검색 결과·도감·전시·주소 폼의 문구가 spec에 없어 화면을 설계할 수 없다 | spec 수정은 이 단계 범위 밖(spec.md는 고치지 않음). 남은 문제 1로 팀 확인 후 spec에 옮긴다 |
