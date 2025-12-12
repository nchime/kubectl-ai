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




현재 코드 기준으로 지연어로 입력된 질문이 응답을 받는 실제 과정을 단계적으로 설명해줘
--------------

사용자 입력 (Frontend -> Backend):
사용자가 채팅창에 질문을 입력하면 브라우저는 /send-message API를 호출합니다.
백엔드(htmlui.go)의 handlePOSTSendMessage 함수가 이 요청을 받아 에이전트의 입력 채널(u.agent.Input)로 메시지를 전달합니다.
질문 접수 및 상태 변경 (Agent Loop):
별도의 고루틴(Goroutine)으로 실행 중인 에이전트의 메인 루프(chatLoop)가 입력 채널에서 질문을 감지합니다.
에이전트 상태가 Running으로 변경되고, 질문이 현재 대화 컨텍스트에 추가됩니다.
LLM 추론 (Thinking):
현재까지의 대화 내용(시스템 프롬프트 + 이전 대화 + 새로운 질문)을 가지고 LLM에 요청을 보냅니다(c.llmChat.SendStreaming).
이때 사용할 수 있는 도구(Kubectl 명령어, MCP 도구 등)의 정의도 함께 전달됩니다.
LLM은 질문을 분석하여 바로 답변할지, 아니면 도구를 사용해야 할지 결정합니다.
도구 실행 (Executing - 필요한 경우):
만약 LLM이 도구 사용이 필요하다고 판단하면(예: kubectl get pods), 함수 호출 응답을 보냅니다.
에이전트는 DispatchToolCalls 함수를 통해 해당 도구를 실제로 실행합니다.
Kubernetes 관련 도구는 pkg/tools를 통해 실행됩니다.
MCP 도구(예: sequential-thinking)는 MCPClient를 통해 외부 서버로 요청을 보냅니다.
도구 실행 결과(Output)나 오류가 다시 대화 컨텍스트에 추가됩니다.
최종 응답 생성 및 전달:
도구 실행 결과가 포함된 새로운 컨텍스트로 LLM이 다시 추론을 수행합니다.
최종적인 자연어 답변이 생성되면 스트리밍 형태로 사용자에게 전달됩니다.
백엔드는 SSE(Server-Sent Events)를 통해 브라우저로 실시간 데이터를 전송하고, 채팅창에 답변이 표시됩니다.


코드 상으로는 MCP 기능이 완전히 구현되어 있습니다. 다만, 앱을 실행할 때 기본적으로 '비활성화(Disabled)' 상태로 시작하도록 설정되어 있어서 사용되지 않고 있었습니다.

구체적으로:

초기화 로직 (Init 함수): 에이전트가 시작될 때 MCPClientEnabled 플래그를 확인합니다. 이 값이 true여야만 MCP 매니저를 초기화하고 도구를 등록합니다 (pkg/agent/conversation.go).
도구 등록: 활성화되면 InitializeMCPClient 함수가 실행되어 sequential-thinking 같은 도구들을 LLM이 사용할 수 있는 도구 목록(a.Tools)에 추가합니다 (pkg/agent/mcp_client.go).
앞서 제가 Makefile을 수정한 것은 이 MCPClientEnabled 플래그를 true로 켜주는 옵션(--mcp-client)을 추가한 것입니다. 이제 앱을 재시작하면 코드가 정상적으로 MCP 기능을 사용하게 됩니다.

------------

make build
kubectl-ai --list-sessions
kubectl-ai --ui-type web
make dev-html  

