---
name: list
description: "Jira 프로젝트 백로그 목록 조회"
category: project-management
complexity: basic
mcp-servers: [jira]
personas: []
---

# /jira:list - 백로그 목록 조회

## Usage
```
/jira:list [PROJECT] [OPTIONS]
```

## Parameters
- **PROJECT**: Jira 프로젝트 키 (예: SMFD)
- **OPTIONS** (선택):
  - `--status [상태]`: 특정 상태만 필터 (예: --status "진행 중")
  - `--type [타입]`: 이슈 타입 필터 (예: --type Bug)
  - `--assignee [담당자]`: 담당자 필터 (예: --assignee 최정석)
  - `--me`: 내 담당 이슈만

## Behavioral Flow

### Phase 0: 설정 파일 확인 (필수 - BLOCKING)

**⛔ 이 단계를 통과하지 않으면 어떤 작업도 진행할 수 없습니다.**

1. `.claude/config/jira.md` 파일 존재 여부 확인
2. **파일 있음**: 설정값 로드 (projectKey, boardId, boardType) → Phase 1 진행
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

### Phase 1: 백로그 조회

1. **프로젝트 확인**
   - `jira_get_agile_boards` 호출하여 보드 ID 확인 (설정값과 일치 확인)

2. **백로그 조회**
   - `jira_get_board_issues` 또는 `jira_search` 호출
   - 필터 옵션 적용

3. **결과 표시**
   - 이슈 목록을 테이블 형식으로 표시
   - 키, 제목, 상태, 담당자, 우선순위 포함

## Tool Coordination
- **jira_get_agile_boards**: 보드 ID 조회
- **jira_get_board_issues**: 백로그 이슈 조회
- **jira_search**: JQL 기반 검색 (필터 적용 시)

## Examples

```bash
# 전체 백로그
/jira:list SMFD

# 버그만 조회
/jira:list SMFD --type Bug

# 진행 중인 이슈만
/jira:list SMFD --status "진행 중"

# 내 담당 이슈만
/jira:list SMFD --me

# 특정 담당자 이슈
/jira:list SMFD --assignee 최정석
```

## Output Format

```
📋 SMFD 백로그 (총 21개)

| 키 | 제목 | 타입 | 상태 | 담당자 | 우선순위 |
|----|------|------|------|--------|----------|
| SMFD-203 | BKRQ History - cancelled... | 버그 | 진행 중 | 김종욱 | High |
| SMFD-184 | 추가 assign 기능 | 작업 | 해야 할 일 | 최정석 | Medium |
| SMFD-38 | Email 본문 작성 화면 | 작업 | 해야 할 일 | 최정석 | Medium |
| ... | ... | ... | ... | ... | ... |

📊 상태별 요약
   해야 할 일: 15개
   진행 중: 3개
   TEST: 2개
   완료: 1개

👉 상세 보기: /jira:show SMFD-184
```

## Status Icons
| 상태 | 아이콘 |
|------|--------|
| 해야 할 일 | ⬜ |
| 진행 중 | 🔵 |
| TEST | 🟡 |
| 완료 | ✅ |

## Error Handling
- 프로젝트가 존재하지 않으면 사용 가능한 프로젝트 목록 표시
- 결과가 없으면 필터 조건 확인 안내
- API 오류 시 상세 에러 메시지 표시

---

## ⛔ 종료 규칙 (CRITICAL)

**이 명령어는 목록 출력 후 반드시 종료됩니다.**

- ❌ 다음 단계(show, start 등)는 **자동 진행 금지**
- ❌ 사용자의 명시적 요청 없이 다른 `/jira:*` 명령 실행 금지
- ❌ 특정 티켓에 대한 추가 조회/분석 **자동 진행 금지**
- ⚠️ "👉 상세 보기"는 **안내일 뿐**, 자동 실행 대상이 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행
