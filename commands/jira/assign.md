---
name: assign
description: "Jira 이슈 담당자 지정"
category: project-management
complexity: basic
mcp-servers: [jira]
personas: []
---

# /jira:assign - 담당자 지정

## Usage
```
/jira:assign [TICKET-ID] [ASSIGNEE]
```

## Parameters
- **TICKET-ID**: Jira 이슈 키 (예: SMFD-184)
- **ASSIGNEE** (선택): 담당자 이름 또는 이메일
  - 생략 시 → 나(현재 사용자)에게 할당
  - 이름 입력 시 → 해당 사용자에게 할당
  - `none` 입력 시 → 담당자 해제

## Behavioral Flow

### ⚠️ 필수 규칙
1. **작업계획 승인 필수**: 실행 전 반드시 사용자에게 작업계획 제시 → 승인 후 진행
2. **검증 통과 필수**: 각 단계 실행 직후 즉시 검증 → 검증 통과 전까지 다음 단계 진행 금지
3. **재시도 규칙**: Critical 단계 실패 시 성공할 때까지 무한 재시도

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
현재 담당자: [CURRENT_ASSIGNEE]
변경 대상: [TARGET_ASSIGNEE]

수행 단계:
1. 🔴 담당자 변경 → [TARGET_ASSIGNEE]

위 계획대로 진행할까요? (예/아니오)
```
**→ 사용자 승인 후에만 Phase 1 진행**

---

### Phase 1: 이슈 확인
- `jira_get_issue` 호출하여 이슈 존재 및 현재 담당자 확인
- 현재 담당자 정보 표시

### Phase 2: 담당자 조회 (이름 입력 시)
- `jira_get_user_profile` 호출하여 사용자 확인
- 사용자가 없으면 유사 이름 제안

### Phase 3: 🔴 담당자 변경 (Critical)
```
LOOP:
  실행: jira_update_issue(assignee="[TARGET_EMAIL_OR_NAME]")
  검증: jira_get_issue → assignee.displayName == "[TARGET_NAME]"
  IF 성공: 출력 "✅ 담당자 변경 완료" → Phase 4로
  IF 실패: 출력 "❌ 담당자 변경 실패 - 재시도..." → LOOP 반복
```

### Phase 4: 결과 반환
```
👤 담당자 지정 완료

📌 [TICKET-ID] 이슈 제목
   변경 전: [PREVIOUS_ASSIGNEE]
   변경 후: ✅ [TARGET_ASSIGNEE]

👉 다음 단계: /jira:start [TICKET-ID] [branch-description]
```

## ⛔ 종료 규칙 (CRITICAL)

**이 명령어는 Phase 4 결과 반환 후 반드시 종료됩니다.**

- ❌ 다음 단계(start 등)는 **자동 진행 금지**
- ❌ 사용자의 명시적 요청 없이 다른 `/jira:*` 명령 실행 금지
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상이 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행

## Tool Coordination
- **jira_get_issue**: 이슈 정보 조회
- **jira_get_user_profile**: 사용자 정보 조회
- **jira_update_issue**: 담당자 필드 업데이트

## Examples

```bash
# 나한테 할당
/jira:assign SMFD-184

# 특정 사람에게 할당
/jira:assign SMFD-184 김종욱
/jira:assign SMFD-184 yhlee@cma.com

# 담당자 해제
/jira:assign SMFD-184 none
```

## Output Format

### 나에게 할당
```
👤 담당자 지정

📌 [SMFD-184] 추가 assign 기능

   변경 전: 미배정
   변경 후: 최정석 (stevenchoi@oceansmart.ai) ✅

✅ 담당자 지정 완료

👉 다음 단계: /jira:start SMFD-184 add-assign-feature
```

### 다른 사람에게 할당
```
👤 담당자 지정

📌 [SMFD-184] 추가 assign 기능

   변경 전: 최정석
   변경 후: 김종욱 (jwkim@oceansmart.ai) ✅

✅ 담당자 변경 완료
```

### 담당자 해제
```
👤 담당자 해제

📌 [SMFD-184] 추가 assign 기능

   변경 전: 최정석
   변경 후: 미배정 ✅

✅ 담당자 해제 완료
```

## Team Members Reference
조회된 팀 멤버 목록 (참고용):
- 최정석 (stevenchoi@oceansmart.ai)
- 김종욱 (jwkim@oceansmart.ai)
- Younghwa Lee (yhlee@oceansmart.ai)

## Verification Procedures (검증 절차)

각 단계는 성공할 때까지 반복 실행됩니다. 실패 시 즉시 재시도하며, 간격 없이 성공할 때까지 계속합니다.

### 🔴 Critical (필수 - 실패 시 전체 프로세스 중단)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| 담당자 변경 | `jira_get_issue` | `assignee.displayName == [TARGET_USER]` |
| 담당자 해제 | `jira_get_issue` | `assignee == null` |

### 검증 상세 명령어

**담당자 변경 검증:**
```
jira_get_issue(issue_key="[TICKET]", fields="assignee")
# 예상 결과 (할당 시):
#   assignee.displayName == "최정석" (또는 지정된 사용자)
# 예상 결과 (해제 시):
#   assignee == null
```

**사용자 확인 검증:**
```
jira_get_user_profile(user_identifier="[USER_EMAIL]")
# 예상 결과: 사용자 정보 반환 (accountId, displayName 등)
```

## Error Handling
- 이슈가 존재하지 않으면 에러 메시지 표시
- 사용자를 찾을 수 없으면 유사 이름 제안
- 권한이 없으면 접근 권한 안내
- API 오류 시 상세 에러 메시지 표시
