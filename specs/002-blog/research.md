# Research: 블로그 (BLOG)

**Feature**: `002-blog` | **Date**: 2026-10-07 | **Plan**: [plan.md](plan.md) | **Spec**: [spec.md](spec.md)

- 기준: 코드 저장소 `main` `feb4c05`, `docs/02-erd.md` v1.4, 공통 맥락 문서(plan-context, 테이블 담당 규칙 3.1·공통 모듈 소유 5.2·구현 순서 5.1).
- 근거 표기: **코드 확인**(파일·줄을 읽고 확인), **라이브러리 동작**(문서·기존 코드 사용례로 아는 동작), **추측**(확인하지 못함). 코드 체크아웃에 `node_modules`가 없어 설치된 패키지 문서를 열 수 없으므로, 라이브러리 세부 API에 기대는 결정은 끝에 "구현 전 확인"을 붙였다 (R-26에 모음).
- 코드 위치는 코드 저장소 기준, 문서는 문서 저장소 기준 상대 경로다.

## 0. Technical Context에서 남은 질문

| 질문 | 결론 | 절 |
|---|---|---|
| 서비스 규모·배포 환경 (NF-08 미정) | 블로그당 글 1,000개, 마을 공개 글 수만 건 이하로 가정. 로컬 프로덕션 빌드로 잰다 | R-01 |
| PostgreSQL 최소 버전 | 15 이상 (`ON DELETE SET NULL (컬럼)`). README 안내는 17 | R-18 |
| 아이디 ↔ 주소·닉네임 겹침의 동시성 | 이름 단위 advisory lock (auth와 같은 키) | R-04 |
| 검색 방식·위치 | 블로그 홈 `?q=` 주인 전용 모드, `ILIKE` 부분 일치 | R-15, R-16 |
| Drizzle이 `SET NULL (컬럼)`을 표현하는지 | 모르면 생성 SQL을 손질 | R-18 |
| 설치된 패키지 문서로 확인할 것 | 목록으로 남김 | R-26 |

`NEEDS CLARIFICATION`으로 남긴 항목은 없다. 팀 확인이 필요한 plan 결정은 [plan.md](plan.md)의 "남은 문제"에 모았다.

---

## R-01 규모와 성능 기준을 재는 방법

- **Decision**: 성능 목표는 spec 그대로 쓴다 — 블로그 홈 1초(배포 환경, 캐시 없는 첫 방문, 글 1,000개, SC-003·NF-07), 검색 결과 첫 페이지 1초(SC-011). 배포 환경이 정해지지 않았으므로(NF-08) 지금 관례대로 `npm run build && npm run start` 대상에서 잰다. 새 검증 `e2e/blog-scale.mjs`가 전용 회원 블로그에 공개 글 1,000개를 DB로 채운 뒤 두 화면의 첫 방문 시간을 잰다. 규모는 블로그당 글 1,000개, 마을 공개 글 수만 건 이하로 가정한다 (**추측**: 3명 팀 과제 서비스).
- **Rationale**: `e2e/nonfunctional.mjs`가 NF-06·NF-07을 같은 방식(프로덕션 빌드, 새 컨텍스트)으로 이미 잰다 (**코드 확인**). 블로그 홈 글 목록은 `posts_blog_created_idx (blog_id, created_at DESC)`를 탄다 (`src/db/schema.ts`). 글 1,000개짜리 블로그는 지금 테스트 데이터에 없어 새로 만들어야 한다.
- **Alternatives considered**: 부하 테스트 도구(k6 등) — 새 의존성, 동시 사용자 목표가 spec에 없다. 측정 없이 코드 확인만 — SC-003·SC-011이 시간 기준이라 재야 한다.

## R-02 예약어 목록과 이름 겹침 검사 모듈

- **Decision**: auth가 단계 1(auth plan U4, auth research R3)에서 새로 만드는 공통 모듈을 쓴다 — `src/lib/names.ts`의 `RESERVED_NAMES`·`normalizeName()`·`isReservedName()`, DB를 쓰는 `src/server/names.ts`의 `lockName(tx, name)`·`findNameConflict(tx, name, { exceptUserId })`(다른 회원의 `users.username`·`blogs.slug`·`lower(profiles.nickname)`이 `lower(name)`과 같은지 `{ username, slug, nickname }`로 돌려줌). blog는 **목록 값의 주인**이다: FR-009의 16개 `admin api town feed shop closet write settings blog onboarding farm attendance tags wallet files notice`. 주소 변경(`updateBlogSlug`)과 닉네임 변경(`updateNickname`)이 이 모듈을 부르되, `findNameConflict` 결과 중 **주소는 `username`·`slug`만, 닉네임은 `username`만** 본다 (R-03). 닉네임은 예약어 검사를 하지 않는다 (FR-019에 없음).
- **Rationale**: 공통 맥락 5.2 "값은 blog, 모듈은 auth". 지금 예약어는 온보딩의 `RESERVED_SLUGS` 11개뿐이고 주소에만 쓴다 (`src/app/onboarding/actions.ts:30`, **코드 확인**). 온보딩이 없어지면 이 상수도 사라지므로 새 모듈이 필요하다. D1은 같은 목록을 가입 아이디와 주소에 쓰라고 정했다.
- **Alternatives considered**: blog 전용 목록을 따로 두기 — 원본 문서에서 이미 AUTH-02와 BLOG-02 목록이 어긋났다(`specs/README.md` "원본 문서에서 고칠 곳"). DB 표로 예약어 관리 — 바꿀 일이 드물고, 코드 리뷰로 관리하는 편이 단순하다 (VII).

## R-03 "다른 회원의 아이디와 같은 값 금지"의 비교 방법

- **Decision**:
  - 주소: 정규화한 새 주소와 `users.username`이 같고 `users.id <> 나`인 회원이 있으면 `이미 있는 주소예요` (`findNameConflict(tx, 새 주소, { exceptUserId: 나 })`의 `username`, 다른 블로그 주소와 같으면 `slug`도 같은 문구. 마지막 보장은 `blogs_slug_unique`). 다른 회원의 닉네임과 같은 주소는 막지 않는다 (spec에 없는 규칙이라 `nickname` 결과는 쓰지 않는다).
  - 닉네임: 앞뒤 공백을 지운 닉네임을 소문자로 바꾼 값과 `users.username`이 같고 다른 회원이면 `이미 있는 닉네임이에요` (`findNameConflict`의 `username`만).
  - 닉네임끼리는 지금 DB 규칙(`profiles.nickname` UNIQUE, 글자 그대로 비교)을 그대로 쓴다. `findNameConflict`의 `nickname` 결과는 대소문자를 무시한 비교라 닉네임 변경에 쓰면 이 규칙과 어긋나므로 쓰지 않는다.
- **Rationale**: `users.username`은 늘 소문자 `[a-z0-9_]`로 저장된다 — 가입의 `usernameSchema`가 `trim().toLowerCase()` 후 정규식을 걸고 (`src/app/(auth)/actions.ts:12-16`, **코드 확인**), auth가 단계 1에서 `users_username_check`(`^[a-z0-9_]{4,20}$`)를 더해 DB도 보장한다 (auth research R4). 그래서 username 쪽에 `lower()`를 씌울 필요가 없고 UNIQUE 인덱스를 그대로 탄다. 닉네임끼리 대소문자를 무시하라는 규칙은 spec·D1에 없다. auth research R4는 "닉네임끼리 대소문자는 blog가 정하고, 무시로 정하면 auth가 `lower(nickname)` UNIQUE로 바꾼다"고 적었다 — plan은 지금 규칙을 유지하고 남은 문제 4로 둔다.
- **Alternatives considered**: `lower(username)` 함수 인덱스 — 값이 이미 소문자라 불필요. `citext` 타입 — 확장·타입 변경이 필요하고 ERD 7장 할 일에 없다. 닉네임끼리도 대소문자 무시 — spec에 없어 plan에서 정하지 않고 남은 문제로 남긴다.

## R-04 가입과 주소·닉네임 변경이 동시에 일어날 때 (이름 잠금)

- **Decision**: 주소·닉네임 변경 트랜잭션은 **바꾸려는 값으로 이름 잠금(`lockName(tx, 값)`)을 먼저 건 뒤** `findNameConflict`로 username을 확인하고 UPDATE한다. 잠금 키는 auth가 정했다 — `pg_advisory_xact_lock(<이름 잠금 번호>, hashtext(lower(name)))` (auth research R3). 두 정수 키 형식이라 `lockUser`(한 정수 키 `hashtext(userId)`)와 키 공간이 섞이지 않는다. auth의 가입 처리도 같은 키로 잠근 뒤 다른 회원의 `blogs.slug`·`profiles.nickname`(대소문자 무시)을 확인하고 회원을 만든다. 한 트랜잭션은 이름 잠금을 하나만 잡는다 (교착 없음).
- **Rationale**: 아이디(`users`)와 주소(`blogs`)·닉네임(`profiles`)은 다른 표라 UNIQUE 하나로 막을 수 없다 (공통 맥락 5.3, auth spec Edge Cases "둘 중 한쪽만 성공"). advisory lock은 이 저장소가 회원 단위 잠금으로 이미 쓰는 방식이다 (`src/server/points.ts`의 `lockUser`, **코드 확인**). PostgreSQL에서 64비트 한 키와 32비트 두 키의 잠금 공간은 겹치지 않는다 (**라이브러리 동작**). 주소끼리·닉네임끼리의 경쟁은 각 UNIQUE가 막으므로 잠금은 아이디와의 경쟁만 줄 세운다.
- **Alternatives considered**: 이름 등록 표(`names`, UNIQUE(name)) — 한 회원이 같은 값을 아이디·주소·닉네임으로 함께 쓰는 기본 상태(가입 직후)를 표현하려면 소유자 묶음과 참조 수 관리가 필요하다. 트리거로 검사 — READ COMMITTED에서 같은 경쟁이 그대로 남는다. SERIALIZABLE 격리 — 재시도 처리가 필요하고 다른 처리와 섞인다.

## R-05 관리자 블로그 `notice`의 주소 변경

- **Decision**: (1) 새 주소가 지금 주소와 같으면 다른 검사 없이 성공으로 끝낸다(저장하지 않음). (2) 예약어 중 `notice`는 `users.role = 'admin'`인 회원에게만 허용한다. 나머지 규칙은 관리자도 같다.
- **Rationale**: FR-009는 "회원 주소"에 예약어를 금지하고, Edge Cases는 `notice`를 "회원이 먼저 쓰지 못하도록" 예약했다 — 목적은 관리자 전용 주소다. 면제가 없으면 관리자가 주소를 한 번 바꾼 뒤 `/@notice`로 돌아올 수 없다. `scripts/create-admin.ts:64-81`은 프로필이 이미 있으면 블로그 만들기를 건너뛰어 다시 실행해도 복구되지 않는다 (**코드 확인**). (1)은 주소 폼을 바꾸지 않고 다시 보냈을 때, 예전 규칙으로 만든 주소(예: 이제 예약어가 된 `tags`)를 가진 회원이 오류 문구를 보지 않게 한다 (주소 폼은 이름·소개와 나뉘어 있다, R-09).
- **Alternatives considered**: 관리자 주소 변경을 막기 — FR-018은 모든 블로그 주인에게 허용한다. 면제 없음 — 공지 블로그 주소를 되돌릴 방법이 없다. spec에 없는 규칙이라 팀 확인을 남은 문제로 남긴다.

## R-06 주소를 바꾸면 사이트 전체가 새 주소를 가리키는 방식

- **Decision**: UPDATE 한 번(`blogs.slug`) + `revalidatePath("/", "layout")`. 예전 주소는 다음 요청부터 `getBlogBySlug()`가 찾지 못해 404가 된다. 자동 이동(redirect·rewrite)은 두지 않는다. 글 본문 HTML 안에 사용자가 직접 적은 예전 주소 링크는 바꾸지 않는다.
- **Rationale**: 주소는 `blogs.slug` 한 곳에만 저장되고 다른 표에 복사되지 않는다 (`src/db/schema.ts`, ERD 3.17 3NF). 사이트의 링크는 모두 그릴 때 slug로 만든다 — `src/components/blog/blog-header.tsx`, `src/components/blog/post-card.tsx`, `src/server/town.ts`(광장 집), `src/components/town/town-menu.tsx`, `src/app/write/actions.ts:137·145`(발행·삭제 뒤 이동) (**코드 확인**). 블로그 화면은 `getViewer()`(쿠키)를 읽어 요청마다 그려진다. D3 "자동 이동 없음". FR-013의 목록(광장·글 카드·댓글 작성자·글 화면·관리)은 사이트가 만드는 링크이고, 본문은 사용자가 쓴 내용이다.
- **Alternatives considered**: 예전 주소 기록 표 + 리다이렉트 — D3로 범위 밖. 본문 링크 일괄 치환 — 다른 회원 글 본문까지 바꿔야 하고 정화·`updated_at`·보상 판단과 얽힌다.

## R-07 글자 수를 세는 기준

- **Decision**: blog 영역의 길이 검사(블로그 이름 1~40, 소개 0~160, 닉네임 2~20, 카테고리 1~20, 검색어 1~50)는 **코드 포인트 수**로 센다. 순수 함수 `charCount()`(`Array.from(s).length`)를 새 파일 `src/lib/blog.ts`(blog가 새로 만듦, plan Project Structure)에 두고 zod 스키마에서 `.refine`으로 쓴다. 문구는 spec 그대로다.
- **Rationale**: DB CHECK는 `char_length`(코드 포인트)로 센다 (`profiles_nickname_check`, `blogs_title_check`, `categories_name_check`, **코드 확인**). JS 문자열 `length`는 UTF-16 단위라 이모지 하나를 2로 센다. zod 4의 문자열 `.min()/.max()`가 `length`를 쓴다면(**라이브러리 동작**, 구현 전 확인) 닉네임 `😀`(1자)은 서버 검사 `min(2)`를 통과하고 DB CHECK(2자 이상)에 걸려 **500 오류**가 난다 — FR-019 문구 대신 오류 화면이다. 최대 길이 쪽은 반대로 이모지를 쓴 이름을 DB보다 엄격하게 막는다.
- 입력칸의 `maxLength`도 브라우저가 UTF-16 단위로 센다. 이모지가 많은 값은 화면에서 먼저 잘릴 수 있지만(서버·DB보다 엄격한 쪽) 500으로 이어지지는 않는다. 서버 판단이 기준이다.
- **Alternatives considered**: 그대로 두기 — 500 위험. 그래핌 단위(`Intl.Segmenter`) — DB `char_length`와 다시 어긋난다.

## R-08 오류가 나도 입력한 값을 남기기

- **Decision**: 블로그 관리의 기본 정보·주소·대분류 추가·소분류 추가 폼과 내 정보의 닉네임 폼은 Server Action 결과(`FormState`)에 `values`를 돌려주고, 칸을 `defaultValue={state.values?.칸 ?? 저장된 값}`으로 그린다. 성공하면 저장된 값(추가 칸은 빈 값)을 보인다. 이름 바꾸기 칸은 지금처럼 폼이 아닌 `useState` 칸이라 그대로 남는다.
- **Rationale**: React 19의 `<form action>`은 처리가 끝나면 비제어 입력을 초기화한다 (**라이브러리 동작**: 원본 요구사항 BLOG-03·BLOG-05 예외 흐름, SOC 열린 질문이 이 저장소에서 이 동작을 기록했다). 같은 저장소의 로그인·가입 폼이 `values`를 돌려받아 `defaultValue={state.values?.username}`으로 남기는 방식을 이미 쓴다 (`src/components/login-buttons.tsx:21·36`, **코드 확인**). spec 기본값 "블로그 관리에서 오류가 나면 입력한 값을 남긴다".
- **Alternatives considered**: 모든 칸을 `useState` 제어 입력으로 — 코드가 늘고 서버가 정규화한 값(소문자 주소)을 따로 반영해야 한다. `onSubmit` + `preventDefault` — 기존 폼들과 방식이 달라진다.

## R-09 주소 변경을 기본 정보 저장과 나누기

- **Decision**: "기본 정보" 카드 안에 이름·소개 폼([저장])과 **별도의 주소 폼**(`/@` + 주소 칸 + [주소 바꾸기])을 둔다. 결과 문구는 각 폼의 버튼 왼쪽에 한 줄. 지금의 `블로그 주소: /@{slug} (주소는 바꿀 수 없어요)` 줄은 없앤다.
- **Rationale**: FR-017은 "오류가 둘 이상이면 이름 → 소개 순으로 첫 번째 하나만"이다. 주소가 같은 폼에 들어오면 순서가 spec에 없다. 주소 변경은 예전 링크를 끊는 다른 성격의 변경이라, 이름만 고치려다 주소가 바뀌는 일을 막는다. 이름 잠금·아이디 확인이 있는 트랜잭션을 이름·소개 UPDATE와 섞지 않는다.
- **Alternatives considered**: 한 폼 — 오류 순서가 모호하고 실수로 주소가 바뀐다. 주소를 별도 화면으로 — 화면이 늘고 FR-018 "블로그 관리에서"와 다르다.

## R-10 소분류 표 설계

- **Decision**: ERD 3.18 그대로 `subcategories` — `id` integer identity PK, `category_id` integer NOT NULL FK → `categories.id` `ON DELETE CASCADE`, `name` text CHECK `char_length` 1~20, `position` integer 기본 0, UNIQUE(`category_id`, `name`), UNIQUE(`category_id`, `id`)(posts 복합 FK 대상). `blog_id`는 두지 않는다. 모두 Drizzle 스키마로 표현된다 — `categories`와 같은 `unique().on()`, `check()`, `.references(..., { onDelete: "cascade" })` 문법 (**코드 확인**: `src/db/schema.ts`의 categories 블록).
- **Rationale**: ERD 3.18, 공통 맥락 3.3. `position`은 ERD 표기(SMALLINT)가 아니라 지금 `categories.position`과 같은 integer로 둔다 (공통 맥락 3.6 "plan에서 타입을 바꾸지 않는다"). UNIQUE(`category_id`, `name`)의 첫 컬럼이 `category_id`라 대분류별 조회·CASCADE 삭제에도 이 인덱스를 쓴다.
- **Alternatives considered**: `categories.parent_id` 자기 참조 — 3단계를 DB가 막지 못해 ERD 3.18이 거부했다. `subcategories.blog_id` — 대분류로 알 수 있는 값의 중복(3NF 위반).

## R-11 대분류를 지울 때 글의 CHECK 위반을 피하기

- **Decision**: post가 `posts.subcategory_id`와 CHECK(`subcategory_id IS NULL OR category_id IS NOT NULL`), 트리거 `posts_clear_subcategory`(`BEFORE UPDATE OF category_id`, 새 `category_id`가 NULL이면 `subcategory_id`도 NULL, post research R4)를 넣은 뒤(post 단계 3)부터, `deleteCategory`는 한 트랜잭션에서 (1) 내 블로그 행을 잠그고 (2) 그 대분류가 내 블로그 것인지 확인한 뒤 (3) 그 대분류 글의 `category_id`·`subcategory_id`를 **함께** NULL로 바꾸고(`updated_at`은 그대로 둔다) (4) 대분류를 지운다(소분류는 CASCADE). post 단계 3 전에는 (3) 없이 지운다. **모든 삭제 경로의 보장은 post의 트리거가 맡고**, blog의 (3)은 트리거와 충돌하지 않는 앱 쪽 처리로 둔다 (먼저 비워졌으면 트리거가 할 일이 없다).
- **Rationale**: 대분류 DELETE 하나에 FK 동작 두 개가 걸린다 — `posts.category_id` SET NULL, 그리고 `subcategories` CASCADE → posts 복합 FK `SET NULL (subcategory_id)`. PostgreSQL은 같은 이벤트의 내부 FK 트리거를 이름 순(`RI_ConstraintTrigger_a_<oid>`, 대개 만든 순)으로 실행한다. `posts.category_id` FK는 `drizzle/0000_init.sql`에서 먼저 만들어졌으므로 먼저 돌아, `subcategory_id`가 남은 중간 행이 CHECK에 걸려 삭제 전체가 실패한다 (**추측**: 이름순 규칙에 근거, 구현 때 실제 DB에서 확인). CHECK는 미룰(DEFERRABLE) 수 없다. **회원 삭제도 같은 문제를 겪는다** — `blogs`를 지우면 `categories` CASCADE(`drizzle/0000_init.sql` 186행)가 `posts` CASCADE(197행)보다 먼저 만들어져 먼저 돌고, 아직 남은 글에 `category_id` SET NULL이 걸린다 (FK 생성 순서는 **코드 확인**, 실행 순서는 위와 같은 **추측**). 그래서 앱 처리만으로는 FR-005("회원이 지워지면 블로그·카테고리·글이 함께 지워진다")와 auth의 탈퇴·관리자 회원 삭제를 지킬 수 없고, post의 트리거가 필요하다. `updated_at`: Drizzle `$onUpdate`가 앱의 UPDATE마다 지금 시각을 넣으므로, `incrementViewCount`처럼 `updatedAt: sql\`updated_at\``로 덮어 둔다 (`src/server/blog.ts:258-264`, **코드 확인**). 카테고리 삭제는 글 내용 변경이 아니다.
- **Alternatives considered**: FK 실행 순서에 맡기기 — 불확실하고 마이그레이션 순서에 따라 달라진다. CHECK 빼기 — ERD 3.18 결정 위반이고 posts는 post 담당. 앱 처리만(트리거 없음) — 회원 삭제 CASCADE 경로와 "글을 비운 뒤·대분류를 지우기 전에 다른 탭이 같은 대분류로 저장한 글"을 막지 못한다 (post research R4). 트리거는 posts 담당인 post가 정했고, 이름·이유를 ERD 3.14·3.18과 마이그레이션 주석에 적어 "규칙이 숨는" 문제를 줄인다.

## R-12 카테고리 변경을 줄 세우고 순서 번호를 지키기

- **Decision**: 대분류·소분류의 추가·삭제·순서 바꾸기는 트랜잭션 첫 줄에서 **내 블로그 행을 `SELECT ... FOR UPDATE`로 잠근다** (Drizzle `.for("update")`, 구현 전 확인). 순서 바꾸기는 지금처럼 같은 단계 목록을 `position, id` 순으로 읽고 맞바꾼 뒤 0, 1, 2…로 다시 매긴다. 삭제 뒤에도 남은 형제를 다시 매긴다. 추가는 `MAX(position) + 1`. 소분류의 ▲▼는 같은 대분류 안에서만 움직인다.
- **Rationale**: 지금 `moveCategory`는 잠금 없이 읽고 다시 매긴다 (`src/app/settings/blog/actions.ts:75-94`, 원본 BLOG-05 무결성 "행 잠금은 따로 걸지 않는다", **코드 확인**). 두 탭에서 추가와 순서 바꾸기가 겹치면 같은 `position`이 생길 수 있고, 삭제는 빈 번호를 남긴다 → FR-039 "빈틈이나 겹침이 없어야". 블로그 행 하나를 잠그면 같은 블로그 요청만 줄을 서고 다른 블로그와는 경쟁하지 않는다.
- **Alternatives considered**: UNIQUE(`blog_id`, `position`) — 맞바꾸는 중간 상태가 위반이라 DEFERRABLE UNIQUE와 그 관리가 필요하다. advisory lock(블로그 ID) — 효과는 같지만 이미 있는 행을 잠그는 편이 단순하다. `FOR NO KEY UPDATE` — 같은 블로그에 글·방문 기록·카테고리를 넣는 요청의 FK 확인(블로그 행 `FOR KEY SHARE`)을 막지 않아 더 가볍다. 카테고리 변경은 짧은 트랜잭션이라 차이가 작아 `FOR UPDATE`로 두고, `e2e/blog-scale.mjs`나 동시 요청 시험에서 대기가 보이면 바꾼다 (둘 다 카테고리 변경끼리는 줄 세운다).

## R-13 블로그 홈 주소 값

- **Decision**: `/@{주소}?category=대분류ID`, `?sub=소분류ID`, `?page=N`, 주인만 `?q=검색어`. 숫자는 모두 `parseId()`. `sub`가 1~2147483647 정수면 소분류로 거르고(`category`는 무시), 아니면 `category`, 둘 다 아니면 전체 글. 페이지 링크는 고른 값을 유지한다. 없는 번호·다른 블로그 번호면 빈 목록 + 제목 `전체 글 0개` + 선택 표시 없음 (지금 대분류 동작과 같게, spec Edge Cases).
- **Rationale**: 지금 코드가 `?category=`를 `parseId`로 받아 이상한 값이면 전체 글을 보인다 (`src/app/blog/[slug]/page.tsx:26`, **코드 확인**). 소분류 ID도 서비스 전체 번호라 값 하나로 충분하다. FR-057.
- **Alternatives considered**: `?category=3&sub=7` 둘 다 요구 — 어긋난 조합 규칙이 더 필요하다. 경로(`/@주소/c/3`) — 글 주소 `/@주소/글ID`와 `next.config.ts` rewrites가 충돌한다.

## R-14 카테고리 트리와 글 수

- **Decision**: `getCategories(blogId, includePrivate)`의 이름·인자는 유지하고 각 대분류에 `subcategories`(이름·순서·글 수)를 더해 돌려준다. 대분류 글 수 = `posts.category_id = 대분류`(소분류 글 포함), 소분류 글 수 = `posts.subcategory_id = 소분류`. 주인이 보면(관리 화면은 늘) 비공개 포함. 쿼리는 대분류 1번 + 소분류 1번(`WHERE category_id IN (...)`)이고, 트리 조립·정렬(`position` → `id`)은 순수 함수 `buildCategoryTree()`(`src/lib/blog.ts`). 소분류 글 수는 post 단계 3 뒤에만 계산할 수 있으므로 단계 2에서는 관리 화면 소분류 줄에 글 수를 그리지 않는다.
- **Rationale**: 소분류를 고른 글은 그 대분류도 함께 가진다 (ERD 3.18 CHECK) → 대분류 수에 소분류 글이 저절로 들어가고 FR-040 "대분류를 누르면 소분류 글까지"와 맞는다. 지금 글 수는 대분류마다 하위 쿼리 COUNT다 (`src/server/blog.ts:81-96`, **코드 확인**). posts에 `category_id` 인덱스는 없지만 블로그당 글 1,000개면 충분하다고 본다 (**추측**) → `e2e/blog-scale.mjs`가 넘으면 post에 `posts(category_id)`·`posts(subcategory_id)` 인덱스를 요청한다. 글쓰기의 대분류·소분류 선택(post 소유, FR-041)도 같은 순서를 쓰도록 이 함수를 쓸 수 있다.
- **Alternatives considered**: 글 수 컬럼(`categories.post_count`) — 반정규화, 글 저장·삭제·공개 전환마다 갱신이 필요하다. 함수 이름을 `getCategoryTree`로 새로 — 호출하는 곳이 두 군데(블로그 홈, 관리)라 이름을 바꿀 이득이 작다.

## R-15 검색 위치와 권한

- **Decision**: 새 주소를 만들지 않고 **블로그 홈의 "검색 모드"**로 한다. 서버가 보는 사람이 그 블로그 주인일 때만 검색창을 그리고 `q`를 읽는다. 주인이 아니면 `q`를 무시하고 보통 블로그 홈을 그린다 (검색 쿼리를 돌리지 않는다). 광장·마을 소식에는 아무것도 더하지 않는다. 검색창은 GET 폼으로 `/@{내 주소}?q=…`로 이동한다 (`next/form`의 `Form`, 구현 전 확인. 없으면 `<form method="get">`).
- **Rationale**: D4·FR-050 "자기 블로그 홈(내 집)에서만". 주인 판정은 이미 서버에서 한다 (`src/app/blog/[slug]/page.tsx:22-24`, `viewerId === blog.ownerId`, **코드 확인**). 방문자가 주소에 `q`를 붙여도 검색이 돌지 않으므로 SC-013과 FR-060을 서버에서 지키고, 로그인하지 않은 요청이 무거운 부분 일치 쿼리를 돌릴 수 없다. 새 최상위 주소가 없어 `ExitButton` 숨김 목록·예약어·rewrites를 바꿀 일이 없다.
- **Alternatives considered**: `/search?q=` 회원 전용 화면 — "내 집 안에서" 결정과 어긋나고, 남의 블로그에서도 링크로 열리는 검색 화면이 생긴다. Route Handler 검색 API — 화면 이동 없는 검색은 spec에 없고, Route Handler는 Origin·회원 확인을 직접 해야 한다.

## R-16 검색 방식

- **Decision**: PostgreSQL `ILIKE '%검색어%'`로 공개 글의 `title`·`content_text`, 블로그의 `title`·주인 `profiles.nickname`을 찾는다. 검색어는 값으로 바인딩하고, `%`·`_`·`\`는 순수 함수 `toLikePattern()`(`src/lib/blog.ts`)이 `\`로 이스케이프한다. 인덱스는 지금 더하지 않는다. `e2e/blog-scale.mjs`의 검색 시간이 1초를 넘으면 `pg_trgm` GIN 인덱스를 더한다 (posts 인덱스는 post에 요청, blogs·profiles 쪽은 각 담당).
- **Rationale**: PostgreSQL 기본 전문 검색 사전에는 한국어 형태소 분석이 없어 `tsvector`는 띄어쓰기 단위로만 나눈다 (**라이브러리 동작**). 사용자가 기대하는 것은 부분 일치(`맛집`으로 `서울맛집` 찾기)다. `posts.content_text`는 검색용으로 이미 저장한다 (`src/db/schema.ts` 주석 "요약, 검색, 글자 수 확인용", **코드 확인**). 이스케이프하지 않으면 `%` 하나로 모든 글이 걸린다. Drizzle `ilike()`는 값을 바인딩한다 (원칙 IV).
- **Alternatives considered**: `tsvector` + `simple` 사전 — 붙여 쓴 말 안의 검색어를 못 찾는다. 외부 검색 엔진 — 범위 밖(VII). `pg_trgm`을 바로 — `CREATE EXTENSION`(PostgreSQL 13+에서 신뢰 확장)과 posts 인덱스(post 담당)가 필요해 측정 전에는 과하다.

## R-17 검색 결과 구성

- **Decision**:
  1. 검색어: 앞뒤 공백을 지운 뒤 0자면 `검색어를 적어 주세요`(결과 없음). 51자 이상이면 조작된 값으로 보고 검색하지 않고 보통 블로그 홈을 보인다 (입력칸 `maxLength=50`이라 화면에서는 보낼 수 없다). 순수 함수 `parseSearchQuery()`.
  2. 블로그 묶음: 이름이나 주인 닉네임에 검색어가 든 블로그를 **1페이지 위쪽에만 최대 8곳**, 최근 공개 글 순(공개 글이 없으면 뒤, 만든 순). 줄마다 캐릭터 얼굴(프로필 사진이 있으면 사진) · `{닉네임} · {블로그 이름}` · `@주소`, 누르면 `/@주소`. 새 함수 `searchBlogs()`(`src/server/blog.ts`).
  3. 글 카드: `listFeed({ page, search })` — 공개 글만, 최신순, 8개씩, 작성자 줄을 보이는 `PostCard showAuthor`(마을 소식과 같은 모양). 작성자 본인 비공개 글도 넣지 않는다.
  4. 둘 다 없으면 `검색 결과가 없어요`.
- **Rationale**: FR-051~053. 여러 블로그의 글이 섞이므로 작성자를 보여야 누구 글인지 안다 (공통 맥락 5.3 "검색 결과 카드 작성자 표시: blog plan이 정한다"). `listFeed`는 이미 공개 글·작성자 칸·8개 페이지·전체 수를 돌려준다 (`src/server/blog.ts:158-172`, **코드 확인**) → 선택 인자 `search`만 더하면 된다 (공통 모듈 추가 규칙, 없으면 지금 동작 그대로). 51자 처리는 "이상한 값이면 기본 화면"이라는 FR-057의 방식을 따랐다.
- **Alternatives considered**: 블로그 결과도 페이지 나누기 — 한 화면에 페이지 번호가 둘이라 헷갈린다. 글·블로그를 한 목록으로 — 카드 모양이 달라 섞기 어렵다. 51자 이상에 새 오류 문구 — spec에 없는 문구가 늘어난다.

## R-18 전시 동물 FK와 `ON DELETE SET NULL (컬럼)`

- **Decision**: `blogs.showcase_animal_id integer NULL` + 복합 FK `blogs_showcase_owned_fk`: (`owner_id`, `showcase_animal_id`) → `user_animals` (`user_id`, `id`) `ON DELETE SET NULL (showcase_animal_id)`. Drizzle 스키마에는 같은 복합 FK를 `foreignKey({...}).onDelete("set null")`로 적고, 생성된 마이그레이션 SQL의 그 FK 줄만 `ON DELETE SET NULL ("showcase_animal_id")`로 손으로 고친 뒤, SQL 맨 위 주석과 `schema.ts` 주석에 이유를 남긴다. 설치된 drizzle-orm이 열 목록을 지원하면 그 문법을 쓴다 (구현 전 확인). PostgreSQL 15 이상이 필요하다.
- **Rationale**: 열 목록 없는 `SET NULL`은 FK의 모든 열(`owner_id` 포함)을 NULL로 바꾸려 하고, `owner_id`는 NOT NULL이라 동물 삭제가 실패한다 (**라이브러리 동작**: PostgreSQL FK 동작). 복합 FK는 "주인 자신의 동물만"을 DB가 지킨다 (FR-031, ERD 3.4·3.11, 배경 장착 `blogs_background_owned_fk`와 같은 방식, **코드 확인**). 대상 UNIQUE(`user_id`, `id`)는 town이 먼저 만든다. 손질한 SQL은 drizzle 스냅숏과 어긋나지 않는다 — 다음 `db:generate`는 스냅숏과 스키마를 비교하므로 이 FK를 다시 만들지 않는다 (**라이브러리 동작**, 구현 전 확인).
- **Alternatives considered**: `showcase_animal_id` → `user_animals.id` 단일 FK + 코드로 주인 확인 — FR-031 "데이터 규칙으로도"를 못 지킨다. 전시 표 따로(블로그당 한 줄) — 칸 하나면 충분하고 표가 늘어난다. 스키마에서 FK를 빼고 직접 쓴 SQL만 — `schema.ts`가 ERD를 그대로 옮긴다는 원칙(파일 첫 줄 주석)과 어긋난다.

## R-19 "다 키운 동물만" 전시하기

- **Decision**: `setShowcaseAnimal(animalId)`는 UPDATE 한 문장으로 처리한다: `blogs.owner_id = 나`이고, `user_animals`에 `id = animalId AND user_id = 나 AND status = 'grown'`인 행이 있을 때만 `showcase_animal_id = animalId`. 맞는 행이 없으면 아무것도 바뀌지 않고 문구도 없다. 비우기(`null`)는 조건 없이 `showcase_animal_id = NULL`. 화면도 전시 동물을 "주인의 다 키운 동물 목록" 안에서만 찾아 그린다.
- **Rationale**: 상태는 다른 표의 값이라 CHECK로 막을 수 없다 (ERD 3.11). 동물 상태는 egg → growing → grown 한 방향이고 되돌리는 코드가 없다 (`src/server/farm.ts:105-107`, `src/app/farm/actions.ts`, **코드 확인**) → 확인과 저장 사이에 상태가 뒤로 갈 일이 없고, 한 문장이라 따로 트랜잭션이 필요 없다. 소유는 복합 FK가 다시 확인한다. 화면에서도 거르므로 Edge Case "전시한 동물이 더 이상 주인 소유가 아니게 됨 → 비어 보인다"를 지킨다.
- **Alternatives considered**: 트리거로 상태 확인 — 규칙이 숨는다. FK만 두고 상태 확인 없음 — 알·자라는 중인 동물을 전시할 수 있다.

## R-20 도감·전시·주인 프로필의 모양과 위치

- **Decision**:
  - 도감: 새 함수 `getGrownAnimals(ownerId)`(`src/server/blog.ts`)가 `status = 'grown'`인 동물의 ID·종류 이름·그림 키·다 키운 시각을 다 키운 시각 최신순으로 돌려준다. 블로그 홈에서 블로그 정보 카드 아래, 카테고리·글 목록 위에 작은 카드(동물 그림·이름·다 키운 날짜)로 줄바꿈해 보인다. 카드 모양은 농장 화면의 다 키운 동물 카드와 맞춘다.
  - 전시: `MiniRoom`에 선택 prop `showcase`(그림 키·이름)를 더해, 미니룸 안 캐릭터 오른쪽에 어른 단계 그림(`animalSvg(키, "adult")`)으로 그린다. 주인에게만 도감 카드마다 전시 버튼이 보인다.
  - 주인 프로필: 블로그 이름 왼쪽에 지름 48px 원형 프로필 사진(`/files/{photo_key}`), 없으면 `CharacterBadge`, 옆에 닉네임. FR-054의 위→아래 순서(미니룸 → 이름 → 소개 → 정보 줄)는 그대로다.
- **Rationale**: spec 기본값 "도감은 블로그 홈 안(미니룸·블로그 정보 아래) 카드 목록", "전시 동물은 미니룸 옆", FR-028 "사진이 없으면 캐릭터 얼굴". 동물 그림은 코드 SVG다 (`src/lib/art/animals.ts:51`, CLAUDE.md 그림 규칙). 농장 화면은 "도감은 2차에서 블로그로"라는 주석과 함께 다 키운 동물 카드를 이미 그린다 (`src/app/farm/farm-view.tsx:129-144`, **코드 확인**). `MiniRoom`은 blog 소유이고 shop도 가구 층을 더할 예정이다 (공통 맥락 5.2) → prop을 더하는 방식이면 서로의 층을 건드리지 않는다. `getGrownAnimals`가 이미 주인 동물만 돌려주므로 전시 동물은 이 목록에서 ID로 찾으면 되고 따로 JOIN하지 않는다. `getBlogBySlug`에는 가벼운 컬럼(`photoKey`, `showcaseAnimalId`)만 더한다 — 글 상세도 이 함수를 쓰기 때문이다.
- **Alternatives considered**: 도감 별도 탭·주소 — spec 기본값이 아니다. 전시 동물을 미니룸 밖 별도 칸에 — 375px에서 "옆"을 지키기 어렵고 미니룸과 떨어진다 ("옆"의 해석은 남은 문제로 남긴다).

## R-21 프로필 사진의 공개

- **Decision**: 사진은 `profiles.photo_key`(auth가 추가)가 가리키는 첨부를 `<img src="/files/{키}">`로 그린다. 사진을 올리는 화면은 이 plan 범위가 아니다. post가 `/files/[key]`에 비공개 글 보호를 넣을 때 **`photo_key`로 쓰는 첨부는 누구에게나 내려주도록** 요청한다.
- **Rationale**: FR-028 "누구에게나". post spec FR-059(정리 작업은 `photo_key` 첨부를 지우지 않음)와 짝이다. 올리기 화면 담당이 없다 (`specs/README.md` "spec끼리 아직 비어 있는 곳").
- **Alternatives considered**: 사진을 DB에 data URI로 — 첨부 구조(ERD 3.9)와 다르고 행이 커진다.

## R-22 지붕 색 칸 (요청: town, TOWN-07)

- **Decision**: `blogs.roof_color text NULL` + CHECK `blogs_roof_color_check`(`roof_color IS NULL OR roof_color IN ('red','orange','yellow','green','sky','blue','purple','brown')`). NULL = 배경 색 따라가기. 코드값 목록은 `src/lib/blog.ts`의 `ROOF_COLORS`(blog)에 두고, 실제 색(hex)·고르는 화면(`🏠 지붕 색`)·저장 Server Action·광장 그리기는 town이 만든다. 선행 조건이 없으므로 **town 단계 11 전에** 넣는다 (공통 맥락 5.1은 단계 12에 두었지만 town이 먼저 필요하다). town plan이 코드값 이름을 바꾸고 싶으면 이 마이그레이션 전에 맞춘다.
- **Rationale**: 공통 맥락 3.2·3.4 "roof_color (권장 위치 blogs)". town spec FR-039 8색(빨강·주황·노랑·초록·하늘·파랑·보라·갈색, 목록 밖은 서버 거부), FR-040 "고르지 않으면 배경 색", Key Entities "집은 회원(블로그)마다 하나" → blogs 컬럼.
- **Alternatives considered**: profiles 컬럼 — 집은 블로그 단위다. hex 저장 — 목록 밖 색 거부를 CHECK로 지키기 어렵다. pgEnum — 값을 더할 때 열거형 마이그레이션이 번거롭다 (CHECK면 제약만 바꾼다).

## R-23 소개 길이 DB CHECK

- **Decision**: `blogs_description_check`(`char_length(description) <= 160`)를 더한다.
- **Rationale**: 지금 소개는 앱 검증뿐이다 (`src/db/schema.ts` blogs에 description CHECK 없음, 원본 BLOG-03 열린 질문 "DB CHECK(0~160자)를 더할까?", **코드 확인**). 블로그 이름(`blogs_title_check`)과 같은 수준으로 맞춘다 (원칙 V "규칙은 DB 제약으로도"). 지금까지 저장된 값은 zod `max(160)`(UTF-16 기준)을 통과했으므로 `char_length` 160을 넘을 수 없다 → 위반 행이 없다고 본다. 마이그레이션 전 점검으로 확인한다 (R-28).
- **Alternatives considered**: 그대로 두기 — 열린 질문이 남는다.

## R-24 누르는 영역 44×44px

- **Decision**: 블로그 홈 왼쪽 카테고리 링크·검색 버튼·도감 전시 버튼·주인 버튼 3개([✏️ 글쓰기] [🎨 꾸미기] [⚙️ 관리], 375px 포함)·[첫 글 쓰기]·블로그 이름 링크, 블로그 관리의 `내 블로그로 →`·▲▼·[이름 바꾸기]·[삭제]·[소분류 추가]·[추가]·[저장]·[주소 바꾸기]·[취소], 내 정보의 닉네임 [저장]을 최소 44×44px(Tailwind `min-h-11 min-w-11`)로 만든다. ▲▼는 위아래로 쌓지 않고 가로로 놓아 줄 높이가 88px이 되지 않게 한다. 페이지 번호(`src/components/pagination.tsx`, post 소유)는 post에 요청한다. 이웃 버튼은 social이 같은 파일(`src/components/blog/blog-header.tsx`)의 이웃 버튼 부분만 `min-h-11`로 키운다 (social plan D-9).
- **Rationale**: FR-059, constitution VI. 지금 ▲▼는 `px-1 text-xs`(`src/app/settings/blog/settings-forms.tsx:40-41`), 카테고리 링크는 `px-2 py-1`(`src/app/blog/[slug]/page.tsx:47·55`), 이름 바꾸기·삭제는 `text-sm` 글자 버튼, `내 블로그로 →`는 `text-sm` 글자 링크(`src/app/settings/blog/page.tsx:23`)라 44px에 못 미친다 (**코드 확인**). `btn` 유틸리티는 위아래 여백 `0.6rem`이라 줄 높이에 따라 약 43px, 375px 주인 버튼은 `max-sm:text-sm`이 붙어 약 39px로 **추정**한다 (`src/app/globals.css`의 `@utility btn`, `src/components/blog/blog-header.tsx:55-57`). 실제 높이는 e2e로 잰다.
- **Alternatives considered**: 그대로 두기 — VI 위반.

## R-25 테스트 방식

- **Decision**: 순수 규칙은 `scripts/test-blog.ts`(`npm run test:blog`, `test` 체인 끝에 붙임) — 주소 정규화·형식, 예약어 판정(auth 모듈 + blog 값 16개), `charCount`, `toLikePattern`, `parseSearchQuery`, `buildCategoryTree`, 순서 맞바꾸기. 화면·DB는 새 e2e 6개(`blog-home`, `blog-address`, `categories`, `blog-search`, `blog-showcase`, `blog-scale`)와 `e2e/params.mjs`·`e2e/nonfunctional.mjs`에 대상 추가. 새 e2e는 모두 **실행마다 새 회원**을 만든다 — `tester1` 주소를 바꾸면 `visits.mjs`·`params.mjs`·`social.mjs`가 깨지기 때문이다.
- **Rationale**: 공통 맥락 6.1·6.2 관례 (`expect`/`check` 줄 출력, 실패 시 `process.exit(1)`, `.env.local`의 `DATABASE_URL`로 `pg` 직접 준비). `e2e/visits.mjs`·`e2e/params.mjs`·`e2e/social.mjs`는 `tester1` 블로그에 기댄다 (**코드 확인**).
- **Alternatives considered**: 테스트 프레임워크 도입 — 새 의존성, 관례와 다르다. 기존 `e2e/blog.mjs`에 모두 넣기 — 이미 다른 시나리오의 전제라 실패가 번진다.

## R-26 설치된 패키지 문서로 구현 전에 확인할 것

| 대상 | 이 plan이 기대는 동작 | 확인할 곳 |
|---|---|---|
| Next.js 16.3.8 | Server Action은 Origin·Host가 다르면 거부한다 (FR-060, NF-12). `revalidatePath("/", "layout")`. `next/form`의 `Form`(GET 이동). `PageProps`의 `searchParams` await | `node_modules/next/dist/docs/` (AGENTS.md) |
| React 19.2.8 | `<form action>`이 끝나면 비제어 칸을 초기화한다. `useActionState`에 `.bind`한 Server Action | React 문서, 기존 `login-buttons.tsx` 동작 |
| drizzle-orm 0.45 | `.for("update")`, `ilike()`, `foreignKey().onDelete()`의 열 목록 지원 여부. 제약 이름은 확인됨 — 컬럼 `.unique()`는 `{표}_{컬럼}_unique`(`blogs_slug_unique`, `profiles_nickname_unique`, `users_username_unique`, `drizzle/0000_init.sql`·`0001_site_login.sql`) | `node_modules/drizzle-orm` 타입 |
| drizzle-kit 0.31 | 생성 SQL을 손질해도 다음 `generate`가 스냅숏 기준으로 비교한다. 직접 쓴 SQL은 `generate --custom` | drizzle-kit 문서 |
| zod 4.6 | 문자열 `.min()/.max()`가 세는 단위, `.trim()`이 길이 검사보다 먼저인지 | zod 문서 |
| better-auth 1.7 | `username` 플러그인이 username을 소문자로 저장한다 (auth의 `users_username_check`가 DB에서도 막으므로 R-03의 결론은 이 확인에 기대지 않는다) | better-auth username 플러그인 문서 |
| PostgreSQL 15+ | `ON DELETE SET NULL (컬럼)`, 두 키 advisory lock 공간 분리, 같은 이벤트 FK 트리거 실행 순서(R-11) | PostgreSQL 문서 + 실제 DB 시험 |

## R-27 가입 때 만드는 블로그 기본값 (auth와의 약속)

- **Decision**: 가입 트랜잭션은 auth가 구현하고 blog 값을 그대로 쓴다 — `blogs`(`owner_id` = 새 회원, `slug` = 아이디, `title` = `{아이디}의 블로그`, `description` = `''`, `background_item_id` = `bg_meadow` 아이템), `categories`(`name` = `일상`, `position` = 0). 지금 온보딩의 블로그·카테고리 INSERT(`src/app/onboarding/actions.ts:74-83`)를 옮기는 것과 같다. 관리자 블로그 `/@notice`(FR-006)는 `scripts/create-admin.ts`가 이미 다시 실행해도 늘지 않게 만든다 — 바꿀 것이 없다. blog는 `e2e/blog-home.mjs`로 결과를 확인한다. auth가 원하면 blog가 도우미(가칭 `insertDefaultBlog(tx, …)`)를 더해 줄 수 있다 (추가만).
- **Rationale**: 공통 맥락 5.2 "가입 트랜잭션 안의 블로그 기본값(blog FR-001·002)은 auth가 구현". 값이 늘 DB 규칙을 만족한다: 아이디 형식 `^[a-z0-9_]{4,20}$`은 주소 형식 `^[a-z0-9_]{3,20}$` 안에 들고, 이름은 최대 20 + 5 = 25자 ≤ 40(`blogs_title_check`), 예약어·남의 주소와 같은 아이디는 auth가 가입에서 막는다 (D1).
- **Alternatives considered**: blog가 가입 함수 일부를 직접 소유 — `src/app/(auth)/actions.ts`는 auth 소유라 두 spec이 한 함수를 고치게 된다.

## R-28 이미 있는 데이터 점검

- **Decision**: 마이그레이션 전에 개발 DB에서 네 가지를 점검해 0건인지 본다 — (1) 새 예약어 목록과 같은 주소(관리자 `notice` 제외), (2) 다른 회원의 아이디와 같은 주소, (3) 다른 회원의 아이디와 대소문자 무시로 같은 닉네임, (4) 160자를 넘는 소개. 0건이 아니면 자동으로 고치지 않고 목록을 PR에 적는다.
- **Rationale**: 새 규칙은 "바꿀 때"만 막으므로 이미 있는 값이 남아도 화면은 깨지지 않는다. 배포 DB가 아직 없고(NF-08) 개발 DB는 `npm run db:reset`으로 비울 수 있다. (4)는 R-23 CHECK가 실패하지 않는지 미리 본다.
- **Alternatives considered**: 겹치는 주소를 자동으로 바꾸기 — 주인이 모르는 사이 링크가 끊긴다 (D3의 정신과 어긋난다).

## R-29 닉네임 칸의 위치 (auth 페이지에 끼우기)

- **Decision**: blog가 `src/app/settings/account/nickname-form.tsx`(클라이언트 폼)와 `src/app/settings/account/nickname-actions.ts`(Server Action `updateNickname`)를 새로 만들고, auth 소유 `src/app/settings/account/page.tsx`에 한 줄로 끼운다. 그 페이지가 아직 없으면 auth가 골격을 먼저 만든다 (공통 맥락 5.1 단계 2 선행). 성공하면 `revalidatePath("/", "layout")` — 미니룸 배지·글 카드·글 상세·광장 이름표(FR-020, US3-6)와 town이 만들 헤더 상태창이 새 닉네임을 그린다.
- **Rationale**: FR-019 "내 정보 화면에서", 공통 맥락 5.2 "`src/app/settings/account/*`는 auth, blog가 닉네임 칸을 추가". 닉네임은 `profiles`에만 있다 (ERD 3.17) → UPDATE 한 번으로 모든 화면에 반영된다.
- **Alternatives considered**: 블로그 관리 화면에 닉네임 칸 — FR-019와 다르다. Server Action을 `src/app/settings/blog/actions.ts`에 — 쓰는 화면과 파일 위치가 떨어진다.
