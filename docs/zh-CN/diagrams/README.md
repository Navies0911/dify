# Dify 架构图

本目录包含 Dify 项目的可视化架构图。

> **English Version** | **英文版本**: [Architecture Diagrams Documentation](../../diagrams/README.md)

## 可用的架构图

### 1. 系统架构
查看完整的系统架构：
- [English Version](../../ARCHITECTURE.md)
- [中文版本](../ARCHITECTURE.md)

### 2. Mermaid 图表

所有图表使用 Mermaid 语法创建，可以在以下环境中查看：
- GitHub（原生支持）
- VS Code（需要 Mermaid 扩展）
- 在线 Mermaid 编辑器

### 3. 组件交互图

```mermaid
graph TB
    subgraph "用户层"
        U[终端用户]
        A[管理员用户]
    end
    
    subgraph "展示层"
        WEB[Next.js Web 界面]
        WEB_CONSOLE[控制台界面]
        WEB_APP[应用界面]
        WEB --> WEB_CONSOLE
        WEB --> WEB_APP
    end
    
    subgraph "API 层"
        NGINX[Nginx 网关]
        API_CONSOLE[控制台 API]
        API_PUBLIC[公共 API]
        NGINX --> API_CONSOLE
        NGINX --> API_PUBLIC
    end
    
    subgraph "服务层"
        APP_SVC[应用服务]
        DATASET_SVC[数据集服务]
        MODEL_SVC[模型服务]
        WORKFLOW_SVC[工作流服务]
        AGENT_SVC[Agent 服务]
    end
    
    subgraph "核心层"
        MODEL_MGR[模型管理器]
        WORKFLOW_ENG[工作流引擎]
        RAG_PIPELINE[RAG 管道]
        AGENT_RUNTIME[Agent 运行时]
        PROMPT_ENG[提示词引擎]
    end
    
    subgraph "数据访问层"
        REPO[数据仓库]
        CACHE[缓存管理器]
        QUEUE[任务队列]
    end
    
    subgraph "存储层"
        DB[(数据库)]
        REDIS[(Redis)]
        VECTOR[(向量数据库)]
        STORAGE[(对象存储)]
    end
    
    subgraph "外部服务"
        LLM[LLM 提供商]
        TOOLS[外部工具]
    end
    
    U --> WEB_APP
    A --> WEB_CONSOLE
    WEB --> NGINX
    API_CONSOLE --> APP_SVC
    API_CONSOLE --> DATASET_SVC
    API_CONSOLE --> MODEL_SVC
    API_PUBLIC --> APP_SVC
    
    APP_SVC --> WORKFLOW_SVC
    APP_SVC --> AGENT_SVC
    DATASET_SVC --> RAG_PIPELINE
    MODEL_SVC --> MODEL_MGR
    WORKFLOW_SVC --> WORKFLOW_ENG
    AGENT_SVC --> AGENT_RUNTIME
    
    WORKFLOW_ENG --> MODEL_MGR
    AGENT_RUNTIME --> MODEL_MGR
    RAG_PIPELINE --> VECTOR
    MODEL_MGR --> LLM
    AGENT_RUNTIME --> TOOLS
    
    APP_SVC --> REPO
    DATASET_SVC --> REPO
    WORKFLOW_SVC --> QUEUE
    
    REPO --> DB
    CACHE --> REDIS
    QUEUE --> REDIS
    RAG_PIPELINE --> STORAGE
    
    style U fill:#e1f5ff
    style A fill:#e1f5ff
    style WEB fill:#d4edda
    style NGINX fill:#fff3cd
    style DB fill:#f8d7da
    style REDIS fill:#f8d7da
    style VECTOR fill:#f8d7da
    style STORAGE fill:#f8d7da
    style LLM fill:#e7d4f5
    style TOOLS fill:#e7d4f5
```

### 4. 数据流 - RAG 查询

```mermaid
sequenceDiagram
    autonumber
    participant User as 用户
    participant Web as Web 前端
    participant API as Flask API
    participant Cache as Redis 缓存
    participant Vector as 向量数据库
    participant LLM as LLM 提供商
    participant DB as PostgreSQL
    
    User->>Web: 发送查询消息
    Web->>API: POST /chat-messages
    API->>Cache: 检查对话上下文
    Cache-->>API: 返回上下文（如果存在）
    
    API->>DB: 加载应用配置
    DB-->>API: 返回配置
    
    API->>Vector: 搜索相关文档
    Vector-->>API: 返回 top-k 文本块
    
    API->>API: 使用上下文组装提示词
    API->>LLM: 流式完成请求
    
    loop 流式响应
        LLM-->>API: Token 片段
        API-->>Web: SSE 流
        Web-->>User: 显示 Token
    end
    
    LLM-->>API: 完整响应
    API->>DB: 保存消息和响应
    API->>Cache: 更新对话上下文
    API-->>Web: 最终消息元数据
```

### 5. 数据流 - 文档索引

```mermaid
sequenceDiagram
    autonumber
    participant User as 用户
    participant Web as Web 前端
    participant API as Flask API
    participant Storage as 对象存储
    participant Queue as Redis 队列
    participant Worker as Celery Worker
    participant Vector as 向量数据库
    participant LLM as LLM 提供商
    participant DB as PostgreSQL
    
    User->>Web: 上传文档
    Web->>API: POST /datasets/{id}/documents
    API->>API: 验证文件
    API->>Storage: 保存原始文件
    Storage-->>API: 返回文件 URL
    
    API->>DB: 创建文档记录
    DB-->>API: 返回文档 ID
    
    API->>Queue: 将索引任务加入队列
    API-->>Web: 返回文档 ID 和 status=queued
    Web-->>User: 显示上传成功
    
    Queue->>Worker: 分发索引任务
    Worker->>Storage: 下载文件
    Storage-->>Worker: 返回文件内容
    
    Worker->>Worker: 解析文档
    Worker->>Worker: 分割成文本块
    
    loop 对于每个文本块
        Worker->>LLM: 生成向量嵌入
        LLM-->>Worker: 返回向量
        Worker->>Vector: 存储文本块和向量
    end
    
    Worker->>DB: 更新文档 status=completed
    Worker->>Queue: 确认任务完成
    
    Note over User,DB: 用户现在可以查询该文档
```

### 6. 工作流执行

```mermaid
stateDiagram-v2
    [*] --> Initialized: 创建工作流运行
    Initialized --> Running: 开始执行
    
    Running --> NodeExecuting: 执行节点
    NodeExecuting --> NodeSuccess: 节点完成
    NodeExecuting --> NodeFailed: 节点错误
    
    NodeSuccess --> HasNextNode: 检查流程
    NodeFailed --> ErrorHandling: 错误处理
    
    ErrorHandling --> Retry: 可以重试
    ErrorHandling --> Failed: 达到最大重试次数
    Retry --> NodeExecuting
    
    HasNextNode --> NodeExecuting: 还有更多节点
    HasNextNode --> Completed: 没有更多节点
    
    Failed --> [*]
    Completed --> [*]
    
    note right of NodeExecuting
        节点类型：
        - LLM 节点
        - 知识检索
        - 代码执行
        - HTTP 请求
        - 条件分支
        - 循环
    end note
```

### 7. Agent 决策流程

```mermaid
flowchart TD
    Start([用户查询]) --> ParseQuery[解析查询意图]
    ParseQuery --> SelectAgent{选择 Agent 类型}
    
    SelectAgent -->|函数调用| FCAgent[函数调用 Agent]
    SelectAgent -->|ReAct| ReactAgent[ReAct Agent]
    
    FCAgent --> AnalyzeTools[分析可用工具]
    AnalyzeTools --> CallLLM1[使用工具调用 LLM]
    CallLLM1 --> HasToolCall{有工具调用？}
    
    HasToolCall -->|是| ExecuteTool[执行工具]
    ExecuteTool --> ToolResult[获取工具结果]
    ToolResult --> CallLLM1
    
    HasToolCall -->|否| FinalAnswer1[最终答案]
    
    ReactAgent --> Thought[LLM 思考]
    Thought --> Action{选择行动}
    
    Action -->|使用工具| SelectTool[选择工具]
    Action -->|最终答案| FinalAnswer2[最终答案]
    
    SelectTool --> ExecuteAction[执行行动]
    ExecuteAction --> Observation[观察结果]
    Observation --> Thought
    
    FinalAnswer1 --> Response[返回响应]
    FinalAnswer2 --> Response
    Response --> End([完成])
    
    style Start fill:#e1f5ff
    style End fill:#d4edda
    style Response fill:#d4edda
```

### 8. 安全层

```mermaid
graph TB
    subgraph "网络安全"
        SSL[TLS/SSL 加密]
        CORS[CORS 策略]
        RATE[速率限制]
    end
    
    subgraph "身份认证"
        JWT[JWT 令牌]
        API_KEY[API 密钥]
        OAUTH[OAuth2 集成]
    end
    
    subgraph "授权"
        RBAC[基于角色的访问控制]
        TENANT[租户隔离]
        SCOPE[权限范围]
    end
    
    subgraph "数据保护"
        ENCRYPT[静态数据加密]
        MASK[数据脱敏]
        AUDIT[审计日志]
    end
    
    subgraph "应用安全"
        XSS[XSS 防护]
        SSRF[SSRF 防护]
        SQL[SQL 注入防护]
        CSRF[CSRF 防护]
    end
    
    Request[传入请求] --> SSL
    SSL --> CORS
    CORS --> RATE
    RATE --> JWT
    JWT --> RBAC
    RBAC --> TENANT
    TENANT --> XSS
    XSS --> SSRF
    SSRF --> SQL
    SQL --> CSRF
    CSRF --> APP[应用逻辑]
    APP --> ENCRYPT
    ENCRYPT --> AUDIT
    
    style Request fill:#fff3cd
    style APP fill:#d4edda
```

## 如何查看

### GitHub
所有 Mermaid 图表在 GitHub 的 Web 界面中自动渲染。

### VS Code
安装 "Markdown Preview Mermaid Support" 扩展：
```
ext install bierner.markdown-mermaid
```

### 在线编辑器
- [Mermaid Live Editor](https://mermaid.live/)
- [Mermaid Chart](https://www.mermaidchart.com/)

## 导出为图片

要将图表导出为 PNG/SVG：

1. 使用 Mermaid CLI：
```bash
npm install -g @mermaid-js/mermaid-cli
mmdc -i diagram.md -o diagram.png
```

2. 使用在线编辑器：
   - 将代码粘贴到 [Mermaid Live Editor](https://mermaid.live/)
   - 点击 "Export" 按钮
   - 选择 PNG 或 SVG 格式

## 贡献

添加新图表时：
1. 使用一致的样式
2. 添加清晰的标签和描述
3. 保持图表专注于特定方面
4. 用新图表描述更新此 README
