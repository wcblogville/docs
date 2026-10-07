# Research: 교류 (SOCIAL)

**Feature**: `004-social` | **Date**: 2026-10-07 | **Plan**: [plan.md](plan.md)

코드 저장소 `main` 커밋 `feb4c05`을 읽고 확인한 사실을 근거로 쓴다. 코드 위치는 코드 저장소 기준 상대 경로다.
이 체크아웃에는 `node_modules`가 없어 라이브러리 문서를 열 수 없었다. 라이브러리 동작에 기대는 부분은 "추측" 또는 "구현 전 확인"으로 표시하고 R19에 모았다.

Technical Context에 `NEEDS CLARIFICATION`으로 남긴 항목은 없다. spec이 정하지 않은 구현 경계(최근 7일의 기준, 탈퇴 자리의 표시 범위, 공감 알림 반복)는 아래에서 정했다.

---

## R1. 답글 저장 구조

**Decision**: 답글을 새 표 `replies`(`comment_id` → `comments.id`)에 둔다. `comments.parent_id`(자기 참조)와 FK `comments_parent_fk`를 지운다. 답글 보상 사유는 새로 만들지 않고 `comment`를 쓴다.

**Rationale**:
- ERD 3.8·7장 2와 요구사항 "2026-10-07 설계 변경" 표가 이미 정한 구조다. 답글을 담는 표가 하나뿐이라 "답글의 답글"(FR-018)은 구조상 만들 수 없다.
- 지금 코드(`src/app/blog/actions.ts`의 `addComment`)는 답글의 답글을 원댓글로 바꿔 붙이는 보정 코드가 필요하고, DB는 깊이를 막지 못한다.
- 보상 사유를 `comment` 하나로 두면 `grantReward()`가 `point_ledger`의 같은 사유 수를 세므로 "댓글과 답글을 합쳐 하루 10번"(FR-023, SC-004)이 그대로 지켜진다. `ledger_reason`(game 담당)에 값을 더할 필요가 없다.

**Alternatives considered**:
- `parent_id` 유지 + 트리거로 깊이 1 강제: ERD 결정과 다르고, 트리거는 이 저장소에 전례가 없다.
- `replies`에 `post_id`도 두기: 원댓글을 거치면 글을 알 수 있어 3NF에 어긋난다(ERD 3.17). "같은 글" 확인은 원댓글의 `post_id`로 한다.
- 답글 보상 사유 `reply` 추가: 상한을 합쳐 세려면 `grantReward`를 고쳐야 하고 game 담당 열거형을 바꿔야 한다.

## R2. 기존 답글 데이터 이전

**Decision**: 마이그레이션을 세 개로 나눈다 (번호는 구현 때 정해진다).
1. (생성) `replies` 만들기, `comments.content` CHECK 임시 해제, `comments.author_id` NULL 허용 + FK `ON DELETE SET NULL`.
2. (직접 쓴 SQL, `drizzle-kit generate --custom`) `parent_id`가 있는 행을 `replies`로 옮기고(원래 `id` 유지), 옮긴 행을 `comments`에서 지우고, 삭제된 댓글·답글의 내용을 빈 글자로 바꾼다.
3. (생성) `comments.parent_id` 삭제, `comments`에 새 CHECK 2개(R3, R4).

세부 규칙(원댓글 찾기, id 유지, 확인 쿼리)은 [data-model.md](data-model.md)의 "백필"에 적었다.

**Rationale**:
- ERD 7장은 "마이그레이션 하나씩 나눠서 한다", plan-context 3.6은 "데이터 이전은 구조 변경과 같은 마이그레이션이나 바로 뒤의 직접 쓴 SQL"이라고 했다. 데이터만 바꾸는 SQL을 따로 두는 관례가 이미 있다(`drizzle/0002_give_all_starters.sql`).
- 생성된 SQL 안의 문장 순서를 손으로 바꾸지 않아도 된다. 한 파일로 생성하면 `parent_id` 삭제가 데이터 이전보다 먼저 나올 수 있다 (drizzle-kit의 문장 순서는 확인하지 못했다, 추측).
- 기존 CHECK(`char_length(content) BETWEEN 1 AND 1000`)가 남아 있으면 2번의 "삭제된 댓글 내용 비우기"가 거부되므로 1번에서 CHECK를 먼저 내린다.
- **`id` 유지**: 답글 보상 원장 행의 `ref_id`는 그때의 댓글 ID다(`grantReward(tx, viewer.userId, "comment", created.id)`). `replies.id`를 같은 값으로 두면 원장(game 담당, 고치지 않음)과 답글의 연결이 남는다. `comments`의 identity는 번호를 다시 쓰지 않으므로, 이전 뒤 `ref_id` 숫자는 `comments`에 있으면 댓글, 없으면 옮겨진 답글이다. 새 답글의 `ref_id`는 `reply:{답글ID}`로 구분한다(R9).

**Alternatives considered**:
- 새 `id`로 옮기기: 단순하지만 지난 원장의 `ref_id`가 가리키는 행이 사라진다.
- 원장 `ref_id`를 새 ID로 고치기: `point_ledger`는 game 담당이고, 원장은 "그때의 기록"을 남기는 표라 고치지 않는다(ERD 3.17).
- 한 마이그레이션 파일에 손으로 문장 순서를 맞춰 넣기: 다시 생성할 때(`npm run db:generate` 재실행, plan-context 3.6) 깨지기 쉽다.

## R3. 삭제 표시 때 내용을 비운다

**Decision**: 댓글·답글을 지우면 같은 `UPDATE`에서 `deleted_at = now()`, `content = ''`로 바꾼다. CHECK를 `(deleted_at IS NULL AND char_length(content) BETWEEN 1 AND 1000) OR (deleted_at IS NOT NULL AND content = '')`로 바꿔 DB가 지킨다. 기존 삭제 행은 R2의 2번에서 비운다.

**Rationale**:
- FR-014·SC-006은 "삭제한 댓글의 원래 내용이 화면과 화면으로 전달되는 데이터 어디에도 나오지 않는다"이다. 지금은 글 상세 페이지가 `content: c.deletedAt ? "" : c.content`로 가리지만(`src/app/blog/[slug]/[postId]/page.tsx`), 원문은 DB에 남고 쿼리(`getComments`)도 읽는다. 원문이 없으면 어느 화면에서 실수해도 새어 나갈 수 없다.
- D2(탈퇴)는 "내용 없이 `삭제된 댓글이에요` 자리만"을 요구한다(auth FR-052, SC-014). 탈퇴 자리와 일반 삭제를 같은 규칙으로 두면 표 규칙이 하나다.
- constitution VII(최소 정보). 지운 내용을 되살리거나 관리자가 보는 기능은 spec에 없다.
- `text NOT NULL`을 유지하므로 Drizzle 타입(`string`)과 기존 코드가 그대로다.

**Alternatives considered**:
- ERD 3.8 그대로 `deleted_at`만 기록: 원문이 DB에 남아 D2의 "내용 없이"를 탈퇴 때만 따로 처리해야 한다.
- `content`를 NULL로: 타입이 `string | null`이 되어 고칠 곳이 늘어난다.
- 삭제 행을 실제로 지우기: FR-014는 자리를 남기라고 한다.

이 결정으로 ERD 3.8의 "행을 지우지 않고 `deleted_at`만 기록한다"를 "`deleted_at`을 기록하고 내용을 비운다"로 고친다 (social이 ERD의 comments·replies 부분 담당, plan-context 3.1 규칙 5).

## R4. 탈퇴(D2)를 위한 `comments.author_id`

**Decision**:
- `comments.author_id`: NULL 허용, FK `ON DELETE SET NULL`, CHECK `author_id IS NOT NULL OR deleted_at IS NOT NULL`.
- `replies.author_id`: NOT NULL, FK `ON DELETE CASCADE`.
- 새 파일 `src/server/social.ts`에 `prepareCommentsForWithdrawal(tx, userId)`를 둔다. auth의 탈퇴 트랜잭션이 회원 행을 지우기 전에 부른다. 하는 일: (1) 그 회원의 댓글 행을 `FOR UPDATE`로 잠근다 (2) 다른 회원의 답글이 하나도 없는 그 회원의 댓글을 지운다(그 아래 자기 답글은 `CASCADE`). 다른 회원의 답글은 삭제 표시된 것도 "있음"으로 센다 — 지우면 남의 행(삭제 자리)까지 `CASCADE`로 사라지기 때문이다(auth FR-052 "다른 회원의 답글은 그대로 둔다") (3) 남은 그 회원의 댓글에 삭제 표시(`deleted_at`, 내용 비우기). 그다음 auth가 회원을 지우면 `replies`의 그 회원 답글은 `CASCADE`로, 남은 댓글의 `author_id`는 `SET NULL`로 바뀐다.

**Rationale**:
- 지금 `comments.author_id`는 `CASCADE`라 회원을 지우면 그 댓글과 거기 달린 남의 답글까지 지워진다(`comments_parent_fk`도 `CASCADE`). D2 "다른 사람의 답글이 달린 댓글은 자리만 남기고 다른 사람의 답글은 그대로"를 지킬 수 없다.
- 자리만 남는 댓글은 작성자가 없어야 한다(auth FR-052: 닉네임·캐릭터 안 보임, SC-014). `SET NULL`이면 회원 행이 지워질 때 DB가 연결을 끊는다.
- CHECK는 안전장치다. auth가 도우미를 부르지 않고 회원을 지우면 살아 있는 댓글의 `author_id`가 NULL이 되려다 CHECK에 걸려 탈퇴 트랜잭션 전체가 취소된다(auth FR-051 "하나라도 실패하면 전부 취소"와 맞다).
- (1)의 `FOR UPDATE`: 다른 회원이 그 순간 답글을 달고 있으면 답글 쪽의 원댓글 `FOR SHARE`(R5)와 부딪혀 한쪽이 기다린다. 그래서 "답글이 없다고 보고 지웠는데 그사이 남의 답글이 달려 함께 지워지는" 일이 없다. 탈퇴가 먼저 끝나면 답글 요청은 원댓글이 없으면 `답글을 달 댓글이 없어요`, 자리만 남았으면 `삭제된 댓글에는 답글을 달 수 없어요`를 받는다(auth Edge Cases와 같다).
- 처리 규칙(어떤 댓글을 남길지)은 댓글 구조를 아는 social이 갖고, auth는 한 줄로 부르기만 한다(plan-context 5.3 "탈퇴와 댓글 구조").

**Alternatives considered**:
- "탈퇴한 회원" 대리 계정으로 `author_id`를 바꾸기: 가짜 회원 행이 생기고, D2가 금지한 "탈퇴한 회원 표시"와 비슷해진다.
- `CASCADE` 유지 + 탈퇴 전에 남길 댓글을 복사: 복잡하고 id가 바뀐다.
- 트리거로 처리: 이 저장소에 트리거가 없고, 규칙이 SQL 안에 숨는다.

## R5. 답글 거부 판정과 동시 요청

**Decision**: 댓글·답글·공감 트랜잭션 안에서 대상 행을 잠그고 다시 확인한다.
- 글: `posts` join `blogs`를 `FOR KEY SHARE OF posts`로 읽어 공개 여부(공개 글 또는 내 글)를 확인한다. 없거나 볼 수 없으면 `글을 찾을 수 없어요`(댓글·답글) 또는 무반응(공감).
- 원댓글(답글만): `comments`를 `id`로 `FOR SHARE` 잠금 후 읽는다. 행이 없거나 `post_id`가 요청 글과 다르면 `답글을 달 댓글이 없어요`, `deleted_at`이 있으면 `삭제된 댓글에는 답글을 달 수 없어요`.
- 잠금 순서: 행 잠금(글 → 원댓글) → 회원 잠금(`lockUser`) → 보상.

**Rationale**:
- D9: 답글 입력칸을 열어 둔 사이 원댓글이 지워져도 저장하면 안 된다(US5-10, FR-018). 삭제는 `UPDATE`라 `FOR SHARE`와 충돌한다. 삭제가 먼저 커밋되면 `FOR SHARE`가 기다린 뒤 바뀐 행을 다시 읽어 거부하고, 답글이 먼저 잠그면 삭제가 기다렸다가 답글 뒤에 처리된다. 어느 쪽이든 "삭제된 원댓글에 새 답글"은 생기지 않는다 (PostgreSQL READ COMMITTED의 행 잠금 재확인 동작, 일반적인 PostgreSQL 동작으로 확인).
- 글을 `FOR SHARE`가 아니라 `FOR KEY SHARE`로 잠그는 이유: `savePost`(`src/app/write/actions.ts`)는 `lockUser(글 주인)` 뒤에 `posts`를 `UPDATE`한다. 공감은 글 행을 잠근 뒤 `lockUser(글 주인)`를 하므로, 글 행을 `FOR SHARE`로 잡으면 "공감: 글 행 → 회원 잠금 대기 / 글 수정: 회원 잠금 → 글 행 대기"로 교착할 수 있다. `FOR KEY SHARE`는 키가 아닌 컬럼 `UPDATE`(수정, 비공개 전환, 조회수)와 충돌하지 않고 글 삭제만 막는다.
- 글 행을 잠그지 않으면 그사이 글이 지워졌을 때 insert의 FK 검사가 23503 오류를 낸다(→ 서버 오류 화면, SC-007 위반). 잠근 뒤 확인하면 "글을 찾을 수 없어요"로 끝난다.
- 비공개 전환과 동시에 들어온 요청은 잠근 순간의 공개 상태로 판단한다. "누르는 사이 비공개로 바뀌었다"(US3-7)는 요청이 처리되는 시점 기준이므로 충분하다.

**Alternatives considered**:
- 잠그지 않고 FK 오류(23503)를 잡아 문구로 바꾸기: 삭제 경합은 해결되지만 "삭제된 원댓글에 답글"은 막지 못한다.
- 글을 `FOR SHARE`: 위의 교착 위험.
- `SERIALIZABLE` 격리 수준: 재시도 코드가 필요하고 이 저장소에 전례가 없다.

## R6. 입력 검증과 오류 문구

**Decision**:
- 글 ID·원댓글 ID는 `parseId()`(`src/lib/ids.ts`)로 검사하고 `null`이면 `잘못된 요청이에요`.
- 내용은 새 파일 `src/lib/social.ts`의 정규화 함수로 줄바꿈 `\r\n`·`\r`을 `\n`으로 바꾸고 앞뒤 공백을 뺀 뒤, 빈 값이면 `댓글을 적어 주세요`, 1000자 초과면 `댓글은 1000자까지예요`.
- 판정 순서: 로그인(`requireMember`, 아니면 `/`로 이동) → 글 ID → (답글) 원댓글 ID → 내용 → 글 공개 여부 → (답글) 원댓글 상태.

**Rationale**:
- 지금 `commentSchema`(`src/app/blog/actions.ts`)는 `z.coerce.number().int().positive().max(MAX_DB_INT, "잘못된 요청이에요")`다. 한국어 문구는 `.max`에만 있어 `abc`(NaN), `0`, `2.5`는 zod 기본 영어 문구가 나온다(zod 기본 문구는 영어라는 일반 동작, 구현 전 확인). `1e3`은 1000으로 바뀌어 통과한다. FR-003·NF-19와 spec Assumptions("형식이 틀린 답글 대상 ID도 `잘못된 요청이에요`")에 어긋난다. `parseId`는 이미 `e2e/params.mjs`로 검증된 규칙(앞자리 0·지수 표기 거부)이다.
- 폼 데이터의 줄바꿈: HTML의 multipart/form-data 인코딩은 줄바꿈을 CRLF로 바꾼다(HTML 명세의 일반 동작, 추측 — React의 Server Action 전송에도 같은지 구현 전 확인). 정규화하지 않으면 화면에서 1000자인 댓글이 서버에서 1000자를 넘을 수 있다. 저장도 `\n`으로 통일한다.
- 길이: JS `length`(UTF-16 단위)는 PostgreSQL `char_length`(문자 단위)보다 같거나 크므로, JS로 1000자 이하면 DB CHECK도 통과한다. 화면 `maxLength={1000}`도 UTF-16 단위다.
- 판정 순서는 spec의 문구 우선순위와 맞다: 조작된 ID는 내용과 상관없이 `잘못된 요청이에요`(US2-7), 내용 오류는 DB를 읽기 전에.

**Alternatives considered**:
- zod 스키마를 유지하고 모든 단계에 한국어 문구 붙이기: `coerce`가 `1e3`을 받아 주는 문제는 남는다.
- 줄바꿈 정규화 생략: 한도 근처에서 화면과 서버 판단이 달라진다(POST 영역이 글자 수 일치를 따로 시험하는 것과 같은 종류의 문제, `scripts/test-text-length.ts`).

## R7. 오류 때 입력 내용 남기기 (React 19 폼)

**Decision**: `addComment`·`addReply`는 거부할 때 `{ error, content }`를 돌려주고, 입력칸은 `defaultValue={state.content ?? ""}`로 그린다. 성공하면 `{ ok }`만 돌려주어 React가 폼을 비운다. 답글 대상 오류뿐 아니라 모든 거부에서 내용을 남긴다.

**Rationale**:
- React 19는 폼 액션이 끝나면 비제어 입력칸을 처음 값(`defaultValue`)으로 되돌린다. 지금 댓글 폼이 오류 때 비워지는 이유다(원본 요구사항이 "입력칸이 비워진다"고 적은 동작).
- 이 저장소는 같은 문제를 글쓰기에서 "오류면 입력값을 돌려주고 `defaultValue`로 그린다"로 풀었다(`src/app/write/actions.ts` 주석 "React 19는 폼 액션 뒤 입력칸을 처음 값으로 되돌리므로 (POST-01, #16)", `e2e/blog.mjs`가 확인). 같은 방법을 쓴다.
- spec은 일반 댓글 거부 때 내용을 남길지 정하지 않았다(Assumptions). 남기는 쪽이 사용자에게 낫고 구현이 하나라 모든 거부에 적용한다. 답글 대상 오류(SC-010)는 반드시 남긴다.
- `<select>`는 `key`를 바꿔 다시 그려야 했다는 주석이 있다(`src/components/editor/post-form.tsx`). `<textarea>`가 `defaultValue` 변경을 바로 따르는지는 구현 전 확인하고, 안 되면 `key={state 번호}`로 다시 그린다.

**Alternatives considered**:
- 제어 입력칸(`useState`): 성공 때 비우는 시점을 따로 맞춰야 하고, 폼 초기화와의 관계를 확인해야 한다.
- `onSubmit`에서 `preventDefault` 후 `startTransition`으로 액션 호출: 자동 초기화를 피하지만 이 저장소의 폼 패턴과 다르다.

## R8. 삭제 권한 (D8)

**Decision**:
- 서버: `deleteComment(commentId)`는 `UPDATE comments SET deleted_at = now(), content = '' WHERE id = $1 AND deleted_at IS NULL AND (author_id = 나 OR 관리자 OR 그 글의 블로그 주인 = 나)` 한 문장이다. 블로그 주인 조건은 `EXISTS (posts join blogs where posts.id = comments.post_id and blogs.owner_id = 나)`. `deleteReply(replyId)`는 `replies → comments → posts → blogs`로 같은 조건을 쓴다. 바뀐 행이 없으면 문구 없이 끝난다(FR-013).
- 관리자 여부는 세션의 `viewer.user.role === "admin"`(`src/lib/auth.ts`의 `additionalFields.role`, `input: false`)으로 판단한다.
- 화면: `src/lib/social.ts`의 순수 함수 `canDeleteComment({ viewerId, isAdmin, authorId, postOwnerId })`로 서버가 각 줄의 `canDelete`를 계산해 넘긴다.

**Rationale**:
- 확인과 변경을 한 문장으로 하면 그 사이에 권한이 바뀌는 경합이 없고, 이미 삭제된 행·범위 밖 ID·남의 댓글 모두 "0행 변경"으로 같은 결과가 된다(US2-11, SC-005).
- 화면 계산과 서버 조건이 같은 규칙인지는 `scripts/test-social.ts`(순수 함수)와 `e2e/comments.mjs`(조작 요청)로 각각 확인한다. 화면 값은 편의일 뿐 서버가 다시 판단한다(constitution IV).
- 지금 화면은 `authorId`를 클라이언트로 넘겨 비교한다(`comment-section.tsx`). 서버가 `canDelete`를 계산하면 회원 ID를 넘기지 않아도 된다(R13).
- 비공개 글의 댓글: 블로그 주인·관리자·작성자는 글 공개 여부와 상관없이 지울 수 있다. 화면에서 비공개 글은 주인만 보므로 실제로는 주인만 해당한다.

**Alternatives considered**:
- 먼저 읽어 권한을 판단하고 따로 `UPDATE`: 두 문장 사이 경합, 코드가 길다.
- 관리자를 위한 별도 액션: 권한만 다르고 동작이 같아 하나로 둔다.

## R9. 공감 동시성과 보상 1번

**Decision**:
- `toggleLike(postId)` 트랜잭션: 글 확인(R5, `FOR KEY SHARE`) → 내 공감 `DELETE ... RETURNING`(있으면 취소로 끝) → `INSERT ... ON CONFLICT DO NOTHING RETURNING` → 새로 들어간 행이 없으면(동시 요청이 먼저 넣음) 끝 → 남의 글이면 `lockUser(글 주인)` → `ref_id = "{글ID}:{공감한 회원ID}"`인 `like_received` 원장 행이 없을 때만 `grantReward`.
- 반환값은 지금처럼 없음. 성공하면 `revalidatePath("/", "layout")`로 서버 값이 다시 그려지고, 실패·무변경이면 다시 그리지 않아 `useOptimistic`이 누르기 전 값으로 돌아간다.
- 새 답글 보상 원장의 `ref_id`는 `reply:{답글ID}`, 댓글은 지금처럼 `{댓글ID}`.
- DB 제약으로도 막기 위해 game에 `point_ledger` 부분 UNIQUE(`user_id`, `ref_id`) WHERE `reason = 'like_received'`를 요청한다 (plan 의존성 D-7). constitution V가 "한 글에 공감 한 번" 같은 규칙을 DB 제약으로도 막으라고 하므로 권장이 아니라 필요한 요청이다. game이 출석 보상에 같은 모양의 `point_ledger_attendance_uq`를 두므로 방식도 같다. 인덱스를 만들기 전에 기존 중복 행 수를 센다(0으로 예상, 추측).

**Rationale**:
- 지금 코드는 `tx.insert(postLikes).values(...)`에 충돌 처리가 없어, 같은 회원이 두 탭에서 동시에 처음 누르면 늦은 쪽이 PK 위반 → 오류 화면(원본 SOC-03 열린 질문). `ON CONFLICT DO NOTHING`이면 늦은 쪽은 먼저 넣은 트랜잭션이 끝나길 기다렸다가 아무것도 하지 않는다(FR-026, SC-003). `toggleFollow`가 이미 같은 방식이다.
- 보상 1번(FR-029, SC-004)은 지금 코드의 `ref_id` 확인을 그대로 쓴다. `lockUser(글 주인)` 안에서 확인하므로 동시 요청에도 한 번이다(ERD 3.6). 공감 행이 새로 들어간 경우에만 보상 단계에 가므로 "동시 첫 공감 두 번 → 보상 두 번 확인"도 생기지 않는다.
- `useOptimistic`은 transition이 끝나면 props 값으로 돌아간다(React 19 동작, 지금 `like-button.tsx`가 이 방식). 다른 탭에서 이미 공감한 뒤 누르면 서버가 취소로 처리하고 다시 그려져 ♡로 돌아온다(Edge Cases).

**Alternatives considered**:
- `SELECT` 후 분기: 동시 요청에서 둘 다 "없음"을 본다.
- 공감 결과(`{ liked, count }`)를 돌려받아 화면에 반영: `revalidatePath`로 이미 같은 결과가 오고, 글 카드 수 등 다른 화면도 함께 맞춰진다. 바꿀 이유가 없다.

## R10. 이웃 추가·취소 요청 검사

**Decision**:
- `toggleFollow(followeeId)`: 인자가 문자열이 아니거나 길이가 1~64자가 아니면 아무것도 하지 않는다. 자기 자신이면 끝(DB CHECK `follows_not_self_check`도 있음).
- 취소: `DELETE ... RETURNING`. 지운 행이 없으면 추가: `INSERT INTO follows (follower_id, followee_id) SELECT 나, owner_id FROM blogs WHERE owner_id = $1 ON CONFLICT DO NOTHING` 한 문장. FK 위반(23503, 그 순간 대상 회원 삭제)은 `try/catch`에서 새 `foreignKeyViolation(err)`로 가려 무시하고, 다른 오류는 다시 던진다. `foreignKeyViolation`은 공통 파일 `src/server/db-errors.ts`에 기존 `uniqueViolation`과 같은 방식(`pgError`로 `code` 확인)으로 하나만 더한다 (공통 모듈 추가, plan-context 5.2 "바꿔야 하면 plan에 이유": 이 경합이 500이 되면 SC-007 위반).
- 보상 없음(FR-041). 처리 뒤 `revalidatePath("/", "layout")`. 화면은 지금처럼 폼 액션 제출 후 다시 그려진 값으로 바뀐다(FR-036, 미리 바꾸지 않음).

**Rationale**:
- Server Action 인자는 클라이언트가 보낸 값이라 문자열이라는 보장이 없다(`e2e/social.mjs`가 이미 인자를 바꿔 보낸다). 객체나 숫자가 DB 쿼리로 가면 드라이버 오류(500)가 날 수 있다 — 추측(드라이버 동작 미확인)이지만, 검사 비용이 작다. `users.id`는 Better Auth 형식 32자(plan-context 3.2), 64자는 ERD 부록 `ref_id` 여유와 같은 기준이다.
- `INSERT ... SELECT`는 "블로그를 가진 회원만"(FR-040)과 저장을 한 문장으로 한다. 지금 코드는 확인과 insert가 두 문장이다.
- 동시 추가는 PK(`follower_id`, `followee_id`)가 막는다(FR-039).

**Alternatives considered**:
- `users`만 확인: FR-040은 "블로그를 가진 회원"이다. 가입 통합(auth 단계 1) 뒤에는 모든 회원이 블로그를 갖지만 규칙을 문장 그대로 둔다.
- 대상 `blogs` 행을 먼저 `FOR KEY SHARE`로 잠가 23503을 없애기: 회원 삭제는 `users` 행을 먼저 지우고 `blogs`로 `CASCADE`하므로, 이쪽은 `blogs` 잠금 → `users` FK 확인 대기, 삭제 쪽은 `users` 삭제 → `blogs` 대기가 되어 교착할 수 있다. 오류 코드로 가리는 쪽이 단순하다.

## R11. 이웃 새 글 순서 (D15)

**Decision**:
- "최근 7일(한국 시간)"은 **오늘을 포함한 한국 날짜 7일**이다. 시작 날짜는 `src/lib/social.ts`의 순수 함수 `favoriteWindowStart(today)`(기간 상수 `FAVORITE_RECENT_DAYS = 7`, `previousDay()`를 6번 적용, `getBlogVisitDays`와 같은 방식)가 `todayKST()`로 계산하고, 쿼리는 그 날짜의 한국 0시와 비교한다: `posts.created_at >= (${시작 날짜}::date::timestamp AT TIME ZONE 'Asia/Seoul')`. 단위 테스트한 함수가 실제 쿼리 경계를 정하도록 SQL 안에서 따로 `interval '6 days'`를 계산하지 않는다.
- 정렬: `(즐겨찾기 이웃 AND 최근 7일) DESC, created_at DESC, id DESC`. 즐겨찾기 여부는 `EXISTS (SELECT 1 FROM follows WHERE follower_id = 나 AND followee_id = blogs.owner_id AND is_favorite)`로 ORDER BY에 넣는다.
- 구현 위치: `src/server/blog.ts`의 `listFeed`에서 `followerId`가 있을 때만 위 정렬 앞부분을 넘긴다. `listFeed`는 `paged(where, page, extra)`를 부르고 `paged`가 `baseList(where)`를 만들므로, `paged`와 `baseList` 둘 다에 선택 인자 `orderFirst?: SQL`을 더해 `paged`가 그대로 넘긴다. 없으면 지금 정렬 그대로(공통 모듈 추가, post와 합의). Drizzle의 동적 모드 없이 만든 select는 `orderBy`를 두 번 부를 수 없어서(타입에서 막힌다, 구현 전 확인) `extra` 콜백에서 정렬을 바꾸는 방법은 쓰지 않는다.

**Rationale**:
- spec·town·post 모두 "최근 7일(한국 시간) 안"이라고만 썼다. 이 저장소에서 "최근 7일"은 이미 "오늘 포함 7일"이다(`getBlogVisitDays`의 7칸, BLOG-06 "마지막 칸 `오늘`"). 한국 날짜 단위라 자정을 지나면 한 번에 바뀌어 설명하기 쉽다. US4-11의 예(3일 전 위, 10일 전 아래)를 만족한다.
- 날짜 계산은 이미 시험된 `todayKST()`·`previousDay()`(`src/lib/game.ts`, `scripts/test-game.ts`)를 쓰고, 한국 0시 변환은 `startOfTodayKST`(`src/server/points.ts`)와 같은 `AT TIME ZONE 'Asia/Seoul'`이다. 한국은 서머타임이 없어 날짜 경계가 하루 24시간으로 일정하다.
- `EXISTS`를 정렬에만 쓰면 지금 `WHERE`(이웃 거르기 `inArray`)와 `paged`의 개수 쿼리(`COUNT`)를 바꾸지 않아도 된다(`paged`는 정렬 인자를 넘겨 주기만 한다). 정렬 키가 모든 행에서 정해져 있어 페이지를 넘겨도 순서가 이어진다(FR-042).
- `listFeed`의 "이웃 거르기·정렬"은 social, 공통 부분은 post 소유다(plan-context 5.2). 정렬 인자 추가는 기존 동작을 바꾸지 않는다.

**Alternatives considered**:
- 지금 시각에서 168시간(롤링 7일): 같은 글이 오후에 갑자기 내려가 보이고, 한국 시간 기준이라는 말과 덜 맞는다.
- 오늘 0시 이전 7일(오늘 + 7일 = 8일): "7일"보다 넓다.
- `follows` inner join으로 바꾸기: `WHERE`와 개수 쿼리까지 바뀌어 post 소유 부분을 더 건드린다.
- 두 쿼리(즐겨찾기 7일 글 / 나머지)를 이어 붙이기: 페이지 경계 계산이 복잡해진다.

이 경계는 spec이 정하지 않아 plan에서 정했다. 팀 확인 권장(plan "남은 문제" Q1, quickstart 기대 결과에 그대로 적음).

## R12. 댓글 수 정의 (FR-016, FR-032)

**Decision**: `src/server/social.ts`에 다음 두 가지를 둔다.
- `liveCommentCountSql(postIdColumn)`: `(삭제 안 된 comments 수) + (삭제 안 된 replies 수, comments를 거쳐 그 글)`을 돌려주는 SQL 조각. `src/server/blog.ts`의 `listColumns.commentCount`가 이 식을 쓴다.
- `countLiveComments()`: 마을 전체 합계. 관리자 통계(auth 소유 `src/app/admin/page.tsx`)가 쓰도록 요청한다(의존성 D-4).
- 글 상세의 `💬 댓글 N`은 `getCommentThread`가 돌려준 목록에서 같은 규칙(삭제 안 된 댓글 + 삭제 안 된 답글)으로 센다.

**Rationale**:
- 세 곳이 같은 수여야 한다(FR-016, POST-05 FR-036). 지금은 세 곳이 따로 `comments`만 센다(`listColumns`, `comment-section.tsx`의 `visibleCount`, 관리자 통계). 답글을 분리하면 세 곳 모두 고쳐야 하므로 정의를 한 곳에 둔다. plan-context 5.2: "댓글 수 계산의 정의는 social이 정하고 post 목록이 쓴다".
- 성능: 목록은 한 페이지 8개라 글마다 상관 서브쿼리 2개면 충분하다. `comments (post_id, created_at)`, `replies (comment_id, created_at)` 인덱스를 탄다.

**Alternatives considered**:
- `posts`에 댓글 수 컬럼(반정규화): ERD 3.17에 없고 증감 코드가 늘어난다.

## R13. 댓글 영역에 넘기는 데이터

**Decision**: `getCommentThread(postId, viewer)`가 화면용 데이터를 만든다.
- 쿼리 두 번(ERD 3.8): 그 글의 댓글(작성자 `profiles`·캐릭터 `items`·`blogs`를 **left join**), 그 글 댓글들의 답글(`replies` join `comments` on `post_id`).
- 각 줄: `id`, 작성 시각 문자열, `deleted`, `content`(삭제면 빈 글자, DB에도 빈 글자), `author`(`{ nickname, characterAsset, blogSlug }` 또는 탈퇴로 `null`), `canDelete`, (댓글만) `canReply`, `replies[]`.
- 작성자 회원 ID는 넘기지 않는다.
- 탈퇴 자리(`author: null`)는 캐릭터·닉네임·작성 시각 없이 `삭제된 댓글이에요`만 보인다.

**Rationale**:
- 지금 `getComments`는 `profiles`·`blogs`를 inner join해 작성자 프로필이 없는 행이 빠진다(plan-context 4.4). 탈퇴 자리(D2)를 보이려면 left join이어야 한다.
- auth FR-052·SC-014: 탈퇴 자리에는 닉네임·캐릭터가 보이지 않고 "`삭제된 댓글이에요`만" 남는다. 일반 삭제 자리(작성자·시각 보임, FR-014)와 다르다.
- `canReply`·`canDelete`를 서버에서 계산하면 화면에 회원 ID를 보낼 필요가 없다(constitution VII). FR-019: [답글]은 회원에게, 삭제되지 않은 원댓글에만.
- 날짜 문자열은 지금처럼 서버에서 `formatDateTime()`(`src/lib/format.ts`, 한국 시간 24시간제)으로 만든다(FR-011).
- `getComments`는 social 소유 함수라(plan-context 5.2) 지우고 새 함수를 `src/server/social.ts`에 둔다.

**Alternatives considered**:
- 쿼리 한 번(댓글 left join 답글): 댓글 정보가 답글 수만큼 반복된다.

## R14. 방문자 공감 안내 (FR-030)

**Decision**: 방문자에게는 비활성 버튼 아래에 `로그인하면 공감할 수 있어요`를 보이는 글자로 둔다(`title`도 유지). 버튼에는 `aria-describedby`로 안내를 잇는다.

**Rationale**: 지금은 `title`에만 있다(`src/components/blog/like-button.tsx`). `.btn:disabled`가 `pointer-events: none`(`src/app/globals.css`)이라 마우스를 올려도 안 보이고 터치 화면에서는 볼 수 없다(원본 SOC-03 열린 질문). spec은 "안내가 붙어 있다"이므로 보여야 한다.

**Alternatives considered**: 방문자가 누르면 로그인 화면으로 보내기 — spec Assumptions가 두지 않기로 했다.

## R15. 알림 발생 (2차, D11)

**Decision**: game이 만든 알림 기록 도우미를 각 Server Action 트랜잭션 안에서 부른다. game plan은 이 도우미를 `notifyActivity(tx, { recipientId, actorId, kind, postId })`(`src/server/notifications.ts`, `kind`는 `like`/`comment`/`reply`)로 정했다(`specs/005-game/contracts/notifications.md` 6.1). 이름·타입은 game 소유라 game이 바꾸면 따라간다.
- 공감: 공감 행이 **새로 들어갔고** 글 주인 ≠ 나일 때, 받는 회원 = 글 주인, 종류 = 공감, 관련 글.
- 댓글: 글 주인 ≠ 나일 때, 받는 회원 = 글 주인, 종류 = 댓글.
- 답글: 원댓글 작성자 ≠ 나일 때, 받는 회원 = 원댓글 작성자(글 주인이 아님), 종류 = 답글. 원댓글 작성자가 없는 경우는 원댓글이 삭제 표시라 답글이 거부되므로 생기지 않는다.
- 보상 상한·보상 여부와 상관없이 위 조건이면 남긴다. 공감 취소 때 알림은 지우지 않는다(game Assumptions).
- 알림을 누르면 가는 곳(game FR-044)을 위해 글 상세 댓글 영역에 `id="comments"`, 각 줄에 `id="comment-{id}"` / `id="reply-{id}"`를 둔다(1차에서 미리 넣는다).

**Rationale**:
- 같은 트랜잭션이면 "공감은 됐는데 알림은 없음" 같은 반쪽 상태가 없다.
- FR-033은 "공감이 새로 저장되면"이다. 공감 → 취소 → 공감을 반복하면 알림도 반복된다. spec 문장 그대로 따르고, 묶음 처리는 game Assumptions("같은 글의 여러 공감을 묶지 않는다")와 같게 두지 않는다. 반복 알림이 문제가 되면 팀이 정한다(plan "남은 문제" Q4).

**Alternatives considered**:
- 트랜잭션 밖에서 알림 기록: 실패하면 알림만 빠진다.
- 공감 알림을 보상처럼 (공감한 사람, 글)당 1번: spec 문장("새로 저장되면")과 다르다.

## R16. 화면 갱신과 "열 때마다 최신" (FR-009, FR-028, FR-054)

**Decision**: 모든 social Server Action은 저장이 끝나면 지금처럼 `revalidatePath("/", "layout")`을 부른다. 마을 소식·이웃 새 글 페이지는 캐시 설정을 더하지 않는다.

**Rationale**:
- CLAUDE.md 규칙: 코인이 바뀌는 Server Action은 레이아웃을 갱신한다(댓글 보상은 쓴 사람의 헤더 코인을 바꾼다).
- `/feed`는 `getViewer()`(`headers()` 사용)를, `/feed/following`은 `requireMember()`를 불러 요청마다 다시 그려진다(Next.js의 동적 렌더링, `next.config.ts`에 `cacheComponents` 같은 캐시 설정 없음 — 구현 전 Next.js 16 문서로 확인). 그래서 새 글·삭제·공감·댓글이 다음에 열 때 반영된다(FR-054).
- 알려진 상호작용: `revalidatePath`가 글 상세를 다시 그리면 `incrementViewCount()`가 다시 불린다. post의 조회수 하루 1번(D5) 작업이 해소한다(의존성 D-6).

## R17. 접근성·모바일 (FR-004, FR-005, constitution VI)

**Decision**:
- [답글]·[답글 취소]·[삭제] 글자 버튼: 보이는 글자 크기는 유지하고 누르는 영역을 최소 44×44px(`min-h-11 min-w-11`)로, `focus-visible` 표시(밑줄 또는 테두리)를 준다.
- [댓글 등록]·[답글 등록]: 지금 `btn py-1.5 text-sm`이라 높이가 약 32px(위아래 여백 6px×2 + 줄 높이 20px, 계산값)다. `min-h-11`을 더한다.
- 댓글·답글 줄의 닉네임 링크와 방문자 안내의 `로그인` 링크: 글자 모양은 그대로 두고 누르는 영역만 44px 높이가 되게 한다(`inline-flex min-h-11 items-center`, 줄 간격이 벌어지면 음수 여백으로 보정 — 구현 때 스크린샷으로 확인).
- 이웃 버튼(`blog-header.tsx`의 이웃 버튼 부분): `min-h-11`(지금 `.btn` 기본 높이는 약 43px — 계산값, 추측).
- 마을 소식 탭(`btn py-1.5 text-sm`, 약 32px)·인기 태그 칩·페이지 번호는 post 소유 파일(`src/components/blog/feed-view.tsx`, `src/components/pagination.tsx`)이라 post에 요청한다 (plan 의존성 D-10).
- 답글 들여쓰기는 지금 `ml-10 border-l-2 pl-4`, 긴 내용 `whitespace-pre-wrap break-words` 유지. 375px에서 답글 줄 폭은 약 375 − 32(바깥 여백) − 48(카드 안 여백) − 58(들여쓰기) ≈ 237px로 내용이 줄바꿈된다(계산값).
- 공감 버튼의 `aria-pressed`(FR-027)는 그대로.
- 등록 버튼·입력칸은 Tab으로 이동되고 `focus:border-sun`으로 선택이 보인다(FR-005, 지금 동작).

**Rationale**: 지금 [답글]·[삭제]는 `text-xs`이고 여백이 없고, 등록 버튼도 `py-1.5`로 여백을 줄여 constitution VI의 44×44px보다 작다(`comment-section.tsx`). constitution VI는 버튼과 링크 모두 44×44px를 요구한다.

## R18. 검증 방식

**Decision**:
- 단위: 새 `scripts/test-social.ts` + `package.json`의 새 스크립트 `test:social`(`test` 체인 끝에 붙임). 대상은 `src/lib/social.ts`: 내용 정규화·검증 문구, `canDeleteComment` 표(작성자/주인/관리자/남), 즐겨찾기 우선 시작 날짜 `favoriteWindowStart`(오늘 포함 7일, `listFeed`가 실제로 쓰는 함수).
- E2E (plan-context 6.2 관례: `dotenv` + `pg` Pool로 DB 준비·확인, `check()` 모음, `❌`면 `exit(1)`, 실행마다 새 회원 `pre${Date.now() % 100_000_000}`):
  - 새 `e2e/comments.mjs`: SOC-01·SOC-02 전체, 삭제 권한, 보상 상한(댓글 6 + 답글 5), 삭제 내용이 HTML·RSC 응답에 없는지, 답글 대상 오류 때 입력 유지, 로그인이 풀린 뒤의 쓰기 요청(FR-002), 375px 스크린샷과 누르는 영역 44px.
  - `e2e/social.mjs`: 이웃(기존) + 동시 이웃 추가, 방문자 `/feed/following` 이동, 공감(동시 첫 공감, 공감·취소 반복 보상 1번, 비공개 글·범위 밖 ID 조작), 이웃 새 글 순서(DB로 `created_at`·`is_favorite` 준비), DB에 직접 위반 데이터 넣기 거부(공감 중복 23505, 자기 이웃 23514, 삭제 행 내용 23514). "온보딩 전 회원" 부분은 auth 단계 1 뒤 "블로그 없는 `users` 행을 DB에 직접 넣어 확인"으로 바꾼다.
  - `e2e/params.mjs`: 답글 대상 ID `99999999999`·`abc` → `잘못된 요청이에요`, `deleteReply` 범위 밖 → 200.
  - 새 `e2e/feed-load.mjs`: 공개 글 1,000개를 SQL로 넣고 `/feed` 응답 시간을 잰 뒤 지운다(SC-001, `next build && next start` 대상).
- 2차: `e2e/comments.mjs`·`e2e/social.mjs`에 알림 행 확인을 더한다(알림함 화면은 game의 e2e).

**Rationale**: 이 저장소는 테스트 프레임워크 없이 위 방식을 쓴다(plan-context 6장). 답글을 확인하는 e2e가 지금 없다.

## R19. 구현 전 설치된 패키지 문서로 확인할 것

`node_modules`가 없어 확인하지 못한 항목이다. 구현자가 `npm install` 뒤 확인하고, 다르면 같은 결과를 내는 다른 방법으로 바꾼다.

| 항목 | 이 plan이 기대하는 동작 | 확인할 곳 | 다를 때 대안 |
|---|---|---|---|
| Drizzle `select().for("key share" / "share", { of })` | `FOR KEY SHARE OF posts`, `FOR SHARE` SQL 생성 | `drizzle-orm` pg-core select builder 타입 | `tx.execute(sql\`... FOR KEY SHARE\`)` |
| Drizzle `insert().onConflictDoNothing().returning()` | 충돌이면 빈 배열 | 같은 패키지 | `uniqueViolation(err)`로 잡기 |
| Drizzle select 빌더에서 `orderBy`를 두 번 부를 수 있는지 | 동적 모드(`$dynamic()`)가 아니면 타입에서 막힌다 (R11의 `paged`·`baseList` 인자 추가 근거) | `drizzle-orm` pg-core select builder 타입 | 막히지 않으면 `paged`를 고치지 않고 `extra`에서 정렬을 다시 지정할 수도 있다 |
| drizzle-kit migrate가 대기 중인 마이그레이션을 한 트랜잭션으로 적용하는지 | 파일 3개(만들기 → 데이터 → 정리)가 함께 성공하거나 함께 취소 (drizzle-orm pg 마이그레이터가 전체를 한 트랜잭션으로 감싼다고 알고 있다 — 추측) | `drizzle-orm` `migrator`·pg dialect `migrate()` | 파일 단위면 data-model 6.3대로 백업에서 되돌린다 |
| `drizzle-kit generate --custom --name=...` | 빈 SQL 파일 + journal 항목 | `drizzle-kit` 도움말 | 생성된 빈 마이그레이션 파일에 직접 쓰기 |
| drizzle-kit의 nullable·FK 동작 변경 생성 | `DROP NOT NULL`, FK 다시 만들기 | 생성 결과 SQL 읽기 | 생성 SQL을 고쳐 커밋 |
| React 19 폼 액션 뒤 `<textarea defaultValue>` 초기화 | 돌려준 `content`로 되돌아감 | React 19 문서, 수동 확인 | `key`로 다시 그리기 (R7) |
| Next.js 16 Server Action 안 `redirect()`(로그인 풀림) | 클라이언트가 `/`로 이동 | `node_modules/next/dist/docs/` | 지금 `requireMember()`가 이미 이 방식이라 위험 낮음 |
| Next.js 16 동적 렌더링(`headers()` 사용 페이지) | 요청마다 다시 그림 | 같은 문서 | `export const dynamic = "force-dynamic"` |
| 폼 데이터 줄바꿈 CRLF 변환 | 서버에 `\r\n`이 올 수 있음 | 실제 요청 확인 | 정규화는 해도 해가 없으므로 그대로 둔다 |
