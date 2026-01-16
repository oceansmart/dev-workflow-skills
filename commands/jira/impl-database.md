---
name: impl-database
description: "Database 작업 구현 지침"
category: implementation-guide
---

# Database 작업 구현 지침

## 🔗 호출 규칙

**이 지침서는 `/jira:implement [TICKET] --type database` 실행 시 자동 로드됩니다.**

```
/jira:implement 호출
  → --type database 확인
  → impl-database.md 로드
  → 작업 계획 제시 → 사용자 승인 → Dev 환경 테스트 → 실행
```

---

## ⚠️ 필수 준수 사항

### 데이터베이스 환경
| 환경 | 포트 | 용도 |
|------|------|------|
| **Dev (기본)** | **15433** | 개발 데이터, 기본 조회 대상 |
| **Stage** | **15432** | 스테이징 안정화 버전 |

**기본 규칙: 환경 명시 없으면 항상 Dev(15433) 사용**

### 접속 명령어
```bash
# Dev 접속 (기본)
PGPASSWORD=$'Dev!@34ev' psql -h 127.0.0.1 -p 15433 -U pa_dev -d pam_db

# Stage 접속 (명시적 요청 시만)
PGPASSWORD=$'Dev!@34ev' psql -h 127.0.0.1 -p 15432 -U pa_dev -d pam_db

# 스키마 설정
SET search_path TO pa_adm;
```

---

## 스키마 & 테이블 규칙

### 네이밍 규칙
```
스키마: pa_adm
테이블 접두사: mt_ (Master Table)
컬럼: snake_case
```

### 공통 컬럼 (모든 테이블 필수)
```sql
tenant_id VARCHAR(50) NOT NULL,
created_by VARCHAR(100) NOT NULL,
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
updated_by VARCHAR(100),
updated_at TIMESTAMP
```

### UUID 생성
```sql
-- 신규 ID 생성
gen_random_uuid()
```

---

## DDL 작업 체크리스트

### 테이블 생성
```sql
CREATE TABLE pa_adm.mt_feature (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id VARCHAR(50) NOT NULL,

    -- 비즈니스 컬럼
    name VARCHAR(200) NOT NULL,
    status VARCHAR(50) DEFAULT 'ACTIVE',
    description TEXT,

    -- 공통 컬럼 (필수)
    created_by VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by VARCHAR(100),
    updated_at TIMESTAMP
);

-- 인덱스 (tenant_id 필수)
CREATE INDEX idx_mt_feature_tenant_id ON pa_adm.mt_feature(tenant_id);
```

### ✅ DDL 체크리스트
- [ ] 스키마 `pa_adm.` 접두사 사용
- [ ] 테이블명 `mt_` 접두사 사용
- [ ] UUID primary key + `gen_random_uuid()` 기본값
- [ ] `tenant_id` 컬럼 필수 포함
- [ ] 공통 컬럼 4개 포함 (created_by, created_at, updated_by, updated_at)
- [ ] `tenant_id` 인덱스 생성
- [ ] 컬럼명 snake_case 사용

---

## DML 작업 체크리스트

### INSERT
```sql
INSERT INTO pa_adm.mt_feature (
    id,
    tenant_id,
    name,
    status,
    created_by,
    created_at
) VALUES (
    gen_random_uuid(),
    'TEST_TENANT',
    '기능명',
    'ACTIVE',
    'system',
    CURRENT_TIMESTAMP
);
```

### UPDATE
```sql
UPDATE pa_adm.mt_feature
SET
    name = '수정된 기능명',
    updated_by = 'system',
    updated_at = CURRENT_TIMESTAMP
WHERE tenant_id = 'TEST_TENANT'
  AND id = 'uuid-value';
```

### DELETE (논리 삭제 권장)
```sql
-- 논리 삭제 (권장)
UPDATE pa_adm.mt_feature
SET
    status = 'DELETED',
    updated_by = 'system',
    updated_at = CURRENT_TIMESTAMP
WHERE tenant_id = 'TEST_TENANT'
  AND id = 'uuid-value';

-- 물리 삭제 (주의 필요)
DELETE FROM pa_adm.mt_feature
WHERE tenant_id = 'TEST_TENANT'
  AND id = 'uuid-value';
```

### ✅ DML 체크리스트
- [ ] 스키마 `pa_adm.` 접두사 사용
- [ ] INSERT: `created_by`, `created_at` 값 지정
- [ ] UPDATE: `updated_by`, `updated_at` 값 지정
- [ ] WHERE 조건에 `tenant_id` 포함
- [ ] DELETE는 논리 삭제 우선 고려

---

## 쿼리 검증 절차

### Step 1: 스키마 확인
```sql
-- 테이블 존재 확인
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'pa_adm'
  AND table_name = 'mt_feature';

-- 컬럼 구조 확인
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_schema = 'pa_adm'
  AND table_name = 'mt_feature';
```

### Step 2: 데이터 확인
```sql
-- 샘플 데이터 조회
SELECT *
FROM pa_adm.mt_feature
WHERE tenant_id = 'TEST_TENANT'
LIMIT 5;

-- 데이터 건수 확인
SELECT COUNT(*)
FROM pa_adm.mt_feature
WHERE tenant_id = 'TEST_TENANT';
```

### Step 3: 쿼리 테스트
```sql
-- SELECT 테스트 (실행 계획)
EXPLAIN
SELECT column1, column2
FROM pa_adm.mt_table
WHERE tenant_id = :tenantId;

-- 실제 실행 (LIMIT 사용)
SELECT column1, column2
FROM pa_adm.mt_table
WHERE tenant_id = 'TEST_TENANT'
LIMIT 10;
```

---

## MyBatis Mapper 작업

### 신규 쿼리 추가
```xml
<!-- Mapper Interface -->
@Mapper
public interface FeatureMapper {
    List<FeatureResponse> findByTenantId(@Param("tenantId") String tenantId);
}

<!-- Mapper XML -->
<mapper namespace="tys.fms.feature.mapper.FeatureMapper">

    <select id="findByTenantId" resultType="tys.fms.feature.dto.response.FeatureResponse">
        SELECT
            id::text as id,  <!-- UUID 캐스팅 -->
            tenant_id as tenantId,
            name,
            status,
            created_at as createdAt
        FROM pa_adm.mt_feature
        WHERE tenant_id = #{tenantId}
        ORDER BY created_at DESC
    </select>

</mapper>
```

### ✅ MyBatis 체크리스트
- [ ] namespace: 전체 패키지 경로
- [ ] resultType: 전체 DTO 경로
- [ ] UUID 컬럼: `::text` 캐스팅
- [ ] 스키마: `pa_adm.` 접두사
- [ ] 파라미터: `#{paramName}` 사용
- [ ] snake_case → camelCase 별칭 지정

---

## Native Query (Repository) 작업

### 신규 쿼리 추가
```java
@Repository
@RequiredArgsConstructor
public class FeatureRepository {
    private final EntityManager entityManager;

    @Transactional(readOnly = true)
    public List<Object[]> findByTenantId(String tenantId) {
        StringBuilder sql = new StringBuilder();
        sql.append("SELECT id::text, name, status ");
        sql.append("FROM pa_adm.mt_feature ");
        sql.append("WHERE tenant_id = :tenantId ");
        sql.append("ORDER BY created_at DESC");

        Query query = entityManager.createNativeQuery(sql.toString());
        query.setParameter("tenantId", tenantId);

        return query.getResultList();
    }
}
```

### ✅ Repository 체크리스트
- [ ] StringBuilder 사용
- [ ] 스키마 `pa_adm.` 접두사
- [ ] Named Parameter `:paramName`
- [ ] UUID: `::text` 캐스팅
- [ ] 읽기: `@Transactional(readOnly = true)`
- [ ] 쓰기: `@Transactional`

---

## 데이터 마이그레이션

### 마이그레이션 스크립트 구조
```sql
-- 마이그레이션: YYYYMMDD_description.sql
-- 작성자: [이름]
-- 설명: [마이그레이션 목적]

-- 1. 백업 (선택적)
CREATE TABLE pa_adm.mt_feature_backup AS
SELECT * FROM pa_adm.mt_feature;

-- 2. DDL 변경
ALTER TABLE pa_adm.mt_feature
ADD COLUMN new_column VARCHAR(100);

-- 3. 데이터 마이그레이션
UPDATE pa_adm.mt_feature
SET new_column = 'default_value',
    updated_by = 'migration',
    updated_at = CURRENT_TIMESTAMP
WHERE new_column IS NULL;

-- 4. 검증
SELECT COUNT(*) FROM pa_adm.mt_feature WHERE new_column IS NULL;
-- 예상 결과: 0
```

### ✅ 마이그레이션 체크리스트
- [ ] 백업 테이블 생성 (중요 데이터)
- [ ] DDL 변경 전 영향 범위 확인
- [ ] 마이그레이션 스크립트 검증
- [ ] 롤백 계획 준비
- [ ] 검증 쿼리 포함

---

## 🔴 절대 금지 사항

### 1. 기존 쿼리 임의 수정 금지
```
❌ 검증된 SQL 쿼리 임의 변경 금지
❌ 테이블 스키마 임의 변경 금지
❌ 인덱스 임의 삭제 금지

✅ 문제 발견 시 사용자 확인 요청
✅ 변경 계획 승인 후 진행
```

### 2. Production 직접 조작 금지
```
❌ 프로덕션 DB 직접 수정 금지
❌ 검증 없이 대량 UPDATE/DELETE 금지

✅ Dev 환경에서 먼저 테스트
✅ LIMIT 사용하여 영향 범위 확인
```

### 3. 스키마 누락 금지
```sql
-- ❌ 잘못된 쿼리
SELECT * FROM mt_feature

-- ✅ 올바른 쿼리
SELECT * FROM pa_adm.mt_feature
```

---

## 쿼리 최적화 가이드

### 인덱스 활용
```sql
-- tenant_id 인덱스 활용 (권장)
SELECT * FROM pa_adm.mt_feature
WHERE tenant_id = 'TEST_TENANT';

-- 복합 조건 시 인덱스 순서 고려
WHERE tenant_id = 'TEST_TENANT'
  AND status = 'ACTIVE';
```

### 대량 데이터 처리
```sql
-- 페이지네이션 사용
SELECT *
FROM pa_adm.mt_feature
WHERE tenant_id = 'TEST_TENANT'
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;

-- 커서 기반 페이지네이션 (대용량)
SELECT *
FROM pa_adm.mt_feature
WHERE tenant_id = 'TEST_TENANT'
  AND created_at < :lastCreatedAt
ORDER BY created_at DESC
LIMIT 20;
```

---

## 작업 전 승인 절차 (사용자 확인 필수)

### 작업 계획 제시 템플릿
```
📋 Database 작업 계획

## 티켓 정보
- 티켓: [TICKET-ID]
- 제목: [이슈 제목]

## 작업 내용
| 작업 유형 | 대상 | 쿼리 요약 |
|----------|------|----------|
| DDL | pa_adm.mt_xxx | ALTER TABLE ADD COLUMN |
| DML | pa_adm.mt_xxx | UPDATE SET ... |

## 영향 범위
- 예상 영향 행 수: [건수]
- 관련 API: [API 목록]
- 롤백 가능 여부: 예/아니오

## 체크리스트
- [ ] pa_adm 스키마 접두사 적용
- [ ] Dev 환경에서 테스트 예정
- [ ] 롤백 계획 준비

---
위 계획대로 진행할까요? (예/아니오)
```

**→ 사용자 승인("예", "진행", "승인") 후에만 작업 시작**

---

## 작업 결과 보고 템플릿

```
## 🗄️ Database 작업 완료

### 작업 요약
- 티켓: [SMFD-XXX]
- 작업 유형: [DDL/DML/마이그레이션]
- 대상 테이블: pa_adm.mt_xxx

### 실행 내용
| 작업 | 쿼리 | 영향 행 |
|------|------|---------|
| DDL | ALTER TABLE ... | - |
| DML | UPDATE ... | 150 |

### 검증 결과
- ✅ 테이블 구조 정상
- ✅ 데이터 무결성 유지
- ✅ 관련 API 정상 동작

### 롤백 방법 (필요 시)
```sql
-- 롤백 쿼리
```
```

---

## ⛔ 종료 규칙 (CRITICAL)

**Database 작업은 결과 보고 후 반드시 종료됩니다.**

- ❌ 다음 단계(verify, PR 등)는 **자동 진행 금지**
- ❌ 요청하지 않은 추가 쿼리 **자동 실행 금지**
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행

```
👉 다음 단계:
   1. 커밋: git add . && git commit -m "[TICKET]: Database 작업"
   2. 검증: /jira:verify [TICKET] backend
   3. PR 생성: /jira:pr [TICKET]
```
