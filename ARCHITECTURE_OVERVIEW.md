# Dify 项目架构总览 / Dify Project Architecture Overview

## 快速导航 / Quick Navigation

- 📖 **完整中文文档**: [docs/zh-CN/ARCHITECTURE.md](./docs/zh-CN/ARCHITECTURE.md)
- 📖 **Full English Documentation**: [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md)

## 核心架构 / Core Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户界面 / User Interface                │
│                    Next.js + React + TypeScript                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓ HTTPS
┌─────────────────────────────────────────────────────────────────┐
│                      API网关 / API Gateway                       │
│                         Nginx (Reverse Proxy)                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓ HTTP
┌─────────────────────────────────────────────────────────────────┐
│                       应用层 / Application Layer                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Flask API   │  │Celery Worker │  │ Celery Beat  │          │
│  │   Service    │  │   (Async)    │  │  (Scheduler) │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
           ↓              ↓              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    数据存储层 / Data Storage Layer               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ PostgreSQL/  │  │    Redis     │  │   Vector DB  │          │
│  │    MySQL     │  │ Cache/Queue  │  │  (Weaviate)  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────────────────────────────────────────┐          │
│  │         对象存储 / Object Storage (S3/Local)       │          │
│  └──────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

## 主要模块 / Main Modules

### 1. 前端 Web / Frontend Web
- **框架**: Next.js 16 + React 19
- **语言**: TypeScript
- **主要功能**:
  - 🎨 可视化工作流编辑器
  - 💬 应用管理和配置
  - 📚 知识库管理
  - 📊 数据分析面板

### 2. 后端 API / Backend API
- **框架**: Flask 3.1 + Python 3.11+
- **架构**: DDD (领域驱动设计)
- **主要功能**:
  - 🔌 RESTful API服务
  - 🤖 LLM模型管理
  - 🔄 工作流执行引擎
  - 🧠 RAG检索管道
  - 🛠️ Agent工具系统

### 3. 异步任务 / Async Tasks
- **框架**: Celery 5.5
- **主要功能**:
  - 📄 文档处理和向量化
  - ⚙️ 工作流异步执行
  - 📧 邮件发送
  - 🗂️ 数据清理

## 核心技术栈 / Core Technology Stack

### 后端 / Backend
```
Python 3.11+
├── Flask 3.1.x          # Web框架 / Web Framework
├── SQLAlchemy 2.0.x     # ORM
├── Celery 5.5.x         # 任务队列 / Task Queue
├── Pydantic 2.11.x      # 数据验证 / Data Validation
├── transformers 4.56.x   # AI模型 / AI Models
└── litellm 1.77.x       # 统一LLM接口 / Unified LLM Interface
```

### 前端 / Frontend
```
Node.js 24+
├── Next.js 16.1.x       # React框架 / React Framework
├── React 19.2.x         # UI库 / UI Library
├── TypeScript 5.9.x     # 类型系统 / Type System
├── Zustand 5.0.x        # 状态管理 / State Management
├── TanStack Query 5.90.x # 服务端状态 / Server State
├── ReactFlow 11.11.x    # 工作流图 / Workflow Graph
└── Tailwind CSS 3.4.x   # 样式框架 / CSS Framework
```

### 基础设施 / Infrastructure
```
Docker & Docker Compose
├── Nginx                # 反向代理 / Reverse Proxy
├── PostgreSQL 15+       # 关系数据库 / RDBMS
├── Redis 6+             # 缓存/队列 / Cache/Queue
├── Weaviate             # 向量数据库 / Vector DB
└── S3-compatible        # 对象存储 / Object Storage
```

## 支持的第三方组件 / Supported Third-party Components

### LLM 提供商 / LLM Providers
- OpenAI (GPT-3.5, GPT-4, GPT-4 Turbo)
- Anthropic Claude
- Google Gemini
- Azure OpenAI
- 开源模型 / Open-source models (Llama, Mistral, etc.)

### 向量数据库 / Vector Databases
- Weaviate ⭐ (默认 / Default)
- Milvus
- Qdrant
- PostgreSQL pgvector
- OpenSearch
- ChromaDB

### 对象存储 / Object Storage
- 本地存储 / Local Storage
- AWS S3
- Azure Blob Storage
- Google Cloud Storage
- MinIO (S3兼容 / S3-compatible)
- 阿里云OSS / Alibaba Cloud OSS
- 腾讯云COS / Tencent Cloud COS

### 数据库 / Databases
- PostgreSQL ⭐ (推荐 / Recommended)
- MySQL
- OceanBase

## 部署选项 / Deployment Options

### 快速开始 / Quick Start
```bash
cd docker
cp .env.example .env
docker compose up -d
```

### 开发模式 / Development Mode
```bash
# 后端 / Backend
cd api
uv sync --group dev
uv run flask run

# 前端 / Frontend
cd web
pnpm install
pnpm dev
```

## 主要特性 / Key Features

✅ **工作流编排** / Workflow Orchestration
- 可视化拖拽编辑器
- 支持LLM、知识检索、条件分支等节点

✅ **RAG知识库** / RAG Knowledge Base
- 支持多种文档格式 (PDF, DOCX, TXT, etc.)
- 向量检索 + 关键词检索
- 智能分块和重排序

✅ **Agent系统** / Agent System
- Function Calling和ReAct两种模式
- 50+内置工具
- 支持自定义工具集成

✅ **模型管理** / Model Management
- 统一接口支持100+模型
- 负载均衡和故障转移
- Token使用追踪

✅ **LLMOps**
- 提示词版本管理
- 对话日志分析
- 性能监控
- 成本追踪

## 系统要求 / System Requirements

### 最小配置 / Minimum
- CPU: 2 Core
- RAM: 4 GB
- 磁盘 / Disk: 10 GB

### 推荐配置 / Recommended
- CPU: 4+ Core
- RAM: 8+ GB
- 磁盘 / Disk: 50+ GB SSD

## 监控和观测 / Monitoring & Observability

- 📊 **指标监控** / Metrics: Prometheus + Grafana
- 🔍 **分布式追踪** / Tracing: OpenTelemetry
- 🐛 **错误追踪** / Error Tracking: Sentry
- 📝 **日志聚合** / Log Aggregation: 结构化日志 / Structured Logging

## 安全特性 / Security Features

- 🔐 JWT认证 / JWT Authentication
- 👥 基于角色的访问控制 / RBAC
- 🔑 API密钥管理 / API Key Management
- 🛡️ SSRF/XSS/SQL注入防护 / Protection
- 🚦 速率限制 / Rate Limiting
- 📋 审计日志 / Audit Logging

## 扩展性 / Scalability

- 🔄 **水平扩展**: API服务和Worker节点可独立扩展
- 🔌 **插件系统**: 支持自定义模型、工具和加载器
- 📦 **模块化设计**: 清晰的领域边界和依赖注入
- ⚖️ **负载均衡**: 支持多实例部署

## 文档资源 / Documentation

- 📚 [官方文档 / Official Docs](https://docs.dify.ai)
- 🏗️ [详细架构文档 / Detailed Architecture](./docs/ARCHITECTURE.md)
- 🇨🇳 [中文架构文档 / Chinese Architecture](./docs/zh-CN/ARCHITECTURE.md)
- 🤝 [贡献指南 / Contributing Guide](./CONTRIBUTING.md)
- 🔧 [开发指南 / Development Guide](./AGENTS.md)

## 社区和支持 / Community & Support

- 💬 [Discord社区](https://discord.gg/FngNHpbcY7)
- 🐛 [GitHub Issues](https://github.com/langgenius/dify/issues)
- 💡 [GitHub Discussions](https://github.com/langgenius/dify/discussions)
- 🐦 [Twitter/X](https://twitter.com/dify_ai)

## 许可证 / License

本项目采用 Dify Open Source License (基于 Apache 2.0 附加条件)

This project is licensed under the Dify Open Source License (based on Apache 2.0 with additional conditions)

---

⭐ 如果这个项目对你有帮助，请给我们一个星标！ / If this project helps you, please give us a star!
