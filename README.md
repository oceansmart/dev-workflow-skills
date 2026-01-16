# dev-workflow-skills

Claude Code plugin for development workflow automation with Jira and Notion integration.

## Installation

```bash
# 1. 마켓플레이스 등록
claude plugin marketplace add oceansmart/dev-workflow-skills

# 2. 플러그인 설치
claude plugin install dev-workflow-skills

# 3. Claude Code 재시작
```

## Available Commands

### Jira Commands

| Command | Description |
|---------|-------------|
| `/jira:list` | 백로그 목록 조회 |
| `/jira:show` | 이슈 상세 정보 조회 |
| `/jira:create` | 티켓 생성 |
| `/jira:start` | 작업 시작 (상태 변경 + 브랜치 생성) |
| `/jira:assign` | 담당자 지정 |
| `/jira:implement` | 티켓 구현 |
| `/jira:verify` | 완료 전 검증 |
| `/jira:pr` | PR 생성 |
| `/jira:merge` | PR 머지 및 브랜치 정리 |

### Implementation Guidelines

| Command | Description |
|---------|-------------|
| `/jira:impl-backend-api` | Backend API 구현 지침 |
| `/jira:impl-frontend` | Frontend 구현 지침 |
| `/jira:impl-database` | Database 작업 지침 |
| `/jira:impl-bugfix` | 버그 수정 지침 |

### Notion Commands

| Command | Description |
|---------|-------------|
| `/notion:search` | 페이지/데이터베이스 검색 |

## Setup Requirements

### 1. MCP Server Configuration

첫 명령어 실행 시 자동 안내됩니다.

**Jira MCP:**
- Jira Instance URL (예: `https://company.atlassian.net`)
- User Email
- API Token ([발급 링크](https://id.atlassian.com/manage-profile/security/api-tokens))

**Notion MCP:**
- Integration Token ([발급 링크](https://www.notion.so/my-integrations))

### 2. Project Configuration

Jira 명령어 사용 시 `.claude/config/jira.md` 파일 필요 (첫 실행 시 자동 생성 안내)

필요한 정보:
- Project Key (예: `SMFD`)
- Board ID
- Board Type (scrum/kanban)

## Plugin Management

```bash
# 설치된 플러그인 목록
claude plugin list

# 플러그인 업데이트
claude plugin marketplace update dev-workflow-skills
claude plugin update dev-workflow-skills

# 플러그인 제거
claude plugin uninstall dev-workflow-skills
```

## Documentation

자세한 설치 및 설정 가이드: [INSTALLATION_GUIDE.md](./INSTALLATION_GUIDE.md)

## License

MIT
