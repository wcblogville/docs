# Data Model: 교류 (SOCIAL)

**Feature**: `004-social` | **Date**: 2026-10-07 | **Plan**: [plan.md](plan.md) | **Research**: [research.md](research.md)

기준: 코드 저장소 `src/db/schema.ts`, `drizzle/0000_init.sql`~`0006_blog_visits.sql`, `docs/02-erd.md` v1.4.
DB의 글자 컬럼은 `text` + 길이 CHECK, 번호는 `integer generated always as identity`, 시각은 `timestamptz`다 (ERD 부록 첫 문단).

## 1. 표 한눈에 (담당 규칙: plan-context 3.1)

| 테이블 | 구분 | 내용 |
|---|---|---|
| `replies` | 변경 (새 테이블) | 답글 1단계 표. `comment_id` → `comments` `CASCADE`, `author_id` → `users` `CASCADE`, 내용 CHECK(삭제면 빈 글자), 인덱스 (`comment_id`, `created_at`) (ERD 3.8, 7장 2) |
| `comments` | 변경 (요청: auth, AUTH-06 포함) | `parent_id`와 `comments_parent_fk` 삭제, 답글 행을 `replies`로 이전, `author_id` NULL 허용 + `ON DELETE SET NULL`, CHECK 2개(내용·작성자), 삭제된 행 내용 비우기 |
| `follows` | 변경 (요청: town, TOWN-08) | `is_favorite` boolean NOT NULL 기본 false 추가. "최대 10명"은 town의 코드 규칙 |
| `post_likes` | 변경 없음 (social 담당) | PK(`post_id`, `user_id`)가 "한 글에 공감 한 번"을 지킨다. 코드만 `ON CONFLICT DO NOTHING`으로 바뀐다 |
| `posts` | 참조 | 공개 여부(`visibility`), 블로그(`blog_id`), 작성 시각. 댓글·답글·공감은 글 삭제 때 `CASCADE` |
| `blogs` | 참조 | 글 주인(`owner_id`), 주소(`slug`), 이름(`title`). 이웃 추가 대상 확인 |
| `profiles` | 참조 | 작성자 닉네임, 장착 캐릭터(`character_item_id`) |
| `items` | 참조 | 캐릭터 그림 키(`asset_key`) |
| `users` | 참조 | 회원 ID, 관리자 여부(`role`). 회원 삭제 때 아래 7절의 동작 (상태 전이는 4.1) |
| `point_ledger` | 참조 (요청: game이 부분 UNIQUE 추가 — constitution V 때문에 필요) | 댓글·답글 보상(`comment`), 받은 공감 보상(`like_received`)을 `grantReward()`로 기록. 요청 내용은 5절 |
| `notifications` (game 새 표) | 참조 (game이 만든다, game 단계 6) | 2차. 받는 회원, 종류(`like`·`comment`·`reply`), 행동한 회원, 관련 글. 받는 회원·행동한 회원 탈퇴, 글 삭제 때 함께 삭제 (game data-model 2.4, game FR-046) |

열거형: social이 담당하는 열거형은 없다. 답글 보상도 `ledger_reason`의 `comment`를 쓴다(새 값 요청 없음).

## 2. 엔터티별 현재 → 목표

### 2.1 댓글 (`comments`) — 변경

| 컬럼 | 현재 | 목표 | 메모 |
|---|---|---|---|
| `id` | integer identity PK | 그대로 | |
| `post_id` | integer NOT NULL FK → `posts.id` `CASCADE` | 그대로 | 글이 지워지면 댓글·답글이 함께 없어진다 (FR-017) |
| `author_id` | text NOT NULL FK → `users.id` `CASCADE` | **text NULL** FK → `users.id` **`ON DELETE SET NULL`** | 탈퇴 자리(D2). 살아 있는 댓글은 항상 작성자가 있다(CHECK) |
| `parent_id` | integer NULL, FK `comments_parent_fk` → `comments.id` `CASCADE` | **삭제** | 답글은 `replies`로 (FR-018) |
| `content` | text NOT NULL, CHECK `char_length BETWEEN 1 AND 1000` | text NOT NULL, **CHECK 바뀜** (아래) | 삭제하면 빈 글자 |
| `created_at` | timestamptz NOT NULL 기본 now() | 그대로 | 정렬 기준 (FR-011) |
| `deleted_at` | timestamptz NULL | 그대로 | 삭제 표시 시각 |

제약·인덱스 (목표)

```sql
CONSTRAINT comments_content_check CHECK (
  (deleted_at IS NULL AND char_length(content) BETWEEN 1 AND 1000)
  OR (deleted_at IS NOT NULL AND content = '')
)
CONSTRAINT comments_author_check CHECK (author_id IS NOT NULL OR deleted_at IS NOT NULL)
INDEX comments_post_created_idx (post_id, created_at)   -- 그대로
```

### 2.2 답글 (`replies`) — 변경 (새 테이블)

| 컬럼 | 타입·제약 | 메모 |
|---|---|---|
| `id` | integer identity PK | 이전한 답글은 원래 댓글 `id`를 그대로 쓴다 (6.2) |
| `comment_id` | integer NOT NULL FK → `comments.id` `ON DELETE CASCADE` | 원댓글. 원댓글 행이 지워지면(글 삭제, 탈퇴 때 남의 답글 없는 댓글) 함께 지워진다 |
| `author_id` | text NOT NULL FK → `users.id` `ON DELETE CASCADE` | 탈퇴하면 자리 없이 지워진다 (D2) |
| `content` | text NOT NULL, CHECK `replies_content_check` (댓글과 같은 식) | |
| `created_at` | timestamptz NOT NULL 기본 now() | |
| `deleted_at` | timestamptz NULL | |

```sql
INDEX replies_comment_created_idx (comment_id, created_at)   -- ERD 3.16 ⏳ 항목
```

- 답글에는 `post_id`가 없다. 답글의 글 = 원댓글의 `post_id` (ERD 3.17 3NF). "같은 글의 원댓글" 확인은 Server Action이 원댓글의 `post_id`와 요청의 글 ID를 비교한다.
- 답글을 가리키는 표가 없으므로 답글의 답글은 구조상 없다 (FR-018).

### 2.3 공감 (`post_likes`) — 변경 없음

| 컬럼 | 현재 = 목표 |
|---|---|
| `post_id` | integer NOT NULL FK → `posts.id` `CASCADE` |
| `user_id` | text NOT NULL FK → `users.id` `CASCADE` |
| `created_at` | timestamptz NOT NULL 기본 now() |
| PK | (`post_id`, `user_id`) — 한 회원·한 글 최대 1개 (FR-026, NF-15) |

### 2.4 이웃 관계 (`follows`) — 변경 (요청: town, TOWN-08)

| 컬럼 | 현재 | 목표 |
|---|---|---|
| `follower_id` | text NOT NULL FK → `users.id` `CASCADE` | 그대로 (이웃 추가한 회원) |
| `followee_id` | text NOT NULL FK → `users.id` `CASCADE` | 그대로 (블로그 주인) |
| `is_favorite` | 없음 | **boolean NOT NULL 기본 false** (즐겨찾는 이웃, town이 값을 바꾼다) |
| `created_at` | timestamptz NOT NULL 기본 now() | 그대로 |
| PK | (`follower_id`, `followee_id`) | 그대로 (FR-039) |
| CHECK `follows_not_self_check` | `follower_id <> followee_id` | 그대로 (FR-038) |
| INDEX `follows_followee_idx` | (`followee_id`) | 그대로 (`이웃 N` 세기) |

- 이웃을 취소하면 행이 지워져 즐겨찾기도 함께 풀린다 (town Edge Cases).
- "즐겨찾기 최대 10명"(town FR-033)은 DB 제약이 아니라 town의 코드 규칙(회원 잠금 안에서 세기)이다.

## 3. 검증 규칙 (서버)

| 대상 | 규칙 | 거부 결과 | FR |
|---|---|---|---|
| 글 ID (댓글·답글·공감) | `parseId()` 1~2147483647 정수 문자열·숫자 | 댓글·답글: `잘못된 요청이에요` / 공감: 무반응 | FR-008, FR-031, FR-003 |
| 원댓글 ID (답글) | `parseId()` | `잘못된 요청이에요` | FR-018, FR-021 |
| 댓글·답글 ID (삭제) | `parseId()` | 무반응 | FR-013 |
| 이웃 대상 회원 ID | 문자열, 1~64자, 나 아님 | 무반응 | FR-038, FR-040 |
| 내용 | `\r\n`·`\r` → `\n`, 앞뒤 공백 제거 뒤 1~1000자 | `댓글을 적어 주세요` / `댓글은 1000자까지예요` | FR-007, FR-008, FR-021 |
| 글 공개 여부 | 공개 글이거나 글 주인 = 나 (트랜잭션 안 `FOR KEY SHARE`) | 댓글·답글: `글을 찾을 수 없어요` / 공감: 무반응 | FR-006, FR-025, FR-031 |
| 원댓글 상태 (답글) | 행 있음 AND `post_id` = 요청 글 (`FOR SHARE`) | `답글을 달 댓글이 없어요` | FR-018 |
| | `deleted_at IS NULL` | `삭제된 댓글에는 답글을 달 수 없어요` | FR-018 |
| 삭제 권한 | 작성자 OR 그 글의 블로그 주인 OR 관리자, 그리고 삭제 안 된 행 | 무반응 | FR-013, FR-024 |
| 이웃 대상 | `blogs.owner_id`로 블로그가 있는 회원 | 무반응 | FR-040 |

내용은 HTML로 해석하지 않는다. 저장은 글자 그대로, 화면은 React 텍스트 노드로 그려 HTML·주소가 글자 그대로 보인다 (FR-007).

## 4. 상태 전이

### 4.1 댓글

```text
            [작성자·블로그 주인·관리자 삭제]
  보임 ─────────────────────────────────────▶ 삭제 자리 (deleted_at, content='', 작성자 보임)
   │                                               │
   │ [작성자 탈퇴: 남의 답글 있음]                     │ [작성자 탈퇴: 남의 답글 있음]
   ▼                                               ▼
  탈퇴 자리 (deleted_at, content='', author_id=NULL) ◀─┘

  보임 / 삭제 자리 ──[글 삭제]──────────────────────▶ 행 없음 (답글도 함께)
  보임 / 삭제 자리 ──[작성자 탈퇴: 남의 답글 없음]────▶ 행 없음 (자기 답글도 함께)
```

- 되돌리기(삭제 취소)는 없다. 모든 전이는 한 방향이다.
- 삭제 자리·탈퇴 자리에는 새 답글을 달 수 없고, 이미 달린 답글은 남는다 (FR-015).
- 탈퇴 자리는 auth의 탈퇴 트랜잭션이 `prepareCommentsForWithdrawal(tx, userId)`를 부른 뒤 회원 행을 지울 때 생긴다 (contracts/comments.md).

### 4.2 답글

```text
  보임 ──[작성자·블로그 주인·관리자 삭제]──▶ 삭제 자리 (deleted_at, content='', 작성자 보임)
  보임 / 삭제 자리 ──[원댓글 행 삭제 · 작성자 탈퇴]──▶ 행 없음
```

### 4.3 공감·이웃

```text
  공감 없음 ◀──[다시 누름]── 공감 있음 ◀──[누름]── 공감 없음      (글·회원 삭제 → 행 없음)
  이웃 아님 ◀──[✓ 이웃]──── 이웃(is_favorite=false) ⇄ 이웃(is_favorite=true)  [town의 ☆]
                                  ▲                    │
                                  └────[+ 이웃 추가]    └──[✓ 이웃] → 이웃 아님 (즐겨찾기도 풀림)
```

## 5. 다른 담당 표에 대한 요청

| 대상 (담당) | 요청 | 이유 | 필수 여부 |
|---|---|---|---|
| `point_ledger` (game) | 부분 UNIQUE INDEX (`user_id`, `ref_id`) WHERE `reason = 'like_received'` | "같은 사람·같은 글 공감 보상 1번"(FR-029)을 회원 잠금 + 원장 확인(ERD 3.6)뿐 아니라 DB 제약으로도 막는다 (constitution V "DB 제약조건으로도"). game이 출석에 둔 `point_ledger_attendance_uq`와 같은 모양이다. 지금 코드가 잠금 안에서 확인해 왔으므로 기존 중복은 없을 것으로 본다(추측 — 만들기 전에 중복 행 수를 세어 확인) | 필요 (game data-model에 아직 없음. 받지 않으면 social plan의 Complexity Tracking에 적는다) |
| `notifications` (game) | 종류 `like`·`comment`·`reply`, 행동한 회원 FK → `users` `CASCADE`, 관련 글 FK → `posts` `CASCADE`, 같은 트랜잭션에서 부를 기록 도우미 `notifyActivity` | 2차 알림 발생 (FR-033, FR-055, FR-056). game data-model 2.4·contracts 6.1에 이미 있다 | 2차 필수 (game 계획에 반영됨) |

## 6. 마이그레이션 순서와 백필

마이그레이션 번호는 박지 않는다 (plan-context 3.6). 1~3은 같은 PR에서 순서대로 만든다. 각 SQL 맨 위에 `-- SOC-02 (2026-10-07): ...` 형식의 한국어 설명 주석을 단다 (`drizzle/0006_blog_visits.sql` 관례).

### 6.1 순서

| 순서 | 종류 | 만드는 방법 | 내용 |
|---|---|---|---|
| 1 | 구조 | `src/db/schema.ts` 수정 → `npm run db:generate` | `replies` 만들기(FK 2개, CHECK, 인덱스). `comments.content` CHECK 삭제(임시). `comments.author_id` NOT NULL 해제 + FK를 `ON DELETE SET NULL`로. 이 단계의 스키마에는 `parent_id`가 아직 있다 |
| 2 | 데이터 | `npx drizzle-kit generate --custom --name=<이름>`(npm 스크립트에 없는 직접 실행, 설치된 `drizzle-kit` 사용) → 빈 파일에 직접 쓴 SQL | 6.2의 백필 |
| 3 | 구조 | `src/db/schema.ts` 수정 → `npm run db:generate` | `comments_parent_fk`·`parent_id` 삭제. `comments_content_check`(새 식), `comments_author_check` 추가 |
| 4 | 구조 | `src/db/schema.ts` 수정 → `npm run db:generate` | `follows.is_favorite` boolean NOT NULL DEFAULT false (기존 행은 false, 백필 없음) |

- 4는 1~3과 독립이다. town이 먼저 필요하면 4만 먼저 merge해도 된다.
- 다른 spec 브랜치가 먼저 merge되었으면 최신 `main`에서 1·3·4의 `npm run db:generate`를 다시 돌려 `drizzle/meta/` 스냅숏 충돌을 피한다. 2의 SQL 파일은 내용을 그대로 옮긴다. 1과 3은 중간 스키마 상태(1: `replies` 있음 + `parent_id` 남음 + `comments` CHECK 없음, 3: 최종)에서 각각 생성해야 하므로, 다시 만들 때도 `src/db/schema.ts`를 1의 상태로 고쳐 생성 → 2 → 최종 상태로 고쳐 생성하는 순서를 지킨다 (PR 설명에 두 상태의 차이를 적어 둔다).
- 적용: `npm run db:migrate`. 1~3은 한 번의 `db:migrate`로 함께 적용되며, 2가 실패하면 3이 적용되지 않는다 (drizzle-orm의 pg 마이그레이터는 대기 중인 파일 전체를 한 트랜잭션으로 감싸는 것으로 알고 있으나 추측이다. 구현 전 확인, research R19. 파일 단위로 나뉘어 있으면 실패 때 DB를 백업에서 되돌린 뒤 고쳐 다시 실행).

### 6.2 백필 규칙 (2번 직접 쓴 SQL)

1. **이전 전 확인** (주석으로 남기고 구현자가 실행해 결과를 PR에 적는다)
   - 답글 행 수: `comments`에서 `parent_id IS NOT NULL`인 행 수 = A
   - 깊이 2 이상 답글 수: 부모 댓글도 `parent_id`가 있는 답글 수 (지금 `addComment`가 답글의 답글을 원댓글에 붙여 저장하므로 0으로 예상)
   - 다른 글 부모: 부모 댓글의 `post_id`가 다른 답글 수 (지금 코드가 같은 글만 허용하므로 0으로 예상)
2. **원댓글 찾기**: 각 답글의 원댓글 = `parent_id`를 따라 올라가 `parent_id IS NULL`인 첫 조상 (재귀 CTE). 깊이 1이면 `parent_id` 그대로다.
3. **옮기기**: `INSERT INTO replies (id, comment_id, author_id, content, created_at, deleted_at) OVERRIDING SYSTEM VALUE SELECT ...` — `id`·`author_id`·`created_at`·`deleted_at`은 그대로, `comment_id`는 2의 원댓글, `content`는 `deleted_at`이 있으면 `''` 아니면 그대로. 원댓글의 `post_id`가 답글의 `post_id`와 다른 행은 옮기지 않는다(그 행은 `comments`에 남아 일반 댓글이 된다, 5 참고. 1의 확인에서 0이면 해당 없음).
4. **identity 맞추기**: `replies.id`의 다음 값을 `MAX(id) + 1`로 (`setval(pg_get_serial_sequence('replies', 'id'), ...)`). 옮긴 행이 없으면 1부터.
5. **옮긴 행 지우기**: 먼저 옮기지 않은 답글 행(3의 다른 글 부모)의 `parent_id`를 NULL로 바꿔 일반 댓글로 만든다. 그다음 `comments`에서 3에서 옮긴 `id`를 지운다. 이제 `parent_id`가 남은 행은 모두 지울 행이라 `comments_parent_fk`의 `CASCADE`가 다른 행을 지우지 않는다.
6. **삭제된 댓글 내용 비우기**: `UPDATE comments SET content = '' WHERE deleted_at IS NOT NULL`.
7. **이전 후 확인**: `replies` 행 수 = A(또는 A − 옮기지 않은 행), `comments`에 `parent_id IS NOT NULL` 행 0, 삭제 행의 `content <> ''` 0. 보상 원장(`point_ledger`)은 건드리지 않는다.

- 여러 번 실행해도 안전하게: 3은 `ON CONFLICT (id) DO NOTHING`, 5·6은 조건이 같아 두 번째 실행에서 0행이다.
- 원장 연결: 옮긴 답글의 보상 원장 행은 `reason = 'comment'`, `ref_id = '{원래 댓글 id}'`이고, 그 값이 이제 `replies.id`와 같다. 새 답글의 보상은 `ref_id = 'reply:{답글 id}'`로 남긴다 (research R2, R9).

### 6.3 되돌리기

되돌리는 마이그레이션은 만들지 않는다 (이 저장소에 전례가 없다). 로컬은 `npm run db:reset`(회원·글 데이터를 비움)으로 다시 시작한다. 배포 DB에 적용하기 전에는 백업을 남긴다.

## 7. 회원·글 삭제 때 (ERD 3.14 갱신 내용)

| 지워지는 것 | `comments` | `replies` | `post_likes` | `follows` |
|---|---|---|---|---|
| 글 | 그 글의 댓글 행 삭제 (`CASCADE`) | 그 댓글들의 답글 행 삭제 (`CASCADE`) | 삭제 (`CASCADE`) | - |
| 회원 (탈퇴, auth가 `prepareCommentsForWithdrawal` 먼저 호출) | 남의 답글 없는 그 회원 댓글: 행 삭제 / 남의 답글(삭제 표시된 답글 포함) 있는 그 회원 댓글: 탈퇴 자리(`author_id` NULL, 내용 빈 글자) / 그 회원 글의 댓글: 글과 함께 삭제 | 그 회원 답글 행 삭제 (`CASCADE`) | 삭제 (`CASCADE`) | 그 회원이 맺은 것·그 회원 대상 모두 삭제 (`CASCADE`) |
| 회원 (도우미 없이 바로 삭제) | 살아 있는 댓글이 있으면 `comments_author_check` 위반으로 삭제 전체가 취소된다 (안전장치) | - | - | - |
| 개발 DB 초기화 (`npm run db:reset`) | `TRUNCATE users, tags ... CASCADE`가 `comments`·`replies`·`post_likes`·`follows`를 함께 비운다 | | | |

## 8. `docs/02-erd.md`에서 social이 고칠 부분

- 1장 관계도: `comments` 블록의 `author_id`에 "탈퇴하면 NULL" 설명을 붙인다. (1장 관계도에는 `replies`·`follows.is_favorite`가 이미 그려져 있고 ⏳ 표시가 없다. ⏳는 3.16 인덱스 표에만 있다.)
- 3.8 답글: "행을 지우지 않고 `deleted_at`만 기록" → "행을 지우지 않고 `deleted_at`을 기록하고 내용을 비운다(CHECK)". 같은 줄의 표시 문구 "삭제된 댓글입니다"를 화면 문구 `삭제된 댓글이에요`로 맞춘다(spec Assumptions). 탈퇴 처리(작성자 NULL + CHECK, auth가 부르는 도우미)를 한 문단 추가.
- 3.14 삭제 규칙: 회원 줄의 "댓글, 답글 ... 삭제(`CASCADE`)"를 7절 표대로 고친다. "댓글·답글" 줄에 "내용을 비운다" 추가.
- 3.16 인덱스: `replies (comment_id, created_at)`의 ⏳ 제거.
- 4장 데이터 마이그레이션 표: 답글 이전 직접 쓴 SQL 한 줄(6.2 요약, 번호는 구현 때 정해짐) 추가.
- 7장 할 일: 2(`replies`)와 6(`follows.is_favorite`)에 완료 표시.
- 부록 컬럼 타입: `comments.content`, `replies.content`의 "1~1000자"를 "1~1000자, 삭제하면 빈 글자"로.
- 부록 NULL 허용 목록: `comments.author_id` 추가.
- (함께 맞추면 좋음) `docs/erdcloud-import.sql`의 `comments` 정의: `author_id` NULL 허용, `ON DELETE SET NULL`. `docs/erdcloud-final.sql`은 ERDCloud 내보내기 결과라 손으로 고치지 않는다.
