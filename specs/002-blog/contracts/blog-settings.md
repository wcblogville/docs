# Contract: 블로그 관리 Server Action (기본 정보·주소·카테고리)

**Feature**: `002-blog` | **Spec**: [../spec.md](../spec.md) | **Data model**: [../data-model.md](../data-model.md) | **Research**: [../research.md](../research.md)

파일: `src/app/settings/blog/actions.ts` (`"use server"`). 화면: `src/app/settings/blog/settings-forms.tsx`. *(plan 임시)* 표시는 spec에 없어 plan이 임시로 정한 문구다.

## 0. 모든 처리에 공통

| 항목 | 약속 |
|---|---|
| 권한 | 첫 줄 `requireMember()`. 로그인하지 않았으면 `/`로 보낸다(리다이렉트). 대상 블로그는 요청 값이 아니라 로그인한 회원의 블로그(`viewer.profile.blogId`, `blogs.owner_id = 나`)다 (FR-021) |
| 다른 사이트 요청 | Next.js Server Action의 Origin 확인으로 거부 (FR-060, NF-12) |
| 숫자 인자 | 카테고리·소분류 ID는 `parseId()`. 순서 방향은 `-1` 또는 `1`만 (FR-042) |
| 입력 검증 | 서버에서 다시. 길이는 앞뒤 공백을 지운 뒤 **코드 포인트**로 센다 (`charCount`, research R-07) |
| 갱신 | 성공하면 `revalidatePath("/", "layout")` — 같은 화면에 머문 채 새 값으로 다시 그려지고, 블로그 홈·광장·글 카드·헤더도 다음 열기부터 새 값 |
| 결과 형식 | 폼 처리는 `FormState = { error?: string; ok?: number; values?: Record<string, string> }`. `error`는 첫 번째 오류 하나. 오류일 때 `values`에 보낸 값을 돌려줘 칸에 남긴다 (spec 기본값, research R-08). 성공하면 `ok = Date.now()` |
| 서버 오류 | 조작된 값으로 500이 나지 않는다 (SC-009) |
| 카테고리 동시성 | 대분류·소분류 추가·삭제·순서 바꾸기는 트랜잭션 첫 줄에서 내 블로그 행을 `FOR UPDATE`로 잠근다 (research R-12) |

## 1. `updateBlogInfo(prev, formData)` — 이름·소개 (바뀜)

| 항목 | 내용 |
|---|---|
| 입력 | `title`, `description` (그 밖의 칸은 읽지 않는다) |
| 검증 (이 순서, 첫 오류만) | 이름 공백 제거 후 0자 → `블로그 이름을 적어 주세요` / 41자 이상 → `블로그 이름은 40자까지예요` / 소개 161자 이상 → `소개는 160자까지예요` (FR-016·017) |
| 처리 | `UPDATE blogs SET title, description WHERE owner_id = 나` |
| 성공 | `{ ok }` → [저장] 왼쪽 초록 `저장했어요 ✓`, 칸은 저장된 값 |
| 실패 | `{ error, values: { title, description } }` → [저장] 왼쪽 빨간 글씨, 칸은 보낸 값 그대로. DB 값은 그대로 |
| 바뀌는 점 | 지금은 `values`를 돌려주지 않아 오류 때 칸이 저장된 값으로 되돌아간다. 길이를 UTF-16으로 센다 |

## 2. `updateBlogSlug(prev, formData)` — 블로그 주소 (새)

| 항목 | 내용 |
|---|---|
| 입력 | `slug` |
| 정규화 | 앞뒤 공백 제거 → 소문자 (FR-009, auth 모듈 `normalizeName()`) |
| 검증·처리 순서 | 1. 형식 `^[a-z0-9_]{3,20}$` 아님 → `주소는 영문 소문자, 숫자, _ 로 3~20자예요`<br>2. 지금 주소와 같음 → 저장 없이 `{ ok }`<br>3. 예약어(`isReservedName()`, FR-009 16개) → `이 주소는 쓸 수 없어요`. 단 `notice`는 관리자(`users.role = 'admin'`)에게 허용 (research R-05)<br>4. 트랜잭션: `lockName(tx, 새 주소)` → `findNameConflict(tx, 새 주소, { exceptUserId: 나 })`의 `username` 또는 `slug`가 참(다른 회원의 아이디·주소와 같음) → `이미 있는 주소예요` → `UPDATE blogs SET slug WHERE owner_id = 나` → `blogs_slug_unique` 위반(23505) → `이미 있는 주소예요`. `nickname` 결과는 보지 않는다 (research R-03) |
| 성공 | `{ ok, values: { slug: 저장된 값 } }` → 주소 폼 버튼 왼쪽 `저장했어요 ✓` *(plan 임시)*, 칸은 정규화된 값(`My_Blog` → `my_blog`). 다음 열기부터 `/@{새 주소}` 200, `/@{예전 주소}` 404, `내 블로그로 →`와 사이트의 모든 링크가 새 주소 |
| 실패 | `{ error, values: { slug: 보낸 값 } }`. 주소는 바뀌지 않는다 |
| 하지 않는 것 | 예전 주소 보호 기간, 변경 횟수·주기 제한, 예전 주소 자동 이동 (FR-018, D3) |
| 동시성 | 같은 값으로 가입(아이디)과 주소 변경이 동시에 오면 한쪽만 성공한다 (이름 잠금, research R-04). 두 회원이 같은 주소로 동시에 바꾸면 UNIQUE가 한쪽을 막는다 |
| 관련 수용 시나리오 | US3-4·5·7·9·10·11, SC-005, SC-012 |

## 3. 대분류

### 3.1 `addCategory(prev, formData)` (바뀜)

| 항목 | 내용 |
|---|---|
| 입력 | `name` |
| 검증 | 공백 제거 후 0자 → `카테고리 이름을 적어 주세요` / 21자 이상 → `카테고리 이름은 20자까지예요` |
| 처리 | 트랜잭션: 블로그 행 잠금 → `INSERT categories (blog_id = 내 블로그, name, position = MAX + 1)` → `categories_blog_name_uq` 위반 → `이미 있는 카테고리예요` |
| 성공 | `{ ok }` → 목록 맨 아래에 새 줄, 추가 칸 비움 (US5-1) |
| 실패 | `{ error, values: { name } }` → 추가 칸 아래 빨간 한 줄, 칸에 보낸 값 남김 |

### 3.2 `renameCategory(categoryId, name)` (바뀜)

| 항목 | 내용 |
|---|---|
| 입력 | `categoryId: number`, `name: string` (폼 아님, 줄 안의 [저장]) |
| 검증 | `parseId` 실패 → `잘못된 요청이에요` / 이름 규칙은 3.1과 같음 |
| 처리 | `UPDATE categories SET name WHERE id AND blog_id = 내 블로그 RETURNING id` → **바뀐 행이 0개면 `잘못된 요청이에요`** (다른 블로그의 ID, US5-10) → UNIQUE 위반 → `이미 있는 카테고리예요` |
| 결과 | `{ ok }`이면 편집 칸이 닫힌다. `{ error }`이면 그 줄 아래 빨간 한 줄, 고친 이름은 칸에 남는다 |
| 바뀌는 점 | 지금은 다른 블로그 ID여도 `{ ok }`를 돌려준다 (`src/app/settings/blog/actions.ts:48-64`) |

### 3.3 `deleteCategory(categoryId)` (바뀜)

| 항목 | 내용 |
|---|---|
| 화면 확인 | `'{이름}' 카테고리를 지울까요? 글은 남고 '카테고리 없음'이 돼요.` 취소하면 요청하지 않는다 (FR-037) |
| 검증 | `parseId` 실패 → 아무것도 안 하고 끝 (문구 없음) |
| 처리 | 트랜잭션: 블로그 행 잠금 → 내 블로그 대분류인지 확인(아니면 끝) → (post 단계 3 이후) 그 대분류 글의 `category_id`·`subcategory_id`를 함께 NULL, `updated_at`은 그대로 → 대분류 DELETE(소분류 CASCADE) → 남은 대분류 `position` 0부터 다시 매김 (research R-11·R-12). 이 처리를 거치지 않는 경로(회원 삭제의 CASCADE, 비운 뒤 다른 탭이 같은 대분류로 저장한 글)는 post의 트리거 `posts_clear_subcategory`가 소분류 칸을 함께 비운다 |
| 결과 | 반환 없음. 글은 남아 "카테고리 없음" (US5-9) |

### 3.4 `moveCategory(categoryId, direction)` (바뀜)

| 항목 | 내용 |
|---|---|
| 검증 | `parseId` 실패 또는 방향이 `-1`·`1`이 아님 → 끝 (문구 없음) |
| 처리 | 트랜잭션: 블로그 행 잠금 → 내 블로그 대분류를 `position, id` 순으로 읽음 → 바로 위·아래와 맞바꿈(끝이면 그대로) → 0부터 다시 매김 |
| 화면 | 맨 위 ▲, 맨 아래 ▼는 비활성. ▲▼는 가로로 놓고 각각 44×44px (research R-24) |
| 바뀌는 점 | 블로그 행 잠금 추가 |

## 4. 소분류 (새)

소분류 이름 규칙·문구는 대분류와 같다 (spec 기본값). 화면의 `aria-label`: 추가 칸 `새 소분류 이름` *(plan 임시)*, 이름 바꾸기 칸 `소분류 이름` *(plan 임시)*, ▲▼ `{이름} 위로` / `{이름} 아래로`.

### 4.1 `addSubcategory(categoryId, prev, formData)`

| 항목 | 내용 |
|---|---|
| 호출 | 대분류 줄의 [소분류 추가]를 누르면 그 아래 입력칸(안내 `새 소분류` *(plan 임시)*) + [추가]가 열린다. `addSubcategory.bind(null, 대분류ID)`를 `useActionState`로 쓴다. Enter로도 추가 |
| 입력 | 묶은 인자 `categoryId`, 폼 `name` |
| 검증 | `parseId(categoryId)` 실패 → `{}` (문구 없음, FR-042) / 이름 규칙 |
| 처리 | 트랜잭션: 블로그 행 잠금 → 대분류가 내 블로그 것인지 확인(아니면 `{}`) → `INSERT subcategories (category_id, name, position = 그 대분류 안 MAX + 1)` → `subcategories_category_name_uq` 위반 → `이미 있는 카테고리예요` |
| 성공 | `{ ok }` → 그 대분류 아래 맨 끝에 새 줄, 칸 비움 (US5-2). "여행"·"공부"에 같은 "맛집"은 둘 다 된다 (US5-4) |
| 실패 | `{ error, values: { name } }` → 그 추가 칸 아래 빨간 한 줄 |

### 4.2 `renameSubcategory(subcategoryId, name)`

| 항목 | 내용 |
|---|---|
| 검증 | `parseId` 실패 → `잘못된 요청이에요` / 이름 규칙 |
| 처리 | `UPDATE subcategories SET name WHERE id AND category_id IN (내 블로그 대분류) RETURNING id` → 0개면 `잘못된 요청이에요` → UNIQUE 위반 → `이미 있는 카테고리예요` |
| 결과 | 3.2와 같은 모양 |

### 4.3 `deleteSubcategory(subcategoryId)`

| 항목 | 내용 |
|---|---|
| 화면 확인 | `'{이름}' 소분류를 지울까요? 글은 '{대분류 이름}'에 남아요.` (FR-038) |
| 검증 | `parseId` 실패 → 끝 |
| 처리 | 트랜잭션: 블로그 행 잠금 → `DELETE subcategories WHERE id AND category_id IN (내 블로그 대분류) RETURNING category_id` → 0개면 끝 → 같은 대분류의 남은 소분류 `position` 다시 매김. 글은 posts 복합 FK가 `subcategory_id`만 NULL로 (post) |
| 결과 | 반환 없음. 글은 대분류에 남는다 (US5-8) |

### 4.4 `moveSubcategory(subcategoryId, direction)`

| 항목 | 내용 |
|---|---|
| 검증 | `parseId` 실패 또는 방향이 `-1`·`1`이 아님 → 끝 |
| 처리 | 트랜잭션: 블로그 행 잠금 → 소분류가 내 블로그 대분류 소속인지 확인 → 같은 대분류의 소분류를 `position, id` 순으로 읽어 맞바꿈 → 다시 매김 |
| 화면 | 같은 대분류 안의 맨 위 ▲, 맨 아래 ▼ 비활성 (US5-6) |

## 5. 블로그 관리 화면이 읽는 데이터

| 함수 (`src/server/blog.ts`) | 돌려주는 것 |
|---|---|
| `getBlogByOwner(나)` | 지금 이름·소개·주소 |
| `getCategories(블로그, true)` | 대분류(`id`, `name`, `position`, 글 수) 배열, 각 대분류에 `subcategories`(`id`, `name`, `position`, 글 수) 배열. 정렬 `position` → `id`. 글 수는 비공개 포함. 소분류 글 수는 post 단계 3 뒤부터 |
| `getBlogVisitDays`, `getBlogVisitStats` | 방문자 카드 (바뀌지 않음) |

## 6. 조작 요청에 대한 약속 (SC-008·009)

| 조작 | 결과 |
|---|---|
| 폼에 `blogId`·`ownerId`·`slug` 같은 다른 칸을 섞어 이름·소개 저장 | 이름·소개만, 내 블로그만 바뀐다 (US3-7) |
| 다른 블로그의 대분류·소분류 ID로 이름 바꾸기 | `잘못된 요청이에요`, 아무것도 안 바뀜 |
| 다른 블로그의 대분류·소분류 ID로 삭제·순서·소분류 추가 | 문구 없이 끝, 아무것도 안 바뀜 |
| ID `99999999999`, `abc`, `1.5`, 방향 `"x"` | 500 없음, 위와 같음 (US5-10) |
| 로그인 없이 Server Action 호출 | `/`로 리다이렉트, 아무것도 안 바뀜 |
| 다른 사이트에서 보낸 Server Action | Next.js가 거부 |
