# Contract: 닉네임·전시 동물·방문 기록 Server Action

**Feature**: `002-blog` | **Spec**: [../spec.md](../spec.md) | **Data model**: [../data-model.md](../data-model.md) | **Research**: [../research.md](../research.md)

공통 규칙(권한, 다른 사이트 요청 거부, `parseId`, 코드 포인트 길이, `FormState`, `revalidatePath("/", "layout")`)은 [blog-settings.md](blog-settings.md) 0절과 같다. *(plan 임시)*는 spec에 없어 plan이 임시로 정한 문구다.

## 1. `updateNickname(prev, formData)` — 닉네임 변경 (새)

- 파일: `src/app/settings/account/nickname-actions.ts` (blog가 새로 만듦), 화면 `src/app/settings/account/nickname-form.tsx`(blog가 새로 만듦) → auth 소유 `src/app/settings/account/page.tsx`(auth가 단계 1 U9에서 새로 만드는 골격)에 끼움.

| 항목 | 내용 |
|---|---|
| 권한 | `requireMember()`. 대상은 로그인한 회원의 프로필(`profiles.user_id = 나`) |
| 입력 | `nickname` (그 밖의 칸은 읽지 않는다) |
| 검증·처리 순서 | 1. 앞뒤 공백 제거 → 2자 미만 `닉네임은 2자 이상이에요` / 21자 이상 `닉네임은 20자까지예요` (코드 포인트)<br>2. 지금 닉네임과 같음 → 저장 없이 `{ ok }`<br>3. 트랜잭션: `lockName(tx, 닉네임)`(auth 모듈, 대소문자 무시 키) → `findNameConflict(tx, 닉네임, { exceptUserId: 나 })`의 `username`이 참(닉네임 소문자 값과 같은 아이디가 다른 회원에게 있음) → `이미 있는 닉네임이에요` → `UPDATE profiles SET nickname WHERE user_id = 나` → `profiles_nickname_unique` 위반(23505) → `이미 있는 닉네임이에요` |
| 성공 | `{ ok }` → [저장] 왼쪽 `저장했어요 ✓` *(plan 임시)*. 다음 열기부터 미니룸 닉네임 배지·블로그 주인 프로필·글 상세 `{블로그 이름} · {닉네임}`·글 카드 `{닉네임} · {블로그 이름}`·광장 집 이름표에 새 닉네임 (FR-020, US3-6) |
| 실패 | `{ error, values: { nickname: 보낸 값 } }` → 빨간 한 줄, 칸에 보낸 값, 닉네임은 그대로 |
| 비교 규칙 | 다른 회원 아이디와는 대소문자 무시(`Tester2` = 아이디 `tester2` → 거부). 자기 아이디와 같은 값(대소문자 달라도)은 본인만 쓸 수 있다. 닉네임끼리는 글자 그대로 — `findNameConflict`의 `nickname` 결과(대소문자 무시)는 쓰지 않고 `profiles_nickname_unique`로만 막는다 (research R-03). 예약어 검사는 하지 않는다 |
| 동시성 | 같은 값으로 가입(아이디)과 닉네임 변경이 동시에 오면 한쪽만 성공 (research R-04). 두 회원이 같은 닉네임으로 동시에 바꾸면 UNIQUE가 한쪽을 막는다 |
| 선행 | auth: `profiles.nickname` CHECK 2~20 (그 전에는 13~20자 저장이 DB에서 거부된다), 이름 잠금 모듈, `/settings/account` 페이지 |

## 2. `setShowcaseAnimal(animalId)` — 전시 동물 고르기·비우기 (새)

- 파일: `src/app/settings/blog/actions.ts`. 화면: 블로그 홈 도감의 주인 전용 버튼 (새 파일 `src/components/blog/animal-collection.tsx`, 클라이언트, `useTransition`).

| 항목 | 내용 |
|---|---|
| 권한 | `requireMember()`. 대상은 로그인한 회원의 블로그 |
| 입력 | `animalId: number \| null` — 숫자면 그 동물을 전시, `null`이면 전시 비우기 |
| 검증 | 값이 정확히 `null`일 때만 비우기로 본다. 그 밖의 값(`undefined`, `"abc"`, `1.5`, 범위 밖 숫자 등)은 모두 `parseId()`를 거쳐 실패하면 아무것도 안 하고 끝 (비우기로 처리하지 않는다) |
| 처리 | 고르기: UPDATE 한 문장 — `blogs.owner_id = 나`이고 `user_animals`에 `id = animalId AND user_id = 나 AND status = 'grown'`이 있을 때만 `showcase_animal_id = animalId`. 비우기: `showcase_animal_id = NULL WHERE owner_id = 나`. DB 복합 FK가 "내 동물만"을 다시 확인한다 |
| 결과 | 반환 없음, 문구 없음. 성공하면 다음 그리기부터 미니룸 옆에 그 동물(늘 최대 1마리), 도감에서 그 카드에 `전시 중` *(plan 임시)* |
| 조작 요청 | 남의 동물 ID, 알·자라는 중인 동물 ID, 없는 ID, `99999999999`, `"abc"` → 아무것도 바뀌지 않는다 (FR-031, US4-9, SC-008) |
| 버튼 *(plan 임시)* | 전시 중이 아닌 카드: [전시하기] / 전시 중인 카드: `전시 중` 표시 + [전시 빼기]. 각 버튼 44×44px 이상 |
| 선행 | town: `user_animals` UNIQUE(`user_id`, `id`), blog: `blogs.showcase_animal_id` 마이그레이션 (data-model B-M3) |

## 3. `recordBlogVisit(blogId)` — 방문 기록 (기존, 바뀌지 않음)

- 파일: `src/app/blog/actions.ts` (이 함수만 blog 소유, 나머지는 social). 호출: `src/components/blog/visit-count.tsx`(블로그 홈), `src/components/blog/record-visit.tsx`(글 상세).

| 항목 | 내용 |
|---|---|
| 권한 | 로그인 불필요 (방문자도 센다). 첫 줄 `requireMember()`가 없는 예외 (`signUp`·`signIn`과 같음) |
| 입력 | `blogId: number` |
| 처리 | `parseId` 실패 → `null`. 없는 블로그 → `null`(아무것도 기록 안 함). 보는 사람이 주인이면 기록하지 않고 숫자만 돌려준다. 쿠키 `bv_visitor`(UUID, httpOnly, SameSite=Lax, 1년, 배포 때 Secure)가 없거나 형식이 틀리면 새로 만든다. `INSERT blog_visits (blog_id, 오늘 한국 날짜, visitor_id) ON CONFLICT DO NOTHING` |
| 결과 | `{ today, yesterday, total } \| null` |
| 보장 | 같은 브라우저·같은 블로그·같은 날 1번 (PK), 회원 정보·IP 저장 없음 (FR-043~049, SC-010, `e2e/visits.mjs`) |

## 4. 화면이 읽는 데이터 (`src/server/blog.ts`)

| 함수 | 바뀌는 점 | 돌려주는 것 |
|---|---|---|
| `getBlogBySlug(slug)` | 칸 추가 | 지금 칸 + `photoKey`(auth의 `profiles.photo_key`), `showcaseAnimalId` |
| `getGrownAnimals(ownerId)` | 새 함수 | 주인의 다 키운 동물 `{ id, name, assetKey, grownAt }[]`, 다 키운 시각 최신순 |
| `searchBlogs(q)` | 새 함수 | 블로그 이름·주인 닉네임 부분 일치 블로그 최대 8곳 `{ slug, title, nickname, characterAsset, photoKey }[]` |
| `listFeed({ page, search })` | 선택 인자 `search` 추가 (post 소유 함수에 추가만) | 지금과 같은 모양, 공개 글 중 제목·본문 부분 일치 |
| `listBlogPosts({ …, subcategoryId })` | 선택 인자 `subcategoryId` — post가 변경 13(단계 3)에서 넣는다. blog는 쓰기만 한다 (먼저 필요하면 같은 모양으로 추가만) | 지금과 같은 모양 (+ post가 더하는 카드의 소분류 이름) |

전시 동물은 `getGrownAnimals` 결과에서 `showcaseAnimalId`로 찾는다. 목록에 없으면(남의 것·다 자라지 않음·지워짐) 전시 자리는 비어 보인다 (spec Edge Cases).
