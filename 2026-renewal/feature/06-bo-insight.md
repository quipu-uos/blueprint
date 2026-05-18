# 06. 백오피스 통계 및 인사이트

## 1) 기능 개요

운영 의사결정을 지원하기 위해 백오피스에 지원자 통계 대시보드를 추가한다.
관리자는 성별·학년·학과·활동유형·개발분야·일자별 지원 추이를 시각적으로 확인할 수 있다.

통계 집계는 서버에서 MongoDB aggregation으로 처리하며, 프론트엔드는 Recharts 라이브러리를 사용해 차트로 시각화한다.
AI 분석은 이번 버전에서 제외하고, 별도 Feature로 분리한다.

---

## 2) 데이터 모델 변경

### 기존 `members` 컬렉션 — `gender` 필드 추가

| 필드 | 타입 | 필수 | 허용값 | 설명 |
|---|---|---|---|---|
| `gender` | String | - | `"male"` / `"female"` / `"other"` | 성별 |

**Member 스키마 수정**
```js
gender: {
  type: String,
  enum: ["male", "female", "other"],
  required: false, // 기존 데이터와의 호환성을 위해 스키마 레벨에서는 optional
  default: null,
}
```

> **기존 데이터 호환성**: `required: true`로 설정하면 기존 Member 도큐먼트를 `.save()` 할 때 Mongoose validation 에러가 발생한다. 스키마 레벨에서는 `required: false`로 두고, 지원 폼(신규 제출)에서 필수 항목으로 받는 방식으로 처리한다. 통계 API에서 `gender`가 `null`인 도큐먼트는 `"unknown"` 으로 처리하여 차트에 노출되지 않도록 한다.

---

## 3) 사전 작업

### 패키지 설치

**백오피스 프론트엔드**
```bash
npm install recharts
```

### 메인 웹 지원 폼 수정 (`main/frontend`)

지원 폼(`src/app/recruit/page.tsx`)은 `POST /recruit` (multipart/form-data)로 Member를 생성한다.

수정 범위:
1. `RecruitFormData` 타입에 `gender: string` 필드 추가
2. `formData` 초기값에 `gender: ""` 추가
3. 폼 UI에 성별 선택 항목 추가 — **라디오 버튼 3개** (남성 / 여성 / 기타)
4. `validateRequiredFields()`에 성별 미선택 시 오류 처리 추가
5. `handleSubmit()`의 FormData에 `fd.append("gender", formData.gender)` 추가

**성별 선택 UI 예시**
```tsx
<div className="flex gap-4">
  {[
    { label: "남성", value: "male" },
    { label: "여성", value: "female" },
    { label: "기타", value: "other" },
  ].map(({ label, value }) => (
    <label key={value} className="flex items-center gap-1 cursor-pointer">
      <input
        type="radio"
        name="gender"
        value={value}
        checked={formData.gender === value}
        onChange={handleChange}
        disabled={!isRecruiting}
      />
      {label}
    </label>
  ))}
</div>
```

### 백오피스 백엔드 — Member 생성 API 수정

`POST /recruit` 처리 로직에서 `gender` 필드를 추가로 받아 저장한다.
유효성 검사: `["male", "female", "other"]` 이외의 값은 `400` 반환.

### `app.js` — insight 라우터 등록

```js
const insightRouter = require("../src/routes/insight");
app.use("/bo/admin", insightRouter);
```

---

## 4) API 명세

### 경로 체계

| 구분 | Prefix | 예시 |
|---|---|---|
| 관리자 API | `/bo/admin/` | `GET /bo/admin/insight/stats` |

모든 인사이트 API는 `isLoggedIn` 미들웨어로 인증을 요구한다.

---

### 통계 조회

```
GET /bo/admin/insight/stats
```

**Response `200`**

모든 집계 항목은 Recharts가 바로 사용할 수 있도록 **배열** 형태로 반환한다.

```json
{
  "total": 42,
  "gender": [
    { "label": "male",    "count": 28 },
    { "label": "female",  "count": 12 },
    { "label": "other",   "count": 1  },
    { "label": "unknown", "count": 1  }
  ],
  "grade": [
    { "label": "1학년", "count": 10 },
    { "label": "2학년", "count": 15 },
    { "label": "3학년", "count": 12 },
    { "label": "4학년", "count": 5  }
  ],
  "major": [
    { "label": "전자전기컴퓨터공학부", "count": 30 },
    { "label": "기계정보공학과",        "count": 5  },
    { "label": "경영학부",              "count": 4  }
  ],
  "activity": [
    { "label": "세미나",   "count": 20 },
    { "label": "개발",     "count": 18 },
    { "label": "스터디",   "count": 15 },
    { "label": "대외활동", "count": 8  }
  ],
  "devField": [
    { "label": "프론트엔드", "count": 8 },
    { "label": "백엔드",     "count": 7 },
    { "label": "기획",       "count": 3 },
    { "label": "디자인",     "count": 2 }
  ],
  "timeline": [
    { "date": "25.05.01", "count": 5  },
    { "date": "25.05.02", "count": 12 }
  ]
}
```

**집계 규칙**

| 항목 | 집계 방식 | 비고 |
|---|---|---|
| `total` | `Member.countDocuments({})` | - |
| `gender` | `$group` by `{ $ifNull: ["$gender", "unknown"] }` | null인 도큐먼트는 `"unknown"` 그룹핑 |
| `grade` | `$group` by `grade`, label에 `"N학년"` 접미사 추가 | 오름차순 정렬 |
| `major` | `$group` by `major` | 카운트 내림차순 정렬 |
| `activity` | `semina/dev/study/external` 필드별 `$sum: { $cond: [필드, 1, 0] }` | 중복 허용 — 지원자 1명이 여러 활동 선택 시 각 항목에 모두 카운트됨 |
| `devField` | `dev: true` 인 도큐먼트의 `field_dev` 필드 `$group` | `field_dev`가 null이거나 빈 문자열이면 집계 제외 |
| `timeline` | `createdAt` 날짜별 `$group`, `$dateToString: { format: "%y.%m.%d", date: "$createdAt" }` | 날짜 오름차순 정렬 |

> **`activity` 중복 집계**: 한 지원자가 세미나·개발·스터디를 모두 선택한 경우 각 항목에 1씩 합산된다. 각 항목의 count는 "해당 활동을 선택한 지원자 수"를 의미하며, 전체 합산이 `total`을 초과할 수 있다.

**에러 응답**

| 상황 | 코드 | body |
|---|---|---|
| 미인증 | `401` | `{ "error": { "message": "UNAUTHORIZED" } }` |
| 서버 오류 | `500` | `{ "error": { "message": "Internal Server Error" } }` |

---

## 5) 운영 정책

- 통계는 실시간 집계한다. 캐싱 없이 요청마다 aggregation을 실행한다.
  - 지원자 수가 수천 명을 넘지 않는 규모에서는 aggregation 성능이 충분하다고 판단.
  - 향후 성능 이슈가 발생하면 서버 사이드 캐싱(TTL 기반) 도입을 검토한다.
- `gender` 필드가 `null`인 기존 도큐먼트는 `"unknown"` 으로 처리하며, 차트에서 별도 항목으로 표시한다.
- `devField` 항목은 `dev: true` 인 지원자만 집계한다. `field_dev` 값이 null이거나 빈 문자열인 경우 해당 도큐먼트는 제외한다.
- `activity` 통계는 중복 허용 집계다. 각 항목의 합산이 `total`을 초과할 수 있으며, 이는 정상 동작이다.

---

## 6) 화면 설계

### 백오피스 (`backoffice/frontend`)

**접근 방법**
- 기존 사이드바(`db-logo-top`)에 "통계/인사이트" 내비 버튼 추가
- 클릭 시 `/insight` 라우트로 이동 (별도 페이지, 코멘트 패널과 달리 오버레이 아님)
- 인사이트 페이지 상단에 "← 돌아가기" 버튼으로 `/recruitDB` 복귀

**대시보드 레이아웃**

```
┌─────────────────────────────────────────┐
│  지원자 통계 대시보드        ← 돌아가기  │
├──────────────┬──────────────────────────┤
│  총 지원자 수 │  성별 비율               │
│    42 명      │  [도넛 차트]             │
├──────────────┴──────────────────────────┤
│  일자별 지원 추이                        │
│  [라인 차트]                             │
├─────────────────┬───────────────────────┤
│  학년 분포       │  활동 유형별 지원 현황 │
│  [바 차트]       │  [바 차트]             │
├─────────────────┴───────────────────────┤
│  학과 분포                               │
│  [가로 바 차트]                          │
├─────────────────────────────────────────┤
│  개발 분야 분포 (dev 지원자만)           │
│  [바 차트]                               │
└─────────────────────────────────────────┘
```

**차트 구성 상세**

| 섹션 | 차트 종류 | 데이터 | 빈 데이터 처리 |
|---|---|---|---|
| 총 지원자 수 | 숫자 카드 | `total` | - |
| 성별 비율 | 도넛 차트 | `gender` | "데이터가 없습니다." |
| 일자별 지원 추이 | 라인 차트 | `timeline` | "데이터가 없습니다." |
| 학년 분포 | 세로 바 차트 | `grade` | "데이터가 없습니다." |
| 활동 유형별 현황 | 세로 바 차트 | `activity` | "데이터가 없습니다." |
| 학과 분포 | 가로 바 차트 | `major` | "데이터가 없습니다." |
| 개발 분야 분포 | 세로 바 차트 | `devField` | "데이터가 없습니다." |

---

## 7) 구현 범위

### 이번 버전 포함
- `Member` 모델에 `gender` 필드 추가 (`required: false`, `default: null`)
- 메인 웹 지원 폼에 성별 라디오 버튼 추가 (`POST /recruit` gender 필드 포함)
- 백오피스 백엔드 `POST /recruit` — `gender` 필드 저장 및 유효성 검사
- `GET /bo/admin/insight/stats` API (MongoDB aggregation, 배열 응답)
- `routes/insight.js` 신규 추가 및 `app.js` 등록
- 백오피스 프론트엔드 통계 대시보드 페이지 (`page/insight.jsx`)
- Recharts 기반 차트 6종 (도넛, 라인, 바, 가로 바)
- 사이드바에 "통계/인사이트" 내비게이션 추가

### 이번 버전 미포함
- AI 기반 텍스트 분석 (지원 동기 분석, 운영 방향 제안) — 별도 Feature로 분리
- 통계 캐싱 (Redis TTL 기반)
- 통계 데이터 엑셀/CSV 내보내기
- 기간 필터 (특정 날짜 범위 통계 조회)
- 지원자 수 추이 비교 (전년도 대비)
- 기존 Member 데이터 gender 일괄 마이그레이션 스크립트
