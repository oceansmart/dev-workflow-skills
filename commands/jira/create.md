---
name: create
description: "Jira 티켓 생성"
category: project-management
complexity: basic
mcp-servers: [jira]
personas: []
---

# /jira:create - Jira 티켓 생성

## Usage
```
/jira:create [PROJECT] [TYPE] [SUMMARY]
```

## Parameters
- **PROJECT**: Jira 프로젝트 키 (예: SMFD, KAN)
- **TYPE**: 이슈 타입 - Task | Bug | Story | Epic (기본: Task)
- **SUMMARY**: 이슈 제목

## Issue Types
| 타입 | 용도 | 예시 |
|------|------|------|
| Task | 일반 작업, 기술적 태스크 | "API 엔드포인트 구현" |
| Bug | 버그 수정 | "로그인 실패 오류 수정" |
| Story | 사용자 스토리 | "사용자는 비밀번호를 변경할 수 있다" |
| Epic | 대규모 기능 묶음 | "사용자 인증 시스템" |

## Behavioral Flow

### ⚠️ 필수 규칙
1. **작업계획 승인 필수**: 실행 전 반드시 사용자에게 작업계획 제시 → 승인 후 진행
2. **검증 통과 필수**: 각 단계 실행 직후 즉시 검증 → 검증 통과 전까지 다음 단계 진행 금지
3. **재시도 규칙**: Critical 단계 실패 시 성공할 때까지 무한 재시도

---

### Phase 0: 설정 파일 확인 (필수 - BLOCKING)

**⛔ 이 단계를 통과하지 않으면 어떤 작업도 진행할 수 없습니다.**

1. `.claude/config/jira.md` 파일 존재 여부 확인
2. **파일 있음**: 설정값 로드 (projectKey 등) → Phase 1 진행
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

프로젝트: [PROJECT]
이슈 타입: [TYPE]
제목: [SUMMARY]

수행 단계:
1. 🟡 프로젝트 유효성 확인
2. 🔴 티켓 생성 → [PROJECT]-XXX

위 계획대로 진행할까요? (예/아니오)
```
**→ 사용자 승인 후에만 Phase 1 진행**

---

### Phase 1: 🟡 프로젝트 확인 (Important)
```
LOOP (최대 3회):
  실행: jira_get_all_projects()
  검증: 프로젝트 목록에 [PROJECT] 존재
  IF 성공: 출력 "✅ 프로젝트 확인 완료" → Phase 2로
  IF 실패 && 재시도<3: 출력 "⚠️ 프로젝트 조회 실패 - 재시도..." → LOOP 반복
  IF 실패 && 재시도>=3: 출력 "❌ 프로젝트가 존재하지 않습니다" → 에러 종료
```

### Phase 2: 🔴 티켓 생성 (Critical)
```
LOOP:
  실행: jira_create_issue(project_key, issue_type, summary)
  검증: jira_get_issue([CREATED_KEY]) → key 존재 AND summary 일치
  IF 성공: 출력 "✅ 티켓 생성 완료" → Phase 3으로
  IF 실패: 출력 "❌ 티켓 생성 실패 - 재시도..." → LOOP 반복
```

### Phase 3: 결과 반환
```
✅ Jira 티켓 생성 완료

📌 [PROJECT-XXX] [SUMMARY]
   타입: [TYPE]
   상태: 해야 할 일
   🔗 https://oceansmart.atlassian.net/browse/[PROJECT-XXX]

👉 다음 단계: /jira:start [PROJECT-XXX] [branch-description]
```

## ⛔ 종료 규칙 (CRITICAL)

**이 명령어는 Phase 3 결과 반환 후 반드시 종료됩니다.**

- ❌ 다음 단계(start, assign 등)는 **자동 진행 금지**
- ❌ 사용자의 명시적 요청 없이 다른 `/jira:*` 명령 실행 금지
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상이 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행

## Tool Coordination
- **jira_get_all_projects**: 프로젝트 유효성 검증
- **jira_create_issue**: 이슈 생성 실행

## Examples

```bash
# Task 생성
/jira:create SMFD Task "로그인 페이지 UI 구현"

# Bug 생성
/jira:create SMFD Bug "홈페이지 로딩 속도 저하 이슈"

# Story 생성
/jira:create SMFD Story "사용자는 카카오 로그인을 할 수 있다"

# Epic 생성
/jira:create SMFD Epic "사용자 인증 시스템 구축"
```

## Output Format

```
✅ Jira 티켓 생성 완료

📌 [SMFD-XXX] 로그인 페이지 UI 구현
   타입: Task
   상태: 해야 할 일
   🔗 https://oceansmart.atlassian.net/browse/SMFD-XXX

👉 다음 단계: /jira:start SMFD-XXX login-page-ui
```

## Verification Procedures (검증 절차)

각 단계는 성공할 때까지 반복 실행됩니다. 실패 시 즉시 재시도하며, 간격 없이 성공할 때까지 계속합니다.

### 🔴 Critical (필수 - 실패 시 전체 프로세스 중단)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| 티켓 생성 | `jira_get_issue` | 이슈 키 반환 (예: SMFD-XXX) |
| 프로젝트 유효성 | `jira_get_all_projects` | 프로젝트 키가 목록에 존재 |

### 검증 상세 명령어

**티켓 생성 검증:**
```
jira_get_issue(issue_key="[CREATED_TICKET]", fields="summary,status,issuetype")
# 예상 결과:
#   key == "SMFD-XXX" (생성된 티켓 번호)
#   summary == [입력한 제목]
#   status.name == "해야 할 일"
#   issuetype.name == [입력한 타입]
```

**프로젝트 유효성 검증:**
```
jira_get_all_projects()
# 예상 결과: projects 배열에 해당 project_key 존재
```

## Error Handling
- 프로젝트가 존재하지 않으면 사용 가능한 프로젝트 목록 표시
- 이슈 타입이 잘못되면 올바른 타입 안내
- API 오류 시 상세 에러 메시지 표시
