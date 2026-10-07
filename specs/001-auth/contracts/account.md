# Contract: 내 정보 — 소셜 연동·해제, 회원 탈퇴

**관련**: FR-036~FR-042, FR-050~FR-052, FR-054 / User Story 4, 7 / [data-model.md](../data-model.md) / [research.md R10·R11](../research.md#r10-소셜-연동해제-fr-036fr-042-user-story-4)

공통 약속:

- 모든 Server Action은 `requireMember()`로 로그인한 본인만 쓴다. 대상은 늘 "로그인한 나"이고 다른 회원 ID를 입력으로 받지 않는다 (FR-042).
- Server Action은 Next.js의 Origin 확인을 받는다. 다른 사이트에서 보낸 해제·탈퇴 요청은 처리되지 않는다 (FR-024, SC-009).
- **(spec에 없음 — 제안)** 은 plan의 남은 문제에 있는 문구다. 구현 전에 spec에서 확정한다.

---

## 1. 화면 라우트 `/settings/account` (새)

| 항목 | 내용 |
|---|---|
| 파일 | `src/app/settings/account/page.tsx`, `actions.ts`, `account-forms.tsx` (클라이언트 폼) |
| 접근 | `requireMember()`. 비로그인·세션 만료 → `redirect("/")` (FR-022, FR-042). 관리자도 같은 화면 |
| 들어오는 길 | 블로그 관리 `/settings/blog`의 `내 정보` 링크 (blog 소유 파일에 한 줄 추가), 헤더의 캐릭터 배지 (town 소유 파일에 추가) (FR-036). TOWN-10 상태창이 생긴 뒤의 입구는 town과 정한다 (town FR-054는 상태창을 누르면 내 블로그 홈으로 간다, plan 남은 문제 15) |
| 탭 제목 | `내 정보` |
| 접근성 | 375px 가로 스크롤 없음, 누르는 영역 44×44px, 버튼 글자 한 줄 (FR-054, SC-011) |

**화면 구성**

| 구역 | 내용 | 근거 |
|---|---|---|
| 로그인 수단 — 아이디 | `아이디 로그인 · {아이디}`. 해제 버튼 없음 | FR-041 |
| 로그인 수단 — 카카오·네이버·Google | 연동됨: `연동됨 ({연동한 날짜})` + [연동 해제]. 아님: [{서비스} 연동하기]. 그 서비스 키가 없으면 [{서비스} 연동하기]를 흐리게 비활성 + `아직 연결 준비 중이에요` (spec에 없음 — 첫 화면 FR-031을 따른 제안) | FR-036, FR-039 |
| 닉네임 | blog가 칸과 Server Action을 끼운다 (BLOG-03 FR-019). auth는 자리만 둔다 | 범위 밖 (spec) |
| 회원 탈퇴 | 지워지는 것 안내, 비밀번호 칸, 버튼 (문구는 4장) | FR-050 |

- 연동한 날짜: `accounts.created_at`을 `formatDate()`(`src/lib/format.ts`)로.
- 서비스 이름: `kakao` → `카카오`, `naver` → `네이버`, `google` → `Google` (첫 화면 버튼 이름 FR-030과 같게. 표기는 남은 문제).

**searchParams (돌아왔을 때)**

| 값 | 보이는 문구 | 근거 |
|---|---|---|
| `linked={서비스}`이고 실제로 그 서비스 연동 행이 있음 | `{서비스} 계정을 연동했어요` (예: `카카오 계정을 연동했어요`) | FR-037 |
| `provider={서비스}&error=<다른 회원 연동 코드>` **(라이브러리 코드 확인, research 확인 목록 8)** | `이미 다른 Blogville 계정에 연동된 {서비스} 계정이에요` | FR-038 |
| `provider={서비스}&error=<그 밖>` (취소·동의 안 함) | 없음. 아무것도 바뀌지 않음 | spec Edge Case |
| `linked=…`인데 실제 연동 행이 없음 (주소를 손으로 만든 경우) | 없음 | |

---

## 2. Server Action `startLinkSocial(provider)`

| 항목 | 내용 |
|---|---|
| 권한 | `requireMember()` |
| 입력 | `provider`: `"kakao" \| "naver" \| "google"` |
| 아무것도 하지 않음 (결과 `{}`) | `provider`가 목록 밖 / 키가 없는 서비스 / 이미 그 서비스를 연동함. 화면에서는 버튼이 `연동됨`이거나 비활성이라 누를 수 없다 (FR-039 "요청을 조작해 보내도 거부") |
| 처리 | `auth.api.linkSocialAccount({ body: { provider, callbackURL: "/settings/account?linked={provider}", errorCallbackURL: "/settings/account?provider={provider}" } })` **(서버 호출 이름 확인, research 확인 목록 9)** |
| 결과 | `{ url: string }` → 화면이 소셜 서비스로 이동 → 로그인·동의 → 콜백 → `/settings/account?linked=…` |
| DB 보장 | `accounts` UNIQUE (`user_id`, `provider_id`): 두 탭에서 동시에 연동해도 서비스마다 1개. UNIQUE (`provider_id`, `account_id`): 한 소셜 계정은 한 회원에만 |

연동 뒤 로그아웃하고 첫 화면의 그 서비스 버튼을 누르면 같은 회원으로 로그인된다 (FR-037, SC-012).

---

## 3. Server Action `unlinkSocial(provider)`

| 항목 | 내용 |
|---|---|
| 권한 | `requireMember()` |
| 입력 | `provider`: `"kakao" \| "naver" \| "google"`만 받는다 |
| 거부 (아무것도 지우지 않음) | `"credential"`이나 그 밖의 값 (FR-041) |
| 화면 확인 창 | `{서비스} 연동을 해제할까요? 아이디 로그인은 그대로 쓸 수 있어요` (spec Assumption) |
| 처리 | `accounts`에서 `user_id = 나 AND provider_id = {provider}` 행 삭제 → `revalidatePath("/settings/account")` |
| 결과 | 그 줄이 [{서비스} 연동하기]로 바뀐다. 따로 성공 문구는 두지 않는다 (spec에 없음) |
| 연동하지 않은 서비스 | 0행 삭제, 변화 없음 |

해제 뒤 그 소셜 계정으로 로그인하면 연동 없음 문구, 아이디 로그인은 그대로 된다 (FR-040).

---

## 4. Server Action `deleteAccount(prev, formData)` (9단계)

| 항목 | 내용 |
|---|---|
| 권한 | `requireMember()` |
| 입력 (FormData) | `password` |
| 화면 | 지워지는 것 안내 (회원·프로필·블로그·글·남의 글에 단 댓글과 답글·코인·경험치 기록·보유 아이템·연동한 소셜 계정 연결), 비밀번호 칸, 버튼 **[회원 탈퇴]** (spec에 없음 — 제안), 처리 중 누를 수 없음 |
| 실패: 비밀번호 비어 있음 | `비밀번호를 적어 주세요` (spec에 없음 — 제안). 아무것도 지우지 않음 |
| 실패: 비밀번호 틀림 | `비밀번호가 맞지 않아요` (spec에 없음 — 제안). 아무것도 지우지 않음 (FR-050, US7 #4) |
| 처리 (성공) | 트랜잭션 하나: `lockUser` → social의 탈퇴용 댓글 정리 (D2) → 실패 기록 삭제 → `users` 행 삭제 (나머지 CASCADE, [data-model.md 3장](../data-model.md#3-탈퇴-때-지워지는-것-fr-051-fr-052)). 커밋 뒤 쿠키 삭제(`auth.api.signOut()`) → `revalidatePath("/", "layout")` → `redirect("/")` |
| 중간 실패 | 아무것도 지워지지 않는다 (FR-051, spec Edge Case) |

**탈퇴 뒤 결과**

| 확인 | 결과 | 근거 |
|---|---|---|
| 그 아이디로 로그인 | `아이디 또는 비밀번호가 맞지 않아요` | US7 #2 |
| 연동했던 소셜 계정으로 로그인 | 연동 없음 문구 | US7 #2 |
| `/@{예전 주소}` | 404 (`길을 잃었어요`) | US7 #3 |
| 남의 글에 단 답글 없는 댓글, 남의 댓글에 단 답글 | 자리 없이 사라짐 | FR-052, US7 #5 |
| 남의 답글이 달린 내 댓글 | 내용·닉네임·캐릭터 없이 `삭제된 댓글이에요` 자리, 남의 답글은 그대로 | FR-052, US7 #6 |
| 화면과 화면으로 전달되는 데이터 | 그 회원의 닉네임과 지운 댓글·답글 원문 0건 | SC-014 |
| 같은 아이디로 다시 가입 | 가능 (그 사이 다른 회원이 그 값을 주소·닉네임으로 쓰면 `이미 있는 아이디예요`) | spec Assumption |

---

## 5. 다른 spec이 이 화면에 끼우는 것

| spec | 무엇 | 방식 |
|---|---|---|
| blog | 닉네임 변경 칸과 Server Action (BLOG-03 FR-019). 이름 겹침 검사는 auth의 `lockName`·`findNameConflict` | 이 페이지에 컴포넌트 한 줄 추가 |
| (담당 없음) | 프로필 사진 올리기 | 만들지 않는다 (공통 맥락 5.3) |
