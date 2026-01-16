---
name: merge
description: "PR 머지 및 브랜치 정리"
category: project-management
complexity: enhanced
mcp-servers: [jira]
personas: []
---

# /jira:merge - PR 머지 및 브랜치 정리

## Usage
```
/jira:merge [PR-NUMBER]
```

## Parameters
- **PR-NUMBER**: GitHub PR 번호 (예: 45)

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

PR: #[PR-NUMBER]
티켓: [TICKET-ID] (PR 본문에서 추출)
브랜치: [BRANCH] → main

수행 단계:
1. 🟡 머지 조건 확인 (CI, 리뷰, 충돌)
2. 🔴 PR 머지 → squash merge
3. 🟡 Jira 댓글 추가
4. 🔴 Jira 상태 변경 → "TEST"
5. 🔴 Jira 담당자 변경 → "Younghwa Lee"
6. 🟡 브랜치 정리 (원격/로컬)

위 계획대로 진행할까요? (예/아니오)
```
**→ 사용자 승인 후에만 Phase 1 진행**

---

### Phase 1: PR 정보 조회
- `gh pr view [PR] --json` 실행하여 PR 정보 조회
- PR 본문에서 Jira 티켓 번호 추출 (SMFD-XXX 패턴)
- 브랜치명 확인

### Phase 2: 🟡 머지 조건 검증 (Important)
```
LOOP (최대 3회):
  실행: gh pr view [PR] --json state,mergeable,statusCheckRollup,reviews
  검증: state=="OPEN" AND mergeable=="MERGEABLE" AND checks 통과
  IF 성공: 출력 "✅ 머지 조건 충족" → Phase 3으로
  IF 실패 && 재시도<3: 출력 "⚠️ 조건 미충족 - 재확인..." → LOOP 반복
  IF 실패 && 재시도>=3: 출력 "❌ 머지 조건 미충족 - 중단" → 에러 종료
```

### Phase 3: 🔴 PR 머지 실행 (Critical)
```
LOOP:
  실행: gh pr merge [PR] --squash --delete-branch
  검증: gh pr view [PR] --json merged → "merged": true
  IF 성공: 출력 "✅ PR 머지 완료" → Phase 4로
  IF 실패: 출력 "❌ PR 머지 실패 - 재시도..." → LOOP 반복
```

### Phase 4: 🟡 Jira 댓글 추가 (Important)
```
LOOP (최대 3회):
  실행: jira_add_comment("PR #[PR] 머지 완료: [PR URL]")
  검증: jira_get_issue(comment_limit=3) → "머지 완료" 포함
  IF 성공: 출력 "✅ 댓글 추가 완료" → Phase 5로
  IF 실패 && 재시도<3: 출력 "⚠️ 댓글 추가 실패 - 재시도..." → LOOP 반복
  IF 실패 && 재시도>=3: 출력 "⚠️ 댓글 추가 실패 - 건너뜀" → Phase 5로
```

### Phase 5: 🔴 Jira 상태 변경 (Critical)
```
LOOP:
  실행: jira_get_transitions → jira_transition_issue("TEST")
  검증: jira_get_issue → status.name == "TEST"
  IF 성공: 출력 "✅ 상태 변경 완료" → Phase 6으로
  IF 실패: 출력 "❌ 상태 변경 실패 - 재시도..." → LOOP 반복
```

### Phase 6: 🔴 Jira 담당자 변경 (Critical)
```
LOOP:
  실행: jira_update_issue(assignee="yhlee@oceansmart.ai")
  검증: jira_get_issue → assignee.displayName == "Younghwa Lee"
  IF 성공: 출력 "✅ 담당자 변경 완료" → Phase 7로
  IF 실패: 출력 "❌ 담당자 변경 실패 - 재시도..." → LOOP 반복
```

### Phase 7: 🟡 브랜치 정리 (Important)
```
삭제_대상_브랜치 = [PR에서 추출한 브랜치명]

## Step 1: main으로 이동 (필수 - 현재 브랜치는 삭제 불가)
실행: git checkout main
IF exit code != 0:
  출력 "❌ main 체크아웃 실패"
  → Phase 8로 (건너뜀)

## Step 2: main 최신화
실행: git pull origin main

## Step 3: 로컬 브랜치 삭제
실행: git branch -D [삭제_대상_브랜치]
# -D 사용 이유: squash merge 후에는 -d가 "not fully merged" 에러 발생
# 이미 PR 머지 완료 확인했으므로 -D 강제 삭제 안전

IF exit code == 0:
  출력 "✅ 로컬 브랜치 삭제 완료"
ELSE:
  출력 "⚠️ 로컬 브랜치 삭제 실패 (수동 삭제 필요)"
  출력 "   git branch -D [삭제_대상_브랜치]"

## Step 4: 검증
실행: git branch --list "[삭제_대상_브랜치]"
IF 출력 == "":
  출력 "✅ 브랜치 정리 완료" → Phase 8로
ELSE:
  출력 "⚠️ 브랜치가 여전히 존재 - 수동 삭제 필요"
  출력 "   git branch -D [삭제_대상_브랜치]"
  → Phase 8로

⚠️ 참고: 원격 브랜치는 Phase 3의 `gh pr merge --delete-branch`에서 이미 삭제됨
   이 단계는 로컬 브랜치 정리만 담당
```

### Phase 8: 결과 반환
```
🔀 PR 머지 완료

📋 PR #[NUMBER]
   머지: ✅ squash merge
   커밋: [COMMIT_SHA]
   🔗 [COMMIT_URL]

📌 [TICKET-ID] 이슈 제목
   댓글 추가: ✅/⚠️
   상태 변경: ✅ TEST
   담당자 변경: ✅ Younghwa Lee

🧹 브랜치 정리
   로컬 삭제: ✅/⚠️

👉 다음 단계:
   1. QA 검증 진행
   2. 새 작업: /jira:start [NEW-TICKET]
```

## ⛔ 종료 규칙 (CRITICAL)

**이 명령어는 Phase 8 결과 반환 후 반드시 종료됩니다.**

- ❌ 다음 단계(새 작업 시작 등)는 **자동 진행 금지**
- ❌ 사용자의 명시적 요청 없이 다른 `/jira:*` 명령 실행 금지
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상이 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행

## Tool Coordination
- **jira_get_issue**: 연결된 Jira 이슈 조회
- **jira_add_comment**: 머지 완료 댓글 추가
- **jira_transition_issue**: 상태를 `TEST`로 변경 (transition_id: 2)
- **jira_update_issue**: 담당자를 `Younghwa Lee`로 변경
- **Bash (gh)**: PR 상태 확인 및 머지 실행
- **Bash (git)**: main 브랜치 업데이트

## Merge Conditions (Required)

| 조건 | 설명 | 필수 |
|------|------|------|
| CI 통과 | 모든 CI 체크 통과 | ✅ |
| 리뷰 승인 | 최소 1개 승인 (설정에 따름) | ⚠️ |
| 충돌 없음 | main과 충돌 없음 | ✅ |
| 검증 완료 | /jira:verify 통과 | 권장 |

## Examples

```bash
# 기본 사용
/jira:merge 45

# PR 번호로 머지
/jira:merge 123
```

## Output Format

### 성공 시
```
🔀 PR 머지 프로세스

📋 PR #45 정보
   제목: [SMFD-184] 추가 assign 기능
   브랜치: feature/SMFD-184-add-assign-feature → main
   상태: Open
   리뷰: ✅ Approved (2)
   CI: ✅ All checks passed

🔍 머지 조건 확인
   CI 통과: ✅
   리뷰 승인: ✅
   충돌 없음: ✅

🚀 머지 실행 중...
   PR 머지: ✅ (squash)

📌 Jira 티켓 처리
   [SMFD-184] 추가 assign 기능
   댓글 추가: ✅
   상태 변경: ✅ (진행 중 → TEST)
   담당자 변경: ✅ (→ Younghwa Lee)

🧹 브랜치 정리
   main 업데이트: ✅
   원격 브랜치 삭제: ✅
   로컬 브랜치 삭제: ✅

✅ 머지 완료!

   커밋: abc1234 - [SMFD-184] 추가 assign 기능 (#45)
   🔗 https://github.com/org/repo/commit/abc1234
```

### 실패 시
```
🔀 PR 머지 프로세스

📋 PR #45 정보
   제목: [SMFD-184] 추가 assign 기능
   상태: Open

🔍 머지 조건 확인
   CI 통과: ❌ (2 failed)
   리뷰 승인: ✅
   충돌 없음: ✅

❌ 머지 불가

🔴 실패한 CI 체크:
   - build: ❌ TypeScript compilation failed
   - test: ❌ 3 tests failed

👉 CI 오류 수정 후 다시 시도하세요.
   1. 오류 확인: gh pr checks 45
   2. 수정 후 push
   3. 다시 실행: /jira:merge 45
```

## Post-Merge Documentation

머지 완료 후 필요한 문서화 작업 안내:

### Backend 변경 시
- Swagger 문서 확인
- API 어노테이션 추가
- doc/backend/api-guide.md 업데이트

### Frontend 변경 시
- 컴포넌트 문서 업데이트
- Storybook 스토리 추가

### 구조 변경 시
- architecture.md 업데이트
- README.md 업데이트

## Verification Procedures (검증 절차)

각 단계는 성공할 때까지 반복 실행됩니다. 실패 시 즉시 재시도하며, 간격 없이 성공할 때까지 계속합니다.

### 🔴 Critical (필수 - 실패 시 전체 프로세스 중단)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| PR 머지 | `gh pr view [PR] --json merged` | `"merged": true` |
| Jira 상태 변경 | `jira_get_issue` | `status.name == "TEST"` |
| Jira 담당자 변경 | `jira_get_issue` | `assignee.displayName == "Younghwa Lee"` |

### 🟡 Important (중요 - 실패 시 경고 후 계속)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| 댓글 추가 | `jira_get_issue` (comments 확인) | 댓글에 "머지 완료" 텍스트 포함 |
| 원격 브랜치 삭제 | `git branch -r \| grep [branch]` | 결과 없음 (빈 출력) |
| 로컬 브랜치 삭제 | `git branch \| grep [branch]` | 결과 없음 (빈 출력) |

### 🟢 Optional (선택 - 실패 시 안내만)

| 단계 | 검증 방법 | 성공 조건 |
|------|----------|----------|
| main 브랜치 pull | `git status` | "Your branch is up to date" 메시지 |

### 검증 실행 로직

```
FOR EACH step IN [PR머지, Jira상태, Jira담당자, 댓글, 원격브랜치, 로컬브랜치, main_pull]:
    WHILE NOT success:
        execute(step)
        result = verify(step)
        IF result == success:
            print("✅ {step} 완료")
            BREAK
        ELSE:
            IF step.priority == CRITICAL:
                print("❌ {step} 실패 - 재시도 중...")
                CONTINUE  # 즉시 재시도
            ELIF step.priority == IMPORTANT:
                print("⚠️ {step} 실패 - 재시도 중...")
                CONTINUE  # 즉시 재시도
            ELSE:  # OPTIONAL
                print("ℹ️ {step} 실패 - 건너뜀")
                BREAK  # 다음 단계로
```

### 검증 상세 명령어

**PR 머지 검증:**
```bash
gh pr view [PR_NUMBER] --json merged --jq '.merged'
# 예상 결과: true
```

**Jira 상태/담당자 검증:**
```
jira_get_issue(issue_key="[TICKET]", fields="status,assignee")
# 예상 결과:
#   status.name == "TEST"
#   assignee.displayName == "Younghwa Lee"
```

**댓글 검증:**
```
jira_get_issue(issue_key="[TICKET]", comment_limit=5)
# 예상 결과: comments 배열에 "머지 완료" 포함된 댓글 존재
```

**브랜치 삭제 검증:**
```bash
# 원격 브랜치
git branch -r | grep "origin/[BRANCH_NAME]"
# 예상 결과: 출력 없음 (exit code 1)

# 로컬 브랜치
git branch | grep "[BRANCH_NAME]"
# 예상 결과: 출력 없음 (exit code 1)
```

## Error Handling
- PR이 존재하지 않으면 에러 메시지 표시
- 머지 조건 미충족 시 상세 사유 표시
- 이미 머지된 PR이면 상태 안내
- Jira 연동 실패 시 수동 처리 안내

## Pre-conditions
- gh CLI 설치 및 인증 완료
- 저장소 쓰기 권한 보유
- Jira MCP 연결 상태
