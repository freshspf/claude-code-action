# Claude Code Action 架构图集

## 1. 整体系统架构图

```mermaid
graph TB
    subgraph "GitHub Events"
        E1[Issue Events]
        E2[PR Events] 
        E3[Comment Events]
        E4[Review Events]
        E5[Workflow Dispatch]
        E6[Schedule Events]
    end

    subgraph "Claude Code Action"
        subgraph "Phase 1: Preparation"
            P1[Context Parsing]
            P2[Mode Selection]
            P3[Trigger Validation]
            P4[Permission Check]
            P5[Data Fetching]
            P6[Prompt Generation]
            P7[MCP Configuration]
        end
        
        subgraph "Phase 2: Execution"
            E1[Base Action]
            E2[Claude CLI]
            E3[MCP Servers]
            E4[GitHub Operations]
        end
    end

    subgraph "External Services"
        EX1[Anthropic API]
        EX2[AWS Bedrock]
        EX3[Google Vertex AI]
        EX4[GitHub API]
    end

    E1 --> P1
    E2 --> P1
    E3 --> P1
    E4 --> P1
    E5 --> P1
    E6 --> P1

    P1 --> P2
    P2 --> P3
    P3 --> P4
    P4 --> P5
    P5 --> P6
    P6 --> P7
    P7 --> E1

    E1 --> E2
    E2 --> E3
    E3 --> E4

    E2 --> EX1
    E2 --> EX2
    E2 --> EX3
    E3 --> EX4
```

## 2. 模式系统架构图

```mermaid
graph TD
    subgraph "Mode Registry"
        MR[Mode Registry]
        MR --> M1[Tag Mode]
        MR --> M2[Agent Mode]
        MR --> M3[Review Mode]
        MR --> M4[Custom Modes...]
    end

    subgraph "Mode Interface"
        MI[Mode Interface]
        MI --> ST[shouldTrigger]
        MI --> PC[prepareContext]
        MI --> GT[getAllowedTools]
        MI --> GD[getDisallowedTools]
        MI --> CT[shouldCreateTrackingComment]
        MI --> GP[generatePrompt]
        MI --> PR[prepare]
        MI --> SP[getSystemPrompt]
    end

    subgraph "Event Processing"
        EP1[Entity Events<br/>issues, PR, comments]
        EP2[Automation Events<br/>workflow_dispatch, schedule]
    end

    EP1 --> M1
    EP2 --> M2
    EP1 --> M3

    M1 --> MI
    M2 --> MI
    M3 --> MI
```

## 3. Prompt 系统架构图

```mermaid
graph TD
    subgraph "Data Sources"
        DS1[GitHub Context]
        DS2[PR/Issue Data]
        DS3[Comments]
        DS4[Review Data]
        DS5[Changed Files]
        DS6[User Inputs]
    end

    subgraph "Data Processing"
        DP1[Data Fetching]
        DP2[Data Formatting]
        DP3[Image Processing]
        DP4[Content Sanitization]
    end

    subgraph "Prompt Generation"
        PG1[Template Selection]
        PG2[Variable Substitution]
        PG3[Mode-specific Logic]
        PG4[Custom Instructions]
    end

    subgraph "Prompt Structure"
        PS1[System Identity]
        PS2[Context Information]
        PS3[Metadata Tags]
        PS4[Workflow Instructions]
        PS5[Tool Information]
        PS6[Capabilities & Limitations]
    end

    DS1 --> DP1
    DS2 --> DP1
    DS3 --> DP1
    DS4 --> DP1
    DS5 --> DP1
    DS6 --> DP1

    DP1 --> DP2
    DP2 --> DP3
    DP3 --> DP4

    DP4 --> PG1
    PG1 --> PG2
    PG2 --> PG3
    PG3 --> PG4

    PG4 --> PS1
    PG4 --> PS2
    PG4 --> PS3
    PG4 --> PS4
    PG4 --> PS5
    PG4 --> PS6
```

## 4. MCP 工具系统架构图

```mermaid
graph TD
    subgraph "Built-in MCP Servers"
        MCP1[GitHub Comment Server<br/>update_claude_comment]
        MCP2[GitHub File Ops Server<br/>commit_files, delete_files]
        MCP3[GitHub Actions Server<br/>get_ci_status, workflow_details]
        MCP4[GitHub Inline Comment Server<br/>create_inline_comment]
        MCP5[Official GitHub MCP Server<br/>Full GitHub API access]
    end

    subgraph "MCP Configuration"
        MC1[Base Configuration]
        MC2[User Configuration]
        MC3[Merged Configuration]
    end

    subgraph "Claude Code CLI"
        CL1[Tool Execution Engine]
        CL2[MCP Client]
    end

    subgraph "Conditions"
        C1[Always Included]
        C2[Commit Signing Enabled]
        C3[Actions Read Permission]
        C4[Experimental Review Mode]
        C5[User Explicitly Enabled]
    end

    C1 --> MCP1
    C2 --> MCP2
    C3 --> MCP3
    C4 --> MCP4
    C5 --> MCP5

    MC1 --> MC3
    MC2 --> MC3
    MC3 --> CL2
    CL2 --> CL1

    MCP1 --> CL2
    MCP2 --> CL2
    MCP3 --> CL2
    MCP4 --> CL2
    MCP5 --> CL2
```

## 5. GitHub 集成层架构图

```mermaid
graph TD
    subgraph "GitHub API Layer"
        API1[REST API Client]
        API2[GraphQL Client]
        API3[Authentication Manager]
    end

    subgraph "Data Layer"
        DL1[Data Fetcher]
        DL2[Data Formatter]
        DL3[Image Downloader]
        DL4[Content Sanitizer]
    end

    subgraph "Operations Layer"
        OL1[Branch Operations]
        OL2[Comment Operations]
        OL3[Git Configuration]
        OL4[File Operations]
    end

    subgraph "Validation Layer"
        VL1[Permission Validation]
        VL2[Actor Validation]
        VL3[Trigger Validation]
    end

    subgraph "GitHub Events"
        GE1[Webhook Events]
        GE2[Action Triggers]
    end

    GE1 --> VL3
    GE2 --> VL3
    VL3 --> VL1
    VL1 --> VL2

    VL2 --> DL1
    DL1 --> API2
    API2 --> API3

    DL1 --> DL2
    DL2 --> DL3
    DL3 --> DL4

    DL4 --> OL1
    OL1 --> OL2
    OL2 --> OL3
    OL3 --> OL4

    OL4 --> API1
    API1 --> API3
```

## 6. 工作流程详细流程图

```mermaid
sequenceDiagram
    participant GH as GitHub Event
    participant AC as Action Container
    participant PR as Prepare Phase
    participant BA as Base Action
    participant CL as Claude CLI
    participant MC as MCP Servers
    participant API as GitHub API

    GH->>AC: Trigger Event
    AC->>PR: Start Preparation
    
    PR->>PR: Parse Context
    PR->>PR: Select Mode
    PR->>API: Check Permissions
    PR->>API: Validate Trigger
    PR->>API: Fetch Data
    PR->>PR: Generate Prompt
    PR->>MC: Configure MCP
    PR->>API: Create Comment
    PR->>API: Setup Branch
    
    PR->>BA: Start Execution
    BA->>BA: Validate Environment
    BA->>BA: Setup Claude Code
    BA->>MC: Start MCP Servers
    BA->>CL: Execute Claude CLI
    
    loop Claude Execution
        CL->>MC: Use Tools
        MC->>API: GitHub Operations
        CL->>CL: Process Response
    end
    
    CL->>BA: Execution Complete
    BA->>API: Update Comment
    BA->>API: Commit Changes
    BA->>AC: Return Results
```

## 7. 错误处理和降级机制图

```mermaid
graph TD
    subgraph "Error Sources"
        ES1[Authentication Errors]
        ES2[Permission Errors]
        ES3[API Rate Limits]
        ES4[Network Failures]
        ES5[Claude API Errors]
        ES6[Configuration Errors]
    end

    subgraph "Error Handling"
        EH1[Error Detection]
        EH2[Error Classification]
        EH3[Retry Logic]
        EH4[Fallback Strategies]
        EH5[User Notification]
    end

    subgraph "Fallback Strategies"
        FB1[Auth: GitHub App → PAT → Fail]
        FB2[Format: Structured → Raw JSON → Error Text]
        FB3[Function: Full → Basic → Read-only]
        FB4[Model: Primary → Fallback → Error]
    end

    ES1 --> EH1
    ES2 --> EH1
    ES3 --> EH1
    ES4 --> EH1
    ES5 --> EH1
    ES6 --> EH1

    EH1 --> EH2
    EH2 --> EH3
    EH3 --> EH4
    EH4 --> EH5

    EH4 --> FB1
    EH4 --> FB2
    EH4 --> FB3
    EH4 --> FB4
```

## 8. 安全和权限架构图

```mermaid
graph TD
    subgraph "Authentication Layer"
        AUTH1[OIDC Token Exchange]
        AUTH2[GitHub App Authentication]
        AUTH3[Personal Access Token]
        AUTH4[Cloud Provider Auth]
    end

    subgraph "Permission Validation"
        PERM1[Repository Write Access]
        PERM2[Actor Human Check]
        PERM3[Trigger Condition Validation]
        PERM4[Additional Permissions]
    end

    subgraph "Security Measures"
        SEC1[Token Scoping]
        SEC2[Network Restrictions]
        SEC3[Tool Restrictions]
        SEC4[Commit Signing]
        SEC5[Input Sanitization]
    end

    subgraph "Access Control"
        AC1[Mode-based Tool Access]
        AC2[Repository-level Permissions]
        AC3[User-defined Restrictions]
    end

    AUTH1 --> PERM1
    AUTH2 --> PERM1
    AUTH3 --> PERM1
    AUTH4 --> PERM1

    PERM1 --> PERM2
    PERM2 --> PERM3
    PERM3 --> PERM4

    PERM4 --> SEC1
    SEC1 --> SEC2
    SEC2 --> SEC3
    SEC3 --> SEC4
    SEC4 --> SEC5

    SEC5 --> AC1
    AC1 --> AC2
    AC2 --> AC3
```

## 9. 扩展性架构图

```mermaid
graph TD
    subgraph "Extension Points"
        EP1[Custom Modes]
        EP2[Custom MCP Servers]
        EP3[Custom Prompt Templates]
        EP4[Custom Triggers]
        EP5[Custom Tools]
    end

    subgraph "Core Interfaces"
        CI1[Mode Interface]
        CI2[MCP Protocol]
        CI3[Prompt Generator Interface]
        CI4[Trigger Validator Interface]
        CI5[Tool Interface]
    end

    subgraph "Registration Systems"
        RS1[Mode Registry]
        RS2[MCP Configuration]
        RS3[Prompt System]
        RS4[Validation System]
        RS5[Tool System]
    end

    subgraph "Plugin Architecture"
        PA1[Plugin Discovery]
        PA2[Plugin Loading]
        PA3[Plugin Validation]
        PA4[Plugin Execution]
    end

    EP1 --> CI1
    EP2 --> CI2
    EP3 --> CI3
    EP4 --> CI4
    EP5 --> CI5

    CI1 --> RS1
    CI2 --> RS2
    CI3 --> RS3
    CI4 --> RS4
    CI5 --> RS5

    RS1 --> PA1
    RS2 --> PA1
    RS3 --> PA1
    RS4 --> PA1
    RS5 --> PA1

    PA1 --> PA2
    PA2 --> PA3
    PA3 --> PA4
```

## 10. 性能优化架构图

```mermaid
graph TD
    subgraph "Data Optimization"
        DO1[GraphQL Batch Queries]
        DO2[Pagination Strategy]
        DO3[Data Caching]
        DO4[Lazy Loading]
    end

    subgraph "Execution Optimization"
        EO1[Parallel Processing]
        EO2[Streaming Outputs]
        EO3[Resource Pooling]
        EO4[Process Management]
    end

    subgraph "Network Optimization"
        NO1[Request Batching]
        NO2[Connection Reuse]
        NO3[Compression]
        NO4[CDN Usage]
    end

    subgraph "Memory Optimization"
        MO1[Object Pooling]
        MO2[Garbage Collection]
        MO3[Memory Limits]
        MO4[Resource Cleanup]
    end

    subgraph "Monitoring"
        MON1[Performance Metrics]
        MON2[Error Tracking]
        MON3[Resource Usage]
        MON4[Latency Monitoring]
    end

    DO1 --> EO1
    DO2 --> EO1
    DO3 --> EO1
    DO4 --> EO1

    EO1 --> NO1
    EO2 --> NO1
    EO3 --> NO1
    EO4 --> NO1

    NO1 --> MO1
    NO2 --> MO1
    NO3 --> MO1
    NO4 --> MO1

    MO1 --> MON1
    MO2 --> MON1
    MO3 --> MON1
    MO4 --> MON1
```