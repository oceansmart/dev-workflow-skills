---
name: start
description: "Jira 티켓 작업 시작 - 상태 변경 및 feature 브랜치 생성"
category: project-management
complexity: enhanced
mcp-servers: [jira]
personas: []
---

## ⛔⛔⛔ HARD STOP RULE (최우선 규칙) ⛔⛔⛔

**이 명령어는 Phase 7 결과 반환 직후 즉시 종료됩니다.**

```
🚨 CRITICAL ENFORCEMENT:
- Phase 7 출력 완료 → STOP → 사용자 입력 대기
- "👉 다음 단계"는 정보 제공용 → 자동 실행 대상 아님
- /jira:pr, /jira:verify, /jira:merge 등 → 사용자가 직접 입력해야만 실행
- 이 규칙 위반 시 → 잘못된 동작으로 간주
```

---

# /jira:start - Jira 티켓 작업 시작

## Usage
```
/jira:start [TICKET-ID] [BRANCH-DESCRIPTION]
```

## Parameters
- **TICKET-ID**: Jira 이슈 키 (예: SMFD-184)
- **BRANCH-DESCRIPTION**: 브랜치 설명 (kebab-case, 예: add-login-page)

## Behavioral Flow

### ⚠️ 필수 규칙
1. **티켓 내용 확인 필수**: 작업 전 반드시 티켓 내용(summary, description)을 조회하고 표시
2. **작업 범위 확인 필수**: 사용자가 설명한 작업 내용과 티켓 내용이 일치하는지 확인
3. **작업계획 승인 필수**: 실행 전 반드시 사용자에게 작업계획 제시 → 승인 후 진행
4. **검증 통과 필수**: 각 단계 실행 직후 즉시 검증 → 검증 통과 전까지 다음 단계 진행 금지
5. **재시도 규칙**: Critical 단계 실패 시 성공할 때까지 무한 재시도

---

### Phase 0: 설정 파일 확인 (필수 - BLOCKING)

**⛔ 이 단계를 통과하지 않으면 어떤 작업도 진행할 수 없습니다.**

1. `.claude/config/jira.md` 파일 존재 여부 확인
2. **파일 있음**: 설정값 로드 (projectKey, branchPattern 등) → Phase 1 진행
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

### Phase 1: 🔴 티켓 내용 확인 (Critical - 가장 먼저 수행)
```
⚠️ 티켓 내용 확인 필수!

1. jira_get_issue(issue_key="[TICKET]", fields="summary,description,status,assignee")
2. 티켓 내용을 사용자에게 표시:

📌 [TICKET-ID] 이슈 정보
   제목: [SUMMARY]
   설명: [DESCRIPTION 요약 - 최대 200자]
   현재 상태: [STATUS]
   현재 담당자: [ASSIGNEE]

⚠️ 확인 요청:
   위 티켓 내용이 진행하려는 작업과 일치합니까?
   - 일치하면 "예" 또는 "진행"
   - 다르면 올바른 티켓 번호를 알려주세요

**→ 사용자가 티켓 내용 일치를 확인한 후에만 Phase 1 진행**
**→ 사용자가 작업 내용을 설명한 경우, 티켓 description과 비교하여 불일치 시 경고**
```

### Phase 1: 작업계획 승인 요청
```
📋 작업계획 확인 요청

티켓: [TICKET-ID] - [SUMMARY]
브랜치: feature/[TICKET-ID]-[DESCRIPTION]

수행 단계:
1. 🔴 담당자 지정 → Steven Choi
2. 🔴 상태 변경 → "진행 중"
3. 🔴 브랜치 생성 → feature/[TICKET]-[DESC]
4. 🟡 댓글 추가 → "작업 시작: ..."

위 계획대로 진행할까요? (예/아니오)
```
**→ 사용자 승인 후에만 Phase 2 진행**

---

### Phase 3: 🔴 담당자 지정 (Critical)
```
LOOP:
  실행: jira_update_issue(assignee="stevenchoi@oceansmart.ai")
  검증: jira_get_issue → assignee.displayName == "Steven Choi"
  IF 성공: 출력 "✅ 담당자 지정 완료" → Phase 4로
  IF 실패: 출력 "❌ 담당자 지정 실패 - 재시도..." → LOOP 반복
```

### Phase 4: 🔴 상태 변경 (Critical)
```
LOOP:
  실행: jira_get_transitions → jira_transition_issue("진행 중")
  검증: jira_get_issue → status.name == "진행 중"
  IF 성공: 출력 "✅ 상태 변경 완료" → Phase 5로
  IF 실패: 출력 "❌ 상태 변경 실패 - 재시도..." → LOOP 반복
```

### Phase 5: 🔴 Git 브랜치 생성 (Critical)
```
LOOP:
  실행: git fetch && git checkout main && git pull && git checkout -b feature/{TICKET}-{DESC}
  검증: git branch --list "feature/{TICKET}-*" → 출력 있음
  IF 성공: 출력 "✅ 브랜치 생성 완료" → Phase 6으로
  IF 실패: 출력 "❌ 브랜치 생성 실패 - 재시도..." → LOOP 반복
```

### Phase 6: 🟡 Jira 댓글 추가 (Important)
```
LOOP (최대 3회):
  실행: jira_add_comment("작업 시작: feature/{TICKET}-{DESC}")
  검증: jira_get_issue(comment_limit=3) → "작업 시작" 포함
  IF 성공: 출력 "✅ 댓글 추가 완료" → Phase 7로
  IF 실패 && 재시도<3: 출력 "⚠️ 댓글 추가 실패 - 재시도..." → LOOP 반복
  IF 실패 && 재시도>=3: 출력 "⚠️ 댓글 추가 실패 - 건너뜀" → Phase 7로
```

### Phase 7: 개발 타입 분류 및 결과 반환
```
🚀 Jira 티켓 작업 시작 완료

📌 [TICKET-ID] 제목
   담당자: ✅ Steven Choi
   상태: ✅ 진행 중
   브랜치: ✅ feature/[TICKET]-[DESC]
   댓글: ✅/⚠️

📂 개발 타입 분류:
   티켓 description을 분석하여 해당하는 타입을 모두 선택:

   | 타입 | 키워드 | 구현 지침서 |
   |------|--------|------------|
   | **backend-api** | API, 엔드포인트, Controller, Service | impl-backend-api.md |
   | **frontend** | 화면, UI, 페이지, 컴포넌트 | impl-frontend.md |
   | **bugfix** | 버그, 오류, 수정, fix | impl-bugfix.md |
   | **database** | 테이블, 컬럼, 쿼리, DDL, DML | impl-database.md |

   ⚠️ 복합 타입 지원:
   - 여러 타입이 해당되면 모두 선택 (예: backend-api + frontend)
   - 복합 타입 시 순서: database → backend-api → frontend 순으로 구현
   - fullstack = backend-api + frontend (편의상 제공)

   🏷️ 분류 결과: [TYPE] 또는 [TYPE1 + TYPE2]
   📖 참조 지침서: [impl-xxx.md] (복합 시 여러 개)

👉 다음 단계:
   1. 구현 시작: /jira:implement [TICKET] --type [TYPE]
      (복합 타입: /jira:implement [TICKET] --type backend-api,frontend)
   2. 검증 필수: /jira:verify [TICKET] [both|frontend|backend]
   3. PR 생성: /jira:pr [TICKET]

⚠️ 중요:
   - 코드 작성 전 반드시 /jira:implement 명령 실행!
   - PR 생성 전 반드시 /jira:verify 통과 필수!
```

## ⛔ 종료 규칙 (CRITICAL)

**이 명령어는 Phase 7 결과 반환 후 반드시 종료됩니다.**

- ❌ 다음 단계(PR 생성, verify, merge 등)는 **자동 진행 금지**
- ❌ 사용자의 명시적 요청 없이 다른 `/jira:*` 명령 실행 금지
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상이 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행

## Tool Coordination
- **jira_get_issue**: 이슈 정보 조회
- **jira_update_issue**: 담당자를 현재 사용자로 지정
- **jira_get_transitions**: 가능한 상태 전환 조회
- **jira_transition_issue**: 상태를 "진행 중"으로 변경
- **jira_add_comment**: 작업 시작 댓글 추가
- **Bash (git)**: 브랜치 생성 및 전환

## Branch Naming Convention
```
feature/{TICKET-ID}-{description}
```
예시:
- `feature/SMFD-184-add-assign-feature`
- `feature/SMFD-38-email-template-editor`

## Examples

```bash
# 기본 사용
/jira:start SMFD-184 add-assign-feature

# 버그 수정
/jira:start SMFD-203 fix-cancelled-container

# 기능 구현
/jira:start SMFD-38 email-template-editor
```

## Output Format

```
🚀 Jira 티켓 작업 시작 완료

📌 [SMFD-184] 추가 assign 기능
   담당자: 미배정 → 최정석 ✅
   상태: 해야 할 일 → 진행 중 ✅

🌿 Git 브랜치 생성
   브랜치: feature/SMFD-184-add-assign-feature ✅
   기반: main (최신)

💬 Jira 댓글 추가 ✅
   "작업 시작: feature/SMFD-184-add-assign-feature"

📂 개발 타입 분류:
   🏷️ 분류 결과: backend-api
   📖 참조 지침서: impl-backend-api.md

👉 다음 단계:
   1. 구현 시작: /jira:implement SMFD-184 --type backend-api
   2. 검증 필수: /jira:verify SMFD-184 backend
   3. PR 생성: /jira:pr SMFD-184

⚠️ 중요:
   - 코드 작성 전 반드시 /jira:implement 명령 실행!
   - PR 생성 전 반드시 /jira:verify 통과 필수!
```

## Verification Procedures (검증 절차)

각 단계는 성공할 때까지 반복 실행됩니다. 실패 시 즉시 재시도하며, 간격 없이 성공할 때까지 계속합니다.

### 🔴 Critical (필수 - 실패 시 전체 프로세스 중단)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| **티켓 내용 확인** | `jira_get_issue` → 사용자 확인 | 사용자가 티켓 내용 일치 확인 |
| 담당자 변경 | `jira_get_issue` | `assignee.displayName == "Steven Choi"` |
| 상태 변경 | `jira_get_issue` | `status.name == "진행 중"` |
| 브랜치 생성 | `git branch --list [branch]` | 브랜치명 출력됨 |

### ⚠️ 티켓 내용 불일치 감지 시

사용자가 작업 내용을 설명한 경우(예: "tenant_id 컬럼 추가 작업"):
1. 티켓의 `description`을 조회
2. 사용자 설명과 티켓 설명 비교
3. **불일치 시 경고 출력 및 확인 요청**:
```
⚠️ 주의: 작업 내용 불일치 가능성

사용자 설명: "tenant_id 컬럼 추가 작업"
티켓 설명: "[실제 티켓 description]"

정말 이 티켓([TICKET-ID])이 맞습니까? (예/아니오)
```

### 🟡 Important (중요 - 실패 시 경고 후 계속)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| 댓글 추가 | `jira_get_issue` (comments 확인) | 댓글에 "작업 시작" 텍스트 포함 |
| main 최신화 | `git log -1 --oneline origin/main` | 최신 커밋 확인 |

### 검증 상세 명령어

**담당자/상태 검증:**
```
jira_get_issue(issue_key="[TICKET]", fields="status,assignee")
# 예상 결과:
#   status.name == "진행 중"
#   assignee.displayName == "Steven Choi"
```

**브랜치 생성 검증:**
```bash
git branch --list "feature/[TICKET]-*"
# 예상 결과: feature/SMFD-XXX-description 출력
```

**댓글 검증:**
```
jira_get_issue(issue_key="[TICKET]", comment_limit=3)
# 예상 결과: comments 배열에 "작업 시작" 포함된 댓글 존재
```

## Error Handling
- 이슈가 존재하지 않으면 에러 메시지 표시
- 이미 "진행 중" 상태면 상태 변경 스킵
- Git 충돌 시 해결 방법 안내
- 브랜치가 이미 존재하면 체크아웃만 수행

## Pre-conditions
- Git 저장소 내에서 실행
- main 브랜치가 존재
- Jira MCP 연결 상태
