# 01. MCP (Model Context Protocol) 소개

## 1. MCP란 무엇인가?

**Model Context Protocol (MCP)**은 AI 모델(LLM)과 다양한 데이터 소스, 개발 도구, 사내 비즈니스 로직을 연결하는 **개방형 표준 프로토콜(Open Standard Protocol)**입니다.

- **발표자**: Anthropic (2024년 11월 오픈소스 공개)
- **비유**: 하드웨어 생태계의 **USB-C 포트**, 소프트웨어 개발 생태계의 **LSP (Language Server Protocol)**
- **핵심 목표**: AI 애플리케이션이 파편화된 인터페이스 없이, 어떤 데이터베이스·개발 도구·사내 시스템과도 일관된 방식으로 대화할 수 있도록 표준화하는 것.

---

## 2. 왜 MCP가 필요하게 되었는가? ($M \times N$ 문제)

### 기존의 도구 연동 방식
LLM이 외부 도구를 쓰거나 데이터를 읽기 위해서는 각 플랫폼마다 고유한 연동 코드를 작성해야 했습니다:
- OpenAI Function Calling 스키마 정의
- Anthropic Tool Use 스키마 정의
- LangChain Tool wrapper 구현
- LlamaIndex Tool spec 작성
- Cursor / VS Code 확장 플러그인 전용 코드 작성

데이터 소스가 $N$개(PostgreSQL, GitHub, Slack, Notion 등)이고, AI 클라이언트가 $M$개(Claude Desktop, Cursor, Custom Agent, LangChain 등)라면, **총 $M \times N$개의 어댑터**를 개발하고 유지보수해야 했습니다.

```
[클라이언트 M개]                [데이터/도구 N개]
Claude Desktop  ───\  /───  PostgreSQL
Cursor IDE      ────><────  GitHub
Custom Agent    ───/  \───  Slack
                 (복잡도: M x N)
```

### MCP의 해결책 (LSP에서 영감을 얻음)
과거 VS Code가 프로그래밍 언어 지원을 위해 **LSP(Language Server Protocol)**를 도입하여 모든 언어 분석기와 에디터를 통일했듯이, MCP는 AI 도구 생태계를 통일합니다.

```
[클라이언트 M개]        [표준 프로토콜]        [서버 N개]
Claude Desktop  ───┐                     ┌─── PostgreSQL
Cursor IDE      ───┼──>  [   MCP   ]  ───┼─── GitHub
Custom Agent    ───┘                     └─── Slack
                        (복잡도: M + N)
```

- **도구 제공자**: MCP 서버를 딱 1번만 만들면 모든 지원 클라이언트에서 동작
- **클라이언트 개발자**: MCP 클라이언트만 구현하면 수많은 오픈소스 MCP 서버를 즉시 활용 가능

---

## 3. MCP의 주요 특징과 장점

1. **표준화된 프리미티브 (Standard Primitives)**
   - 단순히 "함수 호출(Tool)"만 표준화한 것이 아니라, **Resource(읽기 전용 데이터/컨텍스트)**와 **Prompt(사용자 템플릿/워크플로우)**까지 규격화했습니다.
2. **보안과 사용자 주도 통제 (Security & User Control)**
   - 데이터 소스와 자격 증명(API Key 등)이 LLM 제공자의 클라우드로 넘어가지 않고 로컬 또는 사용자 인프라 내에 격리됩니다.
   - 도구 실행 전 사용자의 명시적 승인(Human-in-the-loop)을 강제할 수 있습니다.
3. **유연한 전송 방식 (Transports)**
   - 로컬 프로세스 간 통신: `stdio` (표준 입출력)
   - 원격/클라우드 통신: `HTTP with SSE (Server-Sent Events)`
4. **오픈소스 생태계**
   - 오픈소스 레포지토리와 패키지 생태계가 빠르게 성장 중이며, Docker, SQLite, GitHub, Google Drive, Slack 등 이미 수백 개의 공식/커뮤니티 서버가 존재합니다.

---

## 4. 핵심 용어 정리

| 용어 | 설명 |
| :--- | :--- |
| **MCP Host** | 사용자가 상호작용하는 프론트엔드/런타임 애플리케이션 (예: Claude Desktop, Antigravity, Cursor) |
| **MCP Client** | Host 내부에서 MCP 서버와 1:1로 프로토콜 연결을 맺고 통신하는 주체 |
| **MCP Server** | 특정 데이터나 기능을 노출하는 경량 프로그램 (예: Git 서버, DB 서버) |
| **Transport** | Client와 Server 간에 메시지를 주고받는 통신 채널 (`stdio` 또는 `SSE`) |
