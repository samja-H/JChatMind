# JChatMind

JChatMind 是一个基于 Spring AI 的 Agent + RAG 实验项目，当前仓库已经具备以下闭环能力：

- 智能体配置与管理
- 会话与消息持久化
- Markdown 知识库上传、切片、向量化与检索
- 工具调用（知识库检索、数据库查询、邮件发送）
- 基于 SSE 的聊天消息回推
- React 前端可视化操作界面

项目当前是一个适合本地开发、调试和扩展的原型系统，重点在于把 Agent 运行链路打通，而不是做成生产级 SaaS。

## 核心能力

### 1. 智能体管理

前端支持创建和编辑智能体，当前可配置项包括：

- 名称、描述、系统提示词
- 模型选择：`deepseek-chat`、`glm-4.6`
- 可访问知识库列表
- 可调用工具列表
- 会话窗口长度 `messageLength`
- 预留的采样参数：`temperature`、`topP`

### 2. 知识库与 RAG

- 支持创建知识库
- 支持上传 Markdown 文档
- 上传后自动存储原文件
- 自动解析 Markdown 标题与段落内容
- 调用本地 embedding 服务生成向量
- 将切片结果写入 PostgreSQL + `pgvector`
- 对话过程中通过工具调用执行知识库检索

### 3. Agent 执行链路

用户发送消息后，后端会：

1. 持久化用户消息
2. 发布 `ChatEvent`
3. 异步创建 `JChatMind` 运行时实例
4. 恢复会话记忆、知识库权限和工具权限
5. 执行 think / execute 循环
6. 持久化 Assistant / Tool 消息
7. 通过 SSE 将新增消息推回前端

### 4. 工具体系

当前工具分为两类：

- 固定工具：所有 Agent 默认具备
- 可选工具：创建 Agent 时按需勾选

当前已接入的工具如下：

| 工具名 | 类型 | 说明 |
| --- | --- | --- |
| `KnowledgeTool` | 固定 | 对指定知识库执行语义检索 |
| `terminate` | 固定 | 结束 Agent Loop |
| `databaseQuery` | 可选 | 执行只读 `SELECT` 查询 |
| `sendEmail` | 可选 | 通过 SMTP 异步发送邮件 |

代码中还预留了 `DirectAnswerTool`、`FileSystemTools`，但当前没有注册到运行时工具集合中。

## 技术栈

| 层 | 技术 |
| --- | --- |
| 后端 | Java 17, Spring Boot 3, Spring AI, MyBatis |
| 前端 | React 19, TypeScript, Vite, Ant Design |
| 数据库 | PostgreSQL, `pgvector`, JSONB |
| 向量化 | 本地 embedding 服务，默认走 `http://localhost:11434/api/embeddings` |
| 流式推送 | SSE |
| 文档解析 | flexmark |

## 目录结构

```text
.
├── jchatmind/                # Spring Boot 后端
│   ├── src/main/java/com/kama/jchatmind
│   │   ├── agent/            # Agent runtime、Factory、工具
│   │   ├── controller/       # REST / SSE 接口
│   │   ├── service/          # 业务服务
│   │   ├── mapper/           # MyBatis Mapper
│   │   ├── model/            # request/response/entity/dto/vo
│   │   └── event/            # ChatEvent 及监听器
│   └── src/main/resources
│       ├── application.yaml
│       └── mapper/*.xml
├── ui/                       # React 前端
│   └── src/
│       ├── api/
│       ├── components/
│       ├── hooks/
│       ├── layout/
│       └── contexts/
├── jchatmind_assert/         # SQL 与示例文档
│   ├── jchatmind.sql         # 主业务表结构
│   ├── eshop.sql             # 电商示例业务表
│   ├── eshop_data.sql        # 电商示例数据
│   └── eshop.md              # 可用于知识库导入的示例 Markdown
└── examples/                 # 早期页面示例
```

## 运行前准备

本项目当前默认依赖以下运行环境：

- JDK 17
- Node.js 20+
- PostgreSQL 14+（建议本地安装）
- PostgreSQL 扩展：`pgvector`、`pgcrypto`
- 本地 embedding 服务，默认监听 `http://localhost:11434`
- 至少一个可用的大模型 API Key：
  - DeepSeek
  - 智谱 AI
- 可选：SMTP 邮箱配置（启用邮件工具时需要）

## 快速启动

### 1. 初始化数据库

先创建数据库：

```sql
CREATE DATABASE jchatmind;
```

进入数据库后，先准备扩展：

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS vector;
```

然后导入主表结构：

```bash
psql -d jchatmind -f jchatmind_assert/jchatmind.sql
```

如果你想体验数据库工具，再额外导入电商示例表和数据：

```bash
psql -d jchatmind -f jchatmind_assert/eshop.sql
psql -d jchatmind -f jchatmind_assert/eshop_data.sql
```

### 2. 配置后端

后端的 `application.yaml` 已经改为环境变量占位配置，并会在启动时尝试读取 `jchatmind/.env`。

本地开发可以先复制示例配置：

```bash
cd jchatmind
cp .env.example .env
```

然后按你的环境修改 `jchatmind/.env`。至少需要检查以下变量：

- `JCHATMIND_DB_URL`
- `JCHATMIND_DB_USERNAME`
- `JCHATMIND_DB_PASSWORD`
- `DEEPSEEK_API_KEY`
- `ZHIPUAI_API_KEY`
- `JCHATMIND_MAIL_*`
- `JCHATMIND_DOCUMENT_STORAGE_BASE_PATH`
- `JCHATMIND_EMBEDDING_BASE_URL`
- `JCHATMIND_EMBEDDING_MODEL`

注意：

- `jchatmind/.env.example` 只保留可安全分发的示例值
- `jchatmind/.env` 已加入忽略规则，用来放本地真实数据库密码、API Key、邮箱授权码
- 也可以不创建 `.env`，直接通过系统环境变量注入同名配置

### 3. 启动本地 embedding 服务

后端默认请求：

- 地址：`${JCHATMIND_EMBEDDING_BASE_URL}/api/embeddings`
- 默认地址：`http://localhost:11434/api/embeddings`
- 默认模型：`bge-m3`

如果你使用 Ollama，可参考：

```bash
ollama serve
ollama pull bge-m3
```

如果你使用的不是 Ollama，需要保证接口兼容，或者自行修改 `RagServiceImpl`。

### 4. 启动后端

```bash
cd jchatmind
./mvnw spring-boot:run
```

Windows：

```bash
cd jchatmind
mvnw.cmd spring-boot:run
```

默认地址：

- REST API: `http://localhost:8080/api`
- SSE: `http://localhost:8080/sse`

### 5. 启动前端

```bash
cd ui
cp .env.example .env
npm install
npm run dev
```

默认地址：

- `http://localhost:5173`

前端可通过以下变量切换后端地址：

- `VITE_API_BASE_URL`
- `VITE_SSE_BASE_URL`

## 使用流程

### 知识库问答

1. 启动前后端
2. 进入“知识库”标签页
3. 创建知识库
4. 上传 Markdown 文件（可以直接使用 `jchatmind_assert/eshop.md`）
5. 创建一个 Agent，并把该知识库加入可访问列表
6. 进入聊天页提问，例如：
   - “评论相关表有哪些？”
   - “总结一下这个电商库里和评论分析有关的实体设计”

### 数据库查询问答

1. 导入 `eshop.sql` 与 `eshop_data.sql`
2. 创建一个 Agent，并勾选 `databaseQuery`
3. 在聊天页提问，例如：
   - “列出评分低于 3 分的评论”
   - “统计每个商品的差评数量”

## 主要接口

### Agent

- `GET /api/agents`
- `POST /api/agents`
- `PATCH /api/agents/{agentId}`
- `DELETE /api/agents/{agentId}`

### 会话

- `GET /api/chat-sessions`
- `GET /api/chat-sessions/{chatSessionId}`
- `GET /api/chat-sessions/agent/{agentId}`
- `POST /api/chat-sessions`
- `PATCH /api/chat-sessions/{chatSessionId}`
- `DELETE /api/chat-sessions/{chatSessionId}`

### 消息

- `GET /api/chat-messages/session/{sessionId}`
- `POST /api/chat-messages`
- `PATCH /api/chat-messages/{chatMessageId}`
- `DELETE /api/chat-messages/{chatMessageId}`

### 知识库与文档

- `GET /api/knowledge-bases`
- `POST /api/knowledge-bases`
- `PATCH /api/knowledge-bases/{knowledgeBaseId}`
- `DELETE /api/knowledge-bases/{knowledgeBaseId}`
- `GET /api/documents/kb/{kbId}`
- `POST /api/documents/upload`
- `DELETE /api/documents/{documentId}`

### 工具与 SSE

- `GET /api/tools`
- `GET /sse/connect/{chatSessionId}`

## 数据模型概览

主业务表定义位于：

- `jchatmind_assert/jchatmind.sql`

核心表包括：

- `agent`
- `chat_session`
- `chat_message`
- `knowledge_base`
- `document`
- `chunk_bge_m3`

其中：

- `allowed_tools`、`allowed_kbs`、`chat_options` 使用 `JSONB`
- 向量表 `chunk_bge_m3.embedding` 的维度为 `1024`
- 向量索引使用 `ivfflat`

## 当前实现边界

这部分建议先看，能帮你避免踩当前版本的坑。

1. 前端上传入口当前只接受 `.md` 文件
2. 非 Markdown 文件即使通过后端上传，也只会保存文件，不会进入解析和向量化流程
3. Markdown 切片当前是按标题分段，向量化使用的是“标题文本”，不是整段正文
4. Agent 表单里有 `temperature`、`topP`，这些值会入库，但当前运行时实际主要使用的是 `messageLength`
5. 前端定义了 `AI_PLANNING`、`AI_THINKING`、`AI_EXECUTING`、`AI_DONE` 状态类型，但后端当前主要推送的是新增消息内容
6. 前端 API / SSE 地址已支持环境变量，未配置时默认仍指向本地 `localhost:8080`
7. 文件系统工具在代码中存在，但当前未启用
8. 邮件工具依赖真实 SMTP 配置，数据库工具依赖你本地 PostgreSQL 中确实有可查询的数据

## 后续可以优先补齐的方向

- 把 `temperature`、`topP` 真正接入模型调用
- 支持 PDF / TXT / HTML 等更多知识库文件格式
- 优化 chunk 策略，改为正文级向量化
- 为 SSE 增加更细的 Agent 状态事件
- 为工具调用增加更严格的权限边界和审计日志

## License

本项目使用 MIT License，详见 `LICENSE`。
