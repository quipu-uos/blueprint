# 02. Activity 동적 구성 기능 도입 (Activity CMS)

## 1. 개요 및 목적
기존 메인 웹의 Activity 영역(스터디, 세미나, 개발 등) 데이터는 프론트엔드 코드(`studyData.ts` 등) 내에 **하드코딩**되어 관리되고 있습니다. 이로 인해 새로운 활동이 추가되거나 속성(필드)이 변경될 때마다 개발자가 직접 코드를 수정하고 서버를 재배포해야 하는 비효율이 발생합니다.

본 기능 명세는 백오피스에서 Activity의 **유형(Type)**과 **속성(Field)**, 그리고 실제 들어갈 **콘텐츠(Content)**를 운영진이 직접 정의하고 관리할 수 있는 **동적 CMS(Content Management System)**를 구축하여 운영 효율성(UX)과 시스템 확장성(DX)을 극대화하는 것을 목표로 합니다.

---

## 2. 해결하고자 하는 문제 (As-Is)
- **하드코딩으로 인한 개발자 종속성 및 확장성 한계:** 
오타 수정, 연도별 활동 추가, 혹은 새로운 속성(예: 세미나의 '발표자' 필드)이 필요할 때마다 프론트엔드 코드 내부의 고정된 인터페이스(`ActivityItem`)와 데이터를 직접 수정하고 서버를 재배포해야 하는 구조적 한계.
- **이미지 관리의 비효율성:**
이미지 자원의 URL이 프론트엔드 소스 코드에 하드코딩되어 있어, 리소스 변경 시마다 코드 수정 및 재배포가 강제됨.
현재 외부 스토리지를 활용 중이나 관리 주체가 개발자에게 편중되어 있어 운영 효율성이 낮음.
(수정 방향) 백오피스를 통한 이미지 업로드 및 DB 기반 URL 관리 시스템을 도입하여 리소스 관리와 코드 배포를 완전히 분리함.

---

## 3. 핵심 아키텍처 및 해결 방안 (To-Be)

우리가 만들 CMS는 **"구글 폼을 만드는 관리자 화면"**과 완벽하게 동일한 원리로 동작합니다.

### 3.1. 문제 해결 매핑 (As-Is ➡️ To-Be)
2장에서 정의한 문제점들을 아래와 같은 기술적 전략으로 일대일(1:1) 매칭하여 해결합니다.

| As-Is (해결할 문제점) | To-Be (기술적 해결 전략) | 관련 목차 |
| --- | --- | --- |
| **하드코딩으로 인한 개발자 종속성 및 확장성 한계**<br>(고정된 데이터 구조 & 잦은 재배포) | **데이터베이스(MongoDB) 기반 조회 및 Schema-driven UI 도입**<br>하드코딩 데이터를 DB로 완전 이관(Type/Field/Content 분리)하여, **프론트엔드 코드 수정 및 재배포 없이** 백오피스에서 즉시 메인 웹 화면(신규 필드 포함)을 유연하게 제어. | 3.3, 4, 5.2 |
| **이미지 관리의 비효율성**<br>(코드 의존적 리소스 관리) | **Presigned URL 기반 다이렉트 클라우드 업로드**<br>스토리지 이미지와 DB URL 저장을 결합하여 프론트엔드 코드 배포와 파일 관리를 완전히 분리 | 5.1 |

### 3.2. 주요 용어 사전 (Glossary)
처음 CMS 구조를 접하는 팀원들을 위해, 본 문서와 개발 과정에서 가장 자주 쓰일 핵심 용어들을 정리합니다.

- **Type (활동의 종류) / `ActivityTypes` 컬렉션:** 
  - '스터디', '세미나', '해커톤' 등 메인 웹에서 하나의 **메뉴(탭)** 단위가 되는 큼직한 껍데기를 의미합니다. 
  - 백오피스에서 "새로운 활동 탭을 하나 만들자!"라고 할 때 이 Type을 하나 생성하게 됩니다.
- **Field (질문 문항 / 설계도):** 
  - 특정 Activity Type 안에서 데이터를 받기 위해 뚫어놓은 **입력칸(구멍)**들입니다. 
  - 예시: 스터디 Type 내부의 '기간(Date)', '상세설명(LongText)', '사진(Image)' 필드들.
  - 이 필드들이 모여서 하나의 "설문지 양식(Schema 설계도)"을 이룹니다.
- **Content (실제 게시물 알맹이) / `ActivityContents` 컬렉션:** 
  - 운영진이 만들어진 Field 양식에 맞춰 빈칸에 실제로 타이핑해서 넣은 **진짜 데이터(글, 사진)**입니다.
  - 메인 웹에 예쁘게 그려지는 실제 스터디 카드 하나하나가 바로 이 Content입니다.
- **하드코딩 (Hardcoding):**
  - 데이터를 관리자 웹이나 DB에서 가져오지 않고, 개발자가 소스 코드 파일(예: `.ts`, `.json`) 안에 직접 텍스트로 박아 넣는 행위. 우리가 이 프로젝트에서 **없애려는 가장 큰 문제점(As-Is)**입니다.
- **Schema-driven UI 패러다임:**
  - 프론트엔드가 화면을 그릴 때 코드에 하드코딩된 값(예: `item.topic`)을 찾는 것이 아니라, 백엔드가 주는 설계도(Schema) 문서만 쳐다보고 거기에 있는 필드를 꺼내서 화면을 자동으로 찍어내는(Dynamic Render) 최신 프론트엔드 설계 기법입니다.
- **Presigned URL (사전 발급된 URL):**
  - 무거운 이미지나 파일을 사용자가 업로드할 때, 우리 백엔드 서버를 거치지 않게 하기 위해 **클라우드(AWS S3, Cloudflare R2 등)에서 발급받는 "일회용 클라우드 출입증 티켓"**입니다. 이 티켓을 쓰면 프론트엔드에서 클라우드로 파일을 직행시킬 수 있어 서비 부하가 줄어듭니다.

### 3.3. 역할 분담 (MSA 구조)
퀴푸 프로젝트의 특성을 살려 프론트/백엔드 역할을 명확히 나눕니다.
- **Backoffice Frontend:** 운영진이 Type과 Field를 동적으로 생성(설계)하고, 해당 양식에 Content(데이터)를 입력하는 UI. 드래그 앤 드롭 기능을 통해 필드나 카드의 **순서(order)**를 직관적으로 변경 가능.
- **Backoffice Backend:** 운영진 권한을 확인하고, 입력받은 구조(Schema)와 데이터(JSON)를 MongoDB에 안전하게 저장(C/U/D).
- **Public(Main) Backend:** 메인 웹 사용자들의 요청을 받아, MongoDB에서 데이터(Contents)를 꺼내 프론트엔드에 전달(R).
- **Main Frontend:** 백엔드가 전달준 Type 메타데이터 구조를 파악해 메뉴(탭)를 동적으로 그리고, 그 안에 정의된 Field 설계도에 맞춰 카드를 **코딩 수정 없이 알아서 예쁘게 렌더링(Dynamic Rendering)** 됨.

---

## 4. 데이터베이스 및 스키마 설계 (MongoDB)

RDBMS(MySQL)처럼 고정된 열(Column)을 가지는 대신, 동적으로 변하는 데이터를 담기 위해 유연한 Document 구조(NoSQL)를 채택합니다.
두 가지의 핵심 콜렉션(Collection)으로 구성됩니다.

### 4.1. `ActivityTypes` Collection (양식 설계도 박스)
어떤 활동(Type)이 있고, 그 활동은 어떤 정보(Field)를 필요로 하는지 정의합니다.

```json
{
  "_id": "ObjectId",
  "typeKey": "study",           // 스터디 고유 키값
  "displayName": "스터디",       // 메인 웹 네비게이션에 노출될 이름
  "description": "개발 공부부터 코딩 테스트까지 등등...",
  "order": 1,                   // 👈 메인 웹 네비게이션 메뉴 노출 순서
  "isActive": true,             // 메인 노출 여부 토글
  "fields": [                   // 💡 어떤 입력 항목(질문)을 받을 것인가?
    { 
      "name": "topic",          // 나중에 실제 값을 꺼낼 키
      "label": "주제",          // 백오피스 폼에 보일 질문 이름
      "type": "shortText",      // 입력 타입 (Text, Date, Image 등 프론트엔드 렌더링 기준)
      "required": true,
      "order": 1                // 👈 백오피스 입력 폼에서의 위치
    },
    { "name": "date", "label": "기간", "type": "dateRange", "required": true, "order": 2 },
    // ... 세미나 타입이라면 여기에 "name": "speaker" (발표자) 필드가 동적으로 들어감.
  ]
}
```

### 4.2. `ActivityContents` Collection (실제 데이터 창고)
Type 설계도에 맞춰 운영진이 직접 작성한 실제 게시물이 저장됩니다.

```json
{
  "_id": "ObjectId",
  "typeKey": "study",          // ActivityTypes 참조값
  "order": 1,                  // 👈 해당 Type 내에서 리스트 노출 순서 (드래그 앤 드롭으로 변경됨)
  "isVisible": true,           // 승인/공개 여부
  "data": {                    // 💡 설계도(fields)의 규칙에 얽매이지 않고 들어가는 유연한 데이터 박스!
    "topic": "리액트 기초 스터디",      
    "date": "24.03.01 - 24.06.30",
    "details": "프론트엔드의 대명사 리액트를 다루는 스터디입니다.",
    "images": [
      // DB에는 무거운 이미지 파일이 아닌 클라우드에 직접 올린 흔적(URL)만 저장됨
      "https://pub-e688831b...dev/study-react1.jpeg" 
    ]
  },
  "createdAt": "ISODate",
  "updatedAt": "ISODate"
}
```

---

## 5. 상세 구현 컴포넌트 및 API 흐름도

### 5.1. 이미지 업로드: Presigned URL 방식 도입
서버 부하와 보안을 위해, 무거운 파일은 백엔드를 거치지 않고 프론트엔드에서 클라우드 스토리지(Cloudflare R2 등)로 직접 업로드합니다.
1. `Backoffice 프론트` ➡️ `Backoffice 백엔드`: "나 이미지 올릴 건데 클라우드 일회용 출입증(Presigned URL) 줘!" (권한/토큰 검사)
2. `Backoffice 백엔드` ➡️ `Backoffice 프론트`: 발급된 티켓(URL) 전달
3. `Backoffice 프론트` ➡️ `클라우드 스토리지`: 이미지 100% 직행 전송
4. 업로드 완료 후 생성된 URL 텍스트만 `ActivityContents`의 `data.images` 배열에 저장.

### 5.2. 프론트엔드 동적 렌더링 (Dynamic UI Rendering) 및 신규 필드 추가 대응
메인 프론트엔드는 더 이상 하드코딩된 값(`item.topic`)을 찾지 않습니다. 동적으로 변하는 키 값을 감지하여 예외 상황을 처리하는 **Schema-driven UI** 패턴을 적용합니다.

- **유연한 렌더링 방식 (프론트엔드 무수정 원칙):**
  - 메인 프론트는 `item.data.mentor` 처럼 특정 필드 이름을 하드코딩해서 찾지 않습니다. 
  - 대신, 백엔드가 전달해 준 설계도(`ActivityTypes`의 `fields` 배열)를 순회(`map`)하며, 설계도에 있는 필드 이름(`field.name`)으로 실제 데이터(`item.data[field.name]`)를 동적으로 꺼내서 화면에 그립니다.
  - 이 핵심 로직은 프론트엔드 코드에 아래와 같이 **단 한 번만 작성**됩니다.

  ```tsx
  // [메인 프론트엔드] Schema-driven UI 렌더링 핵심 예시
  function DynamicActivityCard({ schema, itemData }) {
    // schema: ActivityTypes 데이터 (어떤 필드들이 있는지)
    // itemData: ActivityContents의 실제 데이터 박스
    
    return (
      <div className="card">
        {schema.fields.map((field) => {
          // 💡 포인트 1: 필드 이름표(e.g., 'mentor', 'topic')로 실제 데이터를 동적으로 뽑아옴
          const fieldValue = itemData[field.name]; 
          
          // 💡 포인트 2: 만약 이번 글에 이 항목(e.g., 새로 추가된 mentor)이 안 적혀있다면? -> 부드럽게 생략! (Fallback)
          if (!fieldValue) return null; 

          // 💡 포인트 3: 어떤 타입의 질문(field.type)이었느냐에 따라 알아서 알맞은 UI 스위칭
          switch (field.type) {
            case 'shortText':
              return <p key={field.name}><strong>{field.label}:</strong> {fieldValue}</p>;
            case 'longText':
              return <div key={field.name} className="longText-box">{fieldValue}</div>;
            case 'imageList':
              return <img key={field.name} src={fieldValue} alt={field.label} />;
            default:
              return <span key={field.name}>{fieldValue}</span>;
          }
        })}
      </div>
    );
  }
  ```

- **[DB 관점] NoSQL의 유연함 활용:**
  - MongoDB는 스키마리스(Schema-less) 특성을 가집니다. 운영진이 백오피스에서 내년에 '담당 멘토(mentor)'라는 필드를 새로 추가했을 때, **기존 과거 데이터(작년 스터디 글)를 일괄 수정하거나 DB 마이그레이션을 할 필요가 전혀 없습니다.**
  - 기존 문서들은 해당 필드가 없는 채로 얌전히 유지되고, 새로 작성되는 문서에만 `mentor` 필드가 포함된 채로 안전하게 저장됩니다.

- **[백엔드 관점] Data Validation & Schema Delivery:**
  - 백엔드(Express + Mongoose)는 지나치게 자유로운 DB 입력을 제어하기 위해 최소한의 뼈대 스키마(예: `typeKey`, `isActive`, `data: Object`)만을 유지합니다.
  - 백엔드의 역할은 메인 웹 접속자에게 해당 Activity의 구조(`ActivityTypes.fields`)와 내용(`ActivityContents`)을 가공 없이 빠르게 내려주는 API 허브 역할입니다.

- **[프론트엔드 관점] 순수 렌더링 로직 (Zero-Code Modification):**
  - 메인 프론트는 `item.data.mentor` 처럼 특정 필드 이름을 하드코딩해서 찾지 않습니다.
  - 내년에 '담당 멘토(mentor)' 필드가 새로 추가되더라도, 아래 코드가 알아서 `fields` 배열을 돌면서 새 데이터를 그려냅니다. **프론트엔드 개발자는 코드를 단 한 줄도 추가하거나 수정할 필요가 없습니다.**
  - 기존 과거 스터디 데이터들은 `fieldValue`가 `undefined`로 잡히므로 위 코드의 `if (!fieldValue) return null;` 방어 로직에 걸려 에러 없이 자연스럽게 넘어가며(Fallback), 과거의 모습 그대로 안전하게 유지됩니다.

### 5.3. 순서(Order) 제어 방식 상세
운영진이 특정 활동 탭(예: 스터디)이나 내부 게시물(예: 리액트 스터디 카드)의 노출 순서를 마음대로 바꿀 수 있어야 합니다. 각 파트별 역할은 다음과 같습니다.

- **[DB 관점] 정렬의 기준점 (Numeric Index):**
  - `ActivityTypes` (메뉴 탭)와 `ActivityContents` (게시물) 모두에 숫자형 필드인 `order` 를 필수값으로 추가합니다. 

- **[프론트엔드 관점] 직관적인 UX (백오피스 전용):**
  - 운영진이 숫자를 수동으로 입력하게 하지 않습니다. DnD(Drag and Drop) 프론트엔드 라이브러리(e.g., `@hello-pangea/dnd` 또는 `dnd-kit`)를 활용해 마우스로 끌어다 놓는 리스트 정렬 UI를 제공합니다.
  - 마우스 드롭(이동 완료) 이벤트 발생 시, 프론트엔드에서 현재 배열의 인덱스를 바탕으로 새로운 `order` 값(1, 2, 3...)을 일관되게 다시 매깁니다.
  - 즉각적으로 백엔드의 일괄 업데이트 API(`PATCH /api/activity/order`)를 호출하여 변경된 순서표를 전송합니다.

- **[백엔드 관점] 쿼리 정렬 최적화 (DX):**
  - 백오피스에서 전송된 순서 변경 요청을 받아 DB에 일괄 적용(Bulk Write)합니다.
  - 가장 중요한 점은, 메인 웹이 데이터를 요청(`GET`)할 때 **백엔드가 DB에서 데이터를 꺼내 오는 단계(`find()`)에서부터 무조건 `.sort({ order: 1 })` 처리를 거쳐 오름차순으로 정렬된 깔끔한 배열을 프론트엔드에 내려주어야 합니다.**
  - 이렇게 하면 메인 프론트엔드는 서버가 던져준 배열을 믿고 아무 고민 없이 `map()` 만 돌리면 되므로 프론트엔드의 비즈니스 로직(정렬 연산) 부담이 완전히 사라집니다.

---

## 6. 개발 단계별 Action Plan (협업 포인트)

1. **[공통 기반 마련]** (⭐ 3, 4번 기능 담당자와 협업 필수)
   - CMS 기초 뼈대(Dynamic Schema)를 작성합니다. 백엔드의 Mongoose 모델(Schema)과 데이터 저장 구조를 3팀이 모여 확정 짓습니다.
   - 백오피스에서 사용할 공통 '텍스트 입력창', '날짜 선택창', '이미지 업로드창(Presigned URL 적용)' 리액트 컴포넌트를 설계하여 UI 라이브러리처럼 구축합니다.
2. **[Backoffice 뷰 개발]** (재민 담당)
   - Activity 전용 Type 설계 페이지 및 Content 리스트/작성/DnD 정렬 페이지 완성.
3. **[Main 뷰 리팩토링 및 API 연동]** (재민 담당)
   - 기존 `src/lib/activity/*Data.ts` 폴더 완전 제거.
   - 메인 백엔드(Public API) 호출 후 동적으로 Navbar 탭과 Activity 카드가 UI 충돌 없이 렌더링되는 로직(`Schema-driven UI`) 구현.
