# Contract: 헤더 유저 상태창과 지붕 색 (TOWN-10, TOWN-07)

**Feature**: `007-town` | 관련: [../research.md](../research.md) R-10·R-11·R-12, [../data-model.md](../data-model.md) 2.5·3.2

## 1. 헤더 (모든 화면, `src/components/site-header.tsx` — town 소유)

### 1.1 배치

| 화면 | 왼쪽 | 오른쪽 | 높이 |
|---|---|---|---|
| 640px 이상, 회원 | [Blogville 로고] [← 광장으로 나가기] | [`Lv.N`] [`🪙 N`] [🔔 (game)] [`👑 관리자` (관리자만)] [상태창] [로그아웃] | 58px |
| 640px 미만, 회원 | 1줄: [로고] [← 나가기] | 1줄: [👑 (관리자만, 접근 이름 `관리자`)] [로그아웃] | 58 + 48 = 106px |
| | 2줄: [상태창 (프로필 + 닉네임)] | 2줄: [`Lv.N`] [`🪙 N`] [🔔] | |
| 방문자 | [로고] [← 나가기] | [시작하기] → `/` | 58px |

- 다른 화면으로 가는 이동 메뉴는 없다 (FR-007, US1-6).
- 나가기 버튼: `/`·`/town`에서는 없고 그 밖의 화면에 있다 ([town-screen.md](town-screen.md) 7절).
- 로고: 늘 보인다(좁은 화면 포함, FR-053). 회원 → `/town`, 방문자 → `/` (지금과 같음, US8-2).
- `Lv.N`(title `레벨`), `🪙 N`(title `코인`, `/wallet` 링크, 천 단위 쉼표), `👑 관리자`(`/admin`), 로그아웃(auth 컴포넌트)은 그대로 둔다 (FR-055). 🔔·레벨업 팝업·날짜 감시는 game이 끼우는 자리 (game `contracts/notifications.md` 1절).
- 누르는 것(나가기, 🪙, 🔔, 👑, 상태창, 로그아웃, 시작하기)은 모두 44×44px 이상. 375px에서 가로 스크롤 없음 (SC-004, FR-057). 한 줄 배치가 시작되는 640px 바로 위 폭에서도 가로 스크롤이 없어야 한다 — 상태창은 폭 상한 + `truncate`, 그래도 넘치면 두 줄 기준을 768px로 올린다 (research R-11).
- CSS 변수: `--header-h`(방문자·640 이상 회원 58px), `--header-h-member`(640px 미만 106px, 이상 58px) — `src/app/globals.css`. 광장 높이가 이 값을 쓴다.

### 1.2 유저 상태창 (`UserStatus`, `src/components/user-status.tsx`, 새)

| 항목 | 계약 |
|---|---|
| 데이터 | `getViewer()`의 프로필: `nickname`, `blogTitle`, `blogSlug`, `characterAsset`, + `photoKey`(town이 추가, auth의 `profiles.photo_key`), + `outfit`(shop이 추가). 헤더가 이미 부르는 같은 쿼리라 요청이 늘지 않는다 |
| 모양 | 둥근 카드. 왼쪽 원형 36px 프로필, 오른쪽 두 줄: 위 닉네임(굵게), 아래 블로그 제목(작은 회색, 길면 `…`) (FR-054) |
| 프로필 | `photoKey`가 있으면 `<img src="/files/{photoKey}" alt="">`(원형, 가득 채움), 없으면 `CharacterBadge`(지금 장착한 캐릭터 얼굴 + 차림) (US8-6) |
| 640px 미만 | 블로그 제목 숨김, 프로필 + 닉네임(길면 `…`) (FR-057, US8-4) |
| 누르면 | `/@{blogSlug}` 내 블로그 홈 (US8-7). 링크 접근 이름 `내 블로그: {블로그 제목}` *(plan 임시)* |
| 갱신 | 닉네임·블로그 제목·캐릭터·사진을 바꾸는 Server Action이 `revalidatePath("/", "layout")`를 부르므로 저장 직후 화면의 헤더에 새 값 (FR-056, SC-010). 각 Action은 blog(닉네임·제목), shop(캐릭터), auth(사진) 소유 |
| 방문자 | 상태창 없음, [시작하기] (US8-5) |

## 2. 지붕 색 (꾸미기 화면 `/closet`)

### 2.1 화면

| 항목 | 계약 |
|---|---|
| 자리 | shop 소유 `src/app/closet/page.tsx`에 town의 서버 컴포넌트 `<RoofColorSection userId={viewer.userId} />` 한 줄 (공통 모듈 추가). 위치는 꾸미기 내용 아래 (shop `contracts/closet.md`의 "(town이 끼움)" 줄) |
| 접근 | `/closet`은 회원만 (`requireMember()`, shop 소유) |
| 제목 | `🏠 지붕 색` |
| 내용 | 작은 집 미리보기(1단계 그림, 지금 고른 색) + 색 8개(빨강 · 주황 · 노랑 · 초록 · 하늘 · 파랑 · 보라 · 갈색, 각각 색 동그라미 + 이름, 44×44px 이상, 고른 색에 테두리·`aria-pressed`) + [배경 색 따라가기] 버튼(고른 색이 없으면 눌린 상태) (FR-039, US10-1·4) |
| 컴포넌트 | `RoofColorSection`(서버, `src/components/town/roof-color-section.tsx`: 내 블로그의 `roof_color`·장착 배경을 읽음) → `RoofColorPicker`(클라이언트, `roof-color-picker.tsx`, `useTransition`) |
| 비용 | 무료. 코인·원장 변화 없음 (spec 기본값) |

### 2.2 Server Action `setRoofColor`

- 파일: `src/app/town/actions.ts`
- 모양: `setRoofColor(color: unknown): Promise<{ ok: true } | { ok: false; error: string }>`

| 순서 | 조건 | 결과 |
|---|---|---|
| 1 | 로그인 안 함 | `requireMember()` → `/` |
| 2 | `color`가 `null`(배경 색 따라가기) 또는 `red`·`orange`·`yellow`·`green`·`sky`·`blue`·`purple`·`brown` 중 하나가 아님 (zod `z.enum(ROOF_COLORS).nullable()`) | `{ ok: false, error: "고를 수 없는 색이에요" }` *(plan 임시)*, DB 그대로 (FR-039, US10-5) |
| 3 | 통과 | `UPDATE blogs SET roof_color = $color WHERE owner_id = 나` (대상은 요청 값이 아니라 로그인한 회원의 블로그) → `{ ok: true }` |
| 4 | DB CHECK `blogs_roof_color_check` 위반 (앱 검사를 건너뛴 경우) | 오류 → 위 2와 같은 문구 |
| 5 | 성공 뒤 | `revalidatePath("/closet")`, `revalidatePath("/town")` |

- 결과: 다음 광장 방문부터 내 집 지붕이 그 색이다. 내 집을 즐겨찾기한 다른 회원의 광장, 방문자 광장도 같은 색이다(서버가 `TownHouse.roof`를 정함, FR-040, US10-3). 배경을 바꿔도 그대로다(US10-2). `null`이면 장착 배경의 강조색(`backgroundAccent`)으로 돌아간다(US10-4).
- 색 코드값 목록 `ROOF_COLORS`는 blog의 `src/lib/blog.ts`, 코드값 → 색·이름은 town의 `src/lib/town.ts` `ROOF_PALETTE`.

## 3. 증축 메뉴 없음 (US11-7)

상점(`/shop`)과 꾸미기(`/closet`)에 코인으로 집을 키우는 메뉴를 만들지 않는다. 집 단계는 주인 레벨로만 정해진다 (FR-059).
