---
name: impl-frontend
description: "Frontend 개발 구현 지침"
category: implementation-guide
---

# Frontend 개발 구현 지침

## 🔗 호출 규칙

**이 지침서는 `/jira:implement [TICKET] --type frontend` 실행 시 자동 로드됩니다.**

```
/jira:implement 호출
  → --type frontend 확인
  → impl-frontend.md 로드
  → 구현 계획 제시 → 사용자 승인 → 순차 구현
```

---

## 🔴 프로젝트 공통 패턴 (필수 숙지)

**⚠️ 아래 패턴들은 이 프로젝트의 기존 코드에서 일관되게 사용되는 규칙입니다. 새 코드 작성 전 반드시 숙지하세요.**

### 1. 기존 코드 참조 위치 (CRITICAL)
| 유형 | 경로 | 새 코드 작성 전 확인 |
|------|------|---------------------|
| **Types** | `fms-frontend/src/types/` | 기존 타입 정의 재사용 |
| **API 모듈** | `fms-frontend/src/apis/` | 기존 API 패턴 참조 |
| **Store** | `fms-frontend/src/store/` | 기존 Store 패턴 참조 |
| **Components** | `fms-frontend/src/components/` | 재사용 가능 컴포넌트 |

**🔴 새 타입/API/Store 생성 전 반드시 기존 파일 확인! 중복 생성 금지!**

### 2. API 모듈 패턴
```typescript
// ✅ 필수: $axios import (axios 직접 import 금지!)
import { $axios } from "./setupAxios";

// 기존 API 모듈 예시 참조
// - bookingApi.ts, containerApi.ts, vesselApi.ts 등
```

### 3. Zustand Store 패턴 (Action 객체 필수!)
```typescript
// ✅ 프로젝트 표준: Action 객체 패턴
const useFeatureStore = create<StoreInterface>((set, get) => ({
  ...initState,

  // 🔴 반드시 Action 객체 안에 액션 정의!
  Action: {
    fetchData: async (params) => { /* ... */ },
    createData: async (request) => { /* ... */ },
    setSelectedItem: (item) => set({ selectedItem: item }),
  },
}));

// ❌ 금지: Action 객체 없이 직접 액션 정의
const useWrongStore = create((set) => ({
  fetchData: async () => { /* 이렇게 하면 안됨! */ },
}));
```

### 4. 기존 타입 정의 예시
```typescript
// 참조 경로: fms-frontend/src/types/
// 기존 타입 파일들:
// - booking.ts, container.ts, vessel.ts, common.ts 등

// 공통 응답 타입 (이미 정의됨, 재사용!)
interface CommonResponse<T> {
  isSuccess: boolean;
  data: T;
  message?: MessageResponse;
}
```

### 5. 스타일링 규칙 (ESLint 강제)
```typescript
// ✅ 필수: styled-components + 테마 토큰
const Button = styled.button`
  color: ${({ theme }) => theme.colors.primary.main};
  padding: ${({ theme }) => theme.spacing.md}px;
`;

// ❌ 금지 (빌드 에러 발생):
// - 인라인 스타일: style={{ color: 'red' }}
// - 하드코딩 색상: color: #1890ff;
// - CSS import: import './styles.css';
```

### 6. 신규 코드 작성 체크리스트
```
□ types/ 폴더에 동일/유사 타입 있는지 확인
□ apis/ 폴더에 동일/유사 API 모듈 있는지 확인
□ store/ 폴더에 동일/유사 Store 있는지 확인
□ components/ 폴더에 재사용 가능 컴포넌트 있는지 확인
□ 기존 패턴(Action 객체, $axios 등) 준수 확인
```

---

## ⚠️ 필수 준수 사항

### 디렉토리 구조
```
fms-frontend/src/
├── apis/              # API 호출 모듈
├── components/        # 재사용 컴포넌트
├── pages/
│   └── fms/          # FMS 비즈니스 페이지
├── store/            # Zustand 상태 관리
├── types/            # TypeScript 타입 정의
└── utils/            # 유틸리티 함수
```

---

## API 모듈 생성 체크리스트

### 기본 패턴
```typescript
// apis/featureApi.ts
import { $axios } from "./setupAxios";

export interface FeatureRequest {
  tenantId: string;
  field: string;
}

export interface FeatureResponse {
  data: FeatureData;
  isSuccess: boolean;
  message?: string;
}

// GET 요청
export const fetchFeature = async (
  params: FeatureRequest
): Promise<FeatureResponse> => {
  const response = await $axios.get('/api/v1/feature', { params });
  return response.data;
};

// POST 요청
export const createFeature = async (
  request: FeatureRequest
): Promise<FeatureResponse> => {
  const response = await $axios.post('/api/v1/feature', request);
  return response.data;
};

// PUT 요청
export const updateFeature = async (
  id: string,
  request: FeatureRequest
): Promise<FeatureResponse> => {
  const response = await $axios.put(`/api/v1/feature/${id}`, request);
  return response.data;
};

// DELETE 요청
export const deleteFeature = async (
  id: string
): Promise<FeatureResponse> => {
  const response = await $axios.delete(`/api/v1/feature/${id}`);
  return response.data;
};
```

### ✅ API 모듈 체크리스트
- [ ] `$axios` import from `./setupAxios`
- [ ] Request/Response 인터페이스 정의
- [ ] 함수명: `fetchXxx`, `createXxx`, `updateXxx`, `deleteXxx`
- [ ] 반환 타입: `Promise<ResponseType>`
- [ ] GET: `$axios.get(url, { params })`
- [ ] POST: `$axios.post(url, request)`
- [ ] PUT: `$axios.put(url, request)`
- [ ] DELETE: `$axios.delete(url)`

---

## Zustand Store 생성 체크리스트

### 기본 패턴
```typescript
// store/useFeatureStore.ts
import { create } from 'zustand';
import { fetchFeatureApi, createFeatureApi } from '@/apis/featureApi';

// State 인터페이스
interface FeatureState {
  data: FeatureData[];
  selectedItem: FeatureData | null;
  isLoading: boolean;
  error: string | null;
}

// Actions 인터페이스
interface FeatureActions {
  Action: {
    fetchData: (params: FetchParams) => Promise<void>;
    createData: (request: CreateRequest) => Promise<boolean>;
    setSelectedItem: (item: FeatureData | null) => void;
    clearData: () => void;
    setLoading: (loading: boolean) => void;
    setError: (error: string | null) => void;
  };
}

// Store 인터페이스 (export)
export interface FeatureStoreInterface extends FeatureState, FeatureActions {}

// 초기 상태
const initState: FeatureState = {
  data: [],
  selectedItem: null,
  isLoading: false,
  error: null,
};

// Store 생성
const useFeatureStore = create<FeatureStoreInterface>((set, get) => ({
  ...initState,

  Action: {
    fetchData: async (params) => {
      try {
        set({ isLoading: true, error: null });
        const response = await fetchFeatureApi(params);
        if (response.isSuccess) {
          set({ data: response.data, isLoading: false });
        } else {
          set({ isLoading: false, error: response.message || '데이터 조회 실패' });
        }
      } catch (error: any) {
        set({ isLoading: false, error: error.message });
      }
    },

    createData: async (request) => {
      try {
        set({ isLoading: true, error: null });
        const response = await createFeatureApi(request);
        set({ isLoading: false });
        if (response.isSuccess) {
          // 목록 갱신
          await get().Action.fetchData({});
          return true;
        }
        set({ error: response.message || '생성 실패' });
        return false;
      } catch (error: any) {
        set({ isLoading: false, error: error.message });
        return false;
      }
    },

    setSelectedItem: (item) => set({ selectedItem: item }),

    clearData: () => set({ data: [], selectedItem: null }),

    setLoading: (loading) => set({ isLoading: loading }),

    setError: (error) => set({ error }),
  },
}));

export default useFeatureStore;
```

### ✅ Zustand Store 체크리스트
- [ ] State 인터페이스 정의 (데이터, 로딩, 에러)
- [ ] Actions 인터페이스 정의 (Action 객체 패턴)
- [ ] StoreInterface export (State + Actions 합성)
- [ ] initState 정의
- [ ] **Action 객체 패턴 사용** (필수!)
- [ ] 비동기 액션: try-catch + isLoading 관리
- [ ] isSuccess 응답 체크
- [ ] 에러 메시지 상태 저장

---

## TypeScript 타입 정의 체크리스트

### 기본 패턴
```typescript
// types/feature.ts

// 기본 데이터 타입
export interface FeatureData {
  id: string;
  name: string;
  status: 'ACTIVE' | 'INACTIVE';
  createdAt: Date;
  updatedAt?: Date;
}

// API 요청 타입
export interface FeatureRequest {
  tenantId: string;
  name: string;
}

// API 응답 타입 (백엔드 응답 구조)
export interface FeatureApiResponse {
  field_name: string;  // snake_case (백엔드)
  created_at: string;
}

// UI용 타입 (camelCase 변환)
export interface FeatureUIData {
  fieldName: string;   // camelCase (프론트엔드)
  createdAt: Date;
}

// 테이블 행 타입
export interface FeatureTableRow extends FeatureData {
  key: string;  // Ant Design 테이블용
}

// 폼 데이터 타입
export interface FeatureFormData {
  name: string;
  status: 'ACTIVE' | 'INACTIVE';
}
```

### ✅ TypeScript 타입 체크리스트
- [ ] API 응답 타입과 UI 타입 분리
- [ ] snake_case ↔ camelCase 변환 고려
- [ ] 선택적 필드는 `?` 사용
- [ ] 상태값은 Union 타입 사용
- [ ] 테이블용 `key` 필드 포함 (Ant Design)
- [ ] 폼 데이터 타입 별도 정의

---

## 컴포넌트 생성 체크리스트

### 기본 패턴
```typescript
// pages/fms/feature/FeaturePage.tsx
import React, { useEffect, useState } from 'react';
import styled from 'styled-components';
import useFeatureStore from '@/store/useFeatureStore';

const FeaturePage: React.FC = () => {
  const { data, isLoading, error, Action } = useFeatureStore();
  const [searchParams, setSearchParams] = useState<SearchParams>({});

  useEffect(() => {
    Action.fetchData(searchParams);
  }, [searchParams]);

  if (isLoading) return <LoadingSpinner />;
  if (error) return <ErrorMessage message={error} />;

  return (
    <Container>
      <Header>
        <Title>기능 제목</Title>
        <SearchForm onSearch={setSearchParams} />
      </Header>
      <Content>
        <DataTable data={data} />
      </Content>
    </Container>
  );
};

// styled-components (테마 토큰 사용)
const Container = styled.div`
  padding: ${({ theme }) => theme.spacing.lg}px;
`;

const Header = styled.div`
  display: flex;
  justify-content: space-between;
  margin-bottom: ${({ theme }) => theme.spacing.md}px;
`;

const Title = styled.h1`
  color: ${({ theme }) => theme.colors.text.primary};
  font-size: ${({ theme }) => theme.fontSize.xl};
`;

export default FeaturePage;
```

### ✅ 컴포넌트 체크리스트
- [ ] `React.FC` 타입 사용
- [ ] Store에서 구조 분해: `{ data, isLoading, error, Action }`
- [ ] useEffect로 초기 데이터 로드
- [ ] 로딩/에러 상태 처리
- [ ] styled-components 사용
- [ ] **테마 토큰 사용** (필수!)
- [ ] 인라인 스타일 금지

---

## 🔴 스타일링 필수 규칙

### ❌ 절대 금지
```typescript
// ❌ 인라인 스타일 금지
<div style={{ color: 'red' }}>금지</div>

// ❌ 하드코딩 색상 금지
const Button = styled.button`
  color: #1890ff;  // ESLint 에러!
  background: white; // ESLint 에러!
`;

// ❌ CSS 파일 import 금지
import './styles.css';  // ESLint 에러!
```

### ✅ 필수 패턴
```typescript
// ✅ styled-components + 테마 토큰
const Button = styled.button`
  color: ${({ theme }) => theme.colors.primary.main};
  background: ${({ theme }) => theme.colors.neutral.white};
  padding: ${({ theme }) => theme.spacing.md}px;
  border-radius: ${({ theme }) => theme.borderRadius.sm}px;
`;
```

### 테마 토큰 참조
```typescript
// 색상
theme.colors.primary.main
theme.colors.neutral.white
theme.colors.text.primary
theme.colors.border.default

// 간격
theme.spacing.xs  // 4px
theme.spacing.sm  // 8px
theme.spacing.md  // 16px
theme.spacing.lg  // 24px
theme.spacing.xl  // 32px

// 폰트
theme.fontSize.sm
theme.fontSize.md
theme.fontSize.lg

// 테두리
theme.borderRadius.sm
theme.borderRadius.md
```

---

## 파일 생성 순서

1. **TypeScript 타입** → `types/feature.ts`
2. **API 모듈** → `apis/featureApi.ts`
3. **Zustand Store** → `store/useFeatureStore.ts`
4. **페이지 컴포넌트** → `pages/fms/feature/FeaturePage.tsx`
5. **하위 컴포넌트** → `pages/fms/feature/components/`

---

## 구현 전 승인 절차 (사용자 확인 필수)

### 구현 계획 제시 템플릿
```
📋 Frontend 구현 계획

## 티켓 정보
- 티켓: [TICKET-ID]
- 제목: [이슈 제목]

## 생성/수정 파일 목록
| # | 파일 경로 | 작업 | 설명 |
|---|----------|------|------|
| 1 | types/feature.ts | 생성 | TypeScript 타입 |
| 2 | apis/featureApi.ts | 생성 | API 모듈 |
| 3 | store/useFeatureStore.ts | 생성 | Zustand Store |
| 4 | pages/fms/feature/FeaturePage.tsx | 생성 | 메인 페이지 |

## 체크리스트 준수 확인
- [ ] 디렉토리 구조 준수
- [ ] styled-components 사용
- [ ] 테마 토큰 사용 (인라인 스타일 금지)
- [ ] Action 객체 패턴 사용

---
위 계획대로 구현을 진행할까요? (예/아니오)
```

**→ 사용자 승인("예", "진행", "승인") 후에만 구현 시작**

---

## 🔴 절대 금지 사항

1. **인라인 스타일 금지**: `style={{ }}` 사용 불가
2. **하드코딩 색상 금지**: `#1890ff`, `white` 등 직접 사용 불가
3. **CSS 파일 import 금지**: `.css`, `.scss` 파일 import 불가
4. **Action 객체 패턴 필수**: Zustand store에서 `Action: { ... }` 패턴 사용
5. **$axios 사용 필수**: axios 직접 import 금지, `$axios` 사용

---

## ⛔ 종료 규칙 (CRITICAL)

**Frontend 구현은 모든 파일 생성 완료 후 반드시 종료됩니다.**

- ❌ 다음 단계(verify, PR 등)는 **자동 진행 금지**
- ❌ 요청하지 않은 추가 컴포넌트 **자동 생성 금지**
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행

```
👉 다음 단계:
   1. 커밋: git add . && git commit -m "[TICKET]: Frontend 구현"
   2. 검증: /jira:verify [TICKET] frontend
   3. PR 생성: /jira:pr [TICKET]
```
