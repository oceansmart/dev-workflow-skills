# Jira Project Context

## 개요
이 문서는 `/jira:*` 명령어 실행 시 프로젝트 컨텍스트를 로드하는 방법을 정의합니다.

**모든 `/jira:*` 명령어는 이 문서의 Phase 0을 필수로 수행해야 합니다.**

---

## ⛔ Phase 0: 환경 및 설정 확인 (BLOCKING - 필수)

### 핵심 규칙
```
Phase 0-1 실패 → MCP 설정 안내 → 완전 중단
Phase 0-2 실패 → 프로젝트 설정 생성 → 완전 중단
둘 다 통과해야만 → Phase 1 실행 가능
```

---

## Phase 0-1: MCP 연결 확인 (BLOCKING)

### 0-1.1 확인 대상
```
{프로젝트 루트}/.mcp.json
```

### 0-1.2 확인 프로세스
```
1. Read 도구로 .mcp.json 파일 읽기 시도
2. 파일 존재 → "jira" 서버 설정 확인
3. "jira" 설정 존재 → Phase 0-2로 이동
4. 파일 없음 OR "jira" 설정 없음 → 0-1.3 MCP 설정 안내
```

### 0-1.3 MCP 설정 안내 (설정이 없을 때)

```
⛔ Jira MCP 연결이 설정되지 않았습니다.

/jira:* 명령어를 사용하려면 먼저 Jira MCP 서버 연결이 필요합니다.

📋 필요한 정보:
   • Jira Instance URL (예: https://company.atlassian.net)
   • Jira Email (Jira 로그인 이메일)
   • Jira API Token (https://id.atlassian.com/manage-profile/security/api-tokens 에서 생성)

MCP 설정을 진행할까요? (예/아니오)
```

**사용자 "예" 응답 시:**
```
📝 Jira MCP 설정

다음 정보를 입력해주세요:

1️⃣ Jira Instance URL
   예: https://company.atlassian.net

2️⃣ Jira Email
   예: user@company.com

3️⃣ Jira API Token
   ⚠️ https://id.atlassian.com/manage-profile/security/api-tokens 에서 생성
   예: ATATT3xFfGF0...

한 번에 입력하셔도 됩니다:
예: "https://company.atlassian.net, user@company.com, ATATT3xFfGF0..."
```

**입력 수집 후 .mcp.json 생성/업데이트:**
```json
{
  "mcpServers": {
    "jira": {
      "command": "uvx",
      "args": ["mcp-server-jira"],
      "env": {
        "JIRA_INSTANCE_URL": "{입력값}",
        "JIRA_USER_EMAIL": "{입력값}",
        "JIRA_API_TOKEN": "{입력값}"
      }
    }
  }
}
```

**생성 후 메시지:**
```
✅ Jira MCP 설정이 완료되었습니다.

📁 생성된 파일: .mcp.json

⚠️ 중요: MCP 서버 적용을 위해 Claude Code를 재시작해주세요.
   1. 현재 세션 종료 (/exit 또는 Ctrl+C)
   2. Claude Code 다시 시작
   3. /jira:* 명령어 다시 실행

🔒 보안 주의: .mcp.json에 API 토큰이 포함되어 있습니다.
   → .gitignore에 .mcp.json 추가 권장
```

**사용자 "아니오" 응답 시:**
```
❌ 명령어 실행이 취소되었습니다.

/jira:* 명령어를 사용하려면 MCP 설정이 필요합니다.
나중에 설정하려면 다시 /jira:* 명령어를 실행해주세요.
```
→ 명령어 완전 종료

---

## Phase 0-2: 프로젝트 설정 확인 (BLOCKING)

### 0-2.1 설정 파일 위치
```
{프로젝트 루트}/.claude/config/jira.md
```

### 0-2.2 설정 파일 확인 프로세스

```
1. Read 도구로 .claude/config/jira.md 파일 읽기 시도
2. 파일 존재 → 0-2.4 설정값 파싱으로 이동
3. 파일 없음 → 0-2.3 설정 파일 생성 워크플로우 시작
```

### 0-2.3 설정 파일 생성 워크플로우 (파일 없을 때)

```
⛔ Jira 프로젝트 설정 파일이 없습니다.

/jira:* 명령어를 사용하려면 프로젝트별 설정 파일이 필요합니다.

📁 필요한 파일: .claude/config/jira.md

설정 파일을 생성할까요? (예/아니오)
```

**사용자 "예" 응답 시:**
```
📝 Jira 프로젝트 설정 생성

필수 정보를 입력해주세요:

1️⃣ Project Key (필수)
   예: SMFD, PROJ, DEV
   → Jira 티켓 번호 앞부분 (예: SMFD-123의 "SMFD")

2️⃣ Project Name (필수)
   예: OceanSmart Feeder Management System
   → 프로젝트 전체 이름

3️⃣ Board ID (필수)
   예: 3
   → Jira 보드 URL에서 확인 (/board/3 → "3")

4️⃣ Board Type (필수)
   → scrum 또는 kanban

5️⃣ Default Assignee Email (선택)
   예: user@company.com
   → 기본 담당자 이메일

한 번에 입력하셔도 됩니다:
예: "SMFD, OceanSmart FMS, 3, scrum, user@company.com"
```

**입력 수집 후:**
1. `.claude/config/` 폴더 생성 (없으면)
2. `.claude/config/jira.md` 파일 생성
3. 생성 완료 메시지 출력
4. 원래 실행하려던 명령어 다시 실행 안내

**사용자 "아니오" 응답 시:**
```
❌ 명령어 실행이 취소되었습니다.

/jira:* 명령어를 사용하려면 프로젝트 설정 파일이 필요합니다.
나중에 설정하려면 다시 /jira:* 명령어를 실행해주세요.
```
→ 명령어 완전 종료

### 0-2.4 설정값 파싱 규칙

**설정 파일 구조:**
```markdown
# Jira Configuration - {PROJECT_NAME}

## Required Settings (필수 설정)

### Project Information
| 항목 | 값 | 설명 |
|------|-----|------|
| **Project Key** | `SMFD` | Jira 프로젝트 키 |
| **Project Name** | OceanSmart FMS | 프로젝트 전체 이름 |
| **Board ID** | `3` | Scrum/Kanban 보드 ID |
| **Board Type** | `scrum` | 보드 타입 |
```

**파싱 방법:**
1. "Project Information" 테이블에서 값 추출
2. 백틱(`) 안의 값을 실제 설정값으로 사용
3. 추출된 값:
   - `projectKey`: "SMFD"
   - `projectName`: "OceanSmart FMS"
   - `boardId`: "3"
   - `boardType`: "scrum"

**필수값 검증:**
```
IF projectKey 없음 OR boardId 없음 → 설정 파일 불완전 → 재생성 요청
```

---

## Phase 0 완료 조건

```
✅ Phase 0-1 통과: .mcp.json에 jira 서버 설정 존재
✅ Phase 0-2 통과: .claude/config/jira.md 존재 및 필수값 파싱 완료

→ Phase 1 (실제 기능) 실행 가능
```

---

## 설정 파일 형식 정의

### Required Settings (필수)

```markdown
## Required Settings (필수 설정)

### Project Information
| 항목 | 값 | 설명 |
|------|-----|------|
| **Project Key** | `{KEY}` | Jira 프로젝트 키 |
| **Project Name** | {Name} | 프로젝트 전체 이름 |
| **Board ID** | `{ID}` | Scrum/Kanban 보드 ID |
| **Board Type** | `{scrum|kanban}` | 보드 타입 |

### Branch Naming Convention
| 타입 | 패턴 | 예시 |
|------|------|------|
| Feature | `feature/{issueKey}-{summary}` | `feature/{KEY}-123-add-login` |
| Bugfix | `bugfix/{issueKey}-{summary}` | `bugfix/{KEY}-124-fix-crash` |
| Hotfix | `hotfix/{issueKey}-{summary}` | `hotfix/{KEY}-125-urgent-fix` |

### Default Assignee
- **Current User**: {user@email.com}
```

### Optional Settings (선택)

```markdown
## Optional Settings (선택 설정)

### Issue Type Mappings
| Jira Issue Type | 용도 |
|-----------------|------|
| `Story` | 사용자 기능 개발 |
| `Task` | 일반 작업 |
| `Bug` | 버그 수정 |

### Workflow Transitions
| From Status | To Status | Transition Name |
|-------------|-----------|-----------------|
| To Do | In Progress | Start Progress |
| In Progress | In Review | Ready for Review |

### Labels (자주 사용하는 레이블)
- `frontend`
- `backend`
- `database`

### Components
- `FMS-Frontend`
- `FMS-Backend`
```

---

## 설정값 사용 가이드

### 명령어별 사용 설정값

| 명령어 | 사용 설정값 |
|--------|------------|
| `/jira:list` | projectKey, boardId, boardType |
| `/jira:show` | projectKey |
| `/jira:start` | projectKey, branchPattern, defaultAssignee |
| `/jira:create` | projectKey |
| `/jira:assign` | projectKey, defaultAssignee |
| `/jira:implement` | projectKey |
| `/jira:verify` | projectKey |
| `/jira:pr` | projectKey |
| `/jira:merge` | projectKey |

### 설정값 활용 예시

**JQL 쿼리 생성:**
```
project = "{projectKey}" AND status != Done
→ project = "SMFD" AND status != Done
```

**브랜치 생성:**
```
feature/{issueKey}-{summary}
→ feature/SMFD-287-add-vessel-image-url
```

**보드 조회:**
```
jira_get_board_issues(board_id="{boardId}")
→ jira_get_board_issues(board_id="3")
```

---

## 확장성 설계

### Config 폴더 구조
```
.claude/config/
├── jira.md      # ✅ Jira 설정 (구현 완료)
├── notion.md    # 🔜 Notion 설정 (향후)
├── github.md    # 🔜 GitHub 설정 (향후)
├── slack.md     # 🔜 Slack 설정 (향후)
└── linear.md    # 🔜 Linear 설정 (향후)
```

### 새로운 설정 파일 추가 규칙

1. **파일명**: `{tool-name}.md`
2. **구조**: Required Settings / Optional Settings 섹션 구분
3. **값 형식**: 테이블 + 백틱으로 값 표시
4. **검증**: 해당 도구 명령어에서 Phase 0 설정 확인 로직 추가

---

## 설정 파일 생성 템플릿

새 프로젝트에서 복사하여 사용:

```markdown
# Jira Configuration - {프로젝트 이름}

이 파일은 현재 프로젝트의 Jira 연동 설정을 정의합니다.
모든 `/jira:*` 명령어는 실행 전 이 파일을 먼저 읽어 프로젝트 컨텍스트를 파악합니다.

---

## Required Settings (필수 설정)

### Project Information
| 항목 | 값 | 설명 |
|------|-----|------|
| **Project Key** | `{KEY}` | Jira 프로젝트 키 |
| **Project Name** | {Name} | 프로젝트 전체 이름 |
| **Board ID** | `{ID}` | Scrum/Kanban 보드 ID |
| **Board Type** | `{scrum|kanban}` | 보드 타입 (scrum/kanban) |

### Branch Naming Convention
| 타입 | 패턴 | 예시 |
|------|------|------|
| Feature | `feature/{issueKey}-{summary}` | `feature/{KEY}-123-add-login` |
| Bugfix | `bugfix/{issueKey}-{summary}` | `bugfix/{KEY}-124-fix-crash` |
| Hotfix | `hotfix/{issueKey}-{summary}` | `hotfix/{KEY}-125-urgent-fix` |

### Default Assignee
- **Current User**: {user@email.com}

---

## Optional Settings (선택 설정)

### Labels (자주 사용하는 레이블)
- `frontend`
- `backend`
- `database`
- `urgent`

### Components
- `{Component-1}`
- `{Component-2}`

---

*Last Updated: {YYYY-MM-DD}*
*Created by: Claude Code Plugin System*
```

---

## 문제 해결

### MCP 연결 관련 오류

**"Jira MCP 연결이 설정되지 않았습니다":**
→ `.mcp.json` 파일에 jira 서버 설정이 없음
→ Phase 0-1 안내에 따라 MCP 설정 진행

**"MCP 도구를 찾을 수 없습니다":**
→ `.mcp.json`은 있지만 Claude Code 재시작 필요
→ Claude Code 종료 후 다시 시작

**"Jira API 인증 실패":**
→ API Token이 만료되었거나 잘못됨
→ https://id.atlassian.com/manage-profile/security/api-tokens 에서 새 토큰 생성

### 설정 파일 관련 오류

**"프로젝트 설정 파일이 없습니다":**
→ `.claude/config/jira.md` 파일 생성 필요

**"필수 설정값이 없습니다":**
→ Project Key 또는 Board ID가 설정 파일에 없음
→ 설정 파일 확인 및 수정 필요

**"설정 파일 형식이 올바르지 않습니다":**
→ 테이블 형식이 깨졌거나 백틱이 누락됨
→ 템플릿 참고하여 형식 수정

---

## MCP 서버 정보

### Jira MCP Server
| 항목 | 값 |
|------|-----|
| **Package** | `mcp-server-jira` |
| **Command** | `uvx mcp-server-jira` |
| **필수 환경변수** | `JIRA_INSTANCE_URL`, `JIRA_USER_EMAIL`, `JIRA_API_TOKEN` |
| **토큰 발급** | https://id.atlassian.com/manage-profile/security/api-tokens |

### 제공되는 MCP 도구
- `mcp__jira__jira_get_issue` - 이슈 상세 조회
- `mcp__jira__jira_search` - JQL 검색
- `mcp__jira__jira_create_issue` - 이슈 생성
- `mcp__jira__jira_update_issue` - 이슈 수정
- `mcp__jira__jira_transition_issue` - 상태 변경
- `mcp__jira__jira_get_agile_boards` - 보드 조회
- `mcp__jira__jira_get_board_issues` - 보드 이슈 조회
- `mcp__jira__jira_add_comment` - 댓글 추가
- 기타 다수...

---

*Last Updated: 2025-01-16*
*Version: 2.0.0 - Phase 0-1 (MCP 연결 확인) 추가*
