---
name: impl-bugfix
description: "버그 수정 구현 지침"
category: implementation-guide
---

# 버그 수정 구현 지침

## 🔗 호출 규칙

**이 지침서는 `/jira:implement [TICKET] --type bugfix` 실행 시 자동 로드됩니다.**

```
/jira:implement 호출
  → --type bugfix 확인
  → impl-bugfix.md 로드
  → Phase 1-4 순차 진행
```

---

## ⚠️ 버그 수정 원칙

### 핵심 원칙
1. **최소 변경**: 버그 수정에 필요한 최소한의 코드만 변경
2. **원인 분석 우선**: 코드 수정 전 반드시 근본 원인 파악
3. **영향 범위 확인**: 수정으로 인한 사이드 이펙트 분석
4. **테스트 검증**: 수정 후 관련 기능 테스트 수행

---

## Phase 1: 버그 분석 (사용자 확인 필요)

### 1.1 에러 정보 수집
```
📋 버그 분석 체크리스트

1. 에러 메시지/로그 확인
   - [ ] 프론트엔드 콘솔 에러
   - [ ] 백엔드 서버 로그
   - [ ] API 응답 내용

2. 재현 조건 확인
   - [ ] 특정 입력값
   - [ ] 특정 사용자/권한
   - [ ] 특정 환경 (dev/stage/prod)

3. 발생 시점 확인
   - [ ] 언제부터 발생?
   - [ ] 최근 배포/변경 사항?
   - [ ] 특정 조건에서만 발생?
```

### 1.2 코드 추적
```
📍 코드 추적 순서

Frontend 에러:
1. 에러 발생 컴포넌트 확인
2. Store → API 모듈 → 백엔드 추적
3. 타입 정의 확인

Backend 에러:
1. Controller → Service → Repository 추적
2. SQL 쿼리 확인
3. DTO 매핑 확인

데이터 에러:
1. 테이블 스키마 확인
2. 데이터 무결성 확인
3. null/undefined 케이스 확인
```

### 1.3 분석 결과 보고 및 확인 요청
```
📋 Phase 1 분석 결과

## 버그 현상
- 에러 메시지: [에러 내용]
- 발생 위치: [파일:라인]
- 재현 조건: [재현 방법]

## 추적 결과
- 원인 파일: [파일 경로]
- 원인 추정: [추정 원인]

## 확인 요청
위 분석 결과가 맞습니까? (예/수정 필요)
```
**→ 사용자 확인 후 Phase 2 진행**

---

## Phase 2: 원인 분류 (사용자 확인 필요)

### 일반적인 버그 유형

#### 2.1 타입 불일치 (Frontend)
```typescript
// ❌ 문제: API 응답 타입 불일치
interface Response {
  userId: string;  // 실제로는 number 반환
}

// ✅ 해결: 백엔드 응답에 맞게 타입 수정
interface Response {
  userId: number;
}
```

#### 2.2 UUID 매핑 오류 (Backend)
```java
// ❌ 문제: PostgreSQL UUID가 null로 매핑
private UUID proformaId;  // API 응답에서 null

// ✅ 해결: String 타입 + ::text 캐스팅
private String proformaId;

// MyBatis XML
SELECT proforma_id::text AS proformaId
```

#### 2.3 null 체크 누락
```java
// ❌ 문제: NullPointerException 발생
String result = entity.getValue().toString();

// ✅ 해결: null 체크 추가
String result = entity.getValue() != null
    ? entity.getValue().toString() : null;
```

#### 2.4 비동기 상태 관리 오류 (Frontend)
```typescript
// ❌ 문제: 상태 업데이트 누락
const fetchData = async () => {
  const response = await api.fetch();
  // isLoading false 처리 누락
};

// ✅ 해결: try-catch-finally로 상태 관리
const fetchData = async () => {
  try {
    set({ isLoading: true });
    const response = await api.fetch();
    set({ data: response.data });
  } catch (error) {
    set({ error: error.message });
  } finally {
    set({ isLoading: false });
  }
};
```

#### 2.5 SQL 쿼리 오류
```sql
-- ❌ 문제: 스키마 누락
SELECT * FROM mt_table WHERE id = :id

-- ✅ 해결: pa_adm 스키마 추가
SELECT * FROM pa_adm.mt_table WHERE id = :id
```

### 2.6 원인 분류 결과 보고
```
📋 Phase 2 분류 결과

## 버그 유형
- 분류: [타입 불일치 / UUID 매핑 / null 체크 / 비동기 상태 / SQL 오류]
- 참조 패턴: 섹션 2.[번호]

## 예상 수정 범위
- 수정 파일: [파일 수] 개
- 영향 범위: [관련 API/화면]

위 분류가 맞습니까? (예/수정 필요)
```
**→ 사용자 확인 후 Phase 3 진행**

---

## Phase 3: 수정 절차 (승인 필수)

### 3.1 수정 계획 문서화
```
📝 수정 계획 템플릿

## 버그 설명
- 현상: [무엇이 잘못되는지]
- 원인: [왜 발생하는지]
- 영향: [어떤 기능에 영향 미치는지]

## 수정 내용
| 파일 | 변경 내용 | 이유 |
|------|----------|------|
| FeatureResponse.java | UUID → String | PostgreSQL 매핑 |
| FeatureMapper.xml | ::text 추가 | 타입 캐스팅 |

## 영향 범위
- 관련 API: /api/v1/feature
- 관련 화면: FeaturePage
- 테스트 항목: [테스트할 시나리오]

## 승인 요청
위 수정 계획대로 진행할까요?
```

### 3.2 수정 순서
```
1️⃣ 원인이 되는 코드 수정
   - 단일 파일 수정 권장
   - 연쇄적 수정 필요 시 의존성 순서대로

2️⃣ 관련 코드 동기화
   - DTO ↔ Mapper ↔ Response 타입 일치
   - Frontend 타입 ↔ Backend 응답 일치

3️⃣ 검증
   - API 테스트 (Swagger/Postman)
   - 화면 테스트 (브라우저)
```

---

## Phase 4: 검증

### 4.1 검증 체크리스트
```
✅ 버그 수정 검증

1. 직접 검증
   - [ ] 버그 재현 시도 → 수정 확인
   - [ ] 동일 시나리오 정상 동작

2. 사이드 이펙트 검증
   - [ ] 관련 기능 정상 동작
   - [ ] 다른 API 영향 없음
   - [ ] 다른 화면 영향 없음

3. 데이터 검증
   - [ ] 기존 데이터 조회 정상
   - [ ] 신규 데이터 생성 정상
   - [ ] 데이터 무결성 유지
```

### 4.2 데이터베이스 검증 쿼리
```sql
-- Dev 환경 접속
PGPASSWORD=$'Dev!@34ev' psql -h 127.0.0.1 -p 15433 -U pa_dev -d pam_db

-- 스키마 설정
SET search_path TO pa_adm;

-- 데이터 확인 쿼리 예시
SELECT column1, column2
FROM mt_table
WHERE tenant_id = 'TEST_TENANT'
LIMIT 10;
```

---

## 🔴 절대 금지 사항

### 버그 수정 시 금지 행위

1. **임의 리팩토링 금지**
   - 버그와 무관한 코드 정리 금지
   - "이왕 하는 김에" 추가 개선 금지
   - 범위 밖 수정 금지

2. **기존 쿼리 수정 금지**
   - 검증된 SQL 쿼리 임의 변경 금지
   - 쿼리 문제 발견 시 사용자 확인 필수

3. **승인 없는 수정 금지**
   - 수정 계획 제시 필수
   - 사용자 승인 후 진행

4. **테스트 스킵 금지**
   - 수정 후 검증 필수
   - 검증 결과 보고 필수

---

## 일반적인 버그 수정 패턴

### 패턴 1: UUID null 반환
```
원인: PostgreSQL UUID → Java UUID 매핑 실패

수정:
1. Response DTO: UUID → String
2. MyBatis XML: ::text 캐스팅 추가
3. Service: .toString() 변환 (Entity → DTO)
```

### 패턴 2: API 응답 타입 불일치
```
원인: Backend 응답과 Frontend 타입 불일치

수정:
1. Backend Response DTO 구조 확인
2. Frontend 타입 정의 일치시킴
3. API 모듈 반환 타입 수정
```

### 패턴 3: 조건 분기 누락
```
원인: 특정 케이스 처리 로직 없음

수정:
1. 누락된 조건 식별
2. 분기 처리 추가
3. 기존 분기에 영향 없음 확인
```

### 패턴 4: 상태 관리 오류
```
원인: 비동기 처리 중 상태 업데이트 누락

수정:
1. try-catch-finally 구조로 변경
2. 모든 경로에서 isLoading false
3. 에러 상태 적절히 처리
```

---

## 수정 결과 보고 템플릿

```
## 🐛 버그 수정 완료

### 버그 요약
- 티켓: [SMFD-XXX]
- 현상: [버그 설명]
- 원인: [근본 원인]

### 수정 내용
| 파일 | 변경 | 라인 |
|------|------|------|
| file.java | 타입 변경 | L25 |
| mapper.xml | 캐스팅 추가 | L42 |

### 검증 결과
- ✅ 버그 재현 불가 (수정됨)
- ✅ 관련 기능 정상 동작
- ✅ 사이드 이펙트 없음

### 테스트 방법
1. [테스트 시나리오]
2. [확인 방법]
```

---

## ⛔ 종료 규칙 (CRITICAL)

**버그 수정 구현은 Phase 4 결과 보고 후 반드시 종료됩니다.**

- ❌ 다음 단계(verify, PR 등)는 **자동 진행 금지**
- ❌ 버그와 무관한 추가 수정 **자동 진행 금지**
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행

```
👉 다음 단계:
   1. 커밋: git add . && git commit -m "[TICKET]: 버그 수정"
   2. 검증: /jira:verify [TICKET] [both|frontend|backend]
   3. PR 생성: /jira:pr [TICKET]
```
