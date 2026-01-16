---
name: pr
description: "Jira 티켓 기반 PR 생성 - 템플릿 자동 적용 및 Jira 연동"
category: project-management
complexity: enhanced
mcp-servers: [jira]
personas: []
---

# /jira:pr - Jira 티켓 기반 PR 생성

## Usage
```
/jira:pr [TICKET-ID]
```

## Parameters
- **TICKET-ID**: Jira 이슈 키 (예: SMFD-184)

## Behavioral Flow

### ⚠️ 필수 규칙
1. **🔴 verify 통과 필수**: PR 생성 전 /jira:verify 통과 확인 필수
2. **작업계획 승인 필수**: 실행 전 반드시 사용자에게 작업계획 제시 → 승인 후 진행
3. **검증 통과 필수**: 각 단계 실행 직후 즉시 검증 → 검증 통과 전까지 다음 단계 진행 금지
4. **재시도 규칙**: Critical 단계 실패 시 성공할 때까지 무한 재시도

---

### Phase 0: 설정 파일 확인 (필수 - BLOCKING)

**⛔ 이 단계를 통과하지 않으면 어떤 작업도 진행할 수 없습니다.**

1. `.claude/config/jira.md` 파일 존재 여부 확인
2. **파일 있음**: 설정값 로드 → Phase 0.5 진행
3. **파일 없음**: 아래 메시지 출력 후 **완전 중단**

```
⛔ Jira 설정 파일이 없습니다.

/jira:* 명령어를 사용하려면 먼저 설정 파일이 필요합니다.

📁 필요한 파일: .claude/config/jira.md

설정 파일을 생성할까요? (예/아니오)

→ "예" 선택 시: 설정 정보 입력 → 파일 생성 → 다시 명령어 실행
→ "아니오" 선택 시: 명령어 종료
```

**⚠️ 설정 파일이 생성되기 전까지 이후 단계는 절대 실행 불가**

---

### Phase 0.5: 🔴 verify 통과 확인 (Critical)
```
⚠️ verify 통과 확인 필수!

1. 최근 /jira:verify 실행 결과 확인
2. Jira 댓글에서 "검증 완료" 확인:
   jira_get_issue(issue_key="[TICKET]", comment_limit=5)
   → 댓글에 "검증 완료: Frontend ✅ Backend ✅" 포함 여부

IF 검증 통과 기록 없음:
   출력:
   ❌ PR 생성 불가

   /jira:verify 통과 기록이 없습니다.
   먼저 검증을 실행해주세요:

   👉 /jira:verify [TICKET] both

   → 프로세스 중단

IF 검증 통과 기록 있음:
   출력 "✅ verify 통과 확인됨" → Phase 0.5로
```

### Phase 0.5: 작업계획 승인 요청
```
📋 작업계획 확인 요청

티켓: [TICKET-ID]
브랜치: [CURRENT-BRANCH]

수행 단계:
1. 🟡 원격 브랜치 푸시
2. 🔴 PR 생성 → "[TICKET-ID] 이슈 제목"
3. 🟡 Jira 링크 연결
4. 🟡 Jira 댓글 추가

위 계획대로 진행할까요? (예/아니오)
```
**→ 사용자 승인 후에만 Phase 1 진행**

---

### Phase 1: 이슈 정보 조회
- `jira_get_issue` 호출하여 이슈 제목/설명 가져오기
- PR 제목 및 본문 생성에 활용

### Phase 2: Git 상태 확인
- 현재 브랜치 확인
- 커밋 내역 확인
- 푸시되지 않은 변경사항 확인

### Phase 3: 🟡 원격 브랜치 푸시 (Important)
```
LOOP (최대 3회):
  실행: git push -u origin [current-branch]
  검증: git branch -r | grep "origin/[branch]" → 출력 있음
  IF 성공: 출력 "✅ 원격 푸시 완료" → Phase 4로
  IF 실패 && 재시도<3: 출력 "⚠️ 푸시 실패 - 재시도..." → LOOP 반복
  IF 실패 && 재시도>=3: 출력 "❌ 푸시 실패 - 중단" → 에러 종료
```

### Phase 4: 🔴 PR 생성 (Critical)
```
LOOP:
  실행: gh pr create --title "[TICKET-ID] 이슈 제목" --body "..."
  검증: gh pr view --json number,url → PR 번호와 URL 반환
  IF 성공: 출력 "✅ PR 생성 완료: #[NUMBER]" → Phase 5로
  IF 실패: 출력 "❌ PR 생성 실패 - 재시도..." → LOOP 반복
```

### Phase 5: 🟡 Jira 링크 연결 (Important)
```
LOOP (최대 3회):
  실행: jira_create_remote_issue_link(PR URL)
  검증: jira_get_issue(expand="links") → PR URL 포함
  IF 성공: 출력 "✅ Jira 링크 연결 완료" → Phase 6으로
  IF 실패 && 재시도<3: 출력 "⚠️ 링크 연결 실패 - 재시도..." → LOOP 반복
  IF 실패 && 재시도>=3: 출력 "⚠️ 링크 연결 실패 - 건너뜀" → Phase 6으로
```

### Phase 6: 🟡 Jira 댓글 추가 (Important)
```
LOOP (최대 3회):
  실행: jira_add_comment("PR 생성: [PR URL]")
  검증: jira_get_issue(comment_limit=3) → PR URL 포함
  IF 성공: 출력 "✅ 댓글 추가 완료" → Phase 7로
  IF 실패 && 재시도<3: 출력 "⚠️ 댓글 추가 실패 - 재시도..." → LOOP 반복
  IF 실패 && 재시도>=3: 출력 "⚠️ 댓글 추가 실패 - 건너뜀" → Phase 7로
```

### Phase 7: 결과 반환
```
📝 PR 생성 완료

📌 [TICKET-ID] 이슈 제목
   PR: ✅ #[NUMBER]
   🔗 [PR URL]

🔗 Jira 연동
   링크 연결: ✅/⚠️
   댓글 추가: ✅/⚠️

👉 다음 단계:
   1. CodeRabbit 리뷰 확인 및 피드백 반영
   2. 리뷰어 승인 대기
   3. 머지: /jira:merge [PR번호]
```

## ⛔ 종료 규칙 (CRITICAL)

**이 명령어는 Phase 7 결과 반환 후 반드시 종료됩니다.**

- ❌ 다음 단계(verify, merge 등)는 **자동 진행 금지**
- ❌ 사용자의 명시적 요청 없이 다른 `/jira:*` 명령 실행 금지
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상이 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행

## Tool Coordination
- **jira_get_issue**: 이슈 정보 조회
- **jira_create_remote_issue_link**: PR URL을 Jira에 연결
- **jira_add_comment**: PR 생성 알림 댓글
- **Bash (git)**: 브랜치 푸시
- **Bash (gh)**: GitHub PR 생성

## PR Template

```markdown
## Jira Ticket
[{TICKET-ID}](https://oceansmart.atlassian.net/browse/{TICKET-ID})

## Summary
{이슈 제목 및 설명 기반 요약}

## Changes
- {커밋 내역 기반 변경사항}

## Checklist
- [ ] 코드 리뷰 완료
- [ ] 테스트 통과
- [ ] 문서 업데이트 (필요시)

## Test Plan
- [ ] 로컬 테스트 완료
- [ ] 기능 검증 완료
```

## Examples

```bash
# 기본 사용
/jira:pr SMFD-184

# 현재 브랜치의 티켓으로 PR 생성
/jira:pr SMFD-38
```

## Output Format

```
📝 PR 생성 준비

📌 [SMFD-184] 추가 assign 기능
   브랜치: feature/SMFD-184-add-assign-feature
   커밋: 3개

🚀 PR 생성 중...
   원격 푸시: ✅
   PR 생성: ✅

✅ PR 생성 완료

   PR #45: [SMFD-184] 추가 assign 기능
   🔗 https://github.com/org/repo/pull/45

🔗 Jira 연동
   링크 연결: ✅
   댓글 추가: ✅

👉 다음 단계:
   1. CodeRabbit 리뷰 확인
   2. 검증: /jira:verify SMFD-184 both
   3. 머지: /jira:merge 45
```

## Verification Procedures (검증 절차)

각 단계는 성공할 때까지 반복 실행됩니다. 실패 시 즉시 재시도하며, 간격 없이 성공할 때까지 계속합니다.

### 🔴 Critical (필수 - 실패 시 전체 프로세스 중단)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| PR 생성 | `gh pr view --json number,url` | PR 번호와 URL 반환 |

### 🟡 Important (중요 - 실패 시 경고 후 계속)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| 원격 브랜치 푸시 | `git branch -r \| grep [branch]` | 원격 브랜치 존재 |
| Jira 링크 연결 | `jira_get_issue` (links 확인) | PR URL이 링크에 포함 |
| PR 댓글 추가 | `jira_get_issue` (comments 확인) | 댓글에 PR 링크 포함 |

### 검증 상세 명령어

**PR 생성 검증:**
```bash
gh pr view --json number,url,state
# 예상 결과: {"number": 45, "url": "https://...", "state": "OPEN"}
```

**원격 브랜치 푸시 검증:**
```bash
git branch -r | grep "origin/feature/[TICKET]"
# 예상 결과: origin/feature/SMFD-XXX-description 출력
```

**Jira 링크 연결 검증:**
```
jira_get_issue(issue_key="[TICKET]", expand="links")
# 예상 결과: remoteLinks에 PR URL 포함
```

**Jira 댓글 검증:**
```
jira_get_issue(issue_key="[TICKET]", comment_limit=3)
# 예상 결과: comments 배열에 PR URL 포함된 댓글 존재
```

## Error Handling
- 이슈가 존재하지 않으면 에러 메시지 표시
- 커밋이 없으면 PR 생성 불가 안내
- gh CLI 미설치 시 설치 안내
- 이미 PR이 존재하면 기존 PR 링크 제공

## Pre-conditions
- feature 브랜치에서 실행
- 최소 1개 이상의 커밋 존재
- gh CLI 설치 및 인증 완료
- Jira MCP 연결 상태
