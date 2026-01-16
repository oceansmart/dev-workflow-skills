---
name: implement
description: "Jira 티켓 구현 - 타입별 구현 지침에 따른 코드 작성"
category: project-management
complexity: enhanced
mcp-servers: [jira]
personas: []
---

## ⛔⛔⛔ HARD STOP RULE (최우선 규칙) ⛔⛔⛔

**이 명령어는 모든 구현 작업에서 사용자 승인을 필수로 요구합니다.**

```
🚨 CRITICAL ENFORCEMENT:
- 코드 작성 전 → 구현 계획 제시 → 사용자 승인 대기
- 승인 없이 코드 작성 시작 → 즉시 중단
- 각 파일 생성/수정 전 → 변경 계획 제시 → 승인 후 진행
- impl-*.md 지침서 패턴 위반 → 작성 금지
```

---

# /jira:implement - Jira 티켓 구현

## Usage
```
/jira:implement [TICKET-ID] --type [TYPE]
```

## Parameters
- **TICKET-ID**: Jira 이슈 키 (예: SMFD-286)
- **--type**: 개발 타입 (단일 또는 복합)
  - 단일: `backend-api` | `frontend` | `bugfix` | `database`
  - 복합: `backend-api,frontend` | `database,backend-api` 등 (콤마로 구분)
  - 편의: `fullstack` = `backend-api,frontend`

## 개발 타입별 지침서

| 타입 | 지침서 파일 | 적용 범위 |
|------|------------|----------|
| `backend-api` | impl-backend-api.md | Controller, Service, DTO, Repository, Mapper |
| `frontend` | impl-frontend.md | API 모듈, Store, 타입, 컴포넌트 |
| `bugfix` | impl-bugfix.md | 버그 분석 및 최소 수정 |
| `database` | impl-database.md | DDL, DML, 마이그레이션 |
| `fullstack` | impl-backend-api.md + impl-frontend.md | Backend + Frontend 전체 |

---

## Behavioral Flow

### Phase 0: 설정 파일 확인 (필수 - BLOCKING)

**⛔ 이 단계를 통과하지 않으면 어떤 작업도 진행할 수 없습니다.**

1. `.claude/config/jira.md` 파일 존재 여부 확인
2. **파일 있음**: 설정값 로드 → Phase 1 진행
3. **파일 없음**: 아래 메시지 출력 후 **완전 중단**

```
⛔ Jira 설정 파일이 없습니다.

/jira:* 명령어를 사용하려면 먼저 설정 파일이 필요합니다.

📁 필요한 파일: .claude/config/jira.md

설정 파일을 생성할까요? (예/아니오)

→ "예" 선택 시: 설정 정보 입력 → 파일 생성 → 다시 명령어 실행
→ "아니오" 선택 시: 명령어 종료
```

**⚠️ 설정 파일이 생성되기 전까지 Phase 1 이후 단계는 절대 실행 불가**

---

### Phase 1: 🔴 컨텍스트 로드 (Critical)
```
1. project-context.md 로드
   - 프로젝트 구조 확인
   - 기술 스택 확인
   - 필수 규칙 확인

2. 해당 impl-*.md 지침서 로드
   - 파일 생성 순서
   - 필수 패턴
   - 체크리스트

3. Jira 티켓 상세 조회
   - jira_get_issue(issue_key="[TICKET]", fields="*all")
   - description에서 상세 요구사항 추출
```

### Phase 2: 🔴 구현 계획 제시 및 승인 요청 (Critical)
```
📋 구현 계획서

## 티켓 정보
- 티켓: [TICKET-ID]
- 제목: [SUMMARY]
- 타입: [TYPE]

## 구현 범위

### 생성/수정 파일 목록
| # | 파일 경로 | 작업 | 설명 |
|---|----------|------|------|
| 1 | path/to/File.java | 생성 | Request DTO |
| 2 | path/to/File.java | 생성 | Response DTO |
| 3 | ... | ... | ... |

### 상세 구현 계획

#### 1. [첫 번째 파일명]
- **패턴**: [적용할 패턴명]
- **내용 요약**:
  - 필드: field1, field2, ...
  - 검증: @NotBlank, @NotNull, ...

#### 2. [두 번째 파일명]
- **패턴**: [적용할 패턴명]
- **내용 요약**:
  - ...

### 관련 테이블/쿼리
- 테이블: pa_adm.mt_xxx
- 주요 쿼리: SELECT/INSERT/UPDATE

## 체크리스트 준수 확인
- [ ] impl-*.md 필수 패턴 적용
- [ ] project-context.md 규칙 준수
- [ ] 기존 쿼리 수정 없음

---
위 계획대로 구현을 진행할까요? (예/아니오)
```

**→ 사용자 승인("예", "진행", "승인") 후에만 Phase 3 진행**
**→ 수정 요청 시 계획 수정 후 재승인 요청**

### Phase 3: 🔴 파일별 순차 구현 (Critical)
```
각 파일마다:

1. 구현 시작 알림
   "📝 [1/N] FeatureRequest.java 생성 중..."

2. impl-*.md 체크리스트 적용
   - 필수 어노테이션 확인
   - 필수 패턴 적용
   - 금지 사항 준수

3. 파일 생성/수정

4. 완료 알림
   "✅ [1/N] FeatureRequest.java 완료"

5. 다음 파일로 진행
```

### Phase 4: 구현 완료 보고
```
🎯 구현 완료

## [TICKET-ID] 구현 결과

### 생성/수정된 파일
| # | 파일 | 상태 |
|---|------|------|
| 1 | path/to/File.java | ✅ 생성 |
| 2 | path/to/File.java | ✅ 생성 |
| ... | ... | ... |

### 체크리스트 준수
- ✅ impl-*.md 패턴 적용
- ✅ project-context.md 규칙 준수
- ✅ 기존 쿼리 수정 없음

### 다음 단계 (순서 필수!)
1. 커밋: git add . && git commit -m "[TICKET]: ..."
2. 🔴 검증 필수: /jira:verify [TICKET] [both|frontend|backend]
   ⚠️ verify 통과 전까지 PR 생성 금지!
3. PR 생성: /jira:pr [TICKET]
```

## ⛔ 종료 규칙 (CRITICAL)

**이 명령어는 Phase 4 결과 반환 후 반드시 종료됩니다.**

- ❌ 다음 단계(verify, PR 등)는 **자동 진행 금지**
- ❌ 사용자의 명시적 요청 없이 다른 `/jira:*` 명령 실행 금지
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행

---

## 타입별 세부 흐름

### backend-api 타입
```
순서:
1. Request DTO → dto/request/
2. Response DTO → dto/response/
3. Service Interface → service/
4. Repository 또는 Mapper → repository/ 또는 mapper/
5. Service Implementation → service/impl/
6. Controller → controller/

각 단계에서 impl-backend-api.md 체크리스트 적용
```

### frontend 타입
```
순서:
1. TypeScript 타입 → types/
2. API 모듈 → apis/
3. Zustand Store → store/
4. 페이지 컴포넌트 → pages/fms/

각 단계에서 impl-frontend.md 체크리스트 적용
⚠️ 스타일링: styled-components + 테마 토큰 필수!
```

### bugfix 타입
```
순서:
1. 버그 분석 (impl-bugfix.md Phase 1-2)
2. 수정 계획 제시 및 승인
3. 최소 범위 수정
4. 검증 결과 보고

⚠️ 버그와 무관한 코드 수정 금지!
```

### database 타입
```
순서:
1. 스키마/테이블 확인
2. DDL/DML 계획 제시
3. Dev 환경에서 쿼리 테스트
4. 실행 및 검증

⚠️ 기존 쿼리 임의 수정 금지!
```

### fullstack 타입
```
순서:
1. Backend 구현 (backend-api 순서)
2. API 테스트 (Swagger)
3. Frontend 구현 (frontend 순서)
4. 통합 테스트

⚠️ Backend → Frontend 순서 필수!
```

---

## 🔴 절대 금지 사항

### 승인 없는 작업 금지
```
❌ 계획 제시 없이 코드 작성 시작
❌ 승인 없이 파일 생성/수정
❌ "일단 만들어보겠습니다" → 금지
```

### 패턴 위반 금지
```
❌ impl-*.md 체크리스트 미적용
❌ project-context.md 규칙 위반
❌ 기존 쿼리 임의 수정
```

### 범위 초과 금지
```
❌ 요청하지 않은 기능 추가
❌ "개선을 위해" 추가 리팩토링
❌ 범위 밖 파일 수정
```

---

## Error Handling

### 티켓 정보 부족 시
```
⚠️ 티켓 정보 부족

[TICKET-ID]의 description이 불충분합니다.

필요한 정보:
- [ ] 구체적인 기능 요구사항
- [ ] 관련 테이블/API 정보
- [ ] UI 요구사항 (frontend)

추가 정보를 입력해주세요:
```

### 타입 불일치 시
```
⚠️ 타입 불일치 의심

티켓 내용과 지정된 타입이 맞지 않습니다.

티켓 키워드: "화면 추가", "UI 컴포넌트"
지정 타입: backend-api

올바른 타입: frontend

타입을 변경하시겠습니까? (frontend / 현재 유지)
```

### 지침서 위반 감지 시
```
⚠️ 패턴 위반 감지

impl-backend-api.md 위반:
- Response DTO에 UUID 타입 사용 (→ String 필요)

수정 후 계속하시겠습니까?
```

---

## Examples

```bash
# Backend API 구현
/jira:implement SMFD-286 --type backend-api

# Frontend 화면 구현
/jira:implement SMFD-287 --type frontend

# 버그 수정
/jira:implement SMFD-288 --type bugfix

# Database 작업
/jira:implement SMFD-289 --type database

# Fullstack 기능 구현
/jira:implement SMFD-290 --type fullstack
```

---

## Pre-conditions
- `/jira:start` 실행 완료 (feature 브랜치 생성됨)
- 해당 브랜치에서 실행
- impl-*.md 지침서 파일 존재
- project-context.md 파일 존재

## Post-conditions
- 모든 파일이 지침서 패턴을 준수
- 빌드/타입체크 통과
- 사용자 검증 완료
