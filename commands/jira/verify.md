---
name: verify
description: "Jira 티켓 완료 전 검증 프로세스 실행"
category: project-management
complexity: enhanced
mcp-servers: [jira, playwright]
personas: []
---

## ⛔⛔⛔ MANDATORY GATE RULE (최우선 규칙) ⛔⛔⛔

**이 명령어는 모든 검증이 통과될 때까지 탈출 불가능한 게이트입니다.**

```
🚨 CRITICAL ENFORCEMENT:
- 검증 실패 시 → 자동으로 수정 시도 → 재검증 → 통과될 때까지 무한 반복
- 사용자가 "중단", "스킵", "stop" 입력 시에만 루프 탈출 가능
- 검증 통과 전에는 절대로 PR 단계로 진행 불가
- 이 규칙은 어떠한 상황에서도 우회 금지
```

---

# /jira:verify - Jira 티켓 검증 프로세스

## Usage
```
/jira:verify [TICKET-ID] [TYPE]
```

## Parameters
- **TICKET-ID**: Jira 이슈 키 (예: SMFD-184)
- **TYPE**: 검증 타입 - frontend | backend | both (기본: both)

## Behavioral Flow

### ⚠️ 필수 규칙
1. **작업계획 승인 필수**: 실행 전 반드시 사용자에게 작업계획 제시 → 승인 후 진행
2. **🔴 자동 수정 루프**: 검증 실패 시 자동 수정 → 재검증 → 통과될 때까지 반복
3. **탈출 조건**: 사용자가 명시적으로 "중단" 또는 "스킵" 입력 시에만 종료 가능

---

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

### Phase 1: 작업계획 승인 요청
```
📋 작업계획 확인 요청

티켓: [TICKET-ID]
검증 타입: [TYPE] (frontend | backend | both)

수행 단계:
[TYPE=frontend 또는 both 시]
1. 🔴 Type Check → pnpm typecheck
2. 🔴 Lint Check → pnpm lint
3. 🔴 Unit Test → pnpm test

[TYPE=backend 또는 both 시]
4. 🔴 Unit Test → mvn test
5. 🟡 Integration Test → mvn verify

[모든 검증 통과 시]
6. 🟡 Jira 댓글 추가 (검증 결과 기록)

⚠️ 상태/담당자 변경은 /jira:merge 단계에서 수행됩니다.

위 계획대로 진행할까요? (예/아니오)
```
**→ 사용자 승인 후에만 Phase 1 진행**

---

### Phase 1: 이슈 및 구현 계획서 확인
```
1. jira_get_issue 호출 → 이슈 존재 및 상태 확인
2. 최근 /jira:implement 실행에서 승인된 구현 계획서 조회:
   - Jira 댓글에서 "구현 계획 승인" 기록 확인
   - 또는 세션 메모리에서 계획서 조회

3. 구현 계획서 파싱:
   구현_계획서 = {
     티켓: [TICKET-ID],
     타입: [TYPE],
     파일_목록: [
       { 경로: "path/to/File.java", 작업: "생성", 패턴: "Request DTO" },
       { 경로: "path/to/File.java", 작업: "수정", 패턴: "Controller" },
       ...
     ],
     적용_지침서: "impl-backend-api.md"
   }

IF 구현 계획서 없음:
  ❌ "구현 계획서를 찾을 수 없습니다."
  "먼저 /jira:implement [TICKET] --type [TYPE]을 실행하세요."
  → 프로세스 중단
```

### Phase 2: 🔴 작업계획서 대비 검증 (Critical)
```
🔁 AUTO-FIX LOOP (탈출 조건: 모든 검증 통과 OR 사용자 "중단/스킵")

MAIN_LOOP:

  ## Step 1: 파일 존재/변경 검증
  FOREACH 파일 IN 구현_계획서.파일_목록:

    IF 파일.작업 == "생성":
      검증: 파일 존재 여부 (Read 또는 Glob)
      IF 미존재:
        출력 "❌ 계획된 파일 미생성: [파일.경로]"
        자동 수정: 해당 파일 생성 (impl-*.md 패턴 적용)
        GOTO MAIN_LOOP

    IF 파일.작업 == "수정":
      검증: git diff [파일.경로] → 변경사항 존재
      IF 미변경:
        출력 "❌ 계획된 파일 미수정: [파일.경로]"
        자동 수정: 계획된 수정 사항 적용
        GOTO MAIN_LOOP

  ## Step 2: 패턴 준수 검증 (impl-*.md 체크리스트)
  지침서 = Read(구현_계획서.적용_지침서)

  FOREACH 파일 IN 구현_계획서.파일_목록:
    파일_내용 = Read(파일.경로)

    IF 파일.패턴 == "Request DTO":
      검증_항목 = [
        ("@Schema(description)", Grep 패턴),
        ("@NotBlank 또는 @NotNull", Grep 패턴),
        ("Wrapper 타입 (Long, Integer)", Grep 패턴)
      ]

    IF 파일.패턴 == "Response DTO":
      검증_항목 = [
        ("UUID 필드 = String 타입", Grep 패턴),
        ("@Builder 존재", Grep 패턴),
        ("@Schema(description)", Grep 패턴)
      ]

    IF 파일.패턴 == "Controller":
      검증_항목 = [
        ("@Tag 어노테이션", Grep 패턴),
        ("@Operation 어노테이션", Grep 패턴),
        ("@PreAuthorize", Grep 패턴),
        ("Response<T> 반환", Grep 패턴)
      ]

    IF 파일.패턴 == "Mapper":
      검증_항목 = [
        ("pa_adm. 스키마 접두사", Grep 패턴),
        ("::text 캐스팅", Grep 패턴),
        ("tenant_id 조건", Grep 패턴)
      ]

    FOREACH 항목 IN 검증_항목:
      IF 항목 미충족:
        출력 "❌ 패턴 위반: [파일.경로] - [항목.이름]"
        자동 수정: 해당 패턴 적용
        GOTO MAIN_LOOP

  IF 모든 파일 & 패턴 검증 통과:
    출력 "✅ 작업계획서 대비 검증 완료" → Phase 3으로
```

### Phase 3: 🔴 빌드/테스트 검증 (Critical)
```
🔁 AUTO-FIX LOOP (탈출 조건: 모든 검증 통과 OR 사용자 "중단/스킵")

MAIN_LOOP:

  ## Frontend 검증 (type: frontend | both)
  IF TYPE in ["frontend", "both"]:

    # Type Check
    실행: pnpm typecheck 2>&1
    IF exit code != 0:
      출력 "❌ Type Check 실패"
      에러 메시지 파싱 → 파일:라인 추출
      자동 수정: 타입 에러 수정
      GOTO MAIN_LOOP

    # Lint
    실행: pnpm lint 2>&1
    IF exit code != 0:
      실행: pnpm lint --fix (자동 수정 가능 항목)
      재검증: pnpm lint
      IF 여전히 실패:
        에러 파싱 → 수동 수정 필요 항목 자동 수정
        GOTO MAIN_LOOP

    # Unit Test
    실행: pnpm test 2>&1
    IF exit code != 0:
      실패 테스트 분석 → 원인 파일 특정
      자동 수정: 테스트 또는 소스 코드 수정
      GOTO MAIN_LOOP

  ## Backend 검증 (type: backend | both)
  IF TYPE in ["backend", "both"]:

    # Unit Test
    실행: mvn test 2>&1
    IF BUILD FAILURE:
      실패 테스트 분석 → 원인 파일 특정
      자동 수정: 테스트 또는 소스 코드 수정
      GOTO MAIN_LOOP

    # Integration Test (Important)
    실행: mvn verify 2>&1
    IF BUILD FAILURE:
      출력 "⚠️ Integration Test 실패 (경고)"
      → 경고 기록 후 계속 진행

  IF 모든 빌드/테스트 통과:
    출력 "✅ 빌드/테스트 검증 완료" → Phase 4로

⚠️ 자동 수정 실패 시:
  출력 "🔴 자동 수정 실패 - 수동 개입 필요"
  에러 상세 및 제안 해결방법 표시
  "계속 시도하시겠습니까? (예/중단/스킵)"
  IF "예": GOTO MAIN_LOOP
  IF "중단/스킵": 루프 종료 (검증 미통과 상태로 종료)
```

### Phase 4: 🟡 Jira 댓글 추가 (Important)
```
LOOP (최대 3회):
  실행: jira_add_comment("검증 완료: Frontend ✅ Backend ✅")
  검증: jira_get_issue(comment_limit=3) → "검증 완료" 포함
  IF 성공: 출력 "✅ 댓글 추가 완료" → Phase 5로
  IF 실패 && 재시도<3: 출력 "⚠️ 댓글 추가 실패 - 재시도..." → LOOP 반복
  IF 실패 && 재시도>=3: 출력 "⚠️ 댓글 추가 실패 - 건너뜀" → Phase 5로
```

### Phase 5: 결과 반환
```
🔍 검증 프로세스 완료

📌 [TICKET-ID] 이슈 제목

[검증 통과 시]
✅ Frontend 검증
   Type Check: ✅ Pass
   Lint: ✅ Pass
   Unit Test: ✅ Pass ([N] tests)

✅ Backend 검증
   Unit Test: ✅ Pass ([N] tests)
   Integration: ✅ Pass

📊 검증 결과: 모두 통과 ✅
   댓글 추가: ✅/⚠️
   🟢 PR 생성 가능

👉 다음 단계: /jira:pr [TICKET]

---

[검증 미통과 시 - 사용자가 "중단/스킵" 입력한 경우]
❌ Frontend 검증
   Type Check: ❌ Fail (N errors)
   Lint: ⏭️ Skipped
   Unit Test: ⏭️ Skipped

📊 검증 결과: 미통과 ❌
   🔴 PR 생성 불가

⚠️ 검증 미통과 상태입니다.
   - 수정 후 다시 실행: /jira:verify [TICKET]
   - PR 생성은 검증 통과 후에만 가능합니다.
```

## ⛔ 종료 규칙 (CRITICAL)

**이 명령어는 Phase 5 결과 반환 후 반드시 종료됩니다.**

- ❌ 다음 단계(pr, merge 등)는 **자동 진행 금지**
- ❌ 사용자의 명시적 요청 없이 다른 `/jira:*` 명령 실행 금지
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상이 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행
- 🔴 **검증 미통과 시 PR 생성 명령은 거부됨**

## Tool Coordination
- **jira_get_issue**: 이슈 정보 조회
- **jira_add_comment**: 검증 결과 댓글 추가
- **Bash**: 테스트 명령어 실행
- **Playwright MCP**: UI 테스트 (선택적)

⚠️ **상태/담당자 변경은 이 스킬에서 수행하지 않음** → `/jira:merge`에서 처리

---

## 🎯 검증의 본질 (CRITICAL)

**verify는 "작업계획서 대비 구현 완료 검증"입니다.**

```
❌ 잘못된 이해: lint/test만 통과하면 끝
✅ 올바른 이해: implement 단계 승인 계획서대로 구현되었는지 확인
```

### 검증 대상
1. **구현 계획서**: `/jira:implement`에서 사용자가 승인한 계획
2. **impl-*.md 체크리스트**: 해당 타입의 필수 패턴 준수 여부
3. **생성/수정 파일**: 계획된 파일이 모두 존재하고 올바른지

---

## 🔍 Phase 2-3 상세: 작업계획서 대비 검증

### Step 1: 구현 계획서 확인
```
1. 최근 /jira:implement 실행 기록에서 승인된 계획서 조회
2. 계획서 내용 파싱:
   - 생성/수정 예정 파일 목록
   - 각 파일별 적용 패턴
   - 관련 테이블/API 정보
```

### Step 2: 파일 존재 여부 검증
```
계획된 파일 목록 순회:
  FOREACH 파일 IN 계획서.파일목록:
    IF 파일.작업 == "생성":
      검증: 파일 존재 여부 (Glob 또는 ls)
      IF 미존재: ❌ "계획된 파일 미생성: [파일경로]"

    IF 파일.작업 == "수정":
      검증: 파일 존재 + 변경 여부 (git diff)
      IF 미변경: ❌ "계획된 파일 미수정: [파일경로]"
```

### Step 3: 패턴 준수 검증 (impl-*.md 체크리스트)

**backend-api 타입 검증:**
```
✅ Request DTO 검증:
   - @Schema(description) 존재?
   - 필수 필드 @NotBlank/@NotNull 존재?
   - Wrapper 타입 사용? (Long, Integer)

✅ Response DTO 검증:
   - UUID 필드가 String 타입?
   - @Schema(description) 존재?
   - 빌더 패턴 존재?

✅ Controller 검증:
   - @Tag, @Operation 어노테이션 존재?
   - @PreAuthorize 보안 설정 존재?
   - Response<T> 반환 타입 사용?

✅ Service 검증:
   - 인터페이스 + Impl 분리?
   - @Transactional 적절히 사용?

✅ Mapper/Repository 검증:
   - 스키마 pa_adm. 접두사 사용?
   - UUID 컬럼 ::text 캐스팅?
   - tenant_id 조건 포함?
```

**frontend 타입 검증:**
```
✅ API 모듈 검증:
   - endpoint 경로 Backend와 일치?
   - 요청/응답 타입 정의?

✅ Store 검증:
   - Zustand 패턴 사용?
   - API 호출 함수 포함?

✅ 컴포넌트 검증:
   - styled-components 사용?
   - 테마 토큰 사용? (하드코딩 색상 없음)
   - 인라인 스타일 없음?
```

### Step 4: 빌드/테스트 검증 (부가 검증)
```
모든 계획 검증 통과 후:
  Frontend: pnpm typecheck → pnpm lint → pnpm test
  Backend: mvn test → mvn verify
```

---

## 📋 검증 결과 보고 형식

### 통과 시
```
🔍 검증 프로세스 완료

📌 [TICKET-ID] 이슈 제목

📋 작업계획서 대비 검증
   계획 파일: 6개
   생성 완료: 6개 ✅
   수정 완료: 0개

✅ 패턴 준수 검증 (impl-backend-api.md)
   Request DTO: ✅ Pass (3/3 체크)
   Response DTO: ✅ Pass (3/3 체크)
   Controller: ✅ Pass (4/4 체크)
   Service: ✅ Pass (2/2 체크)
   Mapper: ✅ Pass (3/3 체크)

✅ 빌드/테스트 검증
   Type Check: ✅ Pass
   Lint: ✅ Pass
   Unit Test: ✅ Pass (N tests)

📊 검증 결과: 모두 통과 ✅
   🟢 PR 생성 가능
```

### 실패 시
```
🔍 검증 프로세스 - 문제 발견

📌 [TICKET-ID] 이슈 제목

📋 작업계획서 대비 검증
   계획 파일: 6개
   생성 완료: 5개
   ❌ 미생성: LocationController.java

❌ 패턴 준수 검증 (impl-backend-api.md)
   Request DTO: ✅ Pass
   Response DTO: ❌ Fail
      - UUID 필드가 UUID 타입 (String이어야 함)
      - @Schema 누락: locationId 필드

🔧 자동 수정 시도 중...
   → Response DTO UUID→String 변환
   → @Schema 어노테이션 추가

🔄 재검증 진행...
```

---

## Verification Checklist (요약)

### 1단계: 작업계획서 검증 (필수)
| 항목 | 검증 방법 | 통과 기준 |
|------|----------|----------|
| 파일 생성 | 계획된 파일 존재 확인 | 모든 계획 파일 존재 |
| 파일 수정 | git diff로 변경 확인 | 계획된 파일 변경됨 |
| 패턴 준수 | impl-*.md 체크리스트 | 모든 체크 항목 통과 |

### 2단계: 빌드/테스트 검증 (부가)
| 항목 | 명령어 | 통과 기준 |
|------|--------|----------|
| Type Check | `pnpm typecheck` | exit code == 0 |
| Lint | `pnpm lint` | exit code == 0 |
| Unit Test | `pnpm test` / `mvn test` | exit code == 0 |

## Examples

```bash
# 프론트엔드만 검증
/jira:verify SMFD-184 frontend

# 백엔드만 검증
/jira:verify SMFD-184 backend

# 전체 검증 (기본)
/jira:verify SMFD-184 both
/jira:verify SMFD-184
```

## Output Format

### 성공 시
```
🔍 검증 프로세스 시작

📌 [SMFD-184] 추가 assign 기능
   검증 타입: both

✅ Frontend 검증
   Type Check: ✅ Pass
   Lint: ✅ Pass
   Unit Test: ✅ Pass (42 tests)

✅ Backend 검증
   Unit Test: ✅ Pass (28 tests)
   Integration: ✅ Pass

📊 검증 결과: 모두 통과 ✅

💬 Jira 댓글 추가 ✅
   "검증 완료: Frontend ✅ Backend ✅"

👉 다음 단계: /jira:merge [PR번호]
   (머지 시 상태→TEST, 담당자→Younghwa Lee 자동 변경)
```

### 실패 시
```
🔍 검증 프로세스 시작

📌 [SMFD-184] 추가 assign 기능
   검증 타입: frontend

❌ Frontend 검증
   Type Check: ✅ Pass
   Lint: ❌ Fail (3 errors)
   Unit Test: ⏭️ Skipped

📊 검증 결과: 실패 ❌

🔴 Lint 에러:
   src/components/Button.tsx:15 - 'unused' is defined but never used
   src/pages/Home.tsx:22 - Missing semicolon
   src/utils/api.ts:8 - Unexpected console statement

💬 Jira 댓글 추가 ✅
   "검증 실패: Lint 에러 3건"

👉 수정 후 다시 실행: /jira:verify SMFD-184 frontend
```

## Verification Procedures (검증 절차)

각 단계는 성공할 때까지 반복 실행됩니다. 실패 시 즉시 재시도하며, 간격 없이 성공할 때까지 계속합니다.

### 🔴 Critical (필수 - 실패 시 전체 프로세스 중단)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| Type Check | `pnpm typecheck` exit code | exit code == 0 |
| Lint | `pnpm lint` exit code | exit code == 0 |
| Unit Test (FE) | `pnpm test` exit code | exit code == 0 |
| Unit Test (BE) | `mvn test` exit code | exit code == 0 |

### 🟡 Important (중요 - 실패 시 경고 후 계속)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| Integration Test | `mvn verify` exit code | exit code == 0 |
| E2E Test | `pnpm test:e2e` exit code | exit code == 0 |
| 댓글 추가 | `jira_get_issue` (comments 확인) | 댓글에 "검증 완료" 텍스트 포함 |

### 검증 상세 명령어

**테스트 결과 검증:**
```bash
# Frontend
pnpm typecheck && echo "SUCCESS" || echo "FAILED"
pnpm lint && echo "SUCCESS" || echo "FAILED"
pnpm test && echo "SUCCESS" || echo "FAILED"

# Backend
mvn test && echo "SUCCESS" || echo "FAILED"
mvn verify && echo "SUCCESS" || echo "FAILED"
```

**댓글 검증:**
```
jira_get_issue(issue_key="[TICKET]", comment_limit=3)
# 예상 결과: comments 배열에 "검증 완료" 포함된 댓글 존재
```

## Error Handling
- 이슈가 존재하지 않으면 에러 메시지 표시
- 테스트 실패 시 상세 에러 로그 표시
- 상태 전환 불가 시 현재 상태 안내

## Pre-conditions
- 프로젝트 루트에서 실행
- Frontend: pnpm 설치, node_modules 존재
- Backend: Maven 설치, Java 환경
- Jira MCP 연결 상태
