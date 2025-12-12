사용자의 자연어 질의가 LLM을 거쳐 MCP(Model Context Protocol) 툴 실행으로 이어지는 흐름과 관련 파일들을 설명해 드리겠습니다.

핵심 파일 및 역할
pkg/ui/html/htmlui.go (Input Entry Point)
역할: 웹 UI로부터 사용자 입력을 받아 Agent로 전달합니다.
로직: handlePOSTSendMessage 함수가 /send-message 요청을 받아 u.agent.Input 채널로 메시지를 보냅니다.
pkg/agent/conversation.go (Core Agent Logic)
역할: 자연어 처리의 중추입니다. LLM과 대화하고, LLM이 반환한 툴 호출 요청(Function Call)을 분석 및 실행합니다.
로직:
Run(): Input 채널에서 사용자 메시지를 읽어 LLM에 전송(c.llmChat.SendStreaming)합니다.
LLM 응답 분석: 응답에 툴 호출이 포함되어 있으면 analyzeToolCalls로 파싱합니다.
DispatchToolCalls(): 파싱된 툴 호출을 실제로 실행합니다. 이때 MCP 툴도 일반 툴처럼 호출됩니다.
pkg/agent/mcp_client.go (MCP Integration)
역할: Agent 시작 시 MCP 서버들과 연결하고, MCP 툴들을 Agent가 사용할 수 있는 형태로 변환하여 등록합니다.
로직: InitializeMCPClient가 mcp.Manager를 초기화하고, RegisterWithToolSystem을 통해 모든 MCP 툴을 tools.RegisterTool로 등록합니다. 이렇게 등록된 MCP 툴은 LLM 입장에서 일반적인 Go 함수처럼 보입니다.
pkg/mcp/manager.go (MCP Connection Manager)
역할: 실제 MCP 서버 프로세스 관리, 연결, 툴 목록 조회 등을 수행합니다.
로직: ConnectAll로 config에 정의된 서버들에 연결하고, ListAvailableTools로 사용 가능한 툴 목록을 가져옵니다.
전체 호직 흐름 (Logical Flow)
사용자 입력: 사용자가 웹 UI에서 질문을 입력 -> htmlui.go가 받아서 Agent.Input 채널로 전송.
LLM 질의: conversation.go의 Run 루프가 입력을 감지. 시스템 프롬프트(사용 가능한 툴 목록 포함)와 함께 LLM에 질의.
툴 호출 결정: LLM은 질문 해결을 위해 MCP 툴 사용이 필요하다고 판단하면, 응답으로 "특정 MCP 툴을 실행하라"는 Function Call을 반환 (JSON 형식).
분석 및 실행:
conversation.go가 이 응답을 받아 analyzeToolCalls로 분석.
DispatchToolCalls가 해당 툴(MCP 툴)을 실행 (InvokeTool).
MCP 통신:
실행 요청은 등록된 MCPTool 래퍼를 통해 pkg/mcp 패키지로 전달됨.
Manager와 Client가 실제 MCP 서버(예: 별도 프로세스)에 JSON-RPC 메시지를 보내 툴 실행을 요청.
결과 반환: MCP 서버가 실행 결과를 반환하면, 이 결과는 다시 conversation.go를 통해 LLM에게 "Observation"(관찰 결과)으로 전달됨.
최종 응답: LLM은 툴 실행 결과를 바탕으로 최종 답변을 생성하여 사용자에게 보여줌.