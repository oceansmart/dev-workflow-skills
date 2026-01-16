---
name: search
description: "Notion 워크스페이스에서 페이지/데이터베이스를 제목으로 검색합니다."
mcp-server: notion
config: null
---

# /notion:search - Notion 검색

## Usage
```
/notion:search [검색어]
```

## Parameters
- **검색어**: 검색할 페이지/데이터베이스 제목 키워드

## User Input

```text
$ARGUMENTS
```

검색어가 비어있으면 최근 수정된 항목을 조회합니다.

---

## Behavioral Flow

### Phase 0: 환경 확인 (BLOCKING - 필수)

**⛔ 이 단계를 통과하지 않으면 어떤 작업도 진행할 수 없습니다.**

1. **project-context.md 참조**
   - `~/.claude/commands/notion/project-context.md` 문서의 Phase 0 절차 수행

2. **Phase 0-1: MCP 연결 확인**
   - `.mcp.json` 파일에서 "notion" 서버 설정 확인
   - **설정 있음**: Phase 1로 진행
   - **설정 없음**: MCP 설정 안내 → 완전 중단

```
⛔ Notion MCP 연결이 설정되지 않았습니다.

/notion:* 명령어를 사용하려면 먼저 Notion MCP 서버 연결이 필요합니다.

📋 필요한 정보:
   • Notion Integration Token
   • (https://www.notion.so/my-integrations 에서 생성)

MCP 설정을 진행할까요? (예/아니오)
```

**⚠️ MCP 설정이 완료되기 전까지 Phase 1 이후 단계는 절대 실행 불가**

---

### Phase 1: 검색 실행

1. **검색어 확인**
   - `$ARGUMENTS`가 비어있으면 최근 수정된 항목 조회
   - 검색어가 있으면 해당 키워드로 검색

2. **Notion API 호출**
   - `mcp__notion__API-post-search` 도구 사용
   - 파라미터:
     - `query`: 검색어 (비어있으면 생략)
     - `page_size`: 20 (기본값)
     - `sort`: `{ "direction": "descending", "timestamp": "last_edited_time" }`

3. **결과 파싱**
   - 각 결과에서 추출:
     - `id`: 페이지/데이터베이스 ID
     - `object`: 타입 (page 또는 database)
     - `properties.title` 또는 `title`: 제목
     - `last_edited_time`: 최종 수정일
     - `url`: Notion 링크

### Phase 2: 결과 표시

1. **테이블 형식으로 출력**
   - 제목, 타입, 최종 수정일, ID 포함

2. **추가 안내**
   - 상세 조회 방법 안내
   - 데이터베이스 쿼리 방법 안내

---

## Tool Coordination

| 도구 | 용도 |
|------|------|
| `mcp__notion__API-post-search` | 제목 기반 검색 |

---

## Output Format

```
🔍 Notion 검색 결과: "[검색어]"

총 N개 항목을 찾았습니다.

| # | 제목 | 타입 | 최종 수정 | ID |
|---|------|------|----------|-----|
| 1 | 프로젝트 계획 | 📄 page | 2025-01-15 | abc123... |
| 2 | 작업 목록 | 🗄️ database | 2025-01-14 | def456... |
| 3 | 회의록 | 📄 page | 2025-01-13 | ghi789... |

---

👉 **다음 단계**:
- 페이지 상세 보기: `/notion:page [ID]`
- 데이터베이스 쿼리: `/notion:query [ID]`
- 페이지 내용 조회: `/notion:blocks [ID]`
```

---

## Type Icons

| 타입 | 아이콘 |
|------|--------|
| page | 📄 |
| database | 🗄️ |

---

## Error Handling

### 검색 결과 없음
```
🔍 Notion 검색 결과: "[검색어]"

검색 결과가 없습니다.

💡 **팁**:
- 다른 키워드로 검색해 보세요
- 검색어 없이 `/notion:search`를 실행하면 최근 항목을 볼 수 있습니다
```

### API 오류
```
❌ Notion API 오류

오류 내용: [에러 메시지]

💡 **확인 사항**:
- Notion MCP 연결 상태 확인
- API 토큰 유효성 확인
- 해당 페이지에 Integration이 연결되었는지 확인
```

---

## Examples

```bash
# 키워드 검색
/notion:search RFQ
/notion:search 프로젝트
/notion:search meeting

# 최근 수정 항목 조회
/notion:search
```

---

## ⛔ 종료 규칙 (CRITICAL)

**이 명령어는 검색 결과 출력 후 반드시 종료됩니다.**

- ❌ 다음 단계(page, query 등)는 **자동 진행 금지**
- ❌ 사용자의 명시적 요청 없이 다른 `/notion:*` 명령 실행 금지
- ❌ 특정 페이지에 대한 추가 조회/분석 **자동 진행 금지**
- ⚠️ "👉 다음 단계"는 **안내일 뿐**, 자동 실행 대상이 아님
- ✅ 사용자가 **별도 명령으로 요청**해야만 다음 작업 수행
