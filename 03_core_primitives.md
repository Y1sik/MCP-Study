# 03. MCP의 3대 핵심 프리미티브 (Core Primitives)

MCP 서버는 크게 **3가지 핵심 기능(Primitives)**을 클라이언트에게 노출할 수 있습니다. 각 요소는 명확히 다른 목적과 호출 주체를 가집니다.

| 프리미티브 | 비유 | 주도권 (누가 결정하는가?) | 부작용 (Side-effects) | 목적 |
| :--- | :--- | :--- | :--- | :--- |
| **Resources** | 파일 읽기, GET 요청 | **사용자 / 클라이언트** | 없음 (Idempotent/Read-only) | 데이터 및 컨텍스트 제공 |
| **Prompts** | 슬래시 명령어, 템플릿 | **사용자** (UI에서 선택) | 없음 | 구조화된 프롬프트 워크플로우 |
| **Tools** | 함수 호출, POST 요청 | **LLM (AI 모델)** | 발생 가능 (쓰기, 실행 등) | 외부 액션 수행 및 연산 |

---

## 1. Resources (리소스)

### 개념
- 리소스는 클라이언트나 모델에게 **읽기 전용 컨텍스트(데이터)**를 제공하는 방식입니다.
- 웹의 REST GET 요청이나 파일 시스템의 파일 열기와 유사합니다.
- 고유한 **URI (Uniform Resource Identifier)**를 가집니다. (예: `file:///logs/app.log`, `postgres://users/schema`)

### 주요 특징
1. **정적 리소스 (Static Resources)**: 고정된 URI를 가지는 리소스 (예: `system://info`, `project://readme`)
2. **리소스 템플릿 (Resource Templates)**: 동적 매개변수를 받는 URI 패턴 (예: `db://table/{table_name}`, `memo://{id}`)
3. **구독 (Subscriptions)**: 리소스 내용이 변경되면 서버가 클라이언트에 `notifications/resources/updated` 알림을 전송하여 실시간 동기화 가능.

```json
// 리소스 정의 예시
{
  "uri": "config://app/settings",
  "name": "Application Settings",
  "mimeType": "application/json",
  "description": "현재 구동 중인 앱의 환경설정 정보"
}
```

---

## 2. Prompts (프롬프트)

### 개념
- 프롬프트는 서버가 정의해 둔 **재사용 가능한 프롬프트 템플릿**입니다.
- 사용자가 UI 상에서 특정 작업(예: 코드 리뷰, 에러 로그 분석, 번역 등)을 시작할 때 선택하는 슬래시 커맨드(Slash Command) 형태로 자주 사용됩니다.

### 주요 특징
- 매개변수(Arguments)를 선언할 수 있습니다 (예: `language`, `detail_level`).
- 실행 시 하나 이상의 완성된 메시지(역할 `user` 또는 `assistant`, 텍스트 및 임베디드 리소스 포함)를 클라이언트에 반환합니다.

```json
// 프롬프트 정의 예시
{
  "name": "code_review",
  "description": "지정한 파일에 대한 상세 코드 리뷰를 수행합니다.",
  "arguments": [
    {
      "name": "filePath",
      "description": "리뷰할 소스코드 경로",
      "required": true
    }
  ]
}
```

---

## 3. Tools (도구)

### 개념
- 도구는 **LLM(AI 모델)이 자율적으로 선택하여 호출**할 수 있는 실행 가능한 함수(Function)입니다.
- OpenAI나 Anthropic의 "Function Calling" / "Tool Use"와 직접 대응됩니다.
- 계산, 외부 API 호출, 파일 쓰기, 데이터베이스 쿼리 등 **부작용(Side-effect)**을 수반할 수 있습니다.

### 주요 특징
- **JSON Schema 규격**: 도구의 이름, 설명, 입력 인자(Parameters) 스키마를 JSON Schema로 엄격하게 정의합니다.
- **안전 제어 (Human-in-the-Loop)**: 클라이언트는 도구가 부작용을 일으킬 수 있으므로 사용자에게 "이 도구를 실행할까요?"라는 승인 팝업을 띄울 수 있습니다.

```json
// 도구 정의 스키마 예시
{
  "name": "send_slack_message",
  "description": "특정 슬랙 채널에 메시지를 전송합니다.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "channel": { "type": "string", "description": "채널명 (예: #general)" },
      "message": { "type": "string", "description": "보낼 메시지 본문" }
    },
    "required": ["channel", "message"]
  }
}
```

---

## 4. 고급 기능 (Advanced Features)

### 1) Sampling (역방향 LLM 생성)
- 전통적인 구조에서는 클라이언트가 LLM을 호출하고 결과를 서버에 전달합니다.
- MCP의 **Sampling** 기능은 반대로 **서버가 클라이언트에게 "이 프롬프트로 LLM 완료(Generation)를 한 번 돌려줘"라고 요청**할 수 있게 합니다.
- 이를 통해 서버는 자체 API 키 없이도 클라이언트의 모델 능력을 빌려 자율 에이전트 루프(Agentic Loop)를 돌릴 수 있습니다.

### 2) Roots (루트 경로 통지)
- 클라이언트가 서버에게 "현재 사용자가 작업 중인 최상위 프로젝트 폴더 경로는 `/home/ysik/project`야"라고 알려주는 기능입니다.
- 서버가 파일 탐색 범위를 제한하거나 상대 경로를 해석할 때 유용합니다.

### 3) Logging (서버 로그 수집)
- 서버가 `notifications/message`를 통해 `debug`, `info`, `warning`, `error` 로그를 클라이언트에 실시간으로 전달합니다.
