```mermaid
flowchart LR
    %% 1. 사용자 및 클라이언트 계층
    subgraph Clients["사용자 그룹 <User Groups>"]
        Admin[관리자 <Admin 사용자>]
        Dev[개발자/운영자 <Developer 사용자>]
        Viewer[조회 사용자 <Viewer 사용자>]
    end

    %% 2. 웹 대시보드 프론트엔드 계층 (추가된 영역)
    %% 사용자는 브라우저의 웹 UI를 통해 시스템과 상호작용합니다.
    Admin --> WebUI
    Dev --> WebUI
    Viewer --> WebUI

    subgraph FrontendApp["웹 대시보드 프론트엔드 <Web Dashboard Frontend>"]
        WebUI[웹 UI 메인 컨테이너 <React/Vue SPA>]
        
        subgraph UIComponents["UI 컴포넌트 <UI Components>"]
            AIChatUI[AI 자연어 명령어 인터페이스]
            DashViewUI[클러스터 리소스 대시보드 뷰]
            ConfigUI[설정 및 관리자 UI <모델/권한>]
        end
        
        WebUI --> AIChatUI
        WebUI --> DashViewUI
        WebUI --> ConfigUI
    end

    %% 3. 진입점 및 보안 계층
    %% 프론트엔드가 백엔드 API로 요청을 보냅니다.
    WebUI --"API 요청 <HTTPS/WSS>"--> Ingress

    subgraph EntryLayer["진입 및 보안 계층 <Entry & Security Layer>"]
        Ingress[Ingress / Load Balancer]
        APIGW[API Gateway & Auth Guard]
        IDP[Identity Provider <OIDC/LDAP>]
        
        Ingress --> APIGW
        APIGW --"인증/인가 확인 <AuthN/AuthZ>"--> IDP
    end

    %% 4. 웹 대시보드 백엔드 애플리케이션 계층 (핵심 로직)
    APIGW --> BackendSVC

    subgraph BackendApp["웹 대시보드 백엔드 <Web Dashboard Backend>"]
        BackendSVC[Backend API Server <Pod>]
        
        subgraph CoreModules["핵심 기능 모듈 <Core Modules>"]
            RBACMod[사용자/권한 관리 모듈 <RBAC Manager>]
            ModelConfigMod[모델 연동 설정 모듈 <Model Config>]
            MCPOrchestrator[MCP 연동 관리자 <MCP Orchestrator>]
            KubectlAILogic[kubectl-ai 코어 로직 <Prompt Engine>]
        end

        BackendSVC --> RBACMod
        BackendSVC --> ModelConfigMod
        BackendSVC --> MCPOrchestrator
        BackendSVC --> KubectlAILogic
        
        %% 모듈 간 상호작용
        MCPOrchestrator --"프롬프트 참조"--> KubectlAILogic
        MCPOrchestrator --"모델 설정 참조"--> ModelConfigMod
    end

    %% 5. 데이터 저장소 계층
    RBACMod --> MetaDB
    ModelConfigMod --> MetaDB
    MCPOrchestrator --"감사 로그 저장"--> MetaDB

    subgraph Storage["저장소 계층 <Storage Layer>"]
        MetaDB[(Metadata DB\n- 사용자 정보/권한\n- 모델 설정\n- 감사 로그)]
    end

    %% 6. AI 및 MCP 통합 계층 (지능형 처리)
    MCPOrchestrator --"자연어 처리/도구 사용 요청"--> LLMProvider
    LLMProvider --"도구 호출 지시 <Tool Call>"--> K8sMCPServer
    K8sMCPServer --"도구 실행 결과 반환"--> LLMProvider
    LLMProvider --"최종 응답 생성"--> MCPOrchestrator

    subgraph AILayer["AI 및 모델 계층 <AI & Model Layer>"]
        LLMProvider[LLM 공급자 <OpenAI, Anthropic 등>]
    end

    subgraph MCPLayer["MCP 서버 레이어 <MCP Server Layer>"]
        K8sMCPServer[Kubernetes MCP Server <Pod>\n<K8s 도구 제공자>]
    end

    %% 7. 대상 인프라 계층
    K8sMCPServer --> TargetK8s1
    K8sMCPServer --> TargetK8s2

    subgraph TargetInfra["대상 인프라 <Target Infrastructure>"]

```
    
        TargetK8s1[대상 K8s 클러스터 1 <Prod>]
        TargetK8s2[대상 K8s 클러스터 2 <Dev>]
    end
