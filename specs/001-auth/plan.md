# Implementation Plan: 회원 / 인증 (AUTH)

**Branch**: `001-auth` | **Date**: 2026-10-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-auth/spec.md`

**Note**: `/speckit-plan`으로 만든 문서다. 기준은 코드 저장소 `main` 커밋 `feb4c05`, `.specify/memory/constitution.md` v1.0.0, 7개 spec 공통 맥락(7개 plan을 함께 쓰며 정한 작업 문서: 테이블 담당·공통 모듈 소유·구현 순서. 이 저장소에는 없다)이다. 이 plan에 필요한 합의는 [data-model.md 1장](data-model.md#1-테이블-한눈에)과 [의존성](#의존성-다른-spec)에 옮겨 적었고, 본문의 "공통 맥락 N.N"은 그 문서의 절 번호다. 코드 위치는 코드 저장소 기준 상대 경로, 문서 위치는 문서 저장소 기준 상대 경로로 적는다.

## Summary

사이트 아이디로만 가입하고, 가입 한 번으로 회원·로그인 수단·프로필·블로그·기본 아이템·"일상" 카테고리·가입 축하 🪙 100을 **한 트랜잭션**으로 만든다. 그래서 온보딩 화면과 "프로필 없는 회원" 상태를 없앤다 (FR-006, FR-007). 예약어·이름 겹침은 새 공용 모듈(`src/lib/names.ts`, `src/server/names.ts`)이 이름 단위 잠금 안에서 검사하고, blog의 주소·닉네임 변경도 같은 모듈을 쓴다 (D1).

로그인은 [로그인 상태 유지]에 따라 서버 기준 2시간(마지막 사용부터, 브라우저를 닫으면 끝) 또는 7일(쓰는 동안 연장)로 유지한다. `sessions.remember_me` 칸을 두고, 짧은 쪽(마지막 사용 `updated_at` 확인과 연장)은 모든 화면이 지나는 `getSession()`이, 긴 쪽은 라이브러리 기본 연장이 맡는다. 아이디별 5번 연속 실패 → 5분 잠금은 새 표 `login_attempts`에 비밀번호 확인 전 시도를 아이디 단위 잠금으로 예약해 서버에서 센다.

소셜 계정은 가입이 아니라 **연동**으로만 쓴다. 라이브러리의 소셜 가입을 끄고, 새 화면 `/settings/account`(내 정보)에서 서비스마다 하나씩 연동·해제한다. 라이브러리 HTTP 경로는 허용 목록(세션 확인·OAuth 콜백)만 남기고 나머지 인증 동작은 모두 우리 Server Action으로 옮겨, 가입·로그인 규칙을 우회하는 길을 닫는다 (constitution IV). 회원 탈퇴는 비밀번호를 다시 확인한 뒤 한 트랜잭션에서 지우고, 남의 답글이 달린 댓글 처리는 social이 정한 구조를 따른다 (D2). 관리자 화면은 온보딩 흔적을 지우고 비로그인도 404로 숨긴다.

구현은 세 단계다: **1단계 가입 통합**(모든 spec의 e2e 전제), **8단계 로그인 유지·시도 제한·내 정보·연동·관리자 정리**, **9단계 회원 탈퇴**(social 답글 분리와 game 알림 뒤). 근거는 [research.md](research.md), 표 설계는 [data-model.md](data-model.md), 바깥 인터페이스는 [contracts/](contracts/), 검증 순서는 [quickstart.md](quickstart.md).

## Technical Context

**Language/Version**: TypeScript ^5 (`strict: true`, 별칭 `@/*` → `src/*`), Node.js 20.9 이상 (README), React 19.2.8

**Primary Dependencies**: Next.js 16.3.8 (App Router, Server Action, Route Handler), better-auth ^1.7.7 (drizzle 어댑터 `usePlural`, `username` 플러그인, `nextCookies`, 소셜 google·kakao·naver), drizzle-orm ^0.45.3 / drizzle-kit ^0.31.11, zod ^4.6.5, pg ^8.23.1, Tailwind CSS 4. **새 패키지 없음**.

**Storage**: PostgreSQL (README 안내 17). 인증 표 `users`·`accounts`·`sessions`·`verifications`(라이브러리 구조, 컬럼 이름 유지) + `profiles` + 새 `login_attempts`. 가입이 `blogs`·`categories`·`user_items`·`point_ledger`에 행을 만든다. 첨부 파일은 디스크(`UPLOAD_DIR`).

**Testing**: `npx tsc --noEmit`, `npx eslint`, `npm test`(tsx 스크립트, 새 `test:auth`), Playwright `chromium`을 직접 쓰는 Node 스크립트 `e2e/*.mjs`(새 4개: `signup`, `session`, `login-limit`, `account` / 고침 2개: `helpers`, `auth`). CI가 없어 PR 작성자가 직접 돌리고 결과를 PR에 적는다 (NF-23).

**Target Platform**: Node.js 서버 (`next start`, 배포처 미정 NF-08), 로컬 `http://localhost:3000`. 브라우저: 최신 Chrome·Edge·Safari, 모바일 Chrome·Safari (NF-21).

**Project Type**: 웹 애플리케이션 (Next.js 하나에 화면과 서버. 프런트·백엔드 분리 없음)

**Performance Goals**: SC-001 가입 시작 → 광장 도착 1분 이내, 입력 4칸. (plan 목표) 가입·로그인 Server Action의 서버 처리 1초 이내(비밀번호 해시 포함, 로컬). 세션 연장 쓰기는 세션마다 5분에 1번 이하, 요청당 UPDATE 1번 이하. constitution 응답 기준(글 목록 1초)에 부담을 더하지 않는다.

**Constraints**: 권한·검증은 서버에서 (IV). 라이브러리 HTTP는 허용 목록만. 비밀값은 환경 변수(이름만 문서화). 마이그레이션 번호를 박지 않는다 (공통 맥락 3.6). 공용 파일은 소유 spec 규칙을 따른다 (공통 맥락 5.2). 이 체크아웃에 `node_modules`가 없어 Better Auth 1.7·Next.js 16의 세부 동작은 [research.md 구현 전 확인 목록](research.md#구현-전-확인-목록)으로 남겼다.

**Scale/Scope**: 3명 팀, 배포 전. 회원 수 수십~수백 (추측). 화면 3개(`/` 고침, `/settings/account` 새, `/admin` 고침), Server Action 8개(`signUp`, `signIn`, `startSocialSignIn`, `signOut`, `startLinkSocial`, `unlinkSocial`, `deleteAccount`, `adminDeletePost`), Route Handler 1개(허용 목록), 마이그레이션 6개, 새 표 1개, 새 칸 2개.

모든 기술 질문은 research.md에서 결정했다. `NEEDS CLARIFICATION`은 없다. spec 문구가 없어 막히는 것은 [남은 문제](#남은-문제-open-items)에 있다.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 원칙 | 설계 전 | 근거 (이 plan의 설계) | 설계 후 재평가 |
|---|---|---|---|
| I. 글쓰기가 먼저 | 통과 | 온보딩을 없애 가입 → 광장 → 첫 글까지 단계가 줄어든다 (SC-001). 새 게임 요소가 없다. 가입 축하 🪙 100은 기존 규칙이고 유료 결제가 없다 | 통과 |
| II. 요구사항 ID | 통과 | 모든 변경 단위가 AUTH·NF ID와 FR 번호를 가리킨다. 마이그레이션 SQL·코드 주석에 ID를 단다 (관례). ID를 바꾸지 않는다 | 통과 |
| III. 확인할 수 있는 수용 기준 | 통과 (조건) | 수용 시나리오 50개와 SC 14개를 quickstart의 단위 테스트·e2e 행에 연결했다. 문구는 spec 그대로 쓴다. spec에 없는 문구(탈퇴 오류·버튼, 캐릭터 거부, 내 정보 세부)는 확정하지 않고 "제안"으로 표시해 남은 문제에 올렸다 | 통과. 단 남은 문제 1~3의 문구를 spec에서 확정한 뒤 해당 화면을 구현한다 |
| IV. 권한과 검증은 서버에서 (NON-NEGOTIABLE) | 통과 | 아이디 형식·예약어·이름 겹침·기본 캐릭터를 서버에서 검사, 가입 요청의 `role` 무시. 라이브러리 HTTP 허용 목록으로 가입·로그인·사용자 수정·연동·해제 우회 경로를 닫음 (research R5). 관리자 화면·동작은 비로그인 포함 404. 연동·해제·탈퇴는 `requireMember()` 본인만. 쿼리는 Drizzle 값 바인딩. 비밀값은 환경 변수 | 통과 |
| V. 원장·트랜잭션·DB 제약 (NON-NEGOTIABLE) | 통과 | 가입 코인은 `lockUser` + `grantReward`로 원장에. 가입·탈퇴는 각각 한 트랜잭션. DB 제약: `users.username` NOT NULL·형식 CHECK·UNIQUE, `accounts` UNIQUE 2개, `login_attempts` PK, 닉네임 CHECK. 동시 요청: 이름 잠금(가입 ↔ 주소·닉네임 변경), 아이디별 로그인 시도 예약(잠금을 쥔 트랜잭션 안에서 다른 DB 연결을 쓰지 않음). 구조 변경은 마이그레이션 파일로만 | 통과 |
| VI. 모바일에서도 | 통과 | 첫 화면·내 정보·관리자 화면 375px 가로 스크롤 없음(관리자 표 → 휴대폰 폭에서 쌓인 목록), 누르는 영역 44×44px(로그아웃 버튼 포함), 버튼 글자 한 줄. `e2e/nonfunctional.mjs`·`e2e/auth.mjs`가 확인 | 통과 |
| VII. 단순하게, 최소한의 정보 | 통과 | 이메일을 받지 않음(Google도 `openid profile`), 소셜 토큰 저장 안 함, 실패 기록은 아이디 문자열·횟수·시각만(IP 없음), 탈퇴는 즉시 삭제·원문을 남기지 않음. 새 표 1개·새 칸 2개, 새 패키지 없음 | 통과 |
| 보안: 로그인 쿠키 | 통과 | `HttpOnly`, `SameSite=Lax`, 배포(`https`) 때 `Secure`. e2e가 속성 확인 | 통과 |
| 보안: CSRF 거부(403) | 확인 필요 | 상태 변경은 모두 Server Action(Next.js Origin 확인) + `SameSite=Lax`. Next.js가 돌려주는 상태 코드가 403인지는 구현 전 확인 (research 확인 목록 12) | 통과. 검증 기준은 "성공 아님 + 데이터 변경 0". 상태 코드가 403이 아니면 남은 문제로 올린다 |
| 보안: 비밀번호 해시만 | 통과 | `hashPassword`, 원문 저장·출력 없음. SC-010 확인 (덤프·서버 로그 검색) | 통과 |
| 보안: 로그인 유지 2시간 / 7일 | 통과 | research R6. 2시간은 서버의 `sessions.remember_me`와 `updated_at`이 기준이라 쿠키(`dont_remember`)를 지워도 늘어나지 않는다 | 통과 |
| 보안: 실패 문구가 존재를 숨김 | 통과 | 없는 아이디·틀린 비밀번호·잠금이 모두 같은 문구 (SC-004) | 통과 |
| 개발 흐름·품질 관문 | 통과 | 변경 단위마다 tsc·eslint·npm test·e2e를 돌리고 결과를 PR에 적는다 (quickstart) | 통과 |

**결과**: 위반 없음. Complexity Tracking은 비워 둔다.

## Project Structure

### Documentation (this feature)

```text
specs/001-auth/
├── spec.md                  # 기능 명세 (고치지 않음)
├── checklists/
│   └── requirements.md      # spec 품질 점검
├── plan.md                  # 이 파일
├── research.md              # Phase 0: 기술 결정과 근거, 구현 전 확인 목록
├── data-model.md            # Phase 1: 테이블 현재/목표, 변경·참조, 마이그레이션 순서
├── quickstart.md            # Phase 1: 검증 실행 순서와 기대 결과
├── contracts/               # Phase 1: 바깥 인터페이스
│   ├── auth-entry.md        # 첫 화면, signUp·signIn·startSocialSignIn·signOut, /api/auth 허용 목록, 쿠키
│   ├── account.md           # /settings/account, startLinkSocial·unlinkSocial·deleteAccount
│   └── admin.md             # /admin, adminDeletePost, npm run admin:create
```

`tasks.md`는 다음 단계 `/speckit-tasks`가 만든다 (지금은 없다).

### Source Code (repository root)

코드 저장소에서 이 기능이 만지는 파일만 적었다. `[새]` 새 파일, `[고침]` auth 소유 파일 수정, `[삭제]`, `[추가: 소유 spec]` 남의 파일에 한두 줄 추가 (공통 맥락 5.2), `[정리: 소유 spec 합의]` 없어지는 라우트를 가리키는 줄 삭제.

```text
src/
├── app/
│   ├── page.tsx                          [고침] 로그인 회원 → /town, 기본 캐릭터 목록·소셜 오류 문구 전달
│   ├── (auth)/actions.ts                 [고침] signUp(가입 트랜잭션), signIn(유지·시도 제한), [새 함수] startSocialSignIn, signOut
│   ├── onboarding/                       [삭제] page.tsx, actions.ts, onboarding-form.tsx
│   ├── settings/
│   │   ├── account/                      [새] 내 정보
│   │   │   ├── page.tsx                  로그인 수단·연동 상태, (blog 닉네임 자리), 회원 탈퇴
│   │   │   ├── actions.ts                startLinkSocial, unlinkSocial, deleteAccount
│   │   │   └── account-forms.tsx         클라이언트 폼 (해제 확인 창, 탈퇴 비밀번호)
│   │   └── blog/page.tsx                 [추가: blog] 내 정보 링크 한 줄
│   ├── admin/
│   │   ├── page.tsx                      [고침] 카드 5개, 로그인 방식 이름, 휴대폰 목록, 댓글 수(+replies)
│   │   ├── actions.ts                    (그대로. requireAdmin 변경의 영향만)
│   │   └── delete-button.tsx             [고침] 누르는 영역 44px (동작은 그대로)
│   ├── api/auth/[...all]/route.ts        [고침] 허용 목록만 통과, 나머지 404
│   └── town/page.tsx                     [정리: town 합의] 온보딩 redirect 한 줄 삭제
├── components/
│   ├── login-buttons.tsx                 [고침] 캐릭터 고르기, [로그인 상태 유지], 소셜 → Server Action, 안내 문구, 44px
│   ├── sign-out-button.tsx               [고침] Server Action 폼, 누르는 영역 44px
│   ├── session-keeper.tsx                [새] 유지 세션의 7일 연장 (GET /api/auth/get-session). 유지 세션일 때만 그린다
│   ├── site-header.tsx                   [추가: town] SessionKeeper, 캐릭터 배지 → /settings/account 링크
│   └── exit-button.tsx                   [정리: town 합의] HIDDEN_ON에서 /onboarding 삭제
├── db/schema.ts                          [고침] users·accounts·sessions·profiles 블록, loginAttempts [새]
├── lib/
│   ├── auth.ts                           [고침] disableSignUp(이메일·소셜), session 설정·additionalFields, databaseHooks, hooks.after, 소셜 범위·연결 설정
│   ├── auth-client.ts                    (그대로. SessionKeeper가 씀)
│   ├── auth-id.ts                        [새] newAuthId() 32자 (가입·관리자 스크립트 공용)
│   ├── names.ts                          [새] RESERVED_NAMES 16개, normalizeName, isReservedName, USERNAME_RE
│   └── login-limit.ts                    [새] 5번·5분 규칙과 다음 상태 계산 (순수 함수)
└── server/
    ├── dal.ts                            [고침] requireUser 삭제, getViewer(프로필 필수 + sessionId), requireAdmin 404, getSession 2시간 규칙(마지막 사용 확인 + 연장)
    ├── names.ts                          [새] lockName, findNameConflict (blog도 사용)
    ├── signup.ts                         [새] createMember: 가입 트랜잭션
    ├── login-attempts.ts                 [새] 아이디별 실패 기록 (잠금 확인·기록·초기화)
    └── account.ts                        [새] 연동 목록 조회, 탈퇴 트랜잭션
scripts/
├── create-admin.ts                       [고침] 12~64자, 아이디 형식, 아이디와 같은 비밀번호 거부, auth-id 공용
├── reset-dev.ts                          [고침: 공통] login_attempts도 비움 (FK가 없어 CASCADE로 안 비워짐. 표가 있을 때만, data-model 2.4)
└── test-auth.ts                          [새] npm run test:auth
drizzle/                                  [새] 마이그레이션 6개 (data-model.md 4장, 번호는 구현 때)
e2e/
├── helpers.mjs                           [고침] loginDev: 가입 폼에서 캐릭터 고르고 바로 광장
├── auth.mjs                              [고침] 온보딩 확인 삭제, 비로그인 /admin 404, 관리자 375px
├── signup.mjs                            [새]
├── session.mjs                           [새]
├── login-limit.mjs                       [새]
├── account.mjs                           [새]
└── nonfunctional.mjs                     [추가: 공통] /settings/account, 비로그인 /
docs/02-erd.md                            [고침] auth 담당 절 (data-model.md 5장)
package.json                              [추가: 공통] test:auth, test 체인 끝
CLAUDE.md, README.md                      [고침] 로그인·온보딩 규칙(requireUser 삭제, 허용 목록), 기능 표, 스크립트 표
```

**Structure Decision**: 지금의 단일 Next.js 저장소 구조를 그대로 쓴다. 화면·Server Action·Route Handler는 `src/app/`, DB를 쓰는 서버 전용 처리는 `src/server/`(`import "server-only"`), DB 없이 화면·서버·테스트가 같이 쓰는 규칙은 `src/lib/`(순수 함수는 `scripts/test-*.ts`로 시험)이라는 기존 역할을 따른다. 새 서버 모듈(`names`, `signup`, `login-attempts`, `account`)은 auth 영역 파일이고, blog는 `names`를 가져다 쓴다.

## 현재 코드 → spec 목표 변경 단위

단계 번호는 7개 spec 공통 구현 순서(1 → … → 8 → 9)다. 1단계는 다른 모든 spec의 e2e 전제라 가장 먼저 한다.

| # | 단계 | 구분 | 지금 → 목표 | 위치 | 요구사항 | 검증 |
|---|---|---|---|---|---|---|
| U1 | 1 | 마이그레이션 | 가입만 하고 온보딩 전인 회원이 남아 있음 → 정리 (A1) | `drizzle/` (직접 쓴 SQL) | FR-007 | 적용 뒤 프로필 없는 회원 0 |
| U2 | 1 | 마이그레이션 | `users.username` NULL 허용·형식 CHECK 없음, 닉네임 2~12 → NOT NULL + CHECK, 닉네임 2~20 (A2) | `src/db/schema.ts`, `drizzle/` | FR-002, FR-003, FR-009 | signup.mjs 2·4 |
| U3 | 1 | 마이그레이션 | 프로필 사진 칸 없음 → `profiles.photo_key` + FK `SET NULL` (A3, 요청: blog·town·post) | `src/db/schema.ts`, `drizzle/` | ERD 7장 3 | db:migrate |
| U4 | 1 | 서버 | 예약어 11개가 온보딩 안에, 주소만 검사 → 공용 모듈(16개), 가입 아이디와 다른 회원 주소·닉네임 비교, 이름 잠금 | `src/lib/names.ts`, `src/server/names.ts` | FR-003, FR-009, FR-010 | test:auth, signup.mjs 7~9 |
| U5 | 1 | 서버 | `signUpEmail` + 별도 온보딩 트랜잭션 → `createMember` 한 트랜잭션 + 커밋 뒤 `signInUsername` | `src/server/signup.ts`, `src/app/(auth)/actions.ts`, `src/lib/auth-id.ts` | FR-006, FR-008, FR-011, FR-012 | signup.mjs 2·10·11 |
| U6 | 1 | 서버 | 라이브러리 HTTP 모두 열림, 이메일·소셜 가입 가능 → 허용 목록(`sign-in/social` 임시 허용), `disableSignUp`(이메일·소셜), 로그아웃 Server Action | `src/app/api/auth/[...all]/route.ts`, `src/lib/auth.ts`, `src/app/(auth)/actions.ts` | FR-007, FR-011, FR-013, FR-024, FR-028 | signup.mjs 15, session.mjs 12 |
| U7 | 1 | 서버 | `requireUser`·온보딩 redirect, 비로그인 `/admin` → `/` → `requireMember`만, 프로필 필수, `sessionId`, `requireAdmin` 404 | `src/server/dal.ts` | FR-007, FR-022, FR-043 | auth.mjs 5·6 |
| U8 | 1 | 화면 | 가입 폼 3칸, 가입 뒤 온보딩 → 캐릭터 2개 포함 4칸, 가입 뒤 광장. 로그인 회원 `/` → 광장. 로그아웃 버튼 44px | `src/components/login-buttons.tsx`, `src/app/page.tsx`, `src/components/sign-out-button.tsx` | FR-001, FR-005, FR-014, FR-029, FR-054 | signup.mjs 1·6·12·13·16 |
| U9 | 1 | 화면 | 내 정보 화면 없음 → `/settings/account` 골격(`requireMember`, 제목, 닉네임 자리). blog 2단계가 닉네임 칸을 올릴 수 있게 먼저 만든다 | `src/app/settings/account/page.tsx` | FR-036 | account.mjs 1 |
| U10 | 1 | 삭제·정리 | `/onboarding` 화면, 광장·나가기 버튼의 온보딩 참조, 관리자 `주민 (온보딩 완료)`·`온보딩 전` → 없음 | `src/app/onboarding/*`, `src/app/town/page.tsx`·`src/components/exit-button.tsx`(town 합의), `src/app/admin/page.tsx` | FR-007, FR-044, FR-045 | auth.mjs 1·7 |
| U11 | 1 | 스크립트 | 관리자 비밀번호 8자, 아이디 형식 확인 없음 → 12~64자, 형식, 아이디와 다름, `newAuthId` 공용 | `scripts/create-admin.ts` | FR-048, FR-049 | quickstart 4.6 |
| U12 | 1 | 검증·문서 | 온보딩을 기다리는 `loginDev`·`auth.mjs` → 가입 폼 캐릭터 + 광장. `test:auth`(이름 규칙), `signup.mjs`. README 표, CLAUDE.md 로그인 규칙, ERD 3.1·3.2·7장 | `e2e/helpers.mjs`, `e2e/auth.mjs`, `e2e/signup.mjs`, `scripts/test-auth.ts`, `package.json`, 문서 | - | quickstart 2, 4.1, 4.2, 회귀 e2e |
| U13 | 8 | 마이그레이션 | 세션 구분 값·실패 기록 없음 → `sessions.remember_me`, `login_attempts` (A4) | `src/db/schema.ts`, `drizzle/` | FR-019~FR-021, FR-025 | session.mjs, login-limit.mjs |
| U14 | 8 | 마이그레이션 | 소셜 토큰 저장, 서비스마다 1개 보장 없음 → 토큰 비우기(A5), UNIQUE(`user_id`, `provider_id`) (A6) | `drizzle/` | FR-035, FR-039 | account.mjs 6, quickstart 5 |
| U15 | 8 | 서버 | 세션 설정 없음(라이브러리 7일) → 유지 안 함 2시간(`getSession`이 `updated_at` 2시간 지난 세션을 끝내고, 아니면 5분 단위 연장, `disableRefresh`), 유지 7일(라이브러리 연장 + 유지 세션에만 그리는 `SessionKeeper`), 세션 생성 훅 | `src/lib/auth.ts`, `src/server/dal.ts`, `src/components/session-keeper.tsx`, `src/components/site-header.tsx`(town에 추가) | FR-019~FR-023, SC-005 | session.mjs |
| U16 | 8 | 서버 | 시도 제한 없음 → 아이디별 5번·5분. 비밀번호 확인 전에 아이디 단위 잠금으로 시도를 예약(짧은 트랜잭션)하고, 라이브러리 로그인은 트랜잭션 밖에서 부른다(연결 풀 교착 방지, research R8). 성공 시 초기화, `signIn`에 `rememberMe` | `src/server/login-attempts.ts`, `src/lib/login-limit.ts`, `src/app/(auth)/actions.ts`, `scripts/reset-dev.ts` | FR-015~FR-018, FR-025~FR-028, SC-004, SC-006 | login-limit.mjs, test:auth |
| U17 | 8 | 서버 | 소셜: 이메일 요청·토큰 저장·클라이언트에서 직접 로그인 → 이메일 안 받음(Google `email` 범위를 빼므로 Google에도 대체 이메일 `mapProfileToUser`를 더함, research R9), 토큰 안 저장, 자동 연결 끔, `startSocialSignIn` + `bv_remember`, 허용 목록에서 `sign-in/social` 제거 | `src/lib/auth.ts`, `src/app/(auth)/actions.ts`, `src/components/login-buttons.tsx`, route.ts | FR-030~FR-035 | signup.mjs 1, quickstart 6 |
| U18 | 8 | 화면 | 연동·해제 없음 → 내 정보에 서비스별 상태, `startLinkSocial`·`unlinkSocial`, 결과 문구. 블로그 관리·헤더에서 들어오는 링크 | `src/app/settings/account/*`, `src/server/account.ts`, `src/app/settings/blog/page.tsx`(blog에 추가), `src/components/site-header.tsx`(town에 추가) | FR-036~FR-042 | account.mjs 1~11, quickstart 6 |
| U19 | 8 | 화면 | 관리자 표가 375px에서 가로 스크롤, [삭제]가 글자만이라 누르는 영역이 작음, 로그인 방식이 원래 값, 댓글 수가 `comments`만 → 휴대폰 목록, [삭제]·링크 44px, `아이디`/`카카오`…, `comments + replies` (social 5단계 뒤) | `src/app/admin/page.tsx`, `src/app/admin/delete-button.tsx` | FR-044~FR-047, FR-054 | auth.mjs 7·8·9 |
| U20 | 8 | 검증·문서 | `session.mjs`, `login-limit.mjs`, `account.mjs`(연동), `nonfunctional.mjs` 페이지 추가, README·CLAUDE.md, ERD 3.3·새 절 | `e2e/*`, 문서 | - | quickstart 4.3~4.5, 4.7 |
| U21 | 9 | 서버·화면 | 탈퇴 없음 → 비밀번호 확인 + 한 트랜잭션 삭제(social 댓글 정리 → 실패 기록 → 회원 CASCADE) + 쿠키 삭제 | `src/app/settings/account/actions.ts`, `src/server/account.ts`, `account-forms.tsx` | FR-050~FR-052, SC-014 | account.mjs 12~20 |
| U22 | 9 | 검증·문서 | `account.mjs` 탈퇴 부분, ERD 3.14 삭제 규칙 | `e2e/account.mjs`, `docs/02-erd.md` | - | quickstart 4.5 |

### 다른 spec이 쓰는 공용 인터페이스 (auth 제공)

| 위치 | 이름 | 모양 | 쓰는 spec |
|---|---|---|---|
| `src/lib/names.ts` | `RESERVED_NAMES` | 16개 문자열 집합 (값은 blog FR-009 소유. 값을 바꾸는 것은 blog) | blog |
| | `normalizeName(raw)`, `isReservedName(name)`, `USERNAME_RE` | 앞뒤 공백 제거·소문자 / 예약어 여부 / `^[a-z0-9_]{4,20}$` | blog |
| `src/server/names.ts` | `lockName(tx, name)` | 트랜잭션 안에서 이름 단위 잠금 (대소문자 무시) | blog (주소·닉네임 변경) |
| | `findNameConflict(tx, name, { exceptUserId })` | `{ username, slug, nickname }` — 다른 회원의 아이디·주소·닉네임이 대소문자 무시로 같은지 | blog |
| `src/server/dal.ts` | `getViewer()` | `null` 또는 `{ userId, user, sessionId, profile: { nickname, characterAsset, blogId, blogSlug, blogTitle } }` (`profile`은 늘 있음) | 전체 (다른 spec은 필드 추가만) |
| | `requireMember()` | 비로그인 → `/`, 반환은 `getViewer()`와 같음 | 전체 회원 화면·Server Action |
| | `requireAdmin()` | 비로그인·일반 회원 → `notFound()` | auth, social(관리자 댓글 삭제) |
| `src/lib/auth-id.ts` | `newAuthId()` | 32자 영문 대소문자·숫자 | auth (가입, 관리자 스크립트) |

auth가 받기로 한 인터페이스 (social 제공 예정): `removeAuthorComments(tx, userId)` (가칭, `src/server/social.ts`) — 탈퇴 트랜잭션 안에서 D2 규칙대로 그 회원의 댓글·답글을 정리한다.

## 의존성 (다른 spec)

**auth가 먼저 해 줘야 하는 것 (다른 spec이 기다림)**

| 상대 | 무엇 | auth 단계 |
|---|---|---|
| 전체 | 가입 통합과 새 `loginDev`. 모든 e2e의 전제 | 1 |
| blog | 이름 모듈(`lockName`, `findNameConflict`, `RESERVED_NAMES`)과 내 정보 페이지 골격(닉네임 칸 자리) | 1 (U4, U9) |
| post | `profiles.photo_key` — 첨부 정리 작업이 프로필 사진을 제외할 때 쓴다 (post FR-059) | 1 (U3) |
| game | `getViewer().sessionId` — 자동 출석이 "그날 처음 만든 세션"을 가리킬 때 쓴다 | 1 (U7) |
| blog, town | `profiles.photo_key` 표시 (사진이 없으면 캐릭터 얼굴) | 1 (U3) |

**auth가 기다리거나 합의가 필요한 것**

| 상대 | 무엇 | 필요한 auth 단계 |
|---|---|---|
| town | `src/app/town/page.tsx`의 온보딩 redirect 줄과 `src/components/exit-button.tsx` `HIDDEN_ON`의 `/onboarding` 삭제 합의 (없어지는 라우트 참조 정리) | 1 |
| social | `e2e/social.mjs`의 "온보딩 전 회원" 준비 블록 교체. 새 구조에서는 블로그 없는 회원을 만들 수 없어 가입 뒤 멈춘다 (1단계와 같은 시기) | 1 |
| town | 환영 문구 "한 번만" 신호 방식 (TOWN-01 FR-004). 정해지면 `signUp`이 그 신호를 심는다. 그 전에는 `?welcome=1` | 1 이후 |
| town | 헤더(`src/components/site-header.tsx`)에 `SessionKeeper`와 내 정보 링크(지금 캐릭터 배지) 추가. TOWN-10 상태창이 생기면 town FR-054("상태창을 누르면 내 블로그 홈")와 auth FR-036("헤더 프로필에서 내 정보로")이 같은 자리를 두고 부딪히므로, 상태창 안의 내 정보 입구를 town과 정한다 (남은 문제 15) | 8 |
| blog | `src/app/settings/blog/page.tsx`에 내 정보 링크 한 줄 추가 | 8 |
| blog | 닉네임끼리 대소문자를 무시할지 결정 (BLOG-03). 무시로 정하면 auth가 `profiles.nickname` UNIQUE를 `lower(nickname)` 인덱스로 바꾼다 | 결정 시 |
| social | `replies` 표 (관리자 통계 댓글 수에 더함) | 8 (social 5단계 뒤) |
| social | 탈퇴용 댓글 정리 구조와 함수: `comments.author_id` 삭제 동작, 내용 없는 `삭제된 댓글이에요` 자리, 작성자 없는 자리를 `getComments`가 보여 주기 (D2, FR-052) | 9의 선행 (social 5단계) |
| game | `notifications`의 받는 회원·행동한 회원 FK를 `ON DELETE CASCADE`로 (FR-051, game FR-046) | 9의 선행 (game 6단계) |
| game | `attendances.session_id` → `sessions` `ON DELETE SET NULL` (세션이 2시간 만료·로그아웃·탈퇴로 지워져도 출석이 남게). 로그인 유지 변경과 자동 출석을 함께 확인 | 8 ↔ game 6 |
| game | `REWARD_RULES.signup`(🪙 100)과 `grantReward` 유지 | 1 |
| shop | 가입의 기본 캐릭터 기준(`items.type = 'character' AND is_starter`)과 `bg_meadow` 코드를 유지하거나, 판매 중단 표시를 바꾸면 auth와 조율. `user_items.quantity`는 기본값 1 | 1 이후 |
| shop | 새 표(아바타 착용, 가구 배치)가 회원 삭제 때 `CASCADE` | 9 |
| post | 첨부 정리 작업이 "DB에 행이 없는 저장소 파일"도 지운다 (탈퇴로 `attachments` 행이 CASCADE로 사라진 파일, ERD 3.14) | 9 |
| 전체 | 회원(`users`)을 가리키는 새 표는 모두 `ON DELETE CASCADE` 또는 `SET NULL` (탈퇴가 FK로 막히지 않게) | 9 |

## 남은 문제 (Open Items)

spec을 고치지 않았다. 아래는 spec이 모호하거나 문구가 없어 구현 전에 정해야 하는 것이다.

1. **탈퇴 화면 문구가 spec에 없다** (FR-050): 비밀번호가 틀렸을 때·비었을 때의 문구, 버튼 이름, 안내 문장. contracts에 제안(`비밀번호가 맞지 않아요`, `비밀번호를 적어 주세요`, [회원 탈퇴])만 적었다. constitution III에 따라 spec에서 확정한다.
2. **가입 때 기본 캐릭터가 아닌 아이템을 보낸 경우의 문구가 없다** (FR-011): 화면으로는 생기지 않는 조작 요청이다. 제안은 예전 온보딩 문구 `고를 수 없는 캐릭터예요`.
3. **내 정보 화면 세부가 spec에 없다** (FR-036): 화면 제목, 아이디 로그인 줄 표기, 연동 날짜 형식, 서비스 이름 표기(`Google`과 `구글`, 원본 AUTH-05는 "[구글 연동]"), 키가 없는 서비스의 [연동하기] 표시(첫 화면 FR-031을 따른다고 제안), 소셜 오류 문구의 첫 화면 위치.
4. **소셜 로그인에 [로그인 상태 유지]를 적용할지** spec에 없다. plan은 "같은 카드의 체크박스를 따른다"로 정했다 (research R7). 라이브러리 확인 목록 3·10이 안 되면 소셜 세션 쿠키가 7일로 남는다 (서버 2시간 규칙은 지켜짐).
5. **FR-023 문구 해석**: "다른 사이트에서 시작된 요청에는 실리지 않으며"를 constitution 기준 `SameSite=Lax`로 해석했다. Lax는 다른 사이트 링크로 들어오는 최상위 GET 이동에는 쿠키를 보낸다 (OAuth 콜백이 여기에 기댐). 상태 변경은 GET으로 하지 않는다.
6. **CSRF 응답 코드**: constitution은 403을 적었다. Server Action의 Origin 불일치 응답 코드가 403인지 확인이 필요하다. 아니면 그대로 둘지 래퍼를 둘지 팀이 정한다.
7. **로그인 시도 제한의 한계**: "연속"에 시간 창이 없어(spec 그대로) 며칠 전 실패도 센다. 같은 아이디로 동시에 많은 요청이 오면 예약 잠금을 기다리는 요청이 DB 연결을 하나씩 잡는다 (예약 트랜잭션 안에서 라이브러리를 부르지 않으므로 교착은 없다, research R8). 5번째 시도가 성공하기 전에 들어온 같은 아이디의 시도는 잠금 문구를 받을 수 있다. IP 기준 제한은 spec 범위 밖이다.
8. **관리자 탈퇴**: spec에 예외가 없어 허용했다. 관리자가 탈퇴하면 `/@notice`와 공지 글도 지워진다. 막을지 팀 확인.
9. **프로필 사진 올리기 화면을 맡는 spec이 없다**: `profiles.photo_key` 칸만 만든다 (공통 맥락 5.3).
10. **소셜 자동 검증 불가**: SC-007, SC-012, User Story 3 #1·#2·#6, User Story 4 #2·#3은 키가 있어야 한다. 키 발급 전까지 quickstart 6장 수동 확인으로 남는다.
11. **Better Auth 1.7·Next.js 16 세부 동작 16건**: [research.md 구현 전 확인 목록](research.md#구현-전-확인-목록). `node_modules`가 없어 확인하지 못했다. 특히 3(세션 훅), 7(소셜 가입 막기 오류 코드, 이메일 없는 소셜 계정 처리), 9(서버 연동 호출)가 바뀌면 대안 설계로 간다.
12. **기존 소셜 가입 회원**: `username`이 없는 회원이 남아 있으면 마이그레이션 A2가 실패한다. 소셜 키가 발급된 적이 없어 0명으로 본다 (ERD 7장 5). 있으면 로컬은 `npm run db:reset`.
13. **관리자 비밀번호 12자를 개발에서도 강제**: 팀원 `.env.local`의 `ADMIN_PASSWORD`를 바꿔야 한다 (research R13).
14. **`e2e/flow.mjs`가 이미 낡았다**: 없는 `개발용 아이디` 입력을 쓰고 온보딩을 기다린다. 소유 spec이 없다. auth는 고치지 않고 알린다.
15. **헤더에서 내 정보로 들어가는 입구가 town spec과 부딪힌다**: auth FR-036은 "헤더 프로필에서 내 정보로 들어갈 수 있어야" 한다고, town FR-054(TOWN-10)는 "상태창을 누르면 내 블로그 홈으로 간다"고 적었다. 상태창이 생기기 전(8단계)에는 지금 캐릭터 배지를 내 정보 링크로 쓴다. 상태창이 생긴 뒤에는 상태창 안의 별도 버튼 같은 입구를 town과 정해야 하고, 필요하면 두 spec 가운데 하나의 문구를 고친다.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

해당 없음 (Constitution Check 위반 없음).
