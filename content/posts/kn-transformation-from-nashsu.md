---
title: "KN：从 nashsu/llm_wiki 源码出发的企业改造方案"
date: 2026-05-25
tags: ["KN", "nashsu", "llm_wiki", "企业改造", "架构设计", "源码分析"]
summary: "基于 nashsu/llm_wiki v0.4.13 源码的逐模块分析，输出 KN（Knowledge Network）企业服务端改造方案。涵盖可复用模块清单（~70% 核心逻辑可直接复用）、必须重写的部分、分阶段路线图和最终产品形态。"
series: ["KN 企业知识网络"]
series_order: 3
---

> **KN 企业知识网络系列（共三篇）**：
> - **上篇**：[nashsu/llm_wiki 源码架构分析](/posts/nashsu-llmwiki-architecture-deep-dive/) —— 当前架构是什么，核心思想是什么
> - **中篇**：[KN 企业知识网络架构设计](/posts/kn-architecture-design/) —— 如果要改造成企业级，架构应该长什么样
> - **下篇（本文）**：KN 改造方案：从源码出发 —— 具体怎么改，最终产品形态

---

## 〇、源码全景速览

### 物理结构

```
llm_wiki/                         # 当前形态：Tauri v2 桌面应用
├── src/lib/                      # TypeScript 核心逻辑（~35K 行，100+ 模块）
│   ├── ingest.ts (1833 行)       #   核心摄入管线（两步 CoT）
│   ├── llm-client.ts / llm-providers.ts  # LLM 调用层
│   ├── wiki-graph.ts (304 行)    #   知识图谱构建（graphology + Louvain）
│   ├── graph-relevance.ts (312行)#   4 信号关联度模型
│   ├── lint.ts (299 行)          #   7 项健康检查
│   ├── search.ts (82 行)         #   前端搜索 tokenizer（真正搜索在 Rust）
│   ├── embedding.ts              #   多 Provider 嵌入
│   └── ... (90+ 其他模块)
├── src-tauri/src/                # Rust 后端（~5K 行）
│   ├── api_server.rs (1358 行)   #   tiny_http API 服务（端口 19828）
│   ├── commands/search.rs (1119行)#   混合检索（WalkDir 关键词 + LanceDB 向量 + RRF）
│   ├── commands/vectorstore.rs (1018行)# LanceDB 向量存储（per-chunk v2 模式）
│   └── commands/fs.rs            #   文件系统桥接
├── src/stores/                   # Zustand 状态管理（前端）
└── src/components/               # React 19 组件（前端）
```

### Tauri 依赖清单（改造的焦点）

| 依赖 | 影响模块数 | 改造策略 |
|---|---|---|
| `@/commands/fs` (readFile/writeFile/listDirectory/deleteFile/fileExists) | 18 个 | → Node.js `fs/promises` |
| `@tauri-apps/plugin-http` (含 tauri-fetch.ts 封装) | 3 个 | → `undici` |
| `@tauri-apps/plugin-store` (load) | 2 个 | → PostgreSQL |
| `@tauri-apps/api/core` (invoke/listen/convertFileSrc) | 6 个 | → REST API + EventEmitter |

---

## 一、可复用模块清单（无 Tauri 依赖或依赖极浅）

这些模块可以直接复制到 KN 服务端，**修改量 < 5%**：

### 1.1 LLM 层 —— 完全可复用

| 模块 | 行数 | Tauri 依赖 | 复用说明 |
|---|---|---|---|
| `llm-providers.ts` | 760 | 仅 `tauri-fetch.ts` 一处 | 替换 fetch 实现即可 |
| `llm-client.ts` | 274 | 仅 `tauri-fetch.ts` 一处 | 同上 |
| `reasoning-detector.ts` | — | 无 | 纯逻辑，直接复用 |
| `endpoint-normalizer.ts` | — | 无 | 纯逻辑，直接复用 |

### 1.2 摄入管线 —— 核心可复用

| 模块 | 行数 | Tauri 依赖 | 复用说明 |
|---|---|---|---|
| `ingest.ts` (buildAnalysisPrompt) | — | 无 | **纯提示词构建**，`buildAnalysisPrompt()` 和 `buildGenerationPrompt()` 直接复用 |
| `ingest.ts` (parseFileBlocks) | — | 无 | **纯解析**，`parseFileBlocks()` + `isSafeIngestPath()` 直接复用 |
| `ingest.ts` (tryReadFile 辅助) | — | `@/commands/fs` | 改为 `fs.readFile()` |
| `ingest-sanitize.ts` | — | 无 | 纯逻辑，直接复用 |
| `ingest-cache.ts` | 126 | `@/commands/fs` | 改为 PG 或 Redis 缓存 |
| `page-merge.ts` | — | 无 | 纯逻辑，直接复用 |
| `review-utils.ts` | — | 无 | 纯逻辑，直接复用 |
| `sweep-reviews.ts` | — | 无 | 纯逻辑，直接复用 |

### 1.3 图谱层 —— 核心可复用

| 模块 | 行数 | Tauri 依赖 | 复用说明 |
|---|---|---|---|
| `graph-relevance.ts` | 312 | `@/commands/fs`（仅读取 wiki 页面） | 关联度计算逻辑完全独立，仅输入数据来源需改 |
| `wiki-graph.ts` (detectCommunities) | — | 无 | Louvain 算法纯粹在 `graphology` 上运行 |
| `graph-insights.ts` | — | 无 | 惊奇连接 + 知识空白检测，纯算法 |
| `graph-search.ts` | — | 无 | 2-hop 图遍历，纯算法 |

### 1.4 其他纯逻辑模块

| 模块 | 复用方式 |
|---|---|
| `lint.ts` (7 项检查规则) | 去掉 `readFile/listDirectory`，改为从 PG 读取 |
| `wikilink-transform.ts` | 直接复用 |
| `enrich-wikilinks.ts` | 直接复用 |
| `frontmatter.ts` (YAML 解析) | 直接复用 |
| `wiki-filename.ts` | 直接复用 |
| `path-utils.ts` | 直接复用 |
| `dedup.ts` / `dedup-runner.ts` | 改写存储后端 |
| `text-chunker.ts` (CJK bigram + English word) | 直接复用 |
| `context-budget.ts` (60/20/5/15 分配) | 直接复用 |
| `source-identity.ts` | 直接复用 |
| `project-identity.ts` (UUID) | 直接复用 |
| `sources-merge.ts` | 直接复用 |

**小结**：约 **70% 的 TypeScript 核心逻辑可以直接复用**，主要是 LLM 调用、摄入提示词、图谱算法、lint 规则。需要改的是数据源（文件系统 → DB/对象存储）和调用方式（invoke → REST/method call）。

---

## 二、必须重写的部分

### 2.1 Rust 层 → Fastify 服务（最大工程）

当前 Rust 后端承担三类职责，全部需要移植：

#### A. 混合搜索引擎 （commands/search.rs, 1119 行）

当前实现：
```
WalkDir 扫描 wiki/*.md → 关键词分词匹配 → LanceDB 向量检索 → RRF 融合
```

**KN 改造方案**：

| 组件 | 当前 (Rust) | KN (TS/Node) |
|---|---|---|
| 关键词索引 | WalkDir + 内存 tokenizer | PostgreSQL `tsvector` 全文索引（支持 CJK `zhparser`） |
| 向量存储 | 嵌入式 LanceDB | pgvector（PostgreSQL 扩展） |
| 融合算法 | RRF (Reciprocal Rank Fusion) | 同上，TS 实现 |
| 分词器 | Rust CJK bigram | 复用 `text-chunker.ts` |
| 嵌入生成 | Rust `reqwest` 调用 | 复用 `embedding.ts` |

**不需要从头写**：RRF 融合逻辑、分词策略、打分权重都可以从 `search.rs` 翻译为 TS。

#### B. 向量存储 （commands/vectorstore.rs, 1018 行）

当前：LanceDB 嵌入式，per-chunk 存储（`wiki_chunks_v2` 表）。

**KN 改造方案**：pgvector，表结构：
```sql
CREATE TABLE wiki_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id TEXT NOT NULL,
    chunk_index INT NOT NULL,
    chunk_text TEXT NOT NULL,
    heading_path TEXT,
    embedding vector(1536),   -- OpenAI text-embedding-3-small 维度
    project_id UUID REFERENCES projects(id),
    UNIQUE(page_id, chunk_index)
);
CREATE INDEX ON wiki_chunks USING ivfflat (embedding vector_cosine_ops);
```

#### C. API 服务器 （api_server.rs, 1358 行）

当前：tiny_http，6 个端点，120 req/s 限流，64 并发上限。

**KN 改造方案**：Fastify + TypeScript，端点设计：

```
GET  /health                          # 健康检查
GET  /api/v1/projects                 # 列出项目
POST /api/v1/projects                 # 创建项目
GET  /api/v1/projects/:id             # 项目详情
GET  /api/v1/projects/:id/files       # 列出 wiki 文件
GET  /api/v1/projects/:id/files/content?path=wiki/entities/foo.md  # 读取文件
POST /api/v1/projects/:id/search      # 混合检索
GET  /api/v1/projects/:id/graph       # 知识图谱
POST /api/v1/projects/:id/ingest      # 触发摄入（新增）
POST /api/v1/projects/:id/chat        # RAG 对话（新增，当前 Rust 版 501）
POST /api/v1/projects/:id/lint        # 健康检查（新增）
```

### 2.2 文件 I/O 抽象层

当前 18 个模块直接依赖 `@/commands/fs`。KN 需要建立统一的 `StorageProvider` 接口：

```typescript
// kn-server/src/lib/storage-provider.ts
export interface StorageProvider {
  readFile(path: string): Promise<string>;
  writeFile(path: string, content: string): Promise<void>;
  deleteFile(path: string): Promise<void>;
  fileExists(path: string): Promise<boolean>;
  listDirectory(path: string): Promise<FileNode[]>;
  
  // 扩展能力（Tauri 没有的）
  getSignedUrl(path: string, expiresIn: number): Promise<string>;
  getFileStream(path: string): Promise<Readable>;
}

// Phase 1: 本地文件系统（开发用）
export class LocalStorageProvider implements StorageProvider { ... }

// Phase 2: MinIO/S3（生产用）
export class S3StorageProvider implements StorageProvider { ... }
```

**对接方式**：在所有可复用模块中，将 `readFile(path)` 改为 `ctx.storage.readFile(path)`，通过依赖注入传入。

### 2.3 配置/状态持久化

当前 `project-store.ts` 使用 `@tauri-apps/plugin-store`（本地 JSON）。KN 改为 PostgreSQL：

```sql
-- 项目表
CREATE TABLE projects (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID REFERENCES orgs(id),
    name TEXT NOT NULL,
    path TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- 配置表（替代 Tauri Store）
CREATE TABLE project_configs (
    project_id UUID REFERENCES projects(id) PRIMARY KEY,
    llm_config JSONB,       -- {provider, apiKey, model, ...}
    embedding_config JSONB,
    multimodal_config JSONB,
    search_config JSONB,
    api_config JSONB
);
```

---

## 三、关键架构决策

### 3.1 图谱持久化：内存 graphology → Neo4j

当前 `wiki-graph.ts` 的问题：**每次查询都从磁盘扫描所有 markdown 文件重建图谱**。这在企业场景（万级页面）下不可接受。

**KN 方案**：

```
摄入事件 → 解析 wikilink → 写入 Neo4j（增量更新）
查询时 → Neo4j 直接查询图 → 无需重建

Cypher 示例：
MATCH (p:Page {id: 'lightrag'})-[r:LINKS_TO]-(related:Page)
RETURN related.id, related.title, r.weight
```

**关键**：`graph-relevance.ts` 的 4 信号关联度计算需要翻译为 Neo4j 查询或预计算存储。

### 3.2 多租户隔离

nashsu/llm_wiki 是单用户桌面应用，没有租户概念。

**KN 方案**：`Org → Project → 资源` 三级隔离：

```sql
CREATE TABLE orgs (id UUID PRIMARY KEY, name TEXT, created_at TIMESTAMPTZ);
CREATE TABLE org_members (org_id UUID, user_id UUID, role TEXT);
```

所有查询自动带上 `org_id` 过滤器。对象存储路径：`{org_id}/{project_id}/wiki/...`

### 3.3 异步任务队列

当前 `ingest-queue.ts` 是内存队列。KN 改为 BullMQ + Redis：

```typescript
// kn-server/src/queues/ingest-queue.ts
import { Queue, Worker } from 'bullmq';

export const ingestQueue = new Queue('ingest', { connection: redis });
// 每个摄入任务独立执行，支持重试、进度追踪、并发控制
```

### 3.4 LLM 调用收敛

当前每个 Tauri 应用实例直接调 LLM。KN 需要统一的 `LlmGateway`：

```typescript
// kn-server/src/lib/llm-gateway.ts
export class LlmGateway {
  async streamChat(config: LlmConfig, messages: Message[], opts: StreamOpts) {
    // 统一管理：provider 路由、速率限制、成本追踪、fallback
  }
}
```

复用 `llm-providers.ts` 的 provider 配置逻辑，剥离 Tauri HTTP 调用改用 `undici`。

---

## 四、分阶段改造路线图（从源码出发）

### Phase 1：单体服务端验证（4-6 周）

**目标**：把 nashsu/llm_wiki 的核心摄入+检索链路在单进程 Fastify 中跑通。

**具体动作**（按依赖顺序）：

| # | 动作 | 涉及源码 | 产出 |
|---|---|---|---|
| 1.1 | 创建 `kn-server/` 项目骨架 | — | Fastify + TS + Vitest |
| 1.2 | 移植 `llm-client.ts` + `llm-providers.ts` | 改 `tauri-fetch.ts` → `undici` | LlmGateway |
| 1.3 | 建立 `StorageProvider` 接口 + `LocalStorageProvider` | 替 `@/commands/fs` | 文件抽象层 |
| 1.4 | 移植 `ingest.ts` 核心（autoIngest + 两步 CoT 提示词） | 注入 `StorageProvider` | IngestService |
| 1.5 | 移植 `text-chunker.ts` + `embedding.ts` | 剥离 Tauri invoke | EmbeddingService |
| 1.6 | 实现 PostgreSQL 关键词搜索 + pgvector 向量搜索 | 翻译 `search.rs` RRF 逻辑 | SearchService |
| 1.7 | 移植 `graph-relevance.ts` + `wiki-graph.ts`（内存版） | 改数据源 | GraphService |
| 1.8 | 移植 `lint.ts` + `review-utils.ts` | 改数据源 | LintService |
| 1.9 | 实现 Fastify API 端点（6 个基础 + ingest/chat/lint） | 参考 `api_server.rs` | REST API |
| 1.10 | 集成测试：源文件 → 摄入 → 搜索 → 图谱 | — | E2E 验证 |

**验证标准**：给定 10 份测试文档，能完成完整的摄入→检索→图谱→对话链路。

### Phase 2：企业特性叠加（4-6 周）

| # | 动作 | 关键模块 |
|---|---|---|
| 2.1 | PostgreSQL 元数据表（projects/configs/users） | 替 `project-store.ts` |
| 2.2 | JWT 认证 + RBAC | Fastify 中间件 |
| 2.3 | Neo4j 图谱持久化（增量更新） | 替内存 graphology |
| 2.4 | BullMQ 异步任务队列 | 替内存 `ingest-queue.ts` |
| 2.5 | MinIO 对象存储 | 替 `LocalStorageProvider` |
| 2.6 | Webhook + CI 集成 | 参考 Synthadoc hooks |
| 2.7 | OpenTelemetry 可观测性 | 对标 Langfuse |

### Phase 3：MCP Server + 生态集成（2-4 周）

| # | 动作 | 参考 |
|---|---|---|
| 3.1 | MCP Server (`@modelcontextprotocol/sdk`) | 参考 `llm-wiki-compiler` 的 7 工具 + 5 资源 |
| 3.2 | Open WebUI 集成（Ollama 兼容接口） | 参考 LightRAG 的 Ollama 模式 |
| 3.3 | Web UI（React 19 组件复用） | 剥离 Tauri 组件，Next.js 服务端渲染 |

---

## 五、风险点与缓解措施

### 5.1 搜索需求：pgvector 够不够

nashsu 的 LanceDB 是嵌入式向量库，chunk 级存储。pgvector 功能等价但需要关注：
- **维度限制**：pgvector 最大 2000 维，OpenAI text-embedding-3-small 是 1536 维，够用
- **性能**：ivfflat 索引在 10 万级向量时查询 < 50ms。如需更大规模，考虑 pgvector 的 HNSW 索引或独立向量库
- **缓解**：Phase 1 用 pgvector 验证，Phase 3 如有需要再引入 Milvus/Qdrant

### 5.2 图谱重建性能

当前每次重建图谱的逻辑在 Phase 1 保留（快速验证），Phase 2 迁移到 Neo4j 增量更新。迁移时注意：
- `graph-relevance.ts` 的 4 信号权重需要在 Neo4j 中预计算
- Louvain 社区检测有两种路径：(a) Neo4j GDS 插件自带，(b) 导出子图到 graphology 计算

### 5.3 TypeScript 到 TypeScript 的隐形成本

复用 `src/lib/*.ts` 模块时注意：
- 路径别名 `@/` 需要重新映射
- `useWikiStore.getState()` 这种 Zustand store 的跨模块引用需要重构为参数传递
- 建议策略：先把函数签名中的 `useWikiStore` 改为参数注入，减少耦合

### 5.4 nashsu 上游更新

nashsu/llm_wiki 更新频繁（当前 v0.4.13），我们的改造不应 fork 后原地改，而应：
- 建立 `kn-core` 独立包，依赖 nashsu 核心逻辑的"提取版"
- 对 nashsu 的更新保持关注，手动 cherry-pick 有价值的功能

---

## 六、总结：核心复用率

| 层 | 可复用 % | 需改写 | 需重写 |
|---|---|---|---|
| LLM 调用层 | 90% | 替换 fetch 实现 | — |
| 摄入管线（提示词+解析） | 85% | 替换文件 I/O | — |
| 图谱算法 | 90% | 替换数据源 | 图谱持久化（Neo4j） |
| 搜索 | 30% | RRF 逻辑翻译 | WalkDir+LanceDB → PG+pgvector |
| 基础设施 | 5% | — | API 服务器、存储、认证、队列 |

**总体**：约 **70% 的核心领域逻辑可直接复用**，主要工程在基础设施替换（约需 6-8 周完成 Phase 1）。

---

## 参考来源

1. nashsu/llm_wiki v0.4.13 源码。GitHub. <https://github.com/nashsu/llm_wiki>
2. Karpathy, A. "LLM Wiki." Gist. <https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f>
3. atomicstrata/llm-wiki-compiler. GitHub. <https://github.com/atomicstrata/llm-wiki-compiler> — MCP 参考设计
4. axoviq-ai/Synthadoc. GitHub. <https://github.com/axoviq-ai/synthadoc> — 企业级参考：多 wiki 隔离、成本守卫、对抗审查

---

> **系列导航**：
> - **上篇**：[nashsu/llm_wiki 源码架构分析](/posts/nashsu-llmwiki-architecture-deep-dive/) —— 当前架构与核心思想
> - **中篇**：[KN 企业知识网络架构设计](/posts/kn-architecture-design/) —— 企业级架构蓝图
> - **下篇（本文）**：KN 改造方案：从源码出发 —— 具体改法与最终产品形态
