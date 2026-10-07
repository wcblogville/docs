# Data Model: 글 (POST)

**Feature**: `003-post` | **Date**: 2026-10-07 | **Plan**: [plan.md](plan.md) | **Research**: [research.md](research.md)

DB 설계 원본은 코드 저장소 `docs/02-erd.md` v1.4, 실제 스키마는 `src/db/schema.ts`다. 실제 DB는 글자를 `text` + 길이 CHECK, 번호를 `integer`로 둔다 (ERD 부록).
테이블 담당 규칙(공통 맥락 3.1)에 따라 **post 담당 테이블은 '변경'**, 남의 테이블은 **'참조'**로 적는다.

## 1. 테이블 한눈에

| 테이블 | 구분 | 내용 |
|---|---|---|
| `posts` | 변경 | `subcategory_id` 추가 + 복합 FK → `subcategories` (`category_id`, `id`) `ON DELETE SET NULL (subcategory_id)` + CHECK + 트리거 `posts_clear_subcategory` (ERD 3.18, 7장 6-3) |
| `attachments` | 변경 | `post_id` 추가(FK → `posts` `ON DELETE SET NULL`) + 인덱스 (`post_id`) + `detached_at` 추가 + 트리거 `attachments_track_detached` + 데이터 이전 (ERD 3.9, 7장 3. `detached_at`·트리거는 이 plan에서 더함) |
| `post_views` | 변경 (새 테이블) | 조회 기록: PK (`post_id`, `date`, `visitor_id`), FK → `posts` CASCADE (D5, 공통 맥락 3.3 가칭 그대로) |
| `tags` | 변경 없음 (담당) | 구조 그대로. 태그 정리 규칙은 앱(`parseTags`) |
| `post_tags` | 변경 없음 (담당) | 구조 그대로. 글 삭제 시 CASCADE |
| 열거형 `visibility` | 변경 없음 (담당) | `public`, `private` |
| `subcategories` | 참조 (선행: blog가 만듦) | 소분류 이름·소속 대분류를 읽고, `posts` 복합 FK가 UK (`category_id`, `id`)를 가리킨다 |
| `categories` | 참조 | 대분류. 글쓰기 선택지, 배지 이름, "내 블로그 것인지" 확인 |
| `blogs` | 참조 | 글의 블로그·주인(`owner_id`) = 글 작성자 (ERD 3.17). 비공개 첨부 주인 확인 |
| `users` | 참조 | 첨부를 올린 사람 (`attachments.user_id`) |
| `profiles` | 참조 (auth가 `photo_key` 추가 후) | 프로필 사진으로 쓰는 첨부 제외·공개 (research R19). 작성자 줄 닉네임·캐릭터 |
| `point_ledger` | 참조 (game) | 글쓰기 보상 기록(`reason = 'post'`, `ref_id` = 글 ID). 발행 안내의 보상 여부를 여기서 판단 (R13) |
| `comments`, `replies`, `post_likes` | 참조 (social) | 글 삭제 시 CASCADE. 목록 댓글 수는 social이 정의 (`replies`는 social이 만듦) |
| `follows` | 참조 (social) | 이웃 새 글 거르기·즐겨찾기 7일 우선 (D15, social이 구현) |
| `user_animals` | 참조 (town) | `savePost`가 `growForPost`를 계속 부른다 |
| `blog_visits` | 참조하지 않음 | 쿠키 `bv_visitor`만 같이 쓴다 (R3) |

다른 spec에 새로 요청하는 구조 변경은 없다. `subcategories`(blog)와 `profiles.photo_key`(auth)는 이미 정해진 변경이며 의존성으로만 적는다 (plan.md).

---

## 2. `posts` — 변경

### 2.1 현재 → 목표

| 컬럼·제약 | 현재 (`src/db/schema.ts`) | 목표 |
|---|---|---|
| `id` | integer identity PK | 그대로 |
| `blog_id` | FK → `blogs` CASCADE | 그대로 |
| `category_id` | NULL, FK → `categories` `ON DELETE SET NULL` | 그대로 (대분류) |
| `subcategory_id` | 없음 | **추가**: integer NULL (소분류) |
| `title` | text, CHECK 1~100자 | 그대로 |
| `content_html` | text | 그대로 (정화된 HTML) |
| `content_text` | text | 그대로 (요약·글자 수·보상 판단) |
| `visibility` | enum, 기본 `public` | 그대로 |
| `view_count` | integer 기본 0, CHECK ≥ 0 | 그대로. 이제 `post_views`에 새 행이 들어갈 때만 +1 |
| `created_at` / `updated_at` | timestamptz | 그대로. 조회수가 올라도 `updated_at`은 그대로 (FR-046) |
| FK `posts_subcategory_fk` | 없음 | **추가**: (`category_id`, `subcategory_id`) → `subcategories` (`category_id`, `id`) `ON DELETE SET NULL ("subcategory_id")` |
| CHECK `posts_subcategory_check` | 없음 | **추가**: `subcategory_id IS NULL OR category_id IS NOT NULL` |
| 트리거 `posts_clear_subcategory` | 없음 | **추가**: `BEFORE UPDATE OF category_id`, 새 `category_id`가 NULL이면 `subcategory_id`도 NULL (research R4) |
| 인덱스 | (`blog_id`, `created_at` desc), (`visibility`, `created_at` desc) | 그대로. 소분류 거르기는 블로그 인덱스로 좁힌 뒤 거른다 (블로그당 글 수가 작다) |

### 2.2 필드 의미와 검증 규칙 (서버, `savePost`)

| 값 | 규칙 | 위반 시 문구 | 근거 |
|---|---|---|---|
| 제목 | 앞뒤 공백 제거 후 1~100자 | `제목을 적어 주세요` / `제목은 100자까지예요` | FR-006 |
| 본문 HTML | 200,000자 이하 (UTF-16 단위) | `글이 너무 길어요` | FR-006, R11 |
| 본문 글자 | 정화 후 `htmlToText` 결과가 비면 안 됨 (사진·파일 카드·구분선·빈 줄은 0자) | `본문을 적어 주세요` | FR-008 |
| 태그 칸 | 300자 이하 | `태그는 모두 합쳐 300자까지예요` | FR-006 |
| 글·대분류·소분류 번호 | 빈 값이거나 숫자만 적힌 1~2147483647 | `잘못된 요청이에요` | FR-018, R12 |
| 공개 설정 | `public` 또는 `private` | `잘못된 요청이에요` | Edge Cases, R12 |
| 대분류 | 내 블로그 것이 아니거나 없으면 조용히 NULL | (없음) | FR-032 |
| 소분류 | 고른 대분류 소속이어야 함. 다른 대분류 소속이면 거부, 행이 없으면 대분류만 | `잘못된 요청이에요` | FR-032, R15 |
| 본문 첨부 | 붙일 수 있는 첨부만 남김 (§3.3) | (없음, 조용히 뺌) | FR-047, FR-054 |

DB가 막는 것: 제목 길이 CHECK, 소분류-대분류 소속(복합 FK), 소분류만 있는 글(CHECK), `view_count ≥ 0`.

### 2.3 상태 전이

**공개 설정**

```text
(새 글) ──발행──▶ public ◀──수정(공개↔비공개)──▶ private
```

- 새 공개 글이고 본문 100자 이상이면 같은 트랜잭션에서 보상 (`grantReward(tx, 주인, "post", 글ID)`, 하루 3번). private→public으로 바꿔도 보상 없음 (FR-027).
- private이면 주인 말고는 상세 404, 목록·태그·이전/다음 글·인기 태그에서 빠지고 첨부도 주인만 (FR-023~029).

**카테고리**

```text
없음 ──(대분류 고름)──▶ 대분류만 ──(소분류 고름)──▶ 대분류+소분류
대분류+소분류 ──(소분류 삭제: 복합 FK SET NULL(subcategory_id))──▶ 대분류만
대분류만 / 대분류+소분류 ──(대분류 삭제: category_id SET NULL + 트리거)──▶ 없음
```

- 대분류를 지우면 `posts.category_id` SET NULL과 blog의 `subcategories` CASCADE(→ 복합 FK `SET NULL (subcategory_id)`)가 함께 돈다. 어느 쪽이 먼저 돌아도, `category_id`를 NULL로 바꾸는 바로 그 UPDATE에서 트리거가 `subcategory_id`도 비우므로 CHECK 위반이 생기지 않는다 (R4).
  blog의 `deleteCategory`는 앱에서 글 두 칸을 먼저 비운다 (blog research R-11). 그 경로에서는 트리거가 할 일이 없고, 회원 삭제 CASCADE와 삭제와 겹친 저장에서 트리거가 지킨다.
- 글을 지우면 → 태그 연결·댓글·답글·공감·조회 기록 CASCADE 삭제, 첨부는 `post_id` NULL(떨어짐). 보상 기록은 남고 회수하지 않는다 (FR-016, D6).

---

## 3. `attachments` — 변경

### 3.1 현재 → 목표

| 컬럼·제약 | 현재 | 목표 |
|---|---|---|
| `key` | text PK, CHECK `^[a-f0-9]{32}$` | 그대로 (주소 `/files/키`) |
| `user_id` | FK → `users` CASCADE | 그대로 (올린 사람. 내 글에만 붙이는 확인에 쓴다) |
| `post_id` | 없음 | **추가**: integer NULL, FK → `posts` `ON DELETE SET NULL` |
| `detached_at` | 없음 | **추가**: timestamptz NULL (글에서 떨어진 시각. 붙어 있거나 한 번도 안 붙었으면 NULL) |
| `kind`, `name`, `mime`, `size`, `created_at` | CHECK 종류·이름 1~255·크기 > 0 | 그대로 |
| 인덱스 | (`user_id`, `created_at`) | 그대로 + **추가** (`post_id`) (글의 첨부 찾기, 글 삭제 시 SET NULL) |
| 트리거 `attachments_track_detached` | 없음 | **추가**: `BEFORE UPDATE OF post_id`. 값→NULL이면 `detached_at = now()`, NULL→값이면 `detached_at = NULL` (실제로 바뀔 때만) (R7) |

`detached_at`과 트리거는 ERD에 없는 추가다. FR-059 "떨어진 지 하루"를 지키려고 더한다. post가 `docs/02-erd.md`에 함께 적는다 (§7).

### 3.2 상태 전이

```text
              (POST /api/uploads)
                    │
                    ▼
       ┌──── 떨어짐(post_id NULL) ◀────────────────────────────┐
       │        │                                              │
       │        │ 같은 회원의 글 저장 시 본문에 있음             │ 본문에서 빠짐 / 글 삭제
       │        ▼                                              │ (트리거가 detached_at = now())
       │     붙음(post_id = 글) ───────────────────────────────┘
       │
       │ post_id NULL 이고 COALESCE(detached_at, created_at) < now() - 1일
       │ 이고 프로필 사진이 아님
       ▼
   정리됨 (행 삭제 → 저장소 파일 삭제)        * 회원 삭제: 행 CASCADE 삭제 → 파일은 정리 작업의 저장소 훑기가 지움
```

- 붙여 넣기 다시 올리기(R8)는 원본을 바꾸지 않고 **새 행**(새 키, 같은 `name`·`mime`·`size`·`kind`, `post_id` NULL)을 만든다.
- 떨어진 첨부도 같은 회원이 같은 글이나 새 글에서 다시 저장하면 다시 붙을 수 있다 (예: 다른 탭의 수정 화면).

### 3.3 글에 "붙일 수 있는" 첨부 (저장할 때)

한 글 P를 저장할 때 본문의 키 K는 다음을 **모두** 만족해야 남는다 (그 밖은 정화가 본문에서 뺀다):

1. `attachments`에 K가 있다.
2. `user_id` = 저장하는 회원.
3. `post_id IS NULL` 또는 `post_id = P` (새 글은 `post_id IS NULL`만).
4. 지금 어느 `profiles.photo_key`도 K가 아니다 (auth 컬럼이 들어온 뒤, R19).
5. 종류가 쓰인 자리와 맞다 (`<img>`는 `image`, 파일 카드는 `file`, 지금 정화 규칙 그대로).

### 3.4 누가 열 수 있나 (`GET /files/[key]`)

| 첨부 상태 | 열 수 있는 사람 | 그 밖 |
|---|---|---|
| 공개 글에 붙음 | 누구나 (로그인 없이) | - |
| 비공개 글에 붙음 | 그 블로그 주인 | 404 `파일을 찾을 수 없어요` |
| 어느 글에도 없음 | 올린 사람 | 404 |
| 어느 글에도 없음 + 지금 프로필 사진 | 누구나 | - |
| 행 없음 / 저장소에 파일 없음 | - | 404 |

---

## 4. `post_views` — 변경 (새 테이블)

| 컬럼 | 타입 | 규칙 |
|---|---|---|
| `post_id` | integer NOT NULL | FK → `posts` `ON DELETE CASCADE` |
| `date` | date NOT NULL | 한국 날짜 (`todayKST()`, `src/lib/game.ts`) |
| `visitor_id` | uuid NOT NULL | 쿠키 `bv_visitor` 값. 회원 정보와 잇지 않고 IP를 저장하지 않는다 (BLOG-06과 같음, NF-26) |
| `created_at` | timestamptz NOT NULL 기본 now() | |
| PK | (`post_id`, `date`, `visitor_id`) | 같은 브라우저는 글마다 하루 한 줄 = 하루 1번 (D5) |

- 쓰기: `recordPostView`가 `INSERT ... ON CONFLICT DO NOTHING RETURNING`. 행이 들어갔을 때만 같은 트랜잭션에서 `posts.view_count + 1`(`updated_at`은 원래 값을 다시 넣어 그대로).
- 주인·비공개 글·없는 글은 쓰지 않는다 (FR-046).
- 보존: 오늘·어제 행만 필요하다. 정리 작업이 `date < 어제(한국)` 행을 지운다 (R1, R10).
- 회원 삭제: 블로그 → 글 CASCADE로 함께 지워진다. `scripts/reset-dev.ts`의 `TRUNCATE users, tags ... CASCADE`로도 비워진다.
- 숫자와의 관계: `view_count`는 이 표가 생기기 전 조회까지 포함한 누적값이라 `post_views` 행 수와 같지 않다 (ERD 3.17 반정규화 설명을 고친다).

---

## 5. 브라우저에만 있는 데이터 (DB 아님)

| 이름 | 위치 | 모양 | 쓰는 곳 | 근거 |
|---|---|---|---|---|
| 임시 글 | `localStorage` `blogville:draft:{userId}` | `{ title, categoryId, subcategoryId, visibility, contentHtml, tags, savedAt }` (회원당 1개, 덮어쓰기) | `PostForm`(쓰기·불러오기), `PublishNotice`(발행 성공 후 지움), `SignOutButton`(로그아웃 때 지움) | FR-060~064, R14 |
| 방문자 쿠키 | 쿠키 `bv_visitor` (BLOG-06과 같음) | 무작위 UUID, `HttpOnly`, `SameSite=Lax`, `Path=/`, 1년, 배포 `Secure` | `recordPostView`(없으면 만듦), 글 상세 서버 렌더(읽기) | FR-046, R3 |

임시 글에는 길이 제한이 없고, 발행할 때 서버 검사(§2.2)를 그대로 거친다.

---

## 6. 삭제 규칙 (이 기능이 만지는 부분)

| 지워지는 것 | 함께 처리 | 방법 |
|---|---|---|
| 글 (주인 `deletePost`, 관리자 `adminDeletePost`) | 태그 연결·댓글·답글·공감·조회 기록 삭제, 첨부는 떨어짐(`post_id` NULL, `detached_at` = 지금), 보상 기록은 남음 | FK CASCADE / SET NULL + 트리거 |
| 대분류 (blog) | 소분류 CASCADE, 글은 남고 `category_id`·`subcategory_id` NULL | blog `deleteCategory`가 먼저 비움(blog R-11) + FK SET NULL + 트리거 `posts_clear_subcategory`(회원 삭제 CASCADE 등 모든 경로) |
| 소분류 (blog) | 글은 남고 `subcategory_id`만 NULL | 복합 FK `SET NULL (subcategory_id)` |
| 회원 (auth) | 블로그·글·첨부 행·조회 기록 CASCADE, 저장소 파일은 정리 작업이 지움 | FK CASCADE + `npm run posts:cleanup` |
| 붙지 않은 첨부 | 하루 지나면 행 + 파일 삭제 | `npm run posts:cleanup` |

---

## 7. 마이그레이션 순서와 데이터 이전

번호는 적지 않는다 (공통 맥락 3.6). 만들 때 최신 `main`에서 `npm run db:generate`를 돌려 다음 번호를 받는다. 각 SQL 맨 위에 요구사항 ID를 단 한국어 주석을 붙인다.
직접 덧붙이는 문장(트리거 함수·트리거, 손으로 고친 FK)도 생성된 SQL처럼 문장 사이에 `--> statement-breakpoint`를 둔다 (`drizzle/0004_attachments.sql` 관례. 함수 본문 `$$ … $$` 안에는 넣지 않는다).

| 순서 | 마이그레이션 (내용) | 만드는 법 | 선행 |
|---|---|---|---|
| A | 첨부를 글에 잇기: `attachments.post_id` + FK `ON DELETE SET NULL` + 인덱스 (`post_id`) + `detached_at` + 트리거 함수·트리거 `attachments_track_detached` | `db:generate` 후 트리거 부분을 같은 파일 끝에 직접 씀 | 없음 |
| B | 첨부 데이터 이전: 기존 글 본문의 `/files/키`로 `post_id` 채우기 | `drizzle-kit generate --custom`으로 빈 파일을 만들고 직접 씀 (추측: 0.31에 있는 옵션. `0002_give_all_starters.sql`도 직접 쓴 SQL이 저널에 올라 있다) | A |
| C | 조회 기록: `post_views` 새 테이블 | `db:generate` | 없음 |
| D | 소분류: `posts.subcategory_id` + 복합 FK + CHECK + 트리거 함수·트리거 `posts_clear_subcategory` | `db:generate` 후 FK의 `ON DELETE` 줄을 `SET NULL ("subcategory_id")`로 고치고 트리거를 직접 씀 (R5) | **blog의 `subcategories` 마이그레이션** (UK (`category_id`, `id`) 포함) |

A~C는 blog를 기다리지 않는다. D만 blog 단계 2 뒤에 한다.

### 7.1 데이터 이전 B (규칙)

1. `post_id IS NULL`인 첨부 a마다, `blogs.owner_id = a.user_id`인 블로그의 글 중 `content_html`에 `'/files/' || a.key`가 들어 있는 글을 `created_at`, `id` 순으로 찾아 첫 글의 `id`를 넣는다.
2. 찾는 글이 없으면 NULL로 둔다 (붙지 않은 첨부 → `created_at` 기준으로 정리 대상).
3. `post_id IS NULL`인 행만 고치므로 다시 실행해도 결과가 같다.

```sql
-- 규칙 스케치 (실제 파일은 구현 때 작성)
UPDATE attachments a
SET post_id = (
  SELECT p.id FROM posts p JOIN blogs b ON b.id = p.blog_id
  WHERE b.owner_id = a.user_id AND position('/files/' || a.key IN p.content_html) > 0
  ORDER BY p.created_at, p.id LIMIT 1)
WHERE a.post_id IS NULL;
```

`UPDATE ... SET post_id`가 트리거 A를 부르지만 NULL→값이라 `detached_at`은 NULL 그대로다.

### 7.2 이전 뒤 확인 (quickstart에서 실행)

- 붙은 첨부 수, 붙지 않은 첨부 수.
- **남는 참조 수**: 본문에 `/files/키`가 있지만 그 첨부의 `post_id`가 그 글이 아닌 (글, 키) 쌍. 개발 데이터에서 0이 아니어도 된다 (R18). 그 글은 다음 저장 때 주소가 빠진다.
- 첫 정리 작업 전에 이전 B가 끝났는지 확인한다 (안 그러면 기존 글의 첨부가 "붙지 않은 첨부"로 지워진다). 마이그레이션 순서가 이를 보장한다.

### 7.3 그 밖

- `posts.subcategory_id`: 이전할 데이터 없음 (기존 글은 대분류만, ERD 7장 6-3).
- `post_views`: 빈 표로 시작. `view_count`는 그대로.
- PostgreSQL 15 이상이 필요하다 (`ON DELETE SET NULL (컬럼)`). README 설치 안내는 17.

---

## 8. `docs/02-erd.md`에서 post가 고칠 곳 (규칙 5)

| 위치 | 고칠 내용 |
|---|---|
| 1장 관계도 | `posts \|\|--o{ post_views : "조회 기록"`, `post_views` 엔터티, `attachments.detached_at` |
| 2장 테이블 그룹 | 글·교류에 `post_views` |
| 3.7 복합 PK 표 | `post_views` (`post_id`, `date`, `visitor_id`) — 같은 브라우저 글마다 하루 1번 |
| 3.9 첨부 | 붙이는 규칙(§3.3), 누가 여나(§3.4), `detached_at`·트리거, 정리 작업(`npm run posts:cleanup`), 붙여 넣기 다시 올리기 |
| 3.14 삭제 규칙 | 글 → 조회 기록 삭제·첨부 `detached_at`, 대분류 → 트리거 설명 |
| 3.16 인덱스 | `attachments (post_id)` ⏳ 표시 지움 |
| 3.17 반정규화 | `posts.view_count`: "조회 기록 표는 하루 1번 판단용(이틀 보관), 숫자는 목록 비용 때문에 쌓는다" |
| 3.18 카테고리 2단계 | 트리거 `posts_clear_subcategory`와 이유(R4), `SET NULL (subcategory_id)`는 손으로 고친 SQL |
| 4장 데이터 마이그레이션 | 이전 B 한 줄 |
| 7장 할 일 | 3(`attachments.post_id` 부분), 6-3(`posts.subcategory_id` 부분) 완료 표시 |
| 부록 NULL 허용 | `attachments.detached_at` |
