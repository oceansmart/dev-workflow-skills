---
name: impl-backend-api
description: "Backend API 개발 구현 지침"
category: implementation-guide
---

# Backend API 개발 구현 지침

## 🔗 호출 규칙

**이 지침서는 `/jira:implement [TICKET] --type backend-api` 실행 시 자동 로드됩니다.**

```
/jira:implement 호출
  → --type backend-api 확인
  → impl-backend-api.md 로드
  → 구현 계획 제시 → 사용자 승인 → 순차 구현
```

---

## 🔴 프로젝트 공통 패턴 (필수 숙지)

**⚠️ 아래 패턴들은 이 프로젝트의 기존 코드에서 일관되게 사용되는 규칙입니다. 새 코드 작성 전 반드시 숙지하세요.**

### 1. 데이터베이스 스키마 규칙
```
스키마: pa_adm (모든 테이블에 접두사 필수)
테이블: mt_* (Master Table), 예: pa_adm.mt_vessel, pa_adm.mt_booking_container
WHERE 조건: tenant_id 필수 포함
```

### 2. UUID → String 변환 (CRITICAL)
```java
// ❌ 문제: PostgreSQL UUID가 Java에서 null 반환
private UUID proformaId;

// ✅ 해결: Response DTO에서 String 사용
private String proformaId;

// MyBatis XML: ::text 캐스팅 필수
SELECT proforma_id::text AS proformaId
```

### 3. 공통 컬럼 패턴
```java
// 모든 테이블 필수 컬럼
tenant_id VARCHAR(50) NOT NULL  // 테넌트 식별
created_by VARCHAR(100)         // 생성자
created_at TIMESTAMP            // 생성일시
updated_by VARCHAR(100)         // 수정자
updated_at TIMESTAMP            // 수정일시
```

### 4. Response 래핑 패턴
```java
// 모든 API 응답은 CommonResponse로 래핑
return ResponseEntity.ok(CommonResponse.<ResponseDTO>builder()
        .isSuccess(true)
        .data(response)
        .build());
```

### 5. MyBatis resultType 규칙
```xml
<!-- 전체 패키지 경로 사용 필수 -->
resultType="tys.fms.{feature}.dto.response.{ResponseDTO}"

<!-- 예시 -->
resultType="tys.fms.booking.dto.response.BookingResponse"
resultType="tys.fms.container.dto.response.ContainerStatusResponse"
```

### 6. 기존 코드 참조 위치
| 유형 | 참조 경로 | 확인 내용 |
|------|----------|----------|
| DTO | `tys/fms/{feature}/dto/` | 기존 Request/Response 패턴 |
| Service | `tys/fms/{feature}/service/` | 비즈니스 로직 패턴 |
| Repository | `tys/fms/{feature}/repository/` | Native Query 패턴 |
| Mapper | `src/main/resources/mapper/` | MyBatis XML 패턴 |

**🔴 새 DTO/Service/Repository 생성 전 반드시 동일 feature의 기존 파일 확인!**

---

## ⚠️ 필수 준수 사항

### 1. 패키지 구조
```
tys/fms/{feature}/
├── controller/         # REST API Controller
├── service/           # Service Interface
│   └── impl/          # Service Implementation
├── dto/
│   ├── request/       # Request DTO
│   └── response/      # Response DTO
├── repository/        # JPA Repository (Native Query)
└── mapper/           # MyBatis Mapper Interface
```

---

## Controller 생성 체크리스트

### 필수 어노테이션
```java
@Tag(name = "Feature Name", description = "기능 설명")
@Slf4j
@RestController
@RequestMapping("/api/v1/{feature}")
@RequiredArgsConstructor
public class FeatureController {
    private final FeatureService featureService;
}
```

### 메서드 패턴
```java
@Operation(summary = "기능 요약", description = "기능 상세 설명")
@ApiResponses(value = {
    @ApiResponse(responseCode = "200", description = "성공"),
    @ApiResponse(responseCode = "400", description = "잘못된 요청"),
    @ApiResponse(responseCode = "500", description = "서버 에러")
})
@PostMapping("/endpoint")
public ResponseEntity<CommonResponse<ResponseDTO>> methodName(
        @Valid @RequestBody RequestDTO request) {
    try {
        ResponseDTO response = featureService.processMethod(request);
        return ResponseEntity.ok(CommonResponse.<ResponseDTO>builder()
                .isSuccess(true)
                .data(response)
                .build());
    } catch (Exception e) {
        log.error("에러 메시지: {}", e.getMessage(), e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(CommonResponse.<ResponseDTO>builder()
                        .isSuccess(false)
                        .message(MessageResponse.builder()
                                .code("500")
                                .message("에러 메시지")
                                .build())
                        .build());
    }
}
```

### ✅ Controller 체크리스트
- [ ] `@Tag` 어노테이션에 API 그룹명과 설명 포함
- [ ] `@Operation`, `@ApiResponses` Swagger 문서화 완료
- [ ] 반환 타입: `ResponseEntity<CommonResponse<T>>`
- [ ] `@Valid` 어노테이션으로 요청 검증
- [ ] try-catch로 에러 핸들링
- [ ] log.error로 에러 로깅 (스택 트레이스 포함)

---

## Request DTO 생성 체크리스트

### 필수 어노테이션
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Schema(description = "요청 DTO 설명")
public class FeatureRequest {

    @NotBlank(message = "tenantId는 필수입니다")
    @Schema(description = "테넌트 ID", required = true, example = "TEST_TENANT")
    private String tenantId;

    @NotNull(message = "필드는 필수입니다")
    @Schema(description = "필드 설명", required = true, example = "1")
    private Integer field;

    @Valid
    @Schema(description = "중첩 객체")
    private NestedObject nestedObject;

    // 중첩 클래스 (Inner Class)
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Schema(description = "중첩 객체 설명")
    public static class NestedObject {
        @NotBlank(message = "필드는 필수입니다")
        @Schema(description = "필드 설명", required = true)
        private String field;
    }
}
```

### ✅ Request DTO 체크리스트
- [ ] `@Data`, `@NoArgsConstructor`, `@AllArgsConstructor` 포함
- [ ] `@Schema(description = "...")` 클래스 레벨 설명
- [ ] 필수 필드에 `@NotBlank` 또는 `@NotNull` 검증
- [ ] 모든 필드에 `@Schema` 문서화
- [ ] 중첩 객체에 `@Valid` 적용
- [ ] 중첩 클래스는 `public static class`로 선언

---

## Response DTO 생성 체크리스트

### 필수 어노테이션
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Schema(description = "응답 DTO 설명")
public class FeatureResponse {

    @Schema(description = "필드 설명")
    private String field;

    @Schema(description = "중첩 응답 객체")
    private NestedResponse nestedResponse;

    @Schema(description = "목록")
    private List<ItemResponse> items;
}
```

### 🔴 PostgreSQL UUID → Java 매핑 (CRITICAL)
```java
// ❌ 잘못된 방식 - API 응답에서 null 반환
private UUID proformaId;

// ✅ 올바른 방식 - String 사용
private String proformaId;
```

MyBatis XML에서 `::text` 캐스팅 필수:
```xml
<!-- ❌ Before -->
SELECT proforma_id AS proformaId

<!-- ✅ After -->
SELECT proforma_id::text AS proformaId
```

### ✅ Response DTO 체크리스트
- [ ] `@Data`, `@Builder`, `@NoArgsConstructor`, `@AllArgsConstructor` 포함
- [ ] `@Schema(description = "...")` 클래스 및 필드 레벨 설명
- [ ] UUID 타입은 String으로 선언
- [ ] MyBatis XML에서 `::text` 캐스팅 적용

---

## Service 생성 체크리스트

### Interface 패턴
```java
public interface FeatureService {
    /**
     * 기능 설명
     * @param request 요청 정보
     * @return 응답 정보
     */
    FeatureResponse processMethod(FeatureRequest request);
}
```

### Implementation 패턴
```java
@Slf4j
@Service
@RequiredArgsConstructor
public class FeatureServiceImpl implements FeatureService {
    private final FeatureRepository featureRepository;
    private final FeatureMapper featureMapper;

    @Override
    @Transactional
    public FeatureResponse processMethod(FeatureRequest request) {
        // 비즈니스 로직 구현
    }
}
```

### ✅ Service 체크리스트
- [ ] Interface와 Implementation 분리
- [ ] Implementation에 `@Slf4j`, `@Service`, `@RequiredArgsConstructor`
- [ ] 쓰기 작업에 `@Transactional` 적용
- [ ] 읽기 전용 작업에 `@Transactional(readOnly = true)` 적용
- [ ] Javadoc 주석으로 메서드 설명

---

## Repository 생성 체크리스트

### Native Query 패턴
```java
@Repository
@RequiredArgsConstructor
public class FeatureRepository {
    private final EntityManager entityManager;

    @Transactional(readOnly = true)
    public List<Object[]> findData(String tenantId, String param) {
        StringBuilder sql = new StringBuilder();
        sql.append("SELECT column1, column2 ");
        sql.append("FROM pa_adm.table_name ");
        sql.append("WHERE tenant_id = :tenantId ");

        if (param != null && !param.isEmpty()) {
            sql.append("AND column = :param ");
        }

        Query query = entityManager.createNativeQuery(sql.toString());
        query.setParameter("tenantId", tenantId);
        if (param != null && !param.isEmpty()) {
            query.setParameter("param", param);
        }

        return query.getResultList();
    }

    @Transactional
    public int updateData(String tenantId, String value) {
        StringBuilder sql = new StringBuilder();
        sql.append("UPDATE pa_adm.table_name SET column = :value ");
        sql.append("WHERE tenant_id = :tenantId");

        Query query = entityManager.createNativeQuery(sql.toString());
        query.setParameter("tenantId", tenantId);
        query.setParameter("value", value);

        return query.executeUpdate();
    }
}
```

### ✅ Repository 체크리스트
- [ ] `@Repository`, `@RequiredArgsConstructor` 적용
- [ ] EntityManager 주입
- [ ] 읽기: `@Transactional(readOnly = true)`
- [ ] 쓰기: `@Transactional`
- [ ] StringBuilder로 동적 쿼리 구성
- [ ] Named Parameter (`:paramName`) 사용
- [ ] 스키마 `pa_adm.` 접두사 사용

---

## MyBatis Mapper 생성 체크리스트

### Mapper Interface
```java
@Mapper
public interface FeatureMapper {
    List<ResponseDTO> findByTenantId(@Param("tenantId") String tenantId);
    int insert(RequestDTO request);
    int update(RequestDTO request);
}
```

### Mapper XML
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="tys.fms.{feature}.mapper.FeatureMapper">

    <!-- 단건 조회 -->
    <select id="findById" resultType="dto.package.ResponseDTO">
        SELECT
            column1 as field1,
            column2 as field2,
            id::text as id  <!-- UUID는 ::text 캐스팅 -->
        FROM pa_adm.table_name
        WHERE tenant_id = #{tenantId}
          AND id = #{id}
    </select>

    <!-- INSERT -->
    <insert id="insert">
        INSERT INTO pa_adm.table_name (
            id, tenant_id, column1, created_by, created_at
        ) VALUES (
            gen_random_uuid(),
            #{tenantId},
            #{column1},
            #{createdBy},
            CURRENT_TIMESTAMP
        )
    </insert>

    <!-- UPDATE -->
    <update id="update">
        UPDATE pa_adm.table_name
        SET
            column1 = #{column1},
            updated_by = #{updatedBy},
            updated_at = CURRENT_TIMESTAMP
        WHERE tenant_id = #{tenantId}
          AND id = #{id}
    </update>

</mapper>
```

### ✅ MyBatis Mapper 체크리스트
- [ ] Mapper Interface에 `@Mapper` 어노테이션
- [ ] XML 네임스페이스: `tys.fms.{feature}.mapper.FeatureMapper`
- [ ] resultType: 전체 패키지 경로 (예: `tys.fms.feature.dto.response.FeatureResponse`)
- [ ] UUID 컬럼에 `::text` 캐스팅
- [ ] 스키마 `pa_adm.` 접두사 사용
- [ ] INSERT에 `gen_random_uuid()` 사용
- [ ] 공통 컬럼: `created_by`, `created_at`, `updated_by`, `updated_at`

---

## 파일 생성 순서

1. **Request DTO** → `dto/request/FeatureRequest.java`
2. **Response DTO** → `dto/response/FeatureResponse.java`
3. **Service Interface** → `service/FeatureService.java`
4. **Repository 또는 Mapper**
   - 단순 쿼리: Repository (Native Query)
   - 복잡 쿼리: MyBatis Mapper + XML
5. **Service Implementation** → `service/impl/FeatureServiceImpl.java`
6. **Controller** → `controller/FeatureController.java`

---

## 구현 전 승인 절차 (사용자 확인 필수)

### 구현 계획 제시 템플릿
```
📋 Backend API 구현 계획

## 티켓 정보
- 티켓: [TICKET-ID]
- 제목: [이슈 제목]

## 생성/수정 파일 목록
| # | 파일 경로 | 작업 | 설명 |
|---|----------|------|------|
| 1 | dto/request/FeatureRequest.java | 생성 | Request DTO |
| 2 | dto/response/FeatureResponse.java | 생성 | Response DTO |
| 3 | service/FeatureService.java | 생성 | Service Interface |
| 4 | mapper/FeatureMapper.java | 생성 | MyBatis Mapper |
| 5 | service/impl/FeatureServiceImpl.java | 생성 | Service 구현체 |
| 6 | controller/FeatureController.java | 생성 | REST Controller |

## 체크리스트 준수 확인
- [ ] 패키지 구조 준수
- [ ] 필수 어노테이션 적용
- [ ] UUID → String 변환 적용
- [ ] pa_adm 스키마 접두사 적용

---
위 계획대로 구현을 진행할까요? (예/아니오)
```

**→ 사용자 승인("예", "진행", "승인") 후에만 구현 시작**

---

## 🔴 절대 금지 사항

1. **쿼리 임의 수정 금지**: 기존 Repository/Mapper의 SQL 쿼리는 검증된 비즈니스 로직
2. **스키마 누락 금지**: 모든 테이블에 `pa_adm.` 스키마 접두사 필수
3. **UUID 타입 사용 금지**: Response DTO에서 UUID 대신 String 사용
4. **CommonResponse 누락 금지**: 모든 API 응답은 `CommonResponse<T>`로 감싸기
5. **Validation 누락 금지**: Request DTO에 적절한 검증 어노테이션 필수

---

## ⛔ 종료 규칙 (CRITICAL)

**Backend API 구현은 모든 파일 생성 완료 후 반드시 종료됩니다.**

- ❌ 다음 단계(verify, PR 등)는 **자동 진행 금지**
- ❌ 요청하지 않은 추가 기능 **자동 구현 금지**
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행

```
👉 다음 단계:
   1. 커밋: git add . && git commit -m "[TICKET]: Backend API 구현"
   2. 검증: /jira:verify [TICKET] backend
   3. PR 생성: /jira:pr [TICKET]
```
