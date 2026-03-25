# QUIPU Renewal Project: 동아리 공통 정보 관리의 백오피스 일원화 기능 명세서

## 1. 개요
QUIPU Main Web 소스코드 내에 하드코딩되어 있던 동아리의 핵심 정보(임원진, 외부 링크, FAQ 등)를 Admin Web(백오피스)에서 통합 관리하고, Main Web에서 이를 동적으로 불러와 렌더링하는 시스템을 구축합니다. 이를 통해 매년 혹은 수시로 변경되는 동아리 정보를 개발자의 코드 수정과 재배포 없이, 운영진이 직접 유연하게 관리할 수 있도록 전환하는 것이 핵심입니다.

## 2. 목표
- **최적의 UX (사용성):** 코딩 지식이 없는 운영진도 직관적으로 데이터를 입력, 수정, 배치할 수 있는 어드민 UI 제공.
- **최적의 DX (확장성):** 프론트엔드와 백엔드 간의 명확한 타입 공유(TypeScript), 기수별 확장이 용이한 DB 스키마(MongoDB), 직관적이고 재사용 가능한 API 설계.

---

## 3. 핵심 기능 명세

### 3.1. 백오피스 (Admin Web) 데이터 관리 기능

#### 3.1.1. 임원진 (Executive) 정보 관리
- **필드 구성:**
  - `cohort` (Number): 기수 (예: 40기 -> 40)
  - `name` (String): 이름
  - `position` (String): 직책 (예: 회장, 부회장, 학술부장 등)
  - `department` (String): 소속 학과
  - `studentId` (Number): 학번 (예: 20학번 -> 20)
  - `phone` (String): 연락처 (관리자 참고용, 노출 여부 선택 가능)
  - `interview` (Text): 인터뷰 내용 (Main Web의 Interview 섹션 노출용)
  - `profileImage` (URL): 프로필 이미지 주소 (URL 기반 자동 업로드 기능 포함)
  - `order` (Number): 정렬 순서
- **UX/기능 요구사항:**
  - 드래그 앤 드롭(Drag & Drop)을 활용한 임원진 표시 순서(`order`) 직관적 변경.
  - 텍스트가 긴 인터뷰 항목의 경우 Markdown 에디터 또는 텍스트 영역(Textarea)을 통한 편리한 작성 지원.
  - **이미지 업로드 편의성:** 외부 URL을 입력하면 서버가 이를 직접 fetch하여 자체 스토리지로 마이그레이션하는 'URL 업로드' 방식 지원.

#### 3.1.2. 대외 링크 (External Links) 관리
- **필드 구성:**
  - `type` (Enum): 링크 종류 (Homepage, Instagram, GitHub, Email 등)
  - `url` (String): 연결 URL 또는 이메일 주소
  - `label` (String): 사용자에게 보여질 표시 텍스트
  - `order` (Number): 정렬 순서
- **UX/기능 요구사항:**
  - 단일 페이지 내에서 인라인(Inline) 형태의 빠른 CRUD 처리.
  - 올바른 URL 및 이메일 형식에 대한 실시간 유효성 검사.

#### 3.1.3. FAQ (자주 묻는 질문) 관리
- **필드 구성:**
  - `category` (String): 질문 카테고리 (예: 지원 관련, 활동 관련, 기술 관련 등)
  - `question` (String): 질문 내용
  - `answer` (String): 답변 내용
  - `order` (Number): 정렬 순서
- **UX/기능 요구사항:**
  - 카테고리 분류 추가/수정/삭제 기능.
  - 아코디언 형태의 메인 웹 렌더링을 고려한 질문-답변 세트 관리.

#### 3.1.4. 이미지 업로드 기능 (URL 기반)
  1. 프론트엔드 UI 구성 (Admin Web)
   * 입력창 & 버튼: 관리자가 외부 이미지 주소(URL)를 입력할 수 있는 텍스트 창과 '스토리지로 가져오기' 버튼을 만듭니다.
   * API 호출: 버튼 클릭 시, 입력된 외부 URL을 백엔드로 전송합니다.
   * 결과 반영: 백엔드에서 작업이 끝나고 '내부 스토리지 URL'을 돌려주면, 이를 폼 데이터의 profileImage 값으로 자동
     입력하고 화면에 미리보기를 띄워줍니다.

  2. 백엔드 API 구현 (Express)
   * 엔드포인트 생성: POST /api/v1/club-info/upload-by-url API를 만듭니다.
   * 다운로드 및 업로드: 프론트엔드에서 받은 외부 URL로 서버가 직접 접근해 이미지를 다운로드한 뒤, 자체 스토리지(AWS S3,
     Cloudinary 등)에 다시 업로드합니다.
   * URL 반환: 스토리지 업로드가 완료되면 생성된 영구적인 '내부 스토리지 URL'을 프론트엔드로 응답(Response)합니다.

  3. 데이터 저장 (MongoDB)
   * 최종 전송: 관리자가 모든 임원진 정보 입력을 마치고 '저장' 버튼을 누릅니다.
   * DB 반영: 1단계에서 폼 데이터에 세팅해둔 '내부 스토리지 URL'이 다른 임원진 정보(name, position 등)와 함께 기존
     임원진 등록/수정 API로 전송되어 안전하게 DB에 저장됩니다.

### 3.2. 메인 웹 (Main Web) 반영 (렌더링) 영역
- **Interview 영역:** 당해 연도 임원진 정보를 `order` 순으로 조회하여 인터뷰 카드와 프로필 렌더링.
- **Footer 영역:** 활성화된 대외 링크 정보를 조회하여 아이콘 및 텍스트 링크로 렌더링.
- **FAQ 영역:** 카테고리별로 그룹화된 활성 FAQ 데이터를 `order` 순으로 렌더링 (아코디언 UI 적용).

---

## 4. 데이터베이스 및 API 설계 (DX 향상 전략)

### 4.1. 유연하고 확장 가능한 MongoDB 스키마 설계
기존 관계형 DB(MySQL)의 고정된 스키마에서 벗어나, MongoDB의 유연성을 활용합니다. 

```typescript
// 예시: Mongoose Schema - 임원진 정보
const ExecutiveSchema = new mongoose.Schema({
  cohort: { type: Number, required: true },
  name: { type: String, required: true },
  position: { type: String, required: true },
  department: { type: String },
  studentId: { type: Number },
  interview: { type: String },
  profileImage: { type: String },
  order: { type: Number, default: 0 }
}, { timestamps: true });
```

### 4.2. RESTful API 엔드포인트 구성
프론트엔드에서 직관적으로 사용할 수 있는 일관된 엔드포인트를 제공합니다.

- **Executives (임원진)**
  - `GET /api/v1/club-info/executives` (Query: `cohort`)
  - `POST /api/v1/club-info/executives`
  - `PUT /api/v1/club-info/executives/:id`
  - `PATCH /api/v1/club-info/executives/reorder` (배열 형태의 id를 받아 일괄 순서 업데이트)
  - `POST /api/v1/club-info/upload-by-url` (외부 URL을 받아 스토리지 업로드 후 내부 URL 반환)
- **External Links & FAQs** 도 위와 동일한 CRUD 규칙 및 라우트 구조 적용 (`/external-links`, `/faqs`).

---
## 5. 아키텍처 및 구현 전략 (UX & DX 통합)

### 5.1. 타입 안정성 (Type Safety) 확보
- 백엔드(Express)와 프론트엔드(React/Next.js) 간 데이터 교환 시 발생할 수 있는 오류를 차단하기 위해 **Zod** 또는 공통 TypeScript `interface`를 정의하여 재사용합니다. 
- API 요청의 Payload 검증과 UI 렌더링 시 타입 추론을 완벽하게 맞춥니다.

### 5.2. 성능 및 렌더링 최적화
- **Main Web (Next.js):** 동아리 정보는 변경 빈도가 아주 높지는 않으므로, **ISR (Incremental Static Regeneration)** 을 활용하여 정적 페이지의 빠른 로딩 속도와 SEO 최적화를 챙기면서도 DB 변경 사항이 일정 주기마다 반영되도록 구현합니다.
- **에러 핸들링 및 Fallback:** API 장애 시 메인 웹의 UI가 깨지지 않도록 에러 바운더리(Error Boundary)와 기본 스켈레톤(Skeleton) UI를 적용합니다.

### 5.3. 관리자 사용성 (Admin UX) 고도화
- **실수 방지 및 피드백:** 데이터 삭제 및 상태 변경 시 반드시 확인 모달(Confirm Modal)을 띄우고, 작업 성공/실패 시 즉각적인 Toast 알림을 제공합니다.
- **폼 상태 관리:** `react-hook-form`을 사용하여 복잡한 입력 폼의 상태를 성능 저하 없이 관리하고, 제출 전 강력한 클라이언트 사이드 유효성 검사를 수행합니다.

---

## 6. 기대 효과
- **개발 생산성 증가:** 해마다 반복되던 텍스트 수정 및 이미지 교체 목적의 PR 생성, 코드 리뷰, 배포 과정이 사라집니다.
- **운영 민첩성 향상:** 비개발 직군 운영진도 손쉽게 동아리 정보를 최신화할 수 있어 서비스 운영의 자율성이 증대됩니다.
