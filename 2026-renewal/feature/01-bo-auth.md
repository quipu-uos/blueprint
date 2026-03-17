# 01. BO Auth 고도화 명세서

## 0) 문서 목적

본 문서는 `02-feature-roadmap.md`의 1번 기능(백오피스 인증/권한 체계 고도화)에 대한 구현 기준 문서다.  
핵심 목표는 **Google OAuth 단일 로그인**, **사전 등록 계정만 접근 허용**, **라벨 기반 권한 분리**, **슈퍼어드민 전용 사용자 관리**를 통해 운영 보안과 유지보수성을 동시에 높이는 것이다.

이 문서는 실제 코드베이스를 직접 확인한 뒤 작성했다.

- backoffice 현재 인증: `passport-local + 세션` 단일 비밀번호 방식
- main 현재 인증: 관리자 인증 체계 없음(공개 서비스 API 중심)

---

## 1) 현재 구현 진단 (As-Is)

## 1.1 Backoffice

- 인증 방식
  - 비밀번호 입력 후 `POST /bo/auth/login`
  - 서버는 `process.env.PASSWORD`(bcrypt hash)와 비교
  - 성공 시 세션에 `admin` 사용자 직렬화
- 접근 제어
  - `isLoggedIn` 미들웨어로 로그인 여부만 확인
  - 권한 레벨(읽기/쓰기/기능별) 없음
- 문제점
  - 계정 단위 추적 불가(누가 로그인했는지 구분 안 됨)
  - 권한 분리 불가(모든 로그인 사용자가 동일 권한)
  - 감사 로그/관리자 변경 이력 부재
  - 확장성 부족(운영 인원 증가 시 보안 리스크 확대)

## 1.2 Main

- 현재 main은 공개 사용자를 위한 서비스이며 관리자 인증 체계가 없다.
- 모집 ON/OFF 같은 운영 제어는 feature API로 조회하는 구조가 있으나, 관리자 권한 모델과 연결되어 있지 않다.
- 따라서 운영 정책의 단일 진실원(Single Source of Truth)을 backoffice 인증/권한 시스템으로 세우는 것이 필요하다.

---

## 2) 목표 상태 (To-Be)

## 2.1 인증 정책

- 로그인 방식은 **Google OAuth 2.0 단일화**
- 비밀번호 로그인/로컬 전략 완전 제거
- 사전 등록된 Google 계정만 접근 허용 (`allowlist`)

## 2.2 권한 정책

권한 라벨(최소 단위):

- `read/all`
- `write/all`
- `write/activity`
- `write/recruit-form`

권한 해석 규칙:

- `write/all`은 모든 write 라벨을 포함하는 상위 권한
- 조회 권한은 기본적으로 `read/all` 필요

## 2.3 슈퍼어드민 정책

- 슈퍼어드민 Google 계정은 고정값(환경변수)으로 관리
- 슈퍼어드민만 사용자 등록/권한 부여/권한 변경 가능
- 마지막 슈퍼어드민 제거 금지

---

## 3) 사용자 시나리오 (UX 중심)

## 3.1 로그인 UX

- 로그인 화면에서 비밀번호 입력 제거
- `Google로 로그인` 버튼 1개 제공
- 인증 실패 시 사유를 사용자 친화 문구로 안내
  - 허용되지 않은 계정
  - 계정 비활성화
  - 권한 없음

## 3.2 권한 기반 UX

- 읽기 전용 계정은 조회만 가능(수정 버튼 비활성 + 이유 툴팁)
- 기능별 write 권한이 없으면 해당 액션 CTA 숨김 또는 비활성화
- API 403 발생 시 공통 에러 처리(권한 부족 메시지 + 관리자 문의 안내)

## 3.3 관리자 UX

- 슈퍼어드민 전용 사용자 관리 화면 제공
  - 사용자 검색/필터(활성, 비활성, 권한별)
  - 권한 체크박스 기반 부여/회수
  - 변경 내역 감사 로그 조회
- 실수 방지 UX
  - 위험한 작업(예 : 계정 비활성화, 권한 제거 등) 2차 확인창 실행.
  - 자기 자신의 super-admin 해제 불가. super admin이 자기 계정에서 is_super_admin 권한을 스스로 끌 수 없게 막는 것. 마지막 관리자 권한을 실수로 없애서 아무도 사용자 / 권한 관리를 못 하게 되는 사고 방지

---

## 4) 데이터 모델 설계 (DX 중심)

## 4.1 Sequelize 모델

### `users`

- `id` (PK)
- `google_sub` (string, unique, not null)
- `email` (string, unique, not null)
- `name` (string, nullable)
- `picture_url` (string, nullable)
- `is_active` (boolean, default true)
- `is_super_admin` (boolean, default false)
- `last_login_at` (date, nullable)
- timestamps

인덱스:

- unique(`google_sub`)
- unique(`email`)
- index(`is_active`)

### `permission_labels`

- `id` (PK)
- `code` (string, unique)  
  허용값: `read/all`, `write/all`, `write/activity`, `write/recruit-form`
- `description` (string)
- timestamps

### `user_permissions`

- `id` (PK)
- `user_id` (FK -> users.id)
- `permission_label_id` (FK -> permission_labels.id)
- timestamps

인덱스:

- unique(`user_id`, `permission_label_id`)
- index(`permission_label_id`)

### `admin_audit_logs` (권장)

- `id` (PK)
- `actor_user_id` (FK -> users.id)
- `target_user_id` (FK -> users.id, nullable)
- `action` (string)
- `before_json` (JSON)
- `after_json` (JSON)
- `ip` (string)
- `user_agent` (string)
- timestamps

---

## 5) 인증/인가 아키텍처

## 5.1 인증 흐름

1. 프론트: `GET /bo/auth/google` 호출
2. 백엔드: Google OAuth consent 페이지로 리다이렉트
3. Google callback: `GET /bo/auth/google/callback`
4. 서버: `profile.sub`, `email` 추출
5. DB `users` 조회
6. 미등록/비활성 계정이면 403 + 로그인 실패 페이지로 리다이렉트
7. 등록 계정이면 `req.login()` 후 세션 발급
8. 프론트에서 `/bo/auth/me`로 사용자/권한 정보 로드

## 5.2 Passport 구성

- `passport-google-oauth20` 사용
- `serializeUser`: 최소 식별자(`user.id`)만 저장
- `deserializeUser`: `users + permissions` 로딩

## 5.3 세션 저장

기본 권장: Redis 기반 세션 스토어

- 이유
  - 멀티 인스턴스/스케일아웃 대응
  - 메모리 스토어 대비 안정성 우수
  - 세션 만료/강제 로그아웃 관리 용이

쿠키 정책:

- `httpOnly: true`
- `secure: true` (prod)
- `sameSite: "None"` (cross-site 운영 시)
- `maxAge`: 2시간(요구 운영 정책에 따라 조정)

---

## 6) 백엔드 API 명세 (Auth/Admin)

## 6.1 Auth API

### `GET /bo/auth/google`

- 설명: Google OAuth 시작
- 응답: Google로 리다이렉트

### `GET /bo/auth/google/callback`

- 설명: OAuth 콜백 처리
- 성공: FE 성공 URL로 리다이렉트
- 실패: FE 로그인 URL로 리다이렉트(`?reason=unauthorized`)

### `GET /bo/auth/me`

- 설명: 현재 로그인 사용자 정보
- 응답 예:

```json
{
  "id": 12,
  "email": "user@uos.ac.kr",
  "name": "홍길동",
  "is_super_admin": false,
  "permissions": ["read/all", "write/activity"]
}
```

### `POST /bo/auth/logout`

- 설명: 세션 종료
- 응답: 200

## 6.2 Super Admin 전용 User Management API

prefix: `/bo/admin/users`

### `GET /bo/admin/users`

- 사용자 목록 + 권한 + 활성 상태 조회

### `POST /bo/admin/users`

- 사용자 등록
- body:
  - `email` (required)
  - `name` (optional)
  - `permissions` (array)
  - `is_active` (optional, default true)

### `PATCH /bo/admin/users/:id`

- 사용자 기본 정보/활성 상태/권한 일괄 수정

### `POST /bo/admin/users/:id/permissions`

- 권한 추가
- body: `code`

### `DELETE /bo/admin/users/:id/permissions/:code`

- 권한 제거

### `GET /bo/admin/audit-logs`

- 감사 로그 조회(페이징/필터)

---

## 7) 권한 체크 미들웨어 설계

## 7.1 미들웨어 목록

- `requireAuth`
  - 세션 존재 + 사용자 활성 상태 확인
- `requireAnyPermission([...codes])`
  - 권한 교집합 확인
- `requireSuperAdmin`
  - `is_super_admin === true` + 고정 슈퍼어드민 이메일 검증

## 7.2 권한 판정 규칙

- `write/all` 보유 시:
  - `write/activity`, `write/recruit-form` 자동 허용
- 읽기 API:
  - `read/all` 또는 `write/all` 허용(운영 정책 선택)

---

## 8) 기존 API 권한 매핑

- `GET /bo/member` -> `read/all`
- `GET /bo/member/pdf/:filename` -> `read/all`
- `POST /bo/semina` -> `write/activity` 또는 `write/all`
- `GET /bo/feature/recruit` -> `read/all`
- `PATCH /bo/feature/recruit` -> `write/recruit-form` 또는 `write/all`

추가 제안:

- 권한 실패 응답 포맷 통일

```json
{
  "code": "FORBIDDEN",
  "message": "해당 작업 권한이 없습니다.",
  "required_permissions": ["write/recruit-form", "write/all"]
}
```

---

## 9) 프론트 명세 (Backoffice)

## 9.1 로그인 페이지

- 기존 비밀번호 인풋 제거
- 버튼: `Google로 로그인`
- 실패 사유별 UI:
  - `unauthorized_account`
  - `inactive_account`
  - `no_permission`

## 9.2 앱 부팅 흐름

1. 앱 시작 시 `/bo/auth/me` 호출
2. 성공 시 사용자 상태 전역 저장
3. 실패 시 로그인 페이지 유지
4. 라우트 가드에서 권한 확인

## 9.3 권한 기반 컴포넌트 제어

- `Can` 컴포넌트(또는 hook) 도입
  - `can("write/recruit-form")` 형태로 사용
  - 버튼 숨김/비활성화 정책 일관화

---

## 10) 보안 요구사항

- OAuth state 검증 필수(CSRF 방지)
- 세션 고정 공격 방지(`session.regenerate`)
- CORS는 정확한 Origin 화이트리스트만 허용
- 허용 계정 검증은 반드시 서버 DB 기준
- 에러 메시지에 내부 정책 과다 노출 금지
- 사용자/권한 변경 API는 모두 감사 로그 적재

## 10.1) 보안 필수 체크리스트 (출시 게이트)

아래 항목은 운영 배포 전 `모두 충족`되어야 한다.

- OAuth `state` 검증을 서버에서 강제한다.
- Google 사용자 식별은 `email` 단독이 아닌 `google_sub`를 기준으로 처리한다.
- 세션 쿠키는 `HttpOnly=true`, `Secure=true(prod)`, `SameSite` 정책을 명시한다.
- 로그인 성공 시 `req.session.regenerate()`로 세션 고정 공격을 방지한다.
- 모든 쓰기 API는 서버 측 권한 검사(`requireAnyPermission`)를 통과해야 한다.
- 슈퍼어드민 API는 서버 측에서 `requireSuperAdmin`으로만 접근 허용한다.
- 계정/권한 변경 이벤트는 `admin_audit_logs`에 actor/target/ip/user-agent를 기록한다.
- CSRF 보호를 적용한다(특히 쿠키 기반 세션 사용 시 필수).
- CORS 허용 Origin은 정확한 도메인만 화이트리스트로 제한한다.
- Rate limiting을 적용한다(로그인 시도, 권한 실패 반복 요청 포함).
- 권한 오류/로그인 실패/관리자 작업 로그를 모니터링 대시보드에서 확인 가능해야 한다.
- 프론트는 권한 기반 UI 제어를 하되, 권한 통제의 진실원은 서버임을 유지한다.
- XSS 방지를 위해 입력 검증/출력 인코딩/CSP 정책을 함께 적용한다.
- 퇴부원 또는 운영 제외 계정은 즉시 `is_active=false` 처리 후 접근 차단한다.
- 마지막 슈퍼어드민 제거/비활성화는 서버에서 금지한다.

---

## 11) DX(확장성/운영성) 설계 포인트

- 권한 문자열 상수화(`PermissionCodes`)로 오타 방지
- 라우트-권한 매핑 중앙 관리(예: `authorizationMap.ts`)
- 테스트 우선순위:
  - 인증 성공/실패
  - 미등록 계정 차단
  - 권한별 200/403
  - super-admin 전용 API 보호
- 운영 편의:
  - 관리자 변경 로그 검색
  - 계정 비활성화 즉시 반영
  - 세션 만료/강제 로그아웃 정책 문서화

---

## 12) 단계별 구현 계획 (Small Steps)

1. 모델/마이그레이션 추가 (`users`, `permission_labels`, `user_permissions`, `admin_audit_logs`)
2. permission label 시드 데이터 삽입
3. Google OAuth 전략 추가 + local 로그인 제거
4. `/bo/auth/google`, `/callback`, `/me`, `/logout` 구현
5. 세션 스토어 Redis 적용
6. `requireAuth`, `requireAnyPermission`, `requireSuperAdmin` 구현
7. 기존 업무 API에 권한 매핑 적용
8. super-admin 사용자 관리 API 구현
9. 프론트 로그인 UI/라우트가드/권한 가드 반영
10. 통합 테스트 + 운영 점검 + 배포

---

## 13) 수용 기준 (Acceptance Criteria)

- 비밀번호 로그인 경로 제거 및 사용 불가
- 등록되지 않은 Google 계정 로그인 차단
- 권한 없는 사용자의 쓰기 API 호출 시 403
- 슈퍼어드민이 아닌 사용자는 계정/권한 관리 API 접근 불가
- 슈퍼어드민은 사용자 등록 및 권한 변경 가능
- 로그인 후 `/bo/auth/me`에서 권한 라벨 확인 가능
- 감사 로그에 사용자/권한 변경 이력이 남음

---

## 14) 현재 코드 대비 변경 영향 요약

- Backoffice는 인증 코어가 전면 교체된다(`local -> google oauth`).
- 라우트 보호 기준이 `로그인 여부`에서 `로그인 + 권한`으로 강화된다.
- 운영팀의 사용자 관리 기능이 코드/DB/UX 전반에 새로 추가된다.
- Main은 직접 인증 기능 추가 대상은 아니지만, 운영 제어 기능의 신뢰 소스가 Backoffice 권한 모델로 정리된다.
