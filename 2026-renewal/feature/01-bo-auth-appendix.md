# 01. BO Auth 기술 상세 Appendix

## 문서 관계 / 소스 오브 트루스

- 이 Appendix는 기술 구현 기준 문서다.
- 기존 [01-bo-auth.md](./01-bo-auth.md)의 기술 상세(데이터 모델, 인증/토큰, 권한 검사, 인덱스, 에러 코드)는 이 Appendix 기준으로 대체한다.
- PM 의사결정/일정/리스크는 [01-bo-auth-plan.md](./01-bo-auth.md)를 기준으로 본다.

## A) 목표 아키텍처 요약

- 인증: Google OAuth
- API 인증 유지: Authorization Bearer Token
- DB: MongoDB
- 권한: `perm` 비트마스크
- 계정 온보딩: 초대 링크 기반

## B) 데이터 모델

- IP 개인정보 최소화 정책은 `admin_audit_logs`, `refresh_tokens` 모두에 동일 적용한다(원문 IP 미저장, `ipHash` 사용).

## B.1 `admin_users`

- `email` (unique, required)
- `googleSub` (unique, nullable)
- `name` (nullable)
- `pictureUrl` (nullable)
- `perm` (number, default `READ`)
- `isSuperAdmin` (boolean, default false)
- `isActive` (boolean, default true)
- `lastLoginAt`
- `createdAt`, `updatedAt`

## B.2 `admin_invites`

- `email` (required)
- `perm` (number, default `READ`)
- `tokenHash` (unique, required)
- `expiresAt` (required)
- `status` (`pending`, `used`, `expired`, `revoked`)
- `invitedBy` (required)
- `usedByUserId` (nullable)
- `createdAt`, `updatedAt`

## B.3 `admin_audit_logs`

- `actorUserId`
- `targetUserId`
- `action`
- `before`, `after`
- `ipHash` (원문 IP 저장 금지, `HMAC-SHA256(ip, SERVER_SECRET_SALT)` 적용)
- `userAgent`
- `createdAt`

## B.4 `refresh_tokens`

- `userId` (required)
- `tokenHash` (unique, required)
- `expiresAt` (required)
- `revoked` (boolean, default false)
- `revokedAt` (nullable)
- `deviceInfo`
  - `userAgent` (raw string)
  - `os` (parsed)
  - `browser` (parsed)
  - `ipHash` (선택, 원문 IP 저장 금지 권장)
    - 알고리즘: `HMAC-SHA256(ip, SERVER_SECRET_SALT)`
    - 운영 정책: `SERVER_SECRET_SALT`는 운영 중 교체 금지
    - 주의: secret 변경 시 기존 `ipHash` 비교/조회가 불가능해짐
    - 조회/비교: 세션 목록/필터링은 원문 IP가 아닌 `ipHash` 기준으로 처리
    - 비상 교체 절차:
      - 트리거: secret 유출이 확인/강하게 의심되는 경우
      - 조치: `refresh_tokens` 전체 revoke + 전체 재로그인 유도 공지
      - 영향: 기존 `ipHash` 기반 조회/필터링 일시 불가
      - 기록: 교체 시각/사유/조치 내역을 운영 로그 및 감사 로그에 기록
  - `lastSeenAt` (선택)
- `createdAt`, `updatedAt`

## C) 권한 모델

```ts
enum Permission {
  READ = 1 << 0,
  WRITE_ACTIVITY = 1 << 1,
  WRITE_RECRUIT_FORM = 1 << 2,
  WRITE_CLUB_INFO = 1 << 3,
}
```

권한 체크:

- `(perm & WRITE_ACTIVITY) !== 0`
- `(perm & WRITE_RECRUIT_FORM) !== 0`
- `(perm & WRITE_CLUB_INFO) !== 0`
- `(perm & REQUIRED_MASK) === REQUIRED_MASK`

운영 UI:

- 내부 저장은 비트마스크
- 화면은 라벨(`read/all`, `write/activity`, `write/recruit-form`, `write/club-info`, `write/all`)로 표시
- `write/all`은 별도 비트를 두지 않고 `WRITE_ACTIVITY | WRITE_RECRUIT_FORM | WRITE_CLUB_INFO` OR 합산값으로 해석한다.
- UI 역변환 규칙: `(perm & (WRITE_ACTIVITY | WRITE_RECRUIT_FORM | WRITE_CLUB_INFO)) === (WRITE_ACTIVITY | WRITE_RECRUIT_FORM | WRITE_CLUB_INFO)`이면 `write/all`로 표시하고, 아니면 보유한 개별 write 라벨만 표시한다.

비트 관리 규칙:

- 신규 권한은 `2^n` 값만 사용
- 기존 비트 값 변경 금지
- 중앙 enum 파일에서만 권한 정의
- 32bit 초과 시 64bit/bigint 확장 정책 적용

## D) 인증/토큰 정책

## D.1 Access Token

- 만료: 10~15분(최대 15분)
- 저장: 메모리 저장(브라우저 새로고침 시 초기화되는 런타임 메모리; 예: 앱 전역 state/in-memory store)
- `localStorage/sessionStorage` 저장 금지
- `jti` claim 포함(필요 시 replay 탐지 확장)

## D.2 Refresh Token

- 만료: 14일
- 저장: `httpOnly + Secure + SameSite` cookie 권장
- rotation + revoke 적용
- 다중 디바이스 분리 관리
- FE는 refresh token 기반 silent refresh로 페이지 reload 시 세션을 복구한다.
- 도메인 정책(확정):
  - Option A 채택: 리버스 프록시로 FE/BE 요청 경로를 same-site로 정렬
  - FE는 동일 사이트 경로(예: `/api`)로 호출하고 프록시가 BE로 전달
  - refresh cookie는 same-site 기준으로 처리해 Safari ITP 영향 구간을 최소화
  - `SameSite=Lax` 이상을 기본으로 하고 운영 환경은 `Secure` 필수 적용
- 잔여 리스크 및 대응:
  - 프록시 misconfiguration으로 cross-site 전송이 남는 경우 Safari에서 silent refresh 실패 가능
  - Phase 1 QA에서 Safari refresh 실패가 확인되면 프록시 규칙 우선 수정, 인프라 제약으로 불가 시 BFF 패턴 전환

rotation atomic 처리:

- 동시 refresh 요청 중복 방지:
  - FE 1차 방어(필수): 탭 내부 싱글턴 Promise lock으로 refresh 중복 호출을 차단한다.
  - FE 2차 방어(권장): `BroadcastChannel('auth')`로 탭 간 신규 access token을 공유한다.
  - BE 안전망(권장): `revokedAt` 기준 grace window(5~10초) 정책을 적용한다.
    - grace window 이내 재사용: 경쟁 조건으로 판단(`REFRESH_TOKEN_REVOKED`) 후 재로그인 강제 없이 복구 경로 안내
    - grace window 초과 재사용: 탈취 의심으로 판단(`REFRESH_TOKEN_REUSE_DETECTED`) 후 전체 refresh token revoke

1. `revoked=false` 조건으로 토큰 조회/전환
2. 기존 토큰 revoke
3. 새 refresh token 저장
4. 새 access token 발급
5. 2~4 중 하나라도 실패 시 전체 롤백(트랜잭션 정책은 `N) 트랜잭션 정책` 참조)

reuse detection:

- revoke된 refresh token 재사용 시 `REFRESH_TOKEN_REUSE_DETECTED`
- 보안 이벤트 기록
- 해당 사용자 전체 refresh token 강제 revoke

## D.3 CSRF 경계

- 일반 Bearer API는 CSRF 영향 낮음
- refresh 엔드포인트는 cookie 사용으로 CSRF 방어 필수
- refresh 엔드포인트 CSRF 방어:
  - 요청 시 커스텀 헤더(`X-Requested-With: XMLHttpRequest`) 필수 포함
  - 서버는 해당 헤더 부재 시 요청 거부 (`401`)
  - CORS preflight 의존 방식이므로 `Access-Control-Allow-Origin` 설정 엄격 유지 필수
  - 허용 도메인은 운영 allowlist로 관리하며(예: `BO_ALLOWED_ORIGINS`), 와일드카드(`*`) 사용 금지
  - 추가 검증: `Origin`(필수) 및 `Referer`(보조) 값을 allowlist와 대조하고 불일치 시 거부
- 한계 및 보완:
  - `X-Requested-With` 단독 방식은 same-origin/프록시 환경에서 우회 가능성이 있으므로 단독 방어로 간주하지 않는다.
  - 따라서 custom header + origin/referer 검증을 기본 세트로 적용한다.
- refresh 요청 통과 조건(체크리스트):
  - `X-Requested-With: XMLHttpRequest` 헤더 존재
  - `Origin`이 `BO_ALLOWED_ORIGINS` allowlist와 일치
  - `Referer`(존재 시)가 `BO_ALLOWED_ORIGINS` allowlist와 일치
  - 위 조건을 모두 만족할 때만 통과, 하나라도 불일치하면 `401` 거부

## E) Google OAuth 보안

- `state` 검증
- `email_verified` 검증
- ID Token `issuer` 검증
- ID Token `audience(client_id)` 검증
- 필요 시 `hd`/도메인 제한
- 이메일 비교는 `trim + lowercase` 정규화 후 수행

## F) 초대 플로우

1. 슈퍼어드민 이메일 입력
2. 이메일 전용 초대 링크 생성(만료 기본 2일)
3. 사용자 링크 접속 후 Google 로그인
4. 정규화된 이메일 일치 시 승인
5. `googleSub` 바인딩
6. 초대 상태 `used` 처리

보안:

- 토큰 엔트로피 최소 128bit
- 토큰 원문 저장 금지, `tokenHash`만 저장
- 1회 사용 후 즉시 폐기
- 재발급 시 기존 활성 토큰 revoke
- 초대 검증 API rate limit 적용
- 만료 정책:
  - 기본 만료는 2일
  - 허용 범위: 최소 1시간(3600초) ~ 최대 7일(604800초)
  - 하드코딩 금지, 운영 설정값으로 변경 가능
  - 슈퍼어드민은 허용 범위 내에서 초대 생성 시 만료값을 조정할 수 있다.
  - 허용 범위를 벗어난 요청은 `400`으로 거부한다.
- 권한 초기값/지정:
  - 초대 생성 시 기본 권한은 `READ`를 부여
  - 슈퍼어드민은 초대 생성 시 추가 write 권한(`WRITE_ACTIVITY`, `WRITE_RECRUIT_FORM`, `WRITE_CLUB_INFO`)을 사전 지정할 수 있다.

초대 생성 API 스펙(`POST /bo/admin/invites`):

- body:
  - `email` (required, 1개)
  - `perm` (optional, 기본값 `READ`)
  - `expiresInSec` (optional, 기본값 172800, 허용 범위 3600~604800)
- 검증:
  - `perm`에 `READ` 미포함 요청 시 서버에서 `READ`를 강제 포함
  - 범위 외 `expiresInSec` 요청은 `400`으로 거부

동시성:

- `status=pending` 조건에서만 수락 허용
- 트랜잭션 내부에서 `pending -> used` 전환 후 바인딩
- 동시 요청은 1건만 성공

충돌 처리:

- `googleSub`가 이미 다른 계정에 바인딩된 경우 `GOOGLE_SUB_CONFLICT` 반환

## G) 슈퍼어드민 정책

- 런타임 판단 기준: DB `isSuperAdmin` 단일 기준
- 서버 시작 시 `SUPER_ADMIN_EMAIL` bootstrap 보정
- 운영 중 env 자동 덮어쓰기 금지
- 최소 1개 이상 비상 super admin 계정 유지(활성 상태)
- Google OAuth 장애 시 임시 로컬 인증 fallback 경로를 통해 비상 계정만 로그인 허용
- 로컬 인증 fallback 경로는 상시 개방하지 않고 장애 대응 시에만 운영 플래그로 활성화
- 예외 전환 규칙: Phase 1 전환 기간에는 기존 로그인 fallback 경로를 임시 활성화로 유지하고, Phase 3 완료 후에는 비상 플래그 기반 fallback만 허용한다.
- env/DB mismatch:
  - 기본: 경고 + 운영 알림
  - strict mode: fail-fast

## H) DB 재검증/권한 검사

라우트 레벨:

- 모든 보호 API 1차 권한 검사 미들웨어 적용

서비스 레벨:

- 민감 작업 2차 DB 재검증

필수 DB 재검증 대상:

- 권한 변경 API
- 초대 생성/검증/수락 API
- 계정 활성/비활성 API
- 모든 write API
- `/bo/auth/me` (화면 기준 정보 최신화)

성능 기준:

- `userId` 단건 조회 중심
- write/admin API low QPS 가정
- 필요 시 Redis 캐시 확장
- `/bo/auth/me`는 매 요청 DB 재검증을 기본 정책으로 한다.
- 근거: `/bo/auth/me`는 관리자 화면 권한/상태 렌더링의 기준 엔드포인트이므로 stale 권한 노출 방지를 위해 성능보다 최신 권한 일관성을 우선한다.

## I) Rate Limiting

대상:

- OAuth 시작
- OAuth callback
- 초대 검증 전 단계
- 초대 검증 API
- refresh API

초기값 및 조정 범위:

| 대상 | 초기값 | 조정 범위 | 기준 |
| --- | --- | --- | --- |
| OAuth callback | IP당 10 req/min | 5~20 | 로그인 실패율 모니터링 후 |
| refresh API | 사용자당 30 req/min | 20~60 | 다중 디바이스 환경 고려 |
| invite 검증 API | IP당 10 req/min | 5~20 | 초대 남용 시 강화 |

초기값은 백오피스 내부 사용자 규모 기준으로 설정한다. 운영 중 rate limit 초과 알림이 반복되거나 브루트포스 징후 발생 시 하향 조정하고, 정상 사용자 차단 이슈 발생 시 상향 조정한다. 조정 이력은 감사 로그에 준하여 기록한다.
구현 전제: 분산 환경 일관성을 위해 Redis 기반 rate limiter를 기본으로 사용한다(단일 인스턴스 개발 환경은 in-memory fallback 허용).

## J) 감사 로그/모니터링

감사 로그:

- append-only
- 수정/삭제 API 제공 금지
- 필요 시 해시 체인/외부 저장 이중 적재 확장
- retention: 180일

모니터링:

- 권한 변경
- super admin 이벤트
- rate limit 반복 초과
- 로그인 실패 반복

## K) 에러 코드 카탈로그

공통:

- `UNAUTHORIZED` (`401`)
- `FORBIDDEN` (`403`)
- `TOO_MANY_REQUESTS` (`429`)
- `INTERNAL_ERROR` (`500`)

초대/인증:

- `INVITE_EXPIRED` (`410`)
- `INVITE_ALREADY_USED` (`409`)
- `INVITE_REVOKED` (`410`)
- `INVITE_EMAIL_MISMATCH` (`422`)
- `GOOGLE_SUB_CONFLICT` (`409`)
- `OAUTH_STATE_INVALID` (`400`)
- `EMAIL_NOT_VERIFIED` (`403`)

토큰:

- `REFRESH_TOKEN_INVALID` (`401`)
- `REFRESH_TOKEN_REVOKED` (`401`): 이미 revoke된 토큰 사용(grace window 이내 경쟁 조건 포함)
- `REFRESH_TOKEN_EXPIRED` (`401`)
- `REFRESH_TOKEN_REUSE_DETECTED` (`401`): revoke된 토큰의 grace window 초과 재사용(탈취 의심, 전체 토큰 강제 revoke)

권한/계정:

- `ACCOUNT_INACTIVE` (`403`)
- `PERMISSION_DENIED` (`403`)
- `SUPER_ADMIN_REQUIRED` (`403`)

## L) 필수 인덱스

`admin_users`:

- `email` unique
- `googleSub` unique

`admin_invites`:

- `tokenHash` unique
- `email + status` 복합 인덱스
- `expiresAt` TTL 인덱스(정책 적용 시)

`admin_audit_logs`:

- `actorUserId`
- `createdAt`
- 필요 시 `action + createdAt` 복합 인덱스

`refresh_tokens`:

- `tokenHash` unique
- `userId + revoked + revokedAt` 복합 인덱스 (grace window 판정 최적화)
- `expiresAt` TTL 인덱스(정책 적용 시)

## M) 구현 순서 (개발자 기준)

1. [Phase 1] MongoDB 컬렉션/인덱스 생성 (`admin_users`, `admin_invites`, `admin_audit_logs`, `refresh_tokens`)
2. [Phase 1] Google OAuth 검증 로직 구현(`state`, `email_verified`, `issuer`, `audience`)
3. [Phase 1] 토큰 발급/검증 로직 구현(access/refresh, rotation, revoke)
4. [Phase 1] 기존 `passport-local` 및 세션 직렬화/역직렬화 전략 제거
5. [Phase 1] refresh reuse detection 구현 및 보안 이벤트 연계(배치 근거: rotation/revoke와 분리 불가한 동일 인증 경로)
6. [Phase 2] 권한 enum/비트마스크 유틸 구현(라벨 <-> 비트 변환 레이어 포함, `write/all` OR 합산/역변환 규칙 포함)
7. [Phase 2] 초대 생성/검증/수락/재발급/폐기 구현
8. [Phase 2] `googleSub` 바인딩 충돌 정책 구현(`GOOGLE_SUB_CONFLICT`)
9. [Phase 2] 라우트 미들웨어 + 서비스 DB 재검증 적용
10. [Phase 2] `/bo/auth/me` 최신 상태 정책 및 권한 반영 검증
11. [Phase 2] FE 라우트 가드/`useCan()` 훅/silent refresh 실패 처리 구현(`R) FE 구현 기준` 반영)
12. [Phase 3] 감사 로그 적재/조회 및 append-only 정책 적용
13. [Phase 3] rate limiting 및 에러 코드 매핑 적용
14. [Phase 3] 통합 테스트 및 운영 점검

## N) 트랜잭션 정책 (MongoDB)

아래 시나리오는 반드시 트랜잭션으로 처리한다.

- 초대 수락: `invite(pending->used)` + `googleSub 바인딩` + `admin_users.perm 반영`
- 권한 변경: `admin_users.perm 변경` + `admin_audit_logs 기록`
- super-admin 변경: `admin_users.isSuperAdmin 변경` + `admin_audit_logs 기록`
- 계정 비활성화: `isActive=false` + `refresh_tokens revoke` + `audit 기록`
- refresh rotation: `old refresh revoke` + `new refresh 발급` (atomic 전환)

트랜잭션 원칙:

- 실패 시 전체 롤백
- 감사 로그 쓰기 실패도 롤백 조건
- 동시성 경합 지점은 조건부 갱신(`revoked=false`, `status=pending`)으로 원자성 보장
- 운영 영향: 감사 로그 저장소 장애 시 권한 변경/계정 상태 변경 API가 일시 중단될 수 있음(보안 무결성 우선 정책)
- 운영 대응: 장애 알림 즉시 전파, 복구 전까지 읽기 중심 운영 모드 유지, 복구 후 변경 작업 재개

## O) 기존 데이터 마이그레이션 전략

목표:

- 기존 관리자 계정을 신규 인증/권한 체계로 안전하게 전환한다.

적용 시점:

- Phase 1 완료 후, Phase 2 시작 직전 실행
- 실제 운영 관리자 계정 수는 Phase 1 완료 시점에 확정해 본 섹션에 기록한다.
- 예상 소요:
  - 관리자 계정 30개 기준 반나절~1일
  - 관리자 계정 100개 기준 1~2일(초대 수락 속도에 따라 변동)

기존 계정 처리 원칙:

- `googleSub`가 없는 기존 계정은 초대 플로우 재온보딩을 기본 경로로 사용
- 운영상 즉시 전환이 필요한 계정은 제한적으로 마이그레이션 스크립트 사용 가능(슈퍼어드민 승인 필수)
- 비밀번호 로그인 데이터는 신규 인증 기준에서 참조하지 않는다.

실행 절차:

1. 마이그레이션 대상 계정 목록 확정(활성 계정 기준)
2. 기존 권한 -> 신규 `perm` 매핑 테이블 확정
3. 계정별 초대 발송 또는 승인된 스크립트 전환 수행
4. 전환 완료 계정의 `googleSub` 바인딩 및 로그인 검증
5. 권한 대조 리포트 생성(기존 권한 vs 신규 `perm`)

검증 기준:

- 마이그레이션 전/후 활성 관리자 계정 수 일치
- 권한 매핑 불일치 0건
- 전환 대상 계정의 Google 로그인 성공 확인

롤백 기준 및 절차:

- 롤백 임계치:
  - 권한 매핑 불일치 1건 이상 발생 시 즉시 중단
  - 전환 대상 계정 로그인 실패율 5% 초과 시 즉시 중단
- 임계치 초과 시 신규 적용 중단
- 영향 계정에 대해 기존 운영 경로(임시 fallback)로 즉시 복귀
- 원인 수정 후 재실행

## P) 테스트 전략

단위 테스트:

- [Phase 1] OAuth 검증: `state` 불일치, `email_verified=false`, `issuer/audience` 실패
- [Phase 1] 토큰: 발급/검증/만료/revoke/rotation/reuse detection
- [Phase 2] 권한: `perm` 비트 연산, 복합 마스크 검사, 경계값 검증

통합 테스트:

- [Phase 1] refresh rotation atomic 보장(동시 요청 포함)
- [Phase 1] 탭 내부 동시 401 상황에서 refresh API 1회만 호출되는지 검증(싱글턴 Promise lock)
- [Phase 3] 감사 로그 append-only(수정/삭제 거부)
- [Phase 2] 권한 변경 후 `/bo/auth/me` 최신 상태 반영

E2E 테스트:

- [Phase 2] 초대 생성 -> 링크 접속 -> Google 로그인 -> 권한 부여
- [Phase 2] 예외 시나리오: 초대 만료/중복 수락/이메일 불일치/`GOOGLE_SUB_CONFLICT`

Safari silent refresh QA(Phase 1 필수):

- 환경:
  - 스테이징 FE/BE를 운영과 동일한 cross-domain으로 배포해 검증
  - 브라우저/디바이스: macOS Safari, iOS Safari
- 시나리오:
  - [Phase 1] access token 만료 후 자동 refresh 성공 및 원 요청 재시도 성공
  - [Phase 1] 페이지 새로고침 후 `/auth/refresh` 기반 세션 복구 성공
  - [Phase 1] Safari cross-site tracking 활성화 상태에서 refresh cookie 전달 여부 확인
  - [Phase 1] 다중 탭 동시 refresh 시 reuse detection 오탐 여부 확인
- 판정/조치:
  - 시나리오 1,2 통과 시 Phase 1 인증 기준 충족
  - 아래 조건 중 하나라도 충족하면 Safari cross-origin refresh 실패로 판정하고 BFF 패턴 전환 착수:
    - 시나리오 1 실패(access token 만료 후 자동 refresh 미동작 또는 재시도 API 실패)
    - 시나리오 2 실패(새로고침 후 세션 복구 실패)
    - 시나리오 3에서 refresh 요청에 cookie 미첨부 확인
  - reuse detection 오탐 발생 시 동시 요청 제어(debounce/lock) 적용 후 재검증
  - grace window(5~10초) 내 재사용은 `REVOKED`로 처리되고, window 초과 재사용만 `REUSE_DETECTED`로 처리되는지 검증

완료 기준:

- [Phase 1] 인증/OAuth/토큰 테스트 통과 후 Phase 2 진입
- [Phase 2] 권한/초대 E2E 테스트 통과 후 Phase 3 진입
- [Phase 3] 감사로그/rate limit/에러코드 회귀 테스트 통과 후 릴리스

## Q) 기존 API 권한 매핑 (비트마스크 기준)

라벨-비트 기준:

- `read/all` -> `READ`
- `write/activity` -> `WRITE_ACTIVITY`
- `write/recruit-form` -> `WRITE_RECRUIT_FORM`
- `write/club-info` -> `WRITE_CLUB_INFO`
- `write/all` -> `WRITE_ACTIVITY | WRITE_RECRUIT_FORM | WRITE_CLUB_INFO`

라우트 매핑:

| API | 필요 권한(라벨) | 비트마스크 조건 |
| --- | --- | --- |
| `GET /bo/member` | `read/all` | `(perm & READ) !== 0` |
| `GET /bo/member/pdf/:filename` | `read/all` | `(perm & READ) !== 0` |
| `POST /bo/semina` | `write/activity` 또는 `write/all` | `(perm & WRITE_ACTIVITY) !== 0` |
| `GET /bo/feature/recruit` | `read/all` | `(perm & READ) !== 0` |
| `PATCH /bo/feature/recruit` | `write/recruit-form` 또는 `write/all` | `(perm & WRITE_RECRUIT_FORM) !== 0` |
| `GET /bo/feature/club-info` | `read/all` | `(perm & READ) !== 0` (`신규 구현 필요`) |
| `PATCH /bo/feature/club-info` | `write/club-info` 또는 `write/all` | `(perm & WRITE_CLUB_INFO) !== 0` (`신규 구현 필요`) |
| `GET /bo/admin/users` | super-admin only | `isSuperAdmin === true` (`신규 구현 필요`) |
| `POST /bo/admin/invites` | super-admin only | `isSuperAdmin === true` (`신규 구현 필요`) |
| `PATCH /bo/admin/users/:id/perm` | super-admin only | `isSuperAdmin === true` (`신규 구현 필요`) |
| `PATCH /bo/admin/users/:id/super-admin` | super-admin only | `isSuperAdmin === true` (`신규 구현 필요`, 감사 로그 필수) |
| `PATCH /bo/admin/users/:id/active` | super-admin only | `isSuperAdmin === true` (`신규 구현 필요`) |

## R) FE 구현 기준

- 앱 부팅:
  - 앱 시작 시 `GET /bo/auth/me`를 호출해 사용자/권한 정보를 전역 상태(store)로 초기화한다.
  - 실패(`401/403`) 시 인증 상태를 비로그인으로 전환하고 로그인 화면으로 라우팅한다.
- 라우트 가드:
  - 보호 라우트 진입 전 `isAuthenticated`와 `perm/isSuperAdmin`을 검사한다.
  - 가드 실패 시 권한 안내 페이지 또는 로그인 페이지로 리다이렉트한다.
- 권한 기반 UI 제어:
  - `useCan()` 훅 또는 `Can` 컴포넌트로 버튼/CTA 렌더링을 제어한다.
  - `write/all` 표시는 C 섹션의 UI 역변환 규칙을 동일하게 사용한다.
- silent refresh 실패 처리:
  - `/auth/refresh` 실패 시 access token/사용자 상태를 즉시 초기화한다.
  - 현재 페이지에서 재시도 루프 없이 로그인 페이지로 단일 리다이렉트한다.
  - 필요 시 `reason=session_expired` 쿼리로 사용자 안내 문구를 노출한다.

## S) API 응답 포맷 (`GET /bo/auth/me`)

- 응답 원칙:
  - `perm`(number)을 권한 판정의 기준값으로 사용한다.
  - `permLabels`는 UI 편의를 위한 파생 필드로 제공한다.
  - `permLabels`는 서버가 C 섹션의 UI 역변환 규칙(`write/all` 합산 규칙 포함)을 적용해 생성하며, FE는 이를 그대로 표시한다.
  - `isSuperAdmin`는 super-admin 전용 UI/기능 노출 제어에 사용한다.

응답 예시:

```json
{
  "id": "1234567890abcde",
  "email": "admin@uos.ac.kr",
  "name": "홍길동",
  "isSuperAdmin": false,
  "isActive": true,
  "perm": 3,
  "permLabels": ["read/all", "write/activity"]
}
```
