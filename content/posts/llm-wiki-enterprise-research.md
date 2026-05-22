---
title: "企业级 LLM Wiki 方案调研报告"
date: 2026-05-22
tags: ["LLM Wiki", "知识管理", "RAG", "开源调研", "企业架构"]
summary: "基于 Karpathy LLM Wiki 思想，对 nashsu/llm_wiki、VectifyAI/OpenKB、AgriciDaniel/claude-obsidian、atomicstrata/llm-wiki-compiler 四个开源仓库的深度技术调研与横向对比。"
---

## 一、调研背景

### 1.1 Karpathy LLM Wiki 核心思想

Andrej Karpathy 在 [Gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 中提出了一个与 RAG 截然不同的知识管理范式：

> **RAG 在每次查询时重新发现知识；LLM Wiki 则增量构建持久化的知识库，知识随时间复利增长。**

核心架构为三层：

```
Raw Sources（不可变原始文档）→ Wiki（LLM 自动维护的 Markdown 知识库）→ Schema（行为规范文档）
```

三大操作：**Ingest**（摄入源 → 更新 wiki）| **Query**（查询 wiki → 保存答案）| **Lint**（健康检查 → 修复矛盾/孤儿/过时）

### 1.2 企业调研目标

寻找满足以下条件的开源方案：

| 维度 | 要求 |
|------|------|
| 文档共享 | 支持团队协作，非单人工具 |
| 多工作空间 | 多项目 / 多知识域隔离 |
| Web 访问 | 浏览器可访问，非纯桌面/CLI |
| 知识图谱可视化 | 关系图谱，非纯文本列表 |
| 语义检索准确 | 向量/混合检索，非仅关键词 |
| Agent 可用 | MCP / API 接口，可被 AI Agent 调用 |
| 开源 | 许可证友好，可商用 |

---

## 二、四个仓库逐个深度分析

### 2.1 nashsu/llm_wiki

**仓库**：`github.com/nashsu/llm_wiki` | **许可证**：GPL-3.0 | **版本**：v0.4.12

#### 技术栈

Tauri v2 桌面应用（Rust 后端 + React 19 前端）：

| 层 | 技术 |
|---|------|
| 桌面壳 | Tauri v2 (Rust) — 文件系统、LanceDB、HTTP API、文件监听 |
| 前端 | React 19 + TypeScript + Vite 8 + shadcn/ui + Tailwind CSS v4 |
| 编辑器 | Milkdown (ProseMirror) WYSIWYG |
| 知识图谱 | sigma.js + graphology + ForceAtlas2 + Louvain 社区检测 |
| 向量库 | LanceDB (Rust 嵌入式) |
| LLM | OpenAI / Anthropic / Gemini / Ollama / 自定义 + Claude Code CLI 子进程 |

#### Karpathy 模式忠实度：⭐⭐⭐⭐⭐

最完整的三层实现：`raw/sources/` → `wiki/` → `schema.md` + `purpose.md`

**独有创新**：
- **两步 Chain-of-Thought 摄入**：LLM 先分析（提取实体/概念/矛盾），再生成 Wiki 文件
- **持久化摄入队列**：崩溃恢复、重试机制、进度可视化
- **异步 Review 系统**：人工审核 LLM 生成的变更
- **Deep Research**：集成 Tavily/SerpApi/SearXNG 网络搜索

#### 检索机制

**四阶段混合检索管线**：

| 阶段 | 方法 | 说明 |
|------|------|------|
| 1 | 分词搜索 | 中文 CJK 二元组 + 英文分词 + 标题加权 |
| 2 | 向量语义 | LanceDB + 自定义分块器 → RRF 融合 |
| 3 | 图谱扩展 | 4 信号关联度模型 → 2 跳遍历 |
| 4 | 上下文组装 | 预算控制 (60/20/5/15 分配) |

基准召回率：分词 58.2% → 向量融合 71.4%

#### 知识图谱可视化

**4 信号关联度模型**：

| 信号 | 权重 | 含义 |
|------|------|------|
| 直接 wikilink | ×3.0 | `[[页面]]` |
| 来源重叠 | ×4.0 | 共享原始文档 |
| Adamic-Adar | ×1.5 | 共享邻居加权 |
| 类型亲和 | ×1.0 | 同类型加分 |

- sigma.js + ForceAtlas2 力导向渲染
- Louvain 社区检测（自动知识聚类）
- **惊奇连接** 发现（跨社区边） + **知识空白** 检测（孤立页面/稀疏社区）

#### 多格式摄入

PDF（pdfium-render）、DOCX、PPTX、XLSX/XLS/ODS（calamine）、图片/视频/音频、Chrome Web Clipper 扩展

#### API 接口

内置 HTTP API 服务器（端口 19828）：
- `GET /api/v1/projects/{id}/search` — 混合检索
- `GET /api/v1/projects/{id}/graph` — 知识图谱
- Token 鉴权 + 速率限制 (120 req/s) + 并发控制 (64 inflight)

---

### 2.2 VectifyAI/OpenKB

**仓库**：`github.com/VectifyAI/OpenKB` | **许可证**：Apache 2.0 | **版本**：0.3.x

#### 技术栈

Python 3.10+ CLI 工具：

| 组件 | 选型 |
|------|------|
| LLM 网关 | LiteLLM（多 Provider 统一） |
| Agent 框架 | OpenAI Agents SDK |
| 长文档索引 | **PageIndex**（VectifyAI 自研，树索引，非向量） |
| 文档转换 | markitdown + pymupdf + trafilatura |
| CLI | Click + Rich + prompt_toolkit |

#### 核心理念：No Vector DB

**OpenKB 明确不使用向量数据库**。这是它最激进的差异化：

- 短文档：LLM 直接读全文
- 长文档 (PDF ≥ 20 页)：PageIndex 构建层级树索引（章节标题 + 摘要 + 页码范围）→ LLM 导航树 → 按需获取具体页面
- 查询时：读 `index.md` → 跟踪 wikilinks → 综合回答

编译管线为 **5 步流水线**（利用 Anthropic prompt caching）：

1. 构建基础上下文
2. 生成摘要（JSON）
3. Concepts 规划（创建/更新/关联）
4. 并发写概念页
5. 修复 wikilinks + 更新 index.md

#### 独特能力：Skill Factory

`openkb skill new` 将知识库蒸馏为 **Anthropic Skill**，可安装到 Claude Code / Codex / Gemini CLI / Cursor：
- 自动生成 `SKILL.md`（触发描述 + 决策规则）
- 自动生成 `references/`（深度材料）
- 自动生成 `.claude-plugin/marketplace.json`

#### 企业级缺失

| 维度 | 状态 |
|------|------|
| Web UI | ❌ 无（路线图中） |
| API | ❌ 无 REST API（仅 CLI + Skill） |
| MCP | ❌ 无 |
| 多租户 | ❌ 无 |
| 认证 | ❌ 无 |
| 图谱可视化 | ⚠️ 仅通过 Obsidian Graph View |

---

### 2.3 AgriciDaniel/claude-obsidian

**仓库**：`github.com/AgriciDaniel/claude-obsidian` | **许可证**：MIT

#### 本质澄清

**这不是 MCP 服务器，而是 Agent Skills 系统 + 预配置 Obsidian Vault。**

核心工作流完全由 **11 个 Agent Skills**（Markdown 指令文件）驱动，LLM 直接读写本地 Markdown 文件。

#### 11 个 Skills 全景

| # | Skill | 功能 |
|---|-------|------|
| 1 | **wiki** | 编排器：初始化、路由、热缓存 |
| 2 | **wiki-ingest** | 源→8-15 wiki 页面→交叉引用 |
| 3 | **wiki-query** | 三层查询：Quick/Standard/Deep |
| 4 | **wiki-lint** | 10 项健康检查 |
| 5 | **save** | 对话归档为 wiki 页面 |
| 6 | **autoresearch** | 3 轮自主研究循环 |
| 7 | **canvas** | Obsidian Canvas JSON 读写 |
| 8 | **wiki-fold** | DragonScale 日志折叠 |
| 9 | **defuddle** | 网页去噪（节省 40-60% token） |
| 10 | **obsidian-markdown** | Markdown 语法规范 |
| 11 | **obsidian-bases** | `.base` YAML 数据库视图 |

#### Obsidian 深度集成

| 功能 | 利用方式 |
|------|---------|
| Wikilinks `[[  ]]` | 核心链接机制 |
| Graph View | 颜色编码图谱（domain/entity/concept/source） |
| Canvas (.canvas) | 可视化层：自动布局 |
| Callouts | `[!contradiction]` `[!gap]` `[!key-insight]` `[!stale]` |
| Bases (.base) | 原生数据库仪表板 |
| CSS Snippets | 彩色文件浏览器 + callout 样式 |

#### DragonScale 机制

三个独特机制解决 wiki 长期维护问题：

1. **地址稳定化**：`c-000042` 格式永久标识符（页面可重命名但地址不变）
2. **日志提取性折叠**：压缩历史日志为摘要，防止 `log.md` 无限增长
3. **语义平铺**：本地 ollama + `nomic-embed-text` 做页面级余弦去重

#### 跨项目引用

任意 Claude Code 项目通过 `CLAUDE.md` 声明 vault 路径即可引用同一 wiki：

```markdown
## Wiki Knowledge Base
Path: ~/Documents/Obsidian Vault
```

成本极低：hot.md (~500 tokens) → index.md → 领域子索引 → 具体页面

---

### 2.4 atomicstrata/llm-wiki-compiler

**仓库**：`github.com/atomicstrata/llm-wiki-compiler` | **许可证**：MIT | **版本**：v0.7.0

#### 技术栈

TypeScript/Node.js (ESM, strict mode)：

| 组件 | 选型 |
|------|------|
| CLI | Commander v13 |
| 构建 | tsup |
| 测试 | Vitest (850+ 测试) |
| LLM | Anthropic / OpenAI / Ollama / MiniMax / Copilot |
| 嵌入 | Voyage-3-lite / text-embedding-3-small / nomic-embed-text |
| Web 抓取 | jsdom + Readability + Turndown |
| MCP | `@modelcontextprotocol/sdk` |
| 前端 | 原生 JS SPA + D3.js 力导向图 |

#### 三层检索架构

```
第1层：Chunk 级语义检索（嵌入 + BM25 重排序）
  ↓ 不可用
第2层：页面级语义检索
  ↓ 不可用
第3层：LLM 基于索引选择
```

- 分块策略：800 字符目标，余弦相似度 Top-30 → BM25 重排序 → 最终 12 块
- 持久化嵌入存储：`embeddings.json` → chunk 级 SHA-256 变更检测
- 增量架构：仅重新嵌入已变更的 chunk

#### 两阶段编译管线

```
阶段1：概念抽取 → 每源文件 LLM tool_use 提取所有概念
阶段2：页面生成 → 合并同名概念 → 每概念生成完整页面 (frontmatter + markdown)
```

**全成功或全不写**：概念提取全部成功才写入页面。

#### MCP 服务器

**7 个 MCP 工具**：

| 工具 | 需 LLM | 功能 |
|------|--------|------|
| `ingest_source` | ❌ | 抓取 URL 或复制本地文件 |
| `compile_wiki` | ✅ | 增量编译管线 |
| `query_wiki` | ✅ | 两步式答案生成 |
| `search_pages` | ✅ | 语义检索 |
| `read_page` | ❌ | 按 slug 读页面 |
| `lint_wiki` | ❌ | 11 条质量检查 |
| `wiki_status` | ❌ | 统计信息 |

**5 个 MCP 资源**：`llmwiki://index` / `llmwiki://concept/{slug}` / `llmwiki://query/{slug}` / `llmwiki://sources` / `llmwiki://state`

#### 本地 Web 查看器

- 纯 Node.js `http` 模块，无框架
- 默认仅绑定 `127.0.0.1`
- 严格 CSP/CORP + sanitize-html + 路径穿越防护
- D3.js 力导向知识图谱
- SPA 架构：hash 路由，原生 JS

---

## 三、横向对比矩阵

### 3.1 企业级维度评分（1-5 分）

| 维度 | llm_wiki<br>(nashsu) | OpenKB<br>(VectifyAI) | claude-obsidian<br>(AgriciDaniel) | llm-wiki-compiler<br>(atomicstrata) |
|------|:---:|:---:|:---:|:---:|
| **文档共享** | ⭐⭐ 单人桌面 | ⭐ 单人 CLI | ⭐⭐ 跨项目引用 | ⭐⭐ 本地 View + MCP |
| **多工作空间** | ⭐⭐⭐ UUID 项目系统 | ⭐ 目录隔离 | ⭐⭐⭐ 6 模式 + 多项目 | ⭐⭐ 独立目录 |
| **Web 访问** | ⭐⭐ 本地 HTTP API | ⭐ CLI 输出 | ⭐⭐ Obsidian Publish | ⭐⭐⭐ 本地 Web 查看器 |
| **图谱可视化** | ⭐⭐⭐⭐⭐ sigma.js + 社区检测 | ⭐⭐ Obsidian Graph | ⭐⭐⭐⭐ Graph + Canvas | ⭐⭐⭐ D3 力导向图 |
| **语义检索** | ⭐⭐⭐⭐ 分词+向量+图谱 3层 | ⭐⭐ 关键词 + wiki 遍历 | ⭐⭐ 索引导航 | ⭐⭐⭐⭐ Chunk 嵌入+BM25 |
| **Agent 可用** | ⭐⭐⭐ API + Agent Skill | ⭐⭐⭐ Skill Factory | ⭐⭐⭐⭐ Skills 指令集 | ⭐⭐⭐⭐⭐ MCP + 7 工具 |
| **开源友好** | ⚠️ GPL-3.0 | ✅ Apache 2.0 | ✅ MIT | ✅ MIT |

### 3.2 技术架构对比

| 维度 | llm_wiki | OpenKB | claude-obsidian | llm-wiki-compiler |
|------|----------|--------|-----------------|-------------------|
| **语言** | Rust + TypeScript | Python | Markdown (Skills) | TypeScript |
| **形态** | 桌面应用 | CLI 工具 | Agent Skills + Vault | CLI + MCP + Viewer |
| **向量库** | LanceDB (嵌入) | **无** | 无 (ollama 仅去重) | embeddings.json (本地) |
| **嵌入模型** | 多 Provider | 无 | nomic-embed-text | Voyage/OpenAI/Ollama |
| **检索方式** | 混合 3 阶段 | 索引+wikilink 遍历 | 索引导航 | Chunk 嵌入+BM25 |
| **LLM Provider** | 5+ 含自定义 | LiteLLM 全兼容 | 依赖 Agent CLI | 5 个 |
| **代码规模** | 大 (~300+ 文件) | 中 (27 测试) | 极小 (11 Skills) | 中 (850+ 测试) |
| **测试覆盖** | ✅ ~70+ 测试 | ✅ 27 测试 | ⚠️ 无自动化测试 | ✅ 850+ Vitest |

### 3.3 Karpathy 模式忠实度

| Karpathy 概念 | llm_wiki | OpenKB | claude-obsidian | llm-wiki-compiler |
|--------------|:---:|:---:|:---:|:---:|
| 三层架构 (Raw/Wiki/Schema) | ✅ | ✅ | ✅ | ✅ |
| Ingest 操作 | ✅ CoT 两步 | ✅ 5 步流水线 | ✅ 8-15 页 | ✅ 两阶段编译 |
| Query 操作 | ✅ 4 阶段检索 | ✅ Agent 查询 | ✅ 3 层深度 | ✅ 语义+回退 |
| Lint 操作 | ✅ 惊奇/空白检测 | ✅ 结构+语义 | ✅ 10 项检查 | ✅ 11 条规则 |
| index.md | ✅ | ✅ | ✅ | ✅ |
| log.md | ✅ | ✅ | ✅ | ✅ |
| 答案归档 | ✅ | ✅ explorations/ | ✅ save skill | ✅ --save 标志 |
| Obsidian 兼容 | ✅ | ✅ | ✅ 核心依赖 | ⚠️ 部分 |
| **独特增强** | 图谱社区检测 | PageIndex 长文档 | DragonScale 3 机制 | MCP 标准协议 |

---

## 四、企业改造成本评估

### 4.1 改造难度矩阵

| 改造目标 | llm_wiki | OpenKB | claude-obsidian | llm-wiki-compiler |
|----------|:---:|:---:|:---:|:---:|
| 添加 Web 服务端 | 🔴 高（重构 Tauri→Server） | 🟡 中（FastAPI 包装） | 🔴 高（依赖 LLM Agent） | 🟢 低（已有 Viewer 基础） |
| 多用户/认证 | 🔴 高 | 🟡 中 | 🔴 高 | 🟡 中 |
| 数据库后端 | 🟡 中 | 🟡 中 | 🔴 高 | 🟡 中 |
| 替换许可证 | 🔴 高（GPL） | 🟢 低 | 🟢 低 | 🟢 低 |
| 添加 MCP | 🟡 中 | 🔴 高 | 🟡 中 | 🟢 已实现 |
| 图谱增强 | 🟢 低（已完善） | 🔴 高（从零） | 🟡 中（依赖 Obsidian） | 🟡 中（D3 基础） |
| 整体改造成本 | 🔴 极高 | 🟡 中高 | 🔴 极高 | 🟡 中 |

### 4.2 各仓库最佳使用场景

| 仓库 | 最适合 | 不适合 |
|------|--------|--------|
| **llm_wiki** | 个人深度研究、知识图谱重度用户 | 企业多用户部署（GPL + 桌面架构） |
| **OpenKB** | 个人知识管理 + Agent 集成、长文档处理 | Web 化部署（无 UI/API） |
| **claude-obsidian** | Obsidian 生态用户、Agent 重度使用者 | 需要 Web UI 的团队场景 |
| **llm-wiki-compiler** | Agent 知识库后端、需要 MCP 的工作流 | 超大规模语料（自述 < 百份源文件） |

---

## 五、综合评估结论

### 5.1 按企业需求维度排序

#### 🥇 最接近企业级目标：llm-wiki-compiler (atomicstrata)

| 优势 | 劣势 |
|------|------|
| MIT 许可证 | 源文件有限规模（百份级） |
| 标准 MCP 协议 | 无多租户/认证 |
| 安全的本地 Web 查看器 | 图谱仅 wikilink 关系 |
| 5 LLM Provider + 3 嵌入模型 | Node.js ≥ 24 依赖 |
| 850+ 测试，代码质量高 | 快速演进（v0.7.0） |
| 增量编译 + 来源追溯 + 审查队列 | |

**改造路径**：基于现有 Viewer 添加 Fastify/Express 服务端 → 加 JWT 认证 → 加 PostgreSQL 存储 → 图谱引入 neo4j

#### 🥈 知识图谱最强：nashsu/llm_wiki

拥有四个方案中最完善的知识图谱引擎（4 信号模型 + Louvain 社区检测 + 惊奇连接 + 知识空白），但 **GPL-3.0 许可证**是最大障碍。如果企业能接受 GPL 或联系作者获取商业许可，其 API 层（内置 HTTP Server）可被直接集成。

#### 🥉 Agent 集成最优雅：claude-obsidian (AgriciDaniel)

11 个 Skill 的指令设计堪称最佳实践。其核心理念——让 AI Agent 自主维护知识库——对 Agent 友好型企业极具吸引力。但依赖 Obsidian 生态，Web 化困难。

#### 4️⃣ 理念最激进：OpenKB (VectifyAI)

"No Vector DB" 的坚持在理念上很有价值，PageIndex 长文档处理也是独有优势，但企业级特性近乎为零，且没有 Web UI 路线图。

### 5.2 推荐策略

**三个层次的企业方案建议**：

| 层次 | 方案 | 适用场景 |
|------|------|---------|
| **轻量级（立即可用）** | `llm-wiki-compiler` 直接部署 + Obsidian 作为 Web 浏览端 | 小团队 (< 10 人)、Agent 工作流 |
| **中等改造（1-3 月）** | `llm-wiki-compiler` + Fastify 服务端 + SQLite/PostgreSQL + JWT | 中型团队 (< 50 人) |
| **企业级（3-6 月）** | 参考 `llm_wiki` 的图谱引擎 + `llm-wiki-compiler` 的 MCP 架构 + Neo4j + SSO | 全企业知识平台 |

### 5.3 最终推荐

> **推荐以 `llm-wiki-compiler` 为底座进行企业级改造。**

理由：
1. **MIT 许可证**，无法律风险
2. **MCP 协议原生支持**，是四个方案中 Agent 集成最标准的
3. **已有 Web Viewer 基础**，改造为服务端成本最低
4. **代码质量最高**（TypeScript strict + 850+ 测试 + fallow 健康检查）
5. **多 Provider 支持**，可对接企业内部 LLM

核心需要补齐的部分：多租户认证、数据库持久化、图谱关系类型增强、文件共享协作。这四个模块可阶段性迭代开发。

---

*报告完成于 2026-05-22 | 调研仓库截止版本：llm_wiki v0.4.12, OpenKB 0.3.x, claude-obsidian main, llm-wiki-compiler v0.7.0*
