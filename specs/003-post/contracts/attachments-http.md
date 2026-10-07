# Contract: 첨부 올리기·내려주기 (Route Handler) + 정리 작업

**Feature**: `003-post` | 관련: POST-02(첨부 공개 범위), POST-07, POST-09, NF-03, NF-12

코드 위치는 코드 저장소 기준. 30MB까지 받아야 해서 Server Action이 아니라 Route Handler다 (`src/app/api/uploads/route.ts` 맨 위 주석).

## 1. `POST /api/uploads` — 올리기 (계약 변경 없음)

파일: `src/app/api/uploads/route.ts`. 이 plan에서 동작을 바꾸지 않는다. 다시 확인하는 계약:

| 순서 | 검사 | 실패 응답 (JSON `{ error }`) |
|---|---|---|
| 1 | `Origin` 호스트 = `X-Forwarded-Host`(없으면 `Host`). `Origin` 없음·`null`도 거부 | 403 `잘못된 요청이에요` |
| 2 | 로그인한 회원 (`getViewer()`) | 401 `로그인한 회원만 올릴 수 있어요` |
| 3 | `Content-Length`가 있음 (chunked 거부, 본문 읽기 전) | 411 `올리지 못했어요. 다시 시도해 주세요` |
| 4 | `Content-Length` ≤ 30MB + 1MB 여유 | 413 `파일은 30MB까지 올릴 수 있어요` |
| 5 | 폼에 `file` 하나 | 400 `올리지 못했어요. 다시 시도해 주세요` |
| 6 | 이름 정리(`cleanFileName`: 경로·제어 문자 제거, 255자 넘으면 확장자 남기고 줄임) 후 형식 | 415 `올릴 수 없는 형식이에요. 사진은 PNG·JPG·GIF·WEBP, 파일은 PDF·한글·워드·엑셀·파워포인트·키노트·ZIP·TXT·CSV·MD·JSON·MP3·MP4만 올릴 수 있어요` |
| 7 | 빈 파일 아님 | 400 `빈 파일은 올릴 수 없어요` |
| 8 | 사진 10MB / 파일 30MB | 413 `사진은 10MB까지 올릴 수 있어요` / `파일은 30MB까지 올릴 수 있어요` |
| 9 | 사진이면 파일 앞부분이 실제 형식 (PNG·JPG·GIF·WEBP) | 415 `사진 파일이 아니에요. PNG·JPG·GIF·WEBP 사진만 올릴 수 있어요` |
| 10 | 저장소 저장 + `attachments` INSERT | 500 `올리지 못했어요. 다시 시도해 주세요` |

**성공**: 200 `{ key, url: "/files/{key}", kind: "image" | "file", name, size }`. 새 행은 `post_id` NULL(어느 글에도 붙지 않음) — 올린 사람만 열 수 있다 (§2).
글에 붙는 것은 `savePost`가 한다 ([write-actions.md](write-actions.md)).

## 2. `GET /files/{key}` — 내려주기 (권한·캐시 변경)

파일: `src/app/files/[key]/route.ts`.

**순서**

1. `key`가 `^[a-f0-9]{32}$`가 아니면 404.
2. `findReadableAttachment(key)`(`src/server/attachments.ts`, 새): 첨부 행 + 붙은 글의 `visibility` + 그 블로그 `owner_id` (+ auth 컬럼이 들어온 뒤 프로필 사진 여부).
3. 보는 사람 `getViewer()` (로그인 안 했으면 없음). 공개 글에 붙었거나 프로필 사진이면 누구나 볼 수 있으므로 세션을 읽지 않는다 (글 상세의 사진마다 세션 조회가 늘지 않게).
4. `attachmentAccess(...)`(`src/lib/attachments.ts`, 순수 함수)로 판단:

   | 첨부 상태 | 허용 |
   |---|---|
   | 공개 글에 붙음 | 누구나 |
   | 비공개 글에 붙음 | 그 블로그 주인만 (관리자도 안 됨) |
   | 어느 글에도 없음 | 올린 사람만 |
   | 어느 글에도 없음 + 지금 프로필 사진 | 누구나 |

5. 허용 안 됨 / 행 없음 / 저장소에 파일 없음 → 404 `파일을 찾을 수 없어요` (`text/plain; charset=utf-8`). 있는지 여부를 다르게 알리지 않는다 (FR-029).
6. 요청의 `If-None-Match`가 `"{key}"`와 같으면 304 (본문 없음, 4번 권한 확인 뒤).
7. 200 + 파일 스트림.

**응답 머리글 (200)**

| 머리글 | 값 | 바뀜 |
|---|---|---|
| `Content-Type` | DB의 `mime` (저장할 때 서버가 정한 형식) | 그대로 |
| `Content-Length` | DB의 `size` | 그대로 |
| `Content-Disposition` | 사진 `inline`, 파일 `attachment`; `filename="ASCII 대체"; filename*=UTF-8''{인코딩한 원래 이름}` (한글·괄호·따옴표, RFC 8187) | 그대로 |
| `X-Content-Type-Options` | `nosniff` | 그대로 |
| `Content-Security-Policy` | `default-src 'none'; img-src 'self'; media-src 'self'; sandbox` | 그대로 |
| `Cache-Control` | `private, no-cache` | **바뀜** (지금 `public, max-age=31536000, immutable`) |
| `ETag` | `"{key}"` (키가 같으면 내용이 같다) | **새로** |

## 3. 정리 작업 `npm run posts:cleanup` (새 CLI)

파일: `scripts/cleanup-posts.ts` → `src/server/attachments.ts`의 `cleanupPostData()`. `package.json`에 `"posts:cleanup": "tsx --conditions=react-server scripts/cleanup-posts.ts"`.

| 항목 | 내용 |
|---|---|
| 실행 | `npm run posts:cleanup` (세기만: `npm run posts:cleanup -- --dry-run`). `.env.local`의 `DATABASE_URL`, `UPLOAD_DIR`을 읽는다 |
| 1. 붙지 않은 첨부 | `post_id IS NULL AND COALESCE(detached_at, created_at) < now() - interval '1 day'` 이고 프로필 사진이 아닌 행을 `FOR UPDATE SKIP LOCKED`로 골라 `DELETE ... RETURNING key` → 저장소 파일 삭제 (없으면 넘어감) (FR-059, SC-011) |
| 2. 주인 없는 파일 | 저장소에 있지만 `attachments` 행이 없고 수정 시각이 1시간 넘은 파일 삭제 (회원 삭제 CASCADE, 실패한 저장) |
| 3. 조회 기록 | `post_views`에서 `date < 어제(한국)` 행 삭제 |
| 출력 | 한국어 한 줄씩: 지운 첨부 수, 지운 파일 수, 지운 조회 기록 수 |
| 종료 코드 | 성공 0, DB·저장소 오류 1 (부분 실패한 파일은 다음 실행에서 2번이 지운다) |
| 동시성 | 1번은 저장 중인 글이 `FOR UPDATE`로 잡은 행을 기다리지 않고 건너뛴다 (`SKIP LOCKED`, 다음 실행에서 다시 본다). 저장은 키 순서로 잠그므로 둘이 교착하지 않는다 (R6) |
| 모듈 경계 | `src/server/attachments.ts`는 `next/*`·`src/server/dal.ts`·`src/lib/auth.ts`를 import하지 않는다. 스크립트는 `.env.local`을 읽은 뒤 이 모듈을 동적 import한다 (`src/db/index.ts`·`src/server/storage.ts`가 불러올 때 환경 변수를 읽는다, R10) |
| 예약 | 배포 환경에서 하루 1번 실행. 방법은 NF-08(배포 서비스)과 함께 정한다 (plan 남은 문제) |

## 4. 저장소 모듈 `src/server/storage.ts` (서버 내부 계약)

배포 저장소가 정해지면 이 파일만 바꾼다 (NF-08, 지금 파일 주석의 원칙).

| 함수 | 지금 | 이 plan |
|---|---|---|
| `saveAttachment(key, bytes)` | 있음 (같은 키가 있으면 실패) | 그대로 |
| `openAttachment(key)` | 있음 (없으면 `null`) | 그대로 |
| `copyAttachment(fromKey, toKey)` | 없음 | **새로**: 같은 저장소 안에서 복사, 대상이 있으면 실패. 복사본의 수정 시각은 지금 (원본의 옛 시각이 옮겨지면 정리 작업 2번이 행을 넣기 전의 복사본을 지울 수 있다, R10) |
| `deleteAttachment(key)` | 없음 | **새로**: 없으면 조용히 넘어감 |
| `listStoredFiles()` | 없음 | **새로**: `{ key, modifiedAt }[]` (키 형식에 맞는 이름만) |
