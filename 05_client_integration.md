# 05. 클라이언트 연동 및 보안 가이드

작성한 MCP 서버는 다양한 AI 클라이언트(Host)에 등록하여 대화 중 도구로 호출하거나 리소스를 참조하게 할 수 있습니다.

---

## 1. Claude Desktop 연동

Claude Desktop은 MCP의 대표적인 공식 Host입니다. 설정 파일에 서버 실행 명령어를 등록하기만 하면 됩니다.

### 설정 파일 위치
- **Linux**: `~/.config/Claude/claude_desktop_config.json`
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

### 설정 파일 예시

```json
{
  "mcpServers": {
    "my-study-server": {
      "command": "/home/ysik/workspace/study/MCP/.venv/bin/python3",
      "args": [
        "/home/ysik/workspace/study/MCP/server.py"
      ],
      "env": {
        "PYTHONUNBUFFERED": "1"
      }
    }
  }
}
```

> **핵심 팁**: `command`와 `args`에는 상대 경로가 아닌 **반드시 절대 경로**를 사용해야 하며, 가상환경의 python 인터프리터를 직접 지정해야 패키지 의존성 문제가 발생하지 않습니다.

---

## 2. Cursor IDE 연동

Cursor IDE(0.45 버전 이상)에서도 MCP를 지원합니다:

1. **설정 경로**: `Settings` $\rightarrow$ `Features` $\rightarrow$ `MCP`
2. **새 서버 추가 (`+ Add New MCP Server`)**:
   - **Name**: `my-study-server`
   - **Type**: `command` (stdio 방식)
   - **Command**: `/home/ysik/workspace/study/MCP/.venv/bin/python3 /home/ysik/workspace/study/MCP/server.py`
3. 또는 프로젝트 루트의 `.cursor/mcp.json`에 직접 정의할 수도 있습니다.

---

## 3. 자주 쓰이는 공식 & 커뮤니티 MCP 서버

자체 제작 서버 외에도 오픈소스로 공개된 수많은 고품질 MCP 서버를 즉시 활용할 수 있습니다.

| MCP 서버 | 역할 | 실행 명령어 (npx/uvx) |
| :--- | :--- | :--- |
| **Filesystem** | 로컬 지정 디렉토리 내 파일 읽기/쓰기 | `npx -y @modelcontextprotocol/server-filesystem <디렉토리>` |
| **SQLite / Postgres** | 데이터베이스 스키마 조회 및 SQL 실행 | `uvx mcp-server-sqlite --db-path <db파일>` |
| **Git / GitHub** | 커밋 조회, diff 확인, PR 생성 | `npx -y @modelcontextprotocol/server-github` |
| **Brave Search** | 실시간 웹 검색 및 뉴스 조회 | `npx -y @modelcontextprotocol/server-brave-search` |
| **Puppeteer / Playwright** | 브라우저 자동화 및 웹 페이지 스크래핑 | `npx -y @modelcontextprotocol/server-puppeteer` |

---

## 4. MCP 보안 및 개발 시 주의사항

### 1) Stdout 오염 방지 (중요!)
- Stdio 통신 방식에서는 표준 출력(`stdout`)으로 오직 올바른 JSON-RPC 2.0 포맷 문자열만 전송되어야 합니다.
- 만약 코드 내에서 `print("디버깅 로그")` 등을 호출하면 클라이언트가 JSON 파싱 에러를 내며 연결이 끊어집니다.
- **해결책**: 로그는 반드시 `logging` 모듈을 쓰거나 표준 에러(`sys.stderr.write()`)로 출력해야 합니다. FastMCP는 기본적으로 이를 안전하게 처리합니다.

### 2) 최소 권한 원칙 (Principle of Least Privilege)
- 파일시스템 서버나 쉘 실행 서버를 연동할 때는 AI가 시스템 전체에 접근하지 못하도록 특정 디렉토리(예: `/home/ysik/workspace`)로 접근 범위를 제한하십시오.

### 3) Human-in-the-Loop (사용자 승인)
- 파일 삭제, DB 업데이트, 메일 전송 등 상태를 변경하는 작업(Tools)은 클라이언트에서 반드시 사용자 확인 절차를 거치도록 설계하는 것이 안전합니다.
