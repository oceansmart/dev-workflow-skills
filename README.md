# dev-workflow-skills

Claude Code plugin for development workflow automation with Jira and Notion integration.

## Installation

```bash
/plugin install oceansmart/dev-workflow-skills
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

## Setup

### 1. MCP Server Configuration

This plugin requires MCP servers for Jira and Notion. On first use, the plugin will guide you through setup.

**Jira MCP:**
- Jira Instance URL
- User Email
- API Token ([Get token](https://id.atlassian.com/manage-profile/security/api-tokens))

**Notion MCP:**
- Integration Token ([Create integration](https://www.notion.so/my-integrations))

### 2. Project Configuration

For Jira commands, create `.claude/config/jira.md` in your project root with:
- Project Key (e.g., `SMFD`)
- Board ID
- Board Type (scrum/kanban)

## License

MIT
