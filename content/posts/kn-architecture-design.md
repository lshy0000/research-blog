---
title: "KN：基于 nashsu/llm_wiki 的企业知识网络架构设计"
date: 2026-05-23
tags: ["KN", "知识网络", "架构设计", "LLM Wiki", "企业级"]
summary: "以 nashsu/llm_wiki 为核心底座，设计企业级知识网络 KN（Knowledge Network）的完整系统架构，覆盖服务化改造、多租户、图谱引擎、混合检索、MCP Server 和分阶段演进路径。"
---

## 一、KN 项目定位

### 1.1 什么是 KN

**KN（Knowledge Network）** 是基于 nashsu/llm_wiki 改造的企业级知识网络平台。核心理念继承自 Karpathy LLM Wiki 范式——**增量构建持久化知识库，知识随时间复利增长**，并在此基础上叠加企业级能力。

### 1.2 与企业调研的关系

本文档是《[企业级 LLM Wiki 方案调研报告](https://lshy0000.github.io/research-blog/posts/llm-wiki-enterprise-research/)》的延续。调研结论：**选定 nashsu/llm_wiki 作为底座**，理由如下：

| 维度 | nashsu/llm_wiki 优势 |
|------|---------------------|
| 图谱引擎 | 4 信号关联度模型 + Louvain 社区检测 + 惊奇连接 + 知识空白（四方案最强） |
| 混合检索 | 分词 + 向量语义 + 图谱扩展三阶段管线（召回率 71.4%） |
| 摄入管线 | 两步 CoT 摄入 + 持久化队列 + 异步 Review + Deep Research |
| API 层 | 内置 HTTP API Server（端口 19828），可直接复用 |
| 多格式摄入 | PDF/DOCX/PPTX/XLSX/音视频/网页（四方案最全） |
| LLM Provider | OpenAI / Anthropic / Gemini / Ollama / 自定义（5+） |

### 1.3 核心设计目标

| # | 目标 | 说明 |
|---|------|------|
| 1 | **服务化** | 剥离 Tauri 桌面壳，转为纯服务端架构 |
| 2 | **多租户** | 组织/项目/用户三级隔离，SSO 集成 |
| 3 | **协作编辑** | 多人同时维护知识库，冲突解决 |
| 4 | **图谱持久化** | 内存 sigma.js 图谱 → Neo4j 图数据库 |
| 5 | **MCP Server** | 标准 MCP 协议，Agent 生态无缝接入 |
| 6 | **企业安全** | JWT/OAuth2.0、RBAC、审计日志、数据加密 |

---

## 二、系统架构总览

### 2.1 高层架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         KN Gateway                              │
│            (Nginx/Kong — 路由、限流、TLS 终结)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │  Web UI  │  │ MCP Srv  │  │ REST API │  │ Webhook/Event│   │
│  │ (Next.js)│  │ (stdio/  │  │ (Fastify)│  │   (NATS)     │   │
│  │          │  │  HTTP)   │  │          │  │              │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│       │             │             │               │            │
│       └─────────────┴──────┬──────┴───────────────┘            │
│                            │                                    │
│                   ┌────────▼────────┐                          │
│                   │   Core Engine   │                          │
│                   │ (TypeScript/RS) │                          │
│                   └────────┬────────┘                          │
│                            │                                    │
│     ┌──────────┬───────────┼───────────┬──────────┐           │
│     │          │           │           │          │           │
│  ┌──▼──┐  ┌───▼───┐  ┌───▼───┐  ┌───▼───┐  ┌───▼───┐       │
│  │Ingest│  │Query  │  │ Lint  │  │ Graph │  │ Admin │       │
│  │Engine│  │Engine │  │Engine │  │Engine │  │Service│       │
│  └──┬──┘  └───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘       │
│     │         │          │          │          │             │
└─────┼─────────┼──────────┼──────────┼──────────┼─────────────┘
      │         │          │          │          │
   ┌──▼─────────▼──────────▼──────────▼──────────▼──┐
   │                  Data Layer                     │
   │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
   │  │PostgreSQL│ │  Neo4j   │ │  Object Storage  │ │
   │  │(元数据)   │ │(知识图谱) │ │  (S3/MinIO 文件) │ │
   │  └──────────┘ └──────────┘ └──────────────────┘ │
   │  ┌──────────┐ ┌──────────────────────────────┐ │
   │  │  Redis   │ │ LanceDB / pgvector (向量)     │ │
   │  │ (缓存)    │ │                              │ │
   │  └──────────┘ └──────────────────────────────┘ │
   └───────────────────────────────────────────────┘
```

### 2.2 技术选型总表

| 层 | 组件 | 选型 | 对标 nashsu/llm_wiki 现状 |
|---|------|------|--------------------------|
| **网关** | 反向代理 | Nginx / Kong | 无（桌面应用） |
| **Web UI** | 前端框架 | Next.js 15 + React 19 | React 19 + Vite（保留组件） |
| **REST API** | 服务框架 | Fastify (Node.js) | 内置 tiny_http (Rust) |
| **MCP Server** | 协议实现 | `@modelcontextprotocol/sdk` | 无 |
| **Core Engine** | 核心逻辑 | TypeScript（复用 `src/lib/`） | Tauri invoke() → 直接函数调用 |
| **LLM Gateway** | Provider 路由 | 复用 `src/lib/llm-providers.ts` + `llm-client.ts` | 已有，需剥离 Tauri HTTP 插件 |
| **关系数据库** | 元数据/配置 | PostgreSQL 16 | 本地 JSON 文件 (Tauri Store) |
| **图数据库** | 知识图谱 | Neo4j 5 | 内存 graphology + sigma.js |
| **向量存储** | 语义检索 | pgvector / LanceDB Server | 嵌入式 LanceDB (Rust) |
| **对象存储** | 原始文件 | MinIO / S3 | 本地文件系统 |
| **缓存** | 热数据 | Redis 7 | 无 |
| **消息队列** | 异步任务 | BullMQ (Redis) | 内存队列 (dedup-queue.ts) |
| **认证** | 身份验证 | Auth0 / Keycloak + JWT | 无（单人桌面） |
| **监控** | 可观测性 | OpenTelemetry + Grafana | 无 |

---

## 三、核心服务设计

### 3.1 Ingest Engine（摄入引擎）

#### 3.1.1 现状继承

nashsu/llm_wiki 的摄入管线是四个方案中最完善的，核心流程直接继承：

```
源文件 → 格式解析 → 两步 CoT 分析 → Wiki 页面生成 → Review 队列 → 合入 Wiki
```

**两步 Chain-of-Thought**（`src/lib/ingest.ts`）：
- **Step 1 Analysis**：LLM 分析源文件，提取实体、概念、矛盾、关系（`max_tokens: 4096 → 16384`）
- **Step 2 Generation**：基于分析结果生成 Wiki Markdown 文件（`max_tokens: 8192 → 32768`）

#### 3.1.2 企业级改造

```
                    ┌─────────────────┐
  多种入口 ────────→│   Ingest Queue  │
  · Web UI 上传     │   (BullMQ)      │
  · API 提交        │                 │
  · Web Clipper     │  ┌───────────┐  │
  · 文件监听        │  │ Parser    │  │
  · 定时导入        │  │ Workers   │  │
                    │  └─────┬─────┘  │
                    │        │        │
                    │  ┌─────▼─────┐  │
                    │  │ CoT       │  │
                    │  │ Analysis  │  │
                    │  └─────┬─────┘  │
                    │        │        │
                    │  ┌─────▼─────┐  │
                    │  │ Page Gen  │  │
                    │  └─────┬─────┘  │
                    │        │        │
                    │  ┌─────▼─────┐  │
                    │  │ Review    │  │
                    │  │ Queue     │  │
                    │  └─────┬─────┘  │
                    └────────┼────────┘
                             │
                    ┌────────▼────────┐
                    │  Wiki Storage   │
                    │  (PG + MinIO)   │
                    └─────────────────┘
```

**改造要点**：

| 改造项 | 现状 | 目标 |
|--------|------|------|
| 队列持久化 | 内存 `dedup-queue.ts` + IndexedDB | BullMQ + Redis |
| 格式解析器 | Tauri Rust 命令调用 | Worker 进程，支持水平扩展 |
| 源文件存储 | 本地磁盘 `raw/sources/` | MinIO 对象存储 |
| Review 流程 | 单人审核 UI | 多人审批流 + 评论 + 版本对比 |

#### 3.1.3 多格式解析器矩阵

| 格式 | 解析器 | 现状 | 企业版 |
|------|--------|------|--------|
| Markdown/Text | 直接读取 | ✅ | ✅ |
| PDF | pdfium-render (Rust) | ✅ | ✅ + OCR 回退 |
| DOCX | mammoth 提取 | ✅ | ✅ |
| PPTX | 自定义解析 | ✅ | ✅ |
| XLSX/XLS/ODS | calamine (Rust) | ✅ | ✅ |
| 图片 | Vision LLM Caption | ✅ | ✅ |
| 音视频 | Whisper 转写 + Vision | ✅ | ✅ + 说话人分离 |
| 网页 | Readability + Turndown | ✅ Clipper 扩展 | ✅ + Headless Browser |
| 代码仓库 | 无 | ❌ | 新增：AST 解析 + 目录结构 |

---

### 3.2 Query Engine（检索引擎）

#### 3.2.1 现状继承

nashsu/llm_wiki 的四阶段混合检索管线（`src/lib/search.ts` + `src/lib/graph-relevance.ts`）：

```
用户查询
  │
  ▼
阶段1：分词搜索（CJK 二元组 + 英文分词 + 标题加权）
  │
  ▼
阶段2：向量语义检索（LanceDB + 自定义分块器）→ RRF 融合
  │
  ▼
阶段3：图谱扩展（4 信号关联度模型 → 2 跳遍历）
  │
  ▼
阶段4：上下文组装（预算控制 60/20/5/15 分配）
  │
  ▼
LLM 答案生成
```

#### 3.2.2 企业级检索架构

```
                         ┌─────────────┐
  用户查询 ─────────────→│ Query Router │
                         └──────┬──────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
   ┌──────▼──────┐     ┌───────▼───────┐     ┌───────▼──────┐
   │  Full-Text  │     │   Semantic    │     │   Graph      │
   │  Search     │     │   Search      │     │   Search     │
   │  (pg_trgm)  │     │  (pgvector)   │     │  (Neo4j)     │
   └──────┬──────┘     └───────┬───────┘     └───────┬──────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │
                      ┌─────────▼─────────┐
                      │   RRF Fusion      │
                      │ (Reciprocal Rank) │
                      └─────────┬─────────┘
                                │
                      ┌─────────▼─────────┐
                      │   Context Builder │
                      │   预算控制 + 排序   │
                      └─────────┬─────────┘
                                │
                      ┌─────────▼─────────┐
                      │   LLM Answer Gen  │
                      └───────────────────┘
```

**改造要点**：

| 组件 | 现状 | 企业版 |
|------|------|--------|
| 全文索引 | 运行时内存分词 | PostgreSQL `pg_trgm` + GIN 索引 |
| 向量检索 | 嵌入式 LanceDB | pgvector (IVFFlat/HNSW) 或独立 LanceDB Server |
| 图谱检索 | 内存 graphology | Neo4j Cypher 查询 |
| RRF 融合 | TypeScript 实现 | 保留算法，改为数据库端并行查询 |
| 上下文组装 | 60/20/5/15 预算 | 可配置预算模板 + Token 精确计数 |

#### 3.2.3 4 信号关联度模型（保留增强）

来自 `src/lib/graph-relevance.ts`：

| 信号 | 权重 | 企业版增强 |
|------|------|-----------|
| 直接 wikilink `[[page]]` | ×3.0 | + 协同过滤（同组织用户点击） |
| 来源重叠（共享原始文档） | ×4.0 | + 时间衰减 |
| Adamic-Adar（共享邻居加权） | ×1.5 | 保留 |
| 类型亲和（同类型加分） | ×1.0 | + 自定义类型权重 |

---

### 3.3 Graph Engine（图谱引擎）

#### 3.3.1 现状

nashsu/llm_wiki 的图谱全部在**前端内存**中处理（`src/lib/wiki-graph.ts`）：
- graphology 库构建无向图
- Louvain 社区检测
- ForceAtlas2 布局算法
- sigma.js 渲染

**限制**：
- 图谱不可持久化（每次重启重建）
- 节点数受限（浏览器内存上限 ~10K 节点）
- 无法做复杂图查询（无 Cypher/GQL）

#### 3.3.2 企业级图谱架构

```
┌─────────────────────────────────────────┐
│              Neo4j Cluster               │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │  知识图谱 (Knowledge Graph)       │   │
│  │                                  │   │
│  │  (:Page) -[:WIKILINK]-> (:Page)  │   │
│  │  (:Page) -[:SOURCED_FROM]->      │   │
│  │    (:Source)                     │   │
│  │  (:Page) -[:BELONGS_TO]->        │   │
│  │    (:Community)                  │   │
│  │  (:Page) -[:HAS_TYPE]->          │   │
│  │    (:PageType)                   │   │
│  │  (:Page) -[:CITES]->             │   │
│  │    (:ExternalResource)           │   │
│  └──────────────────────────────────┘   │
└──────────────┬──────────────────────────┘
               │
    ┌──────────▼──────────┐
    │   Graph Service     │
    │  (Node.js Driver)   │
    │                    │
    │  · 写入：Ingest 触发 │
    │  · 查询：Cypher API │
    │  · 分析：GDS 算法库  │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │   Graph API         │
    │                    │
    │  · 社区检测         │
    │  · 惊奇连接         │
    │  · 知识空白         │
    │  · 影响传播         │
    │  · 路径发现         │
    └─────────────────────┘
```

#### 3.3.3 Neo4j 图模型

```cypher
// 核心节点
(:Page {
  id: String,          // UUID
  slug: String,        // URL 友好标识
  title: String,       // 页面标题
  content_hash: String,// 内容 SHA-256
  type: String,        // concept/entity/source/howto/reference
  project_id: String,  // 所属项目
  created_at: DateTime,
  updated_at: DateTime
})

(:Source {
  id: String,
  filename: String,
  format: String,      // pdf/docx/markdown/url
  storage_key: String, // MinIO object key
  ingested_at: DateTime
})

(:Community {
  id: Integer,
  label: String,       // Louvain 社区标签
  cohesion: Float,     // 内聚度
  project_id: String
})

// 核心关系
(:Page)-[:WIKILINK {weight: Float, signal_type: String}]->(:Page)
(:Page)-[:SOURCED_FROM {relevance: Float}]->(:Source)
(:Page)-[:BELONGS_TO]->(:Community)
(:Page)-[:HAS_TYPE]->(:PageType)
```

#### 3.3.4 GDS 算法替代

| nashsu/llm_wiki 内存算法 | Neo4j GDS 替代 |
|--------------------------|----------------|
| graphology-communities-louvain | `gds.louvain.mutate()` |
| graphology-layout-forceatlas2 | 前端渲染用 D3/Cytoscape 替代 |
| 4 信号关联度（内存计算） | Cypher 加权查询 + GDS Node Similarity |
| 惊奇连接（跨社区边） | `gds.nodeSimilarity` 跨社区过滤 |
| 知识空白（孤立节点/稀疏社区） | `gds.degree` + `gds.triangleCount` |

---

### 3.4 Lint Engine（健康检查）

#### 3.4.1 继承自 nashsu/llm_wiki

`src/lib/lint.ts` 的检查项完整继承：

| 检查项 | 说明 |
|--------|------|
| 孤儿页面 | 无 wikilink 指向的页面 |
| 矛盾检测 | 两页面声称矛盾的事实 |
| 过时内容 | 源文件更新但 wiki 未更新 |
| 空页面 | 内容过短的 stub 页面 |
| 死链 | wikilink 指向不存在的页面 |
| 重复页面 | 语义高度相似的页面对 |
| 知识空白 | 稀疏社区 / 孤立节点 / 低内聚社区 |

#### 3.4.2 企业级 Lint 架构

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ Cron Trigger│────→│  Lint Queue  │────→│ Lint Worker │
│ (每小时/每天) │     │   (BullMQ)   │     │             │
└─────────────┘     └──────────────┘     │ · 结构检查   │
                                          │ · 语义检查   │
                                          │ · 图谱检查   │
                                          └──────┬──────┘
                                                 │
                                    ┌────────────▼────────────┐
                                    │     Lint Report          │
                                    │                         │
                                    │  · 健康分数 (0-100)      │
                                    │  · 问题清单 (按严重度)    │
                                    │  · 修复建议               │
                                    │  · 趋势图 (历史对比)      │
                                    └────────────┬────────────┘
                                                 │
                                    ┌────────────▼────────────┐
                                    │  Notification           │
                                    │  · Webhook              │
                                    │  · 钉钉/飞书/企微        │
                                    │  · Email                │
                                    └─────────────────────────┘
```

**新增企业检查**：

| 检查项 | 说明 |
|--------|------|
| 权限一致性 | 引用无权访问的页面 |
| 合规检查 | 包含敏感词（PII/密钥/内部代号） |
| 多语言覆盖率 | 关键页面是否有多语言版本 |
| 引用新鲜度 | 外部引用是否仍然有效 |

---

## 四、MCP Server 设计

### 4.1 设计原则

参考 llm-wiki-compiler 的 MCP 实现（7 工具 + 5 资源），为 KN 设计标准 MCP Server。

### 4.2 MCP 工具列表

| # | 工具名 | 需 LLM | 功能 |
|---|--------|--------|------|
| 1 | `kn_ingest_source` | ❌ | 提交源文件（URL/文件路径/文本）到摄入队列 |
| 2 | `kn_compile` | ✅ | 触发增量编译管线（概念抽取 → 页面生成） |
| 3 | `kn_query` | ✅ | 四阶段混合检索 + LLM 答案生成 |
| 4 | `kn_search` | ✅ | 语义检索（不含 LLM 答案生成） |
| 5 | `kn_read_page` | ❌ | 按 slug/project 读取 wiki 页面全文 |
| 6 | `kn_list_pages` | ❌ | 列出项目下所有页面（支持过滤） |
| 7 | `kn_graph_explore` | ❌ | 图谱探索：邻居查询、社区发现、路径检索 |
| 8 | `kn_lint` | ❌ | 运行健康检查（可指定检查项） |
| 9 | `kn_status` | ❌ | 项目统计：页面数、源文件数、图谱节点、健康分 |
| 10 | `kn_save_answer` | ❌ | 将 Query 答案归档为新 wiki 页面 |

### 4.3 MCP 资源

| URI 模板 | 说明 |
|----------|------|
| `kn://{project}/index` | 项目知识索引导航 |
| `kn://{project}/pages/{slug}` | 单个 wiki 页面 |
| `kn://{project}/query/{slug}` | 历史查询及答案缓存 |
| `kn://{project}/sources` | 源文件清单 |
| `kn://{project}/graph/communities` | 社区检测结果 |
| `kn://{project}/lint/report` | 最新 Lint 报告 |

---

## 五、多租户与安全

### 5.1 租户模型

```
Organization (组织)
  └── Project (项目/知识域)
       └── Wiki Pages (知识页面)
            └── Versions (版本历史)
```

```
User (用户)
  ├── Organization Role (组织角色: Admin/Member/Viewer)
  └── Project Role (项目角色: Owner/Editor/Reader)
```

### 5.2 认证架构

```
┌──────────┐     ┌──────────────┐     ┌────────────┐
│  Client  │────→│ API Gateway  │────→│ Auth Service│
│          │     │ (JWT 验证)    │     │             │
└──────────┘     └──────────────┘     │ · Keycloak  │
                                       │ · Auth0     │
                                       │ · LDAP      │
                                       └─────┬───────┘
                                             │
                                    ┌────────▼───────┐
                                    │  RBAC Engine   │
                                    │                │
                                    │ · 组织级权限    │
                                    │ · 项目级权限    │
                                    │ · 页面级 ACL    │
                                    └────────────────┘
```

### 5.3 数据隔离策略

| 数据层 | 隔离方式 |
|--------|---------|
| PostgreSQL | 项目级 Schema 或 `project_id` 行级安全 (RLS) |
| Neo4j | 项目级子图，通过 `project_id` 属性过滤 |
| MinIO | `/{organization_id}/{project_id}/` 路径前缀 |
| Redis | Key 前缀 `{org}:{project}:` |
| LanceDB/pgvector | 项目级表/索引 |

---

## 六、数据流设计

### 6.1 整体数据流

```
                      ┌─────────────────┐
  外部输入 ──────────→│   Write Path    │
  · 文件上传           │                 │
  · URL 提交           │  Ingest Queue   │
  · API 调用           │  → Parser       │
  · MCP 调用           │  → CoT Analysis │
  · Web Clipper        │  → Page Gen     │
  · 定时任务           │  → Review       │
                      └────────┬────────┘
                               │
                      ┌────────▼────────┐
                      │   Storage       │
                      │                 │
                      │  PG: 元数据     │
                      │  Neo4j: 图谱    │
                      │  MinIO: 文件    │
                      │  Vector: 嵌入   │
                      └────────┬────────┘
                               │
                      ┌────────▼────────┐
                      │   Read Path     │
                      │                 │
                      │  Query Engine   │
                      │  → Full-Text    │
                      │  → Semantic     │
                      │  → Graph        │
                      │  → RRF Fusion   │
                      │  → LLM Answer   │
                      └─────────────────┘
```

### 6.2 事件驱动架构

```
Ingest Complete ──→ Graph Update ──→ Search Reindex ──→ Webhook Notify
       │                  │                 │
       ▼                  ▼                 ▼
   Lint Trigger    Community Detect    Cache Invalidate
```

使用 NATS/JetStream 或 Redis Pub/Sub 作为事件总线。

---

## 七、前端架构

### 7.1 组件复用策略

nashsu/llm_wiki 前端（React 19 + shadcn/ui + Tailwind）的大量组件可**直接迁移**：

| 组件 | 路径 | 复用方式 |
|------|------|---------|
| 知识图谱渲染 | sigma.js + graphology | 保留渲染层，数据源改为 Neo4j API |
| Milkdown 编辑器 | `@milkdown/kit` | 完整保留 |
| 聊天面板 | `chat-panel.tsx` | 保留，后端改为 REST API |
| 文件树 | `file-tree.tsx` | 保留，数据改从 API 获取 |
| Review 视图 | `review-view.tsx` | 增强为多人协作 |
| Deep Research | `deep-research.ts` | 保留逻辑 |

### 7.2 企业版新增页面

| 页面 | 功能 |
|------|------|
| 组织管理 | 成员邀请、角色分配、SSO 配置 |
| 项目管理 | 创建/归档项目、跨项目搜索 |
| Lint 仪表板 | 健康分趋势、问题热力图 |
| 审计日志 | 操作记录、版本对比、回滚 |
| 图谱分析 | Neo4j Bloom 风格的可视化探索 |
| 协作空间 | 多人实时编辑 + 评论 |

---

## 八、部署架构

### 8.1 单机部署（Phase 1）

```
┌─────────────────────────────────┐
│         Docker Compose          │
│                                 │
│  ┌──────────┐  ┌─────────────┐  │
│  │ KN Core  │  │ KN Web UI   │  │
│  │ :3000    │  │ :3001       │  │
│  └────┬─────┘  └─────────────┘  │
│       │                          │
│  ┌────┴──────────────────────┐  │
│  │ PostgreSQL + pgvector     │  │
│  │ Neo4j Community           │  │
│  │ Redis                     │  │
│  │ MinIO                     │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### 8.2 集群部署（Phase 3）

```
                    ┌──────────────┐
                    │   LB (Nginx) │
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
   │ KN Core #1  │ │ KN Core #2  │ │ KN Core #3  │
   └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
   │ PostgreSQL  │ │ Neo4j       │ │ Redis       │
   │ (Patroni)   │ │ (Cluster)   │ │ (Cluster)   │
   └─────────────┘ └─────────────┘ └─────────────┘
                           │
                  ┌────────▼────────┐
                  │ MinIO (Dist.)   │
                  └─────────────────┘
```

---

## 九、分阶段演进路线

### Phase 1：核心服务化（第 1-2 月）

**目标**：nashsu/llm_wiki 能作为纯服务端运行

| 任务 | 说明 |
|------|------|
| 剥离 Tauri 调用 | `invoke()` → 直接函数调用 / REST API |
| 剥离 Tauri HTTP 插件 | `@tauri-apps/plugin-http` → `fetch`/`undici` |
| Fastify REST API | 替代内置 `tiny_http` |
| PostgreSQL 存储 | 替代 JSON 文件 + IndexedDB |
| Docker 化 | 单容器可运行 |

**产出**：`kn-core` Docker 镜像，保留原前端但通过 API 通信

### Phase 2：多租户 + 图谱持久化（第 3-4 月）

| 任务 | 说明 |
|------|------|
| 多租户模型 | Organization → Project 层级 |
| JWT 认证 | Keycloak 集成 |
| Neo4j 图谱 | 替代内存 graphology |
| pgvector 向量 | 替代嵌入式 LanceDB |
| BullMQ 任务队列 | 替代内存队列 |
| MCP Server | 10 工具 + 6 资源的标准 MCP |

**产出**：企业级 KN v1.0，支持多团队

### Phase 3：协作 + 规模化（第 5-6 月）

| 任务 | 说明 |
|------|------|
| 协作编辑 | OT/CRDT 实时同步 |
| SSO 集成 | LDAP/SAML/OIDC |
| 审计日志 | 全操作记录 |
| 集群部署 | K8s Helm Chart |
| 监控告警 | OpenTelemetry + Grafana |
| 知识蒸馏 | Skill Factory（参考 OpenKB） |

**产出**：KN v2.0，全企业知识平台

---

## 十、关键决策记录

| # | 决策 | 理由 |
|---|------|------|
| 1 | **底座选 nashsu/llm_wiki 而非 llm-wiki-compiler** | 功能维度全面领先：图谱引擎、混合检索、多格式摄入、API 层。GPL-3.0 不构成本项目障碍 |
| 2 | **保留 TypeScript 核心逻辑** | `src/lib/` 下 ~100+ 模块测试覆盖完善，重写成本高 |
| 3 | **Neo4j 而非 graphology** | 持久化、Cypher 查询、GDS 算法库、集群支持 |
| 4 | **pgvector 而非 LanceDB Server** | 减少运维组件，统一 SQL 查询，PostgreSQL 生态成熟 |
| 5 | **Fastify 而非 Express** | 性能更优，TypeScript 原生支持，插件生态好 |
| 6 | **Next.js 替代 Vite** | SSR/SSG、API Routes、中间件、企业级生态 |
| 7 | **bullMQ + Redis 替代内存队列** | 任务持久化、重试、延迟、优先级、可观测 |

---

## 十一、风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| Tauri API 剥离不彻底 | 核心逻辑无法独立运行 | Phase 1 阶段逐模块验证，保留集成测试 |
| 前端组件迁移工作量大 | 进度延迟 | 先服务化后端，前端保留 Vite 版本并行开发 |
| Neo4j 学习曲线 | 图谱查询性能不足 | Phase 2 初期使用 Neo4j AuraDB 免费版验证 |
| GPL-3.0 合规风险 | 法律风险 | 明确 KN 作为独立项目，通过 API 调用 nashsu/llm_wiki（或 fork 后闭源，需评估） |
| 团队规模不足 | 交付延迟 | 优先 Phase 1 MVP，Phase 2/3 按需推进 |

---

*文档版本 v0.1 | 2026-05-23 | 基于 nashsu/llm_wiki v0.4.12 架构分析*
