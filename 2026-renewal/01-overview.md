# QUIPU Renewal Project Overview

본 문서는 팀 공통 실행 기준을 명확히 하기 위해 작성되었습니다.  
프로젝트 참여 인원은 아래 항목을 사전에 숙지하고 동일한 원칙으로 진행해주시기 바랍니다.

---

## 1. 프로젝트 개요

QUIPU Renewal Project는 기존 QUIPU 서비스의 운영 효율성과 확장성을 강화하기 위한 단기 리뉴얼 프로젝트입니다.

- 대상 서비스
  - Main Web
  - Backoffice(Admin) Web
- 대상 레포지토리
  - Main: <https://github.com/quipu-uos/main>
  - Backoffice: <https://github.com/quipu-uos/backoffice>
- 주요 목표
  1. 운영진 주도 콘텐츠 관리 체계 구축
  2. 연도별 요구사항 변화에 유연한 모집 프로세스 구현
  3. 관리자 접근 권한 체계 강화
  4. 메인/백오피스 기능 고도화

---

## 2. 프로젝트 기간

- **2026년 3월 16일 ~ 2026년 4월 5일**

---

## 3. 필수 준비 사항

프로젝트 시작 전 아래 항목을 반드시 준비해주시기 바랍니다.

- 개발 환경
  - VSCode 또는 Cursor
  - Node.js
  - Git
  - MongoDB Compass
- AI 도구
  - ChatGPT Plus 또는 Cursor Pro 등 유료 AI 구독
  - Codex, Cursor, Claude Code, Antigravity 등 **에이전트형 AI 도구 최소 1개 이상 필수 사용**

---

## 4. 운영 원칙

1. 본 프로젝트는 PM이 모든 과정을 단계별로 직접 안내하는 방식으로 운영하지 않습니다.
2. 이슈 발생 시, 우선 ChatGPT/Gemini/검색 등을 활용하여 자율적으로 해결을 시도해야 합니다.
3. 자체 해결이 어려운 경우, Discord에 아래 내용을 포함하여 상세히 공유해야 합니다.
   - 시도한 방법
   - 발생한 에러 또는 현상
   - 현재 진행 상태
4. 팀원 간 상호 지원을 기본 원칙으로 하며, Discord 내 질문은 공동으로 해결합니다.
5. 본 프로젝트는 “AI 없이 순차적으로 직접 구현”하는 방식이 아닌, **에이전트형 AI를 적극 활용한 개발 방식**을 지향합니다.
6. 모든 팀원은 개발 과정에서 항상 “AI를 통해 어떻게 더 빠르고 효율적으로 구현할 수 있는가”를 염두하여 개발해주시기 바랍니다.

---

## 5. Git/GitHub 경험이 없는 경우

Git/GitHub 경험이 부족한 팀원은 아래 범위까지만 가볍게 사전 학습 후 바로 실습 중심으로 진행해주세요.

- 학습 방법
  - ChatGPT 또는 Gemini에 아래 키워드 질의
  - 암기보다 실습 중 필요 시 즉시 질의하는 방식 권장
- 키워드
  - `remote`
  - `add`
  - `commit`
  - `push`
  - `clone`
  - `pull`
  - `checkout`
  - `branch`
  - `merge`

---

## 6. 기술 스택

- Frontend: **React, TypeScript**
- Backend: **Express, TypeScript**
- Database: **MongoDB**
- Deployment (변경 가능)
  - 프론트엔드: Vercel
  - 백엔드: Render
  - DB: MongoDB Atlas

---

## 7. 운영 방식

### 7.1 스크럼

- 일정: **매주 월요일**
- 방식: PM이 월요일에 게시한 안건을 기준으로, 월요일 당일 의견 수렴 및 파트 분배 진행

### 7.2 정기 브리핑

- 일정: **매주 목요일, 일요일**
- 제출 기한: **당일 자정 전**
- 공유 항목
  - 본인 담당 파트
  - 상세 진행 현황

---

## 8. GitHub 협업 절차

1. 담당 파트 작업 시작 전, 해당 작업에 대한 **Issue를 먼저 등록**합니다.
2. 개인 브랜치에서 구현 후, 완료 기능을 기준으로 `develop` 브랜치 대상 **Pull Request(PR)** 를 생성합니다.
3. **PM 1인 + 팀원 2인 이상 승인(approve)** 후 병합(merge)합니다.
4. Commit message / Issue / PR 작성 시 반드시 프로젝트 내 `CONTRIBUTING.md` 기준을 준수합니다.

### 8.1 CONTRIBUTING.md 준수 항목(요약)

- Commit Message / Issue 제목 / PR 제목은 **영어**로 작성
- Issue 본문 / PR 본문은 **한글**로 작성
- Type 예시: `feat`, `fix`, `refactor`, `docs`, `test`, `chore` 등
- 커밋 제목 형식
  - `<type>(<scope>): <subject>` 또는 `<type>: <subject>`
- PR 제목 형식
  - `<type>(<scope>): <subject> (#<issue-number>)`

> 작성 효율을 위해, AI에게 작업 맥락과 `CONTRIBUTING.md`를 함께 제공하여 추천안을 받은 뒤 검토/수정하는 방식을 권장합니다.

---

## 9. 현재 코드베이스 기준 사전 참고사항

아래 내용은 현재 저장소 구조를 기준으로 확인한 사항이며, 리뉴얼 범위 정의 시 참고 목적입니다.

### 9.1 Main 저장소

- 프론트엔드
  - Next.js 기반
  - TypeScript 사용
- 백엔드
  - Express + Sequelize 구조
- 데이터베이스
  - 모델 및 의존성 기준 MySQL 사용 구조 존재
- 운영 관점 이슈
  - 모집/FAQ/인터뷰 등 일부 데이터가 코드 상수로 관리됨
  - 예시 파일
    - `main/frontend/src/lib/recruitData.ts`
    - `main/frontend/src/lib/interviewData.ts`
  - 지원자 모델이 고정 필드 중심으로 구성되어 있어 연도별 확장성 측면에서 제약 존재
    - `main/backend/src/models/Member.js`

### 9.2 Backoffice 저장소

- 프론트엔드
  - React 기반 관리자 UI
- 백엔드
  - Express + session/passport 로그인 구조
- 데이터베이스
  - Sequelize + MySQL 구조
- 현재 확인 가능한 운영 기능
  - 지원자 목록/상세 조회
  - 포트폴리오 PDF 다운로드
  - Excel 내보내기
  - 모집 ON/OFF 상태 제어

### 9.3 리뉴얼 적용 시사점

- 현재 구조는 단기 운영에 유효하나, 연도별 정책 변경 대응 및 운영 자동화 관점에서 구조적 개선 필요
- 리뉴얼에서는 **TypeScript + MongoDB 중심 아키텍처**를 기준으로 단계적 전환/재설계를 추진

---

## 10. 에이전트 툴 사용 가이드

에이전트 툴을 사용할 때는 **토큰 관리가 매우 중요**합니다.  
프롬프트 구성에 따라 불필요한 파일 탐색, 의도치 않은 코드 수정/삭제 등이 발생할 수 있으며, 이 과정에서 토큰이 매우 빠르게 소진될 수 있습니다.

따라서 다음 원칙을 준수해주시기 바랍니다.

- 에이전트에게 허용할 행동은 명확히 지정하고, 의도치 않은 행동은 프롬프트에서 제한합니다.
- 처음부터 최소한의 정보로 기능 구현을 요청하지 않습니다.
- 작업 규모가 큰 기능은 아래 2단계로 진행하는 것을 권장합니다.

1. 기능 구현 방향 문서(`.md`)를 에이전트와 함께 먼저 작성합니다.
   - 포함 항목: 기능 내용, 구현 로직, 디렉토리 구조, 네이밍 규칙
2. 문서 내용이 확정되면, 해당 문서 기준으로 정확히 구현하도록 에이전트에 요청합니다.

> 위 방식은 불필요한 탐색과 재작업을 줄여 토큰 낭비를 크게 줄일 수 있습니다.

추가로, 자주 사용하는 네이밍/구조/규칙을 사전에 등록해 프롬프트에 맞춰 자동 로드하도록 구성할 수 있습니다.

- 관련 개념: `agent skills`
- 권장 도구: `heymark` npm package
- 참고 문서: <https://github.com/MosslandOpenDevs/heymark/blob/main/README.ko.md>

아직 본인에게 맞는 컨벤션이 명확하지 않다면, `agent skills`는 무리해서 즉시 적용하지 않아도 괜찮습니다.

---

## 11. 마무리

본 프로젝트는 짧은 기간 내 완성도와 실행력을 동시에 요구합니다.  
모든 팀원은 AI를 단순 보조 수단이 아니라, 개발 생산성을 높이는 핵심 파트너로 활용해주시기 바랍니다.