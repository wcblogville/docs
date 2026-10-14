# Blogville 개발 설명서

2026-10-13 · 진행

> Claude Doc 원본: https://claude.ai/code/artifact/5863531f-cdf2-495a-b15f-c25309c37b26

## 한눈에 보기

다섯 주제 중 파일 업로드와 무중단 배포는 사이트에서 이미 돌아가고, 오늘은 자동 검사(CI)·Git Flow·소셜 로그인 키 연결을 더해요.

| 주제 | 지금 상태 | 오늘 더하는 것 | 사용자가 할 일 |
| --- | --- | --- | --- |
| 개발환경 | Next.js 16 · PostgreSQL · Docker로 구성 | 이 문서로 정리 | 발표때 아래 그림으로 설명 |
| Git Flow | 기능 브랜치를 main에 바로 합침 | develop 브랜치 만듦, 규칙 문서·PR 템플릿 | GitHub에서 기본 브랜치·보호 규칙 설정 |
| 파일 업로드 | 사진 10MB · 파일 30MB, 권한 검사까지 동작 | 없음 | 없음 |
| OAuth2 소셜 로그인 | 구글·카카오·네이버 코드 완성, 서버에 키 없음 | 배포 때 키를 서버로 넘기는 연결 | 각 개발자 센터에서 키 발급 → GitHub secret |
| CI/CD (무중단) | main push → 블루-그린 배포, 전환 중 실패 0번 | PR마다 검사, 검사 통과해야 배포 | 없음 |

## 개발환경

내 컴퓨터에서는 Node.js 22로 Next.js 개발 서버를 띄우고 로컬 PostgreSQL에 붙이며, 서버에서는 같은 코드를 Docker 이미지로 만들어 띄워요.

| 구분 | 쓰는 것 | 고른 이유 |
| --- | --- | --- |
| 언어 | TypeScript | 화면과 서버를 한 언어로 쓰고, 타입 검사로 실수를 미리 잡음 |
| 화면·서버 | Next.js 16.3 (App Router, Server Action), React 19.2 | 페이지와 서버 동작을 한 프로젝트에 둘 수 있음 |
| 디자인 | Tailwind CSS 4 | 클래스 이름만으로 빠르게 화면을 꾸밀 수 있음 |
| 게임 | Phaser 4, 코드로 그린 도트 그림 | 마을 이동·충돌을 맡기고, 그림 파일 없이 수정 가능 |
| DB | PostgreSQL 17 + Drizzle ORM | 표 구조를 코드로 관리하고 마이그레이션 파일로 변경 기록 |
| 로그인 | Better Auth 1.7 (아이디 로그인 + 소셜 연동) | 세션·비밀번호 암호화·OAuth2를 직접 구현하지 않아도 됨 |
| 글쓰기 | Tiptap 3 에디터 + sanitize-html | 서식 있는 글을 쓰고, 저장 전에 위험한 HTML 제거 |
| 검사 | tsc · ESLint · 단위 테스트 · Playwright e2e | 타입·문법·규칙·실제 화면 동작을 단계별로 확인 |
| 배포 | Docker, GitHub Actions, nginx | 내 컴퓨터와 서버에서 같은 환경으로 실행 |

**내 컴퓨터에서 실행하는 순서**

1. `git clone` 후 `npm install`
2. `.env.example`을 `.env.local`로 복사하고 DB 주소와 `BETTER_AUTH_SECRET`을 채움
3. `npm run db:migrate` → `npm run db:seed` → `npm run admin:create`
4. `npm run dev` 후 `http://localhost:3000` 열기

**환경 변수**

- `DATABASE_URL`: DB 주소. 서버는 크로우풋 PostgreSQL의 전용 스키마를 써요.
- `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`: 로그인 암호화 키와 사이트 주소예요.
- `GOOGLE_*`, `KAKAO_*`, `NAVER_*`: 소셜 로그인 키예요. 비어 있으면 그 버튼이 "아직 연결 준비 중이에요"로 잠겨요.
- `ADMIN_USERNAME`, `ADMIN_PASSWORD`: 관리자 계정이에요.
- `UPLOAD_DIR`: 첨부 파일 폴더예요.

비밀 값은 `.env.local`(git에 안 올림)과 GitHub secret, 서버의 `~/blogville/.secrets`에만 둡니다.

**주요 폴더**

- `src/app`: 페이지와 Server Action
- `src/server`: DB를 다루는 서버 코드 (권한 검사 `dal.ts`)
- `src/lib`: 화면·서버 공용 규칙, `src/lib/art` 도트 그림
- `src/db/schema.ts` + `drizzle/`: 표 구조와 마이그레이션
- `scripts/`: 시드·관리자·배포 스크립트, `e2e/`: 화면 시험
- `docs/`: 요구사항 명세서와 ERD

## Git Flow 브랜치 전략

오늘부터 기능은 `feature/*`에서 만들어 `develop`에 모으고, `main`에는 `release/*`와 `hotfix/*`만 합쳐요. 예전에는 기능 브랜치를 main에 바로 합쳤어요.

```mermaid
gitGraph
  commit id: "v0.21" tag: "v0.21"
  branch develop
  checkout develop
  commit id: "develop 시작"
  branch feature/ci-gitflow
  checkout feature/ci-gitflow
  commit id: "CI 추가"
  commit id: "가이드"
  checkout develop
  merge feature/ci-gitflow id: "PR → develop"
  branch feature/farm-skins
  checkout feature/farm-skins
  commit id: "스킨"
  checkout develop
  merge feature/farm-skins
  branch release/v0.22
  checkout release/v0.22
  commit id: "릴리스 노트 v0.22"
  checkout main
  merge release/v0.22 id: "PR → main (배포)" tag: "v0.22"
  checkout develop
  merge main id: "main → develop"
  checkout main
  branch hotfix/login-500
  checkout hotfix/login-500
  commit id: "급한 수정"
  checkout main
  merge hotfix/login-500 id: "PR → main (배포)" tag: "v0.22.1"
  checkout develop
  merge hotfix/login-500 id: "PR → develop"
```

| 브랜치 | 만드는 곳 | 합치는 곳 | 역할 |
| --- | --- | --- | --- |
| `main` | — | — | 서비스 중인 코드. 들어가면 자동 배포 |
| `develop` | main | — | 다음 배포를 모으는 곳 |
| `feature/<이름>` | develop | develop (PR) | 기능 하나 |
| `release/<버전>` | develop | main + develop | 배포 전 마지막 확인·버전 정하기 |
| `hotfix/<이름>` | main | main + develop | 서비스 중 급한 오류 고침 |

**규칙**: main과 develop에는 직접 push하지 않고 PR로만 합쳐요. PR은 자동 검사를 통과해야 합칠 수 있어요. main에 합칠 때마다 릴리스 노트와 같은 버전 태그(`v0.22` 등)를 붙여요. `develop` 브랜치는 2026-10-13에 만들었어요.

## 파일 업로드

글에 사진(10MB)과 파일(30MB)을 올리면 서버가 무작위 32자 이름으로 디스크에 저장하고, DB `attachments` 표에 주인·원래 이름·형식을 기록해요.

1. 글쓰기 화면에서 사진이나 파일을 고르면 `POST /api/uploads`로 하나씩 보내요. 30MB를 받아야 해서 1MB 상한이 있는 Server Action 대신 Route Handler를 썼어요.
2. 서버는 다른 사이트에서 온 요청인지(Origin), 로그인한 회원인지, 크기가 넘치지 않는지 먼저 봐요.
3. 허용한 확장자만 받아요. 사진은 파일 앞부분(매직 넘버)까지 확인해서 확장자만 바꾼 HTML·SVG를 막아요.
4. 저장 이름은 `randomBytes(16)`으로 만들고, 주소는 `/files/<키>`예요. 올린 파일 이름은 주소에 쓰지 않아요.
5. 글을 저장하면 본문에 든 내 첨부가 그 글에 붙어요(`attachments.post_id`).

**보는 권한**: 공개 글의 첨부는 누구나, 비공개 글은 주인만, 이웃 공개 글은 이웃까지, 아직 글에 안 붙은 첨부는 올린 사람만 열 수 있어요. 파일을 내려받을 때는 원래 이름으로 받아요.

**정리와 보관**: 하루가 지나도 글에 안 붙은 첨부는 `npm run posts:cleanup`이 지워요. 서버에서는 첨부 폴더를 컨테이너 밖 `~/blogville/uploads`에 연결해서, 새 버전을 배포해도 파일이 남아요.

읽어 볼 코드: `src/app/api/uploads/route.ts`(올리기), `src/app/files/[key]/route.ts`(내려주기), `src/lib/attachments.ts`(형식·크기 규칙), `src/server/storage.ts`(디스크 저장).

## OAuth2 소셜 로그인

구글·카카오·네이버 로그인은 Better Auth의 OAuth2(Authorization Code) 흐름으로 동작하고, 아이디로 가입한 회원이 내 정보에서 먼저 연동한 뒤 로그인에 쓸 수 있어요. 발표용 자료와 대본은 따로 만들고 있어요.

1. 사용자가 「구글로 로그인」을 누르면 구글 로그인·동의 화면으로 가요.
2. 동의하면 구글이 `/api/auth/callback/google?code=…`로 돌려보내요.
3. 서버가 code와 비밀 키(client secret)로 토큰을 받고, 계정 번호를 확인해요.
4. 연동된 회원이면 세션 쿠키를 만들어 로그인돼요.

**우리 설계**

- 이메일을 요청하지 않고 계정 번호만 써요. 개인정보를 덜 모으려는 선택이에요.
- 인증 주소(`/api/auth/*`)는 허용 목록(`get-session`, `callback/*`, `error`)만 열고 나머지는 404예요. 가입·연동·해제는 Server Action이 서버에서 처리해요.
- 키가 없는 제공자는 버튼이 "아직 연결 준비 중이에요"로 잠겨요.

**키 발급과 설정**: 각 개발자 센터(Google Cloud Console, Kakao Developers, Naver Developers)에 앱을 만들고 리다이렉트 주소 `https://wcblogville.java21.net/api/auth/callback/<제공자>`를 등록해요. 받은 키는 내 컴퓨터는 `.env.local`, 서버는 GitHub secret(`GOOGLE_CLIENT_ID` 등)에 넣으면 배포 때 서버로 넘어가요.

## GitHub Actions CI/CD와 무중단 배포

PR을 올리면 "검사" 워크플로가 코드를 확인하고, main에 들어가면 같은 검사를 통과해야만 서버에 배포돼요. 2026-10-12 배포에서 파랑 앱이 초록 앱으로 바뀔 때 요청 68번 중 실패는 0번이었어요.

**CI — 검사 (`.github/workflows/ci.yml`)**: develop·main으로 가는 PR과 develop push마다 돌아요.

| 작업 | 하는 일 | 로컬 시간 |
| --- | --- | --- |
| 코드 검사 | 타입 검사(tsc), 린트(ESLint), 단위 테스트 | 약 1분 |
| 빌드·E2E | PostgreSQL을 띄우고 빌드 → 표·아이템·관리자 준비 → 브라우저로 로그인·블로그 화면 시험 | 빌드 1분 29초 + 시험 약 2분 |

시험이 실패하면 스크린샷과 앱 로그를 올려 두어서 원인을 볼 수 있어요. 시험용 비밀 값은 실행마다 새로 만들고 로그에서 가려요.

**CD — 배포 (`.github/workflows/deploy.yml`)**: main에 들어가면 아래 순서로 돌아요.

1. 검사(CI)를 먼저 돌리고, 통과해야만 다음으로 가요.
2. Docker 이미지를 만들고 nginx 이미지와 함께 서버 `~/blogville`로 보내요.
3. SSH로 `scripts/deploy.sh`를 실행해요: DB 접속 확인 → 마이그레이션·아이템·관리자 준비.
4. 비어 있는 색(파랑 8421 또는 초록 8422)에 새 앱을 띄우고, 첫 화면이 정상 응답할 때까지 최대 60초 기다려요.
5. 우리 nginx(8420)가 가리키는 곳을 새 앱으로 바꾸고 `nginx -s reload`해요. 진행 중인 요청은 끊기지 않아요.
6. 10초 뒤 옛 앱을 끄고, 전환 중 요청 수와 실패 수를 기록에 남겨요.

**실패하면**: 새 앱이 응답하지 않으면 전환하지 않고 옛 앱이 계속 손님을 받아요. 앱은 뜨는데 기능이 고장났다면 서버에서 `APP_PORT=8420 bash scripts/deploy.sh rollback`으로 바로 전 버전에 돌아가요. DB는 되돌리지 않으므로 마이그레이션은 "추가 먼저, 삭제는 다음 배포" 규칙을 지켜요.

**포트 구성**: 강사님 nginx(443) → 우리 nginx 컨테이너(8420) → 파랑 앱(8421) 또는 초록 앱(8422).
