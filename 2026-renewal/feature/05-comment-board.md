# 05. 동아리 코멘트 게시판

## 1) 기능 개요

메인 웹에서 누구나 로그인 없이 한 줄 코멘트를 남길 수 있고, 백오피스 관리자가 승인한 코멘트만 메인 웹에 공개되는 게시판 기능을 추가한다.
스팸·악성 게시물 방지를 위해 등록 즉시 공개하지 않고 승인 대기 상태로 저장하며, 관리자가 승인/거절/삭제를 처리한다.

---

## 2) 데이터 모델

**Collection**: `comments`

| 필드 | 타입 | 필수 | 기본값 | 설명 |
|---|---|---|---|---|
| `_id` | ObjectId | - | auto | MongoDB 기본 ID |
| `content` | String | ✓ | - | 코멘트 내용 (최대 200자) |
| `author` | String | ✓ | - | 작성자 이름 자유 기재 (최대 20자) |
| `status` | String | - | `"pending"` | `pending` / `approved` / `rejected` |
| `createdAt` | Date | - | now | 등록 시각 |
| `updatedAt` | Date | - | now | 최종 수정 시각 |

> `createdAt` / `updatedAt`은 Mongoose 스키마 옵션 `{ timestamps: true }`로 자동 관리한다. 수동으로 `$set: { updatedAt }` 을 지정하지 않는다.

**Collection**: `auditlogs` (신규 생성)

감사 로그용 컬렉션. `develop` 브랜치에 auditLogService가 없으므로 이번 기능과 함께 간단하게 구현한다.

| 필드 | 타입 | 설명 |
|---|---|---|
| `actorUserId` | ObjectId | 액션을 수행한 관리자 ID (`req.user._id`) |
| `targetId` | ObjectId | 대상 코멘트의 `_id`. 삭제 후에도 이력 추적 가능 |
| `action` | String | `COMMENT_APPROVED` / `COMMENT_REJECTED` / `COMMENT_DELETED` |
| `before` | Object | 변경 전 상태 |
| `after` | Object | 변경 후 상태 (삭제 시 생략) |
| `createdAt` | Date | 로그 생성 시각 |

### 인덱스 설계

**`comments` 컬렉션**

```js
// 공개 API: approved 필터 + 최신순 정렬
{ status: 1, createdAt: -1, _id: -1 }

// 관리자 API: status 선택 필터 + 최신순 정렬
{ createdAt: -1, _id: -1 }
```

**`auditlogs` 컬렉션**

```js
{ targetId: 1 }    // 특정 코멘트의 이력 조회
{ actorUserId: 1 } // 특정 관리자의 행동 이력 조회
```

> 인덱스 미생성 시 컬렉션 풀 스캔이 발생하므로, 모델 정의 시 `schema.index()`로 반드시 선언한다.

---

## 3) 사전 작업

### 패키지 설치
```bash
npm install express-rate-limit
```
`package.json`에 `express-rate-limit` 의존성 추가 후 구현 시작.

### `isLoggedIn` 미들웨어 수정
현재 `src/middlewares/index.js`의 `isLoggedIn`은 미인증 시 `403 + 문자열`을 반환한다.
이번 기능의 에러 응답 규격(`401 + JSON`)에 맞게 수정한다.

```js
// 수정 전
res.status(403).send('로그인 하지 않았음');

// 수정 후
res.status(401).json({ error: { message: "UNAUTHORIZED" } });
```

### CORS 설정 — 공개 API에 메인 웹 origin 추가
`POST /comments`, `GET /comments`는 메인 웹(`localhost:3000`)에서 호출된다.
현재 `app.js`의 CORS는 백오피스 프론트(`CLIENT_ORIGIN_DEV`)만 허용하므로, 공개 comment 라우터는 별도 CORS 옵션을 라우터 레벨에서 적용한다.

```js
// comment 라우터에 직접 cors 미들웨어 적용
const cors = require("cors");
const commentCors = cors({
  origin: [process.env.CLIENT_ORIGIN_DEV, process.env.MAIN_ORIGIN_DEV].filter(Boolean),
  methods: ["GET", "POST", "OPTIONS"],
  credentials: false, // 공개 API는 쿠키 불필요
});
router.use(commentCors);
router.options("*", commentCors); // preflight
```

`docker-compose.yml`에 환경변수 추가:
```yaml
# backoffice_backend 서비스
- MAIN_ORIGIN_DEV=http://localhost:3000

# main_frontend 서비스
- NEXT_PUBLIC_BO_API_URL=http://localhost:3003
```

> `NEXT_PUBLIC_BO_API_URL`은 메인 웹 프론트엔드가 공개 comment API를 호출할 때 사용하는 백오피스 백엔드 주소다. `main_frontend` 서비스의 `environment` 블록에 추가한다.

---

## 4) API 명세

### 경로 체계

공개 API는 `/bo/` prefix 없이 백오피스 백엔드(포트 3003)에서 직접 제공한다.
메인 웹 프론트엔드는 `NEXT_PUBLIC_BO_API_URL` (예: `http://localhost:3003`) 환경변수를 통해 호출한다.

| 구분 | Prefix | 예시 |
|---|---|---|
| 공개 API | (없음) | `POST /comments` |
| 관리자 API | `/bo/admin/` | `GET /bo/admin/comments` |

---

### 공개 API (인증 불필요 — 메인 웹에서 호출)

#### 코멘트 등록
```
POST /comments
```
- **Rate Limit**: IP당 분당 5회 (`express-rate-limit`, in-memory)
  - `app.js`의 `trust proxy`는 `production` 모드에서만 활성화되어 있어, 개발 환경(Docker)에서 `req.ip`가 컨테이너 내부 IP로 고정될 수 있다. rate limiter 설정 시 `{ trustProxy: true }` 옵션을 명시적으로 적용한다.
- Request Body
```json
{
  "content": "정말 멋진 동아리네요!",
  "author": "컴맹"
}
```
- Response `201`
```json
{ "ok": true }
```
- Validation
  - `content`: 필수, 공백 trim 후 1~200자
  - `author`: 필수, 공백 trim 후 1~20자
- 등록 직후 `status = "pending"` 으로 저장
- 에러 응답

| 상황 | 코드 | body |
|---|---|---|
| content/author 누락 또는 길이 초과 | `400` | `{ "error": { "message": "INVALID_INPUT" } }` |
| rate limit 초과 | `429` | `{ "error": { "message": "TOO_MANY_REQUESTS" } }` |

---

#### 승인된 코멘트 목록 조회
```
GET /comments?page=1&limit=20
```
- `status = "approved"` 인 항목만 반환
- `limit` 최대 100, 초과 시 100으로 clamping
- 정렬: `createdAt` 내림차순 (최신순). 동일 createdAt은 `_id` 내림차순으로 보조 정렬하여 페이지네이션 순서 안정성 보장
- 잘못된 page/limit 값은 기본값으로 fallback (`page=1`, `limit=20`)
- Response `200`
```json
{
  "data": [
    {
      "_id": "...",
      "content": "정말 멋진 동아리네요!",
      "author": "컴맹",
      "createdAt": "2026-05-11T00:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 42,
    "totalPages": 3
  }
}
```

---

### 관리자 API (인증 필요 — 백오피스에서 호출)

- **권한**: 로그인한 관리자 전원 (`isLoggedIn` 미들웨어). 코멘트 승인/거절/삭제는 모든 로그인 관리자가 수행 가능하다.
  - **운영 합의**: 코멘트 내용은 공개 게시물이므로 별도 write 권한 분리 없이 운영하는 것으로 결정. 역할 분리가 필요해지면 BO Auth(Feature #1) 통합 시 `WRITE_COMMENT` 비트를 신설하여 교체한다.

#### 코멘트 목록 조회 (전체, 상태 필터 가능)
```
GET /bo/admin/comments?status=pending&page=1&limit=20
```
- `status` 쿼리: `pending` / `approved` / `rejected` / 생략 시 전체. 유효하지 않은 값은 무시하고 전체 조회
- `limit` 최대 100, 잘못된 page/limit 값은 기본값 fallback
- 정렬: `createdAt` 내림차순, `_id` 내림차순 보조 정렬
- Response `200`
```json
{
  "data": [
    {
      "_id": "...",
      "content": "정말 멋진 동아리네요!",
      "author": "컴맹",
      "status": "pending",
      "createdAt": "2026-05-11T00:00:00.000Z",
      "updatedAt": "2026-05-12T09:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 42,
    "totalPages": 3
  }
}
```
- 에러 응답

| 상황 | 코드 | body |
|---|---|---|
| 미인증 | `401` | `{ "error": { "message": "UNAUTHORIZED" } }` |

---

#### 코멘트 승인
```
PATCH /bo/admin/comments/:id/approve
```
- 허용 전이: `pending → approved`, `rejected → approved`
- Response `200`: `{ "ok": true }`
- 감사 로그: `actorUserId`, `targetId: comment._id`, `action: "COMMENT_APPROVED"`, `before: { status }`, `after: { status: "approved" }`
- 에러 응답

| 상황 | 코드 | body |
|---|---|---|
| 유효하지 않은 id 형식 (ObjectId 파싱 불가) | `400` | `{ "error": { "message": "INVALID_ID" } }` |
| 형식은 맞지만 해당 id 미존재 | `404` | `{ "error": { "message": "NOT_FOUND" } }` |
| 미인증 | `401` | `{ "error": { "message": "UNAUTHORIZED" } }` |

> **400 vs 404 구분 기준**: `:id`가 MongoDB ObjectId 형식(24자 hex)이 아니면 DB 조회 없이 즉시 `400` 반환. 형식은 유효하나 DB에 없으면 `404` 반환.

---

#### 코멘트 거절
```
PATCH /bo/admin/comments/:id/reject
```
- 허용 전이: `pending → rejected`, `approved → rejected`
- Response `200`: `{ "ok": true }`
- 감사 로그: `actorUserId`, `targetId: comment._id`, `action: "COMMENT_REJECTED"`, `before: { status }`, `after: { status: "rejected" }`
- 에러 응답: 승인과 동일

---

#### 코멘트 삭제
```
DELETE /bo/admin/comments/:id
```
- Response `200`: `{ "ok": true }`
- **하드 삭제** (DB에서 완전 제거, 복구 불가). 삭제 전 감사 로그에 내용 보존.
- 감사 로그: `actorUserId`, `targetId: comment._id`, `action: "COMMENT_DELETED"`, `before: { content, author, status }`
- 에러 응답: 승인과 동일

---

### 상태 전이 정리

| 현재 상태 | approve | reject | delete |
|---|---|---|---|
| `pending` | ✓ → approved | ✓ → rejected | ✓ |
| `approved` | 멱등 (변화 없음) | ✓ → rejected | ✓ |
| `rejected` | ✓ → approved | 멱등 (변화 없음) | ✓ |

> `approved → pending`, `rejected → pending` 복귀는 미지원. 승인 취소가 필요한 경우 거절 처리.

> **멱등 전이 시 감사 로그 처리**: `approved → approve`, `rejected → reject` 와 같이 상태 변화가 없는 경우 DB 업데이트 및 감사 로그 기록을 모두 생략하고 `200 { "ok": true }` 만 반환한다 (no-op).

---

## 5) 운영 정책

- 코멘트 내용 최대 200자, 작성자 이름 최대 20자
- 등록 시 앞뒤 공백 trim 처리
- 빈 문자열 등록 불가
- 공개 등록 API: IP당 분당 5회 rate limit (in-memory, 운영 환경은 Redis 전환 권장)
- 페이지네이션: 기본 20개, 최대 100개 (초과 시 clamping, 잘못된 값은 기본값 fallback)
- 하드 삭제로 확정. 삭제된 항목은 복구 불가이며, 감사 로그에만 내용이 보존됨
- **XSS 방지**: `content`, `author`는 plain text로만 저장하고 HTML 태그를 허용하지 않는다. 프론트엔드에서 렌더링 시 `innerHTML` / `dangerouslySetInnerHTML` 사용 금지. React의 기본 텍스트 렌더링(`{value}`)으로 출력한다.

---

## 6) 화면 설계

### 메인 웹 (`main/frontend`)
- **코멘트 제출 폼**: 작성자 이름 입력 + 내용 입력 + 등록 버튼
  - 등록 후 "코멘트가 등록되었습니다. 검토 후 공개됩니다." 안내 문구 표시
- **코멘트 목록**: 승인된 코멘트를 최신순으로 표시 (작성자 이름 + 내용 + 등록일)

### 백오피스 (`backoffice/frontend`)
- AdminPanel에 **코멘트 관리 탭** 추가
- 상태별 필터 (전체 / 대기 / 승인 / 거절)
- 각 항목에 **승인 / 거절 / 삭제** 버튼
- 기본 뷰: 대기(pending) 목록 우선 표시

---

## 7) 구현 범위

### 이번 버전 포함
- `express-rate-limit` 패키지 설치
- `isLoggedIn` 미들웨어 응답 형식 수정 (403 문자열 → 401 JSON)
- CORS — comment 라우터에 메인 웹 origin 허용
- AuditLog 모델 + 서비스 (간단 구현)
- Comment 모델 및 백오피스 백엔드 API 전체
- 공개 등록 API rate limit (IP당 분당 5회)
- 승인/거절/삭제 감사 로그
- 백오피스 프론트엔드 — 코멘트 관리 탭
- 메인 웹 프론트엔드 — 제출 폼 + 승인된 목록 표시

### 이번 버전 미포함
- 신고 기능
- 승인 취소 (approved → pending)
- 관리자 알림 (새 코멘트 등록 시 push/email)
- captcha
- BO Auth(Feature #1) 통합 권한 비트 적용 (현재는 isLoggedIn으로 처리)
- `author` 예약어 필터링 ("관리자", "운영진" 등 사칭 방지)
- `auditlogs` TTL 인덱스 적용 (장기 운영 시 자동 만료 정책 수립 필요)
- cursor 기반 페이지네이션 전환 (`skip()` 방식은 대용량 시 O(n) 성능 저하)
- 멱등 전이 응답에 `"changed": false` 필드 추가 (no-op 여부를 클라이언트에서 구분 가능하게)
