# 실습: Python FastMCP로 나만의 MCP 서버 만들기

공식 Python SDK는 FastAPI와 매우 유사한 직관적인 데코레이터 방식의 **`FastMCP`**를 제공합니다. 이번 실습에서는 도구(Tool), 리소스(Resource), 프롬프트(Prompt)를 모두 포함하는 나만의 서버를 직접 작성해 봅니다.

---

## 1. 실습 환경 준비

Python 가상환경을 생성하거나 `uv`를 사용해 필요한 패키지를 설치합니다.

```bash
cd /home/ysik/workspace/study/MCP/examples/python-fastmcp

# 가상환경 생성 및 활성화
python3 -m venv .venv
source .venv/bin/activate

# 의존성 패키지 설치
pip install -r requirements.txt
```

> **참고**: `uv`를 사용한다면 `uv init` 후 `uv add "mcp[cli]"`로 더 빠르게 설정할 수 있습니다.

---

## 2. 서버 코드 작성 (`server.py`)

같은 폴더에 [server.py](./server.py) 파일을 생성하고 아래와 같이 작성합니다:

```python
"""
MCP 실습용 서버 (FastMCP 활용)
"""
from datetime import datetime
from mcp.server.fastmcp import FastMCP

# 1. 서버 인스턴스 초기화
mcp = FastMCP(
    "My Study MCP Server",
    dependencies=["mcp"],
)

# 메모 저장소 (메모리 딕셔너리)
MEMO_STORE: dict[str, str] = {
    "welcome": "MCP 학습을 환영합니다! 이것은 샘플 리소스입니다.",
    "todo": "1. MCP 개념 이해\n2. 서버 구현\n3. 클라이언트 연동",
}


# ==========================================
# 2. Tools (도구) 정의
# ==========================================
@mcp.tool()
def add_numbers(a: float, b: float) -> float:
  """두 숫자의 합을 계산합니다.

  Args:
      a: 첫 번째 숫자
      b: 두 번째 숫자
  """
  return a + b


@mcp.tool()
def save_memo(title: str, content: str) -> str:
  """새로운 메모를 메모리에 저장합니다.

  Args:
      title: 메모 제목
      content: 메모 본문 내용
  """
  MEMO_STORE[title] = content
  return f"메모 '{title}'가 성공적으로 저장되었습니다."


@mcp.tool()
def get_current_time(timezone_name: str = "Asia/Seoul") -> str:
  """현재 시각을 반환합니다.

  Args:
      timezone_name: 타임존 이름 (기본값: Asia/Seoul)
  """
  now = datetime.now()
  return f"현재 시각 ({timezone_name}): {now.strftime('%Y-%m-%d %H:%M:%S')}"


# ==========================================
# 3. Resources (리소스) 정의
# ==========================================
@mcp.resource("memo://all")
def list_all_memos() -> str:
  """현재 저장된 모든 메모 목록을 반환합니다."""
  if not MEMO_STORE:
    return "저장된 메모가 없습니다."
  items = [f"- [{k}]: {v}" for k, v in MEMO_STORE.items()]
  return "\n".join(items)


@mcp.resource("memo://item/{title}")
def read_memo(title: str) -> str:
  """특정 제목의 메모 내용을 읽어옵니다.

  Args:
      title: 조회할 메모의 제목
  """
  if title not in MEMO_STORE:
    return f"오류: '{title}' 메모를 찾을 수 없습니다."
  return MEMO_STORE[title]


# ==========================================
# 4. Prompts (프롬프트 템플릿) 정의
# ==========================================
@mcp.prompt()
def study_summary(topic: str, difficulty: str = "초급") -> str:
  """학습한 주제에 대해 요약 정리를 요청하는 프롬프트 템플릿입니다.

  Args:
      topic: 학습한 주제 (예: MCP Architecture)
      difficulty: 난이도 (초급, 중급, 고급)
  """
  return (
      f"당신은 친절한 AI 튜터입니다.\n"
      f"사용자가 방금 '{topic}'에 대해 공부했습니다.\n"
      f"난이도({difficulty})에 맞춰 핵심 내용을 3가지 포인트로 요약하고,\n"
      f"이해도를 점검할 수 있는 퀴즈 1개를 출제해주세요."
  )


# ==========================================
# 5. 실행 엔트리포인트 (stdio 모드 기본)
# ==========================================
if __name__ == "__main__":
  # stdio 모드로 서버 구동
  mcp.run(transport="stdio")
```

---

## 3. MCP Inspector로 서버 테스트 및 디버깅

서버를 실제 클라이언트에 붙이기 전에, Anthropic이 제공하는 웹 기반 디버깅 도구인 **MCP Inspector**로 직접 테스트할 수 있습니다.

```bash
# MCP Inspector 실행 명령어
npx @modelcontextprotocol/inspector python3 /home/ysik/workspace/study/MCP/examples/python-fastmcp/server.py
```

또는 Python mcp CLI가 설치되어 있다면:
```bash
mcp dev server.py
```

### Inspector 브라우저 화면에서 확인할 내용:
1. **Tools 탭**: `add_numbers`, `save_memo`, `get_current_time`이 등록되어 있는지 확인하고, 파라미터를 입력해 직접 실행(Run)해 봅니다.
2. **Resources 탭**: `memo://all` 및 템플릿 `memo://item/{title}`의 데이터를 읽어봅니다.
3. **Prompts 탭**: `study_summary` 프롬프트를 렌더링해보고 반환되는 텍스트를 확인합니다.
4. **Console/Notification**: 실시간 교환되는 JSON-RPC 2.0 패킷 로그를 관찰합니다.
