---
name: postgresql-code-analysis
description: 深度分析 PostgreSQL 14.4 源代码。增强版集成结构体内存布局、调用图生成、
  AST 模式搜索、静态分析、DeepWiki 交叉验证等工具。触发：分析内部机制、解释模块原理、
  生成技术文档、可视化数据结构、分析结构体布局、生成调用图、回答 PostgreSQL 核心问题。
---

# PostgreSQL 源代码分析 Skill（增强版）

## 作用域

**仅限 PostgreSQL 核心开源项目**（本仓库源码 + contrib 模块）。

拒绝以下范围的问题，并简要说明 postgres-analysis 仅处理 PostgreSQL 核心问题：
- 第三方扩展、插件、外部工具（如 pg_repack、pgvector 等非 contrib 模块）
- 客户端驱动（JDBC、psycopg2、node-postgres 等）
- 云服务（RDS、CloudSQL、Supabase 等）
- PostgreSQL 分支或衍生产品
- 生态系统工具（Patroni、PgBouncer 等独立项目）

## 核心原则

1. **中文输出**：所有说明和解释使用中文
2. **代码引用**：严格按源代码展示，格式 `src/path/file.c:行号`
3. **SVG 规范**：内联样式，禁止 `<style>` 块。详见 `references/svg-conventions.md`
4. **输出目录**：`C:/Users/swrd/Desktop/markdown/postgresql/`
5. **委托不重写**：通过 `Skill` 工具调用子 skill，不内联其指令
6. **工具感知**：使用前检查工具可用性，不可用时使用降级方案
7. **研究纪律**：
   - 本地 PostgreSQL 文档和源码是权威来源，禁止凭记忆回答可验证的问题
   - 先搜索本地代码/文档，再使用 DeepWiki 辅助验证
   - DeepWiki 是交叉验证工具，不替代本地证据
   - 不浏览网页（除非用户明确要求且仍限于 PG 核心范围）
   - 回答问题时不修改 PostgreSQL 源文件

## 工具可用性矩阵

| 工具 | 状态 | 调用方式 |
|------|------|---------|
| `ast_grep_search/replace` | **可用** | MCP 工具直接调用，language="c" |
| `struct-offset-analyzer` | **可用** | `Skill`: `skill="struct-offset-analyzer"` |
| `static-analysis` | **可用** | `Skill`: `skill="static-analysis"` |
| `grepai-trace-graph` | **可用**（方法指导） | `Skill`: `skill="grepai-trace-graph"`（CLI 不支持 C，仅用方法论） |
| `systems-programming:c-pro` | **可用** | `Agent`: `subagent_type="systems-programming:c-pro"` |
| clang-tidy | **可用** (LLVM 22.1.3) | Bash: `clang-tidy -p compile_commands.json` |
| grepai CLI | **不适用** | 仅支持 TypeScript/TSX，不支持 C 代码分析 |
| LSP (clangd) | **未安装** | 降级：Grep + Read |
| **DeepWiki MCP** | **可用（官方）** | `mcp__deepwiki__ask_question` 问答 + `mcp__deepwiki__read_wiki_contents` 页面 + `mcp__deepwiki__read_wiki_structure` 结构 |
| **Context7 MCP** | **可用** | `mcp__context7__resolve-library-id` + `mcp__context7__query-docs` — 用于查询外部库文档 |

## 触发条件

- 分析 PostgreSQL 内部机制（内存管理、事务系统、查询执行、存储引擎等）
- 解释特定模块或函数的工作原理
- 理解 PostgreSQL 源代码架构和设计模式
- 生成 PostgreSQL 技术文档
- 可视化 PostgreSQL 内部数据结构或流程
- **分析结构体内存布局和偏移计算**
- **生成函数调用图和依赖追踪**
- **运行静态分析检查代码质量**
- **回答 PostgreSQL 核心问题**（行为、配置、SQL、性能、存储、规划器、执行器、复制等）

## 路由表

根据用户意图，加载对应工作流文件并执行：

| 用户意图 | 路由到 |
|---------|--------|
| **PostgreSQL 问答**（行为、原理、配置、SQL、性能等） | `workflows/pgfaq-answer.md` |
| 组件深度分析（内存、事务、执行器等） | `workflows/component-deep-dive.md` |
| "分析结构体内存布局" / 偏移 / 字节对齐 | `workflows/struct-layout-analysis.md` |
| "生成调用图" / 追踪调用链 / 谁调用了 | `workflows/call-graph-trace.md` |
| "静态分析" / 代码质量 / clang-tidy | `workflows/static-analysis-run.md` |

**路由规则**：
- 明确单一意图 → 直接路由到对应工作流
- 综合分析请求 → 加载 `component-deep-dive.md`，它会按需分发到其他工作流
- 多个意图同时出现 → 依次加载对应工作流
- **问答类请求**（"为什么..."、"怎么..."、"XXX 是什么"） → 优先路由到 `pgfaq-answer.md`

## 子 Skill 调用指南

```
通过 Skill 工具调用，传递函数名和上下文：

struct-offset-analyzer  → C 结构体内存布局计算
  调用：Skill(skill="struct-offset-analyzer")
  提供：结构体名称、源文件位置、目标平台（64-bit）

static-analysis         → 静态代码分析指导
  调用：Skill(skill="static-analysis")
  提供：目标文件/目录、检查类别

grepai-trace-graph      → 调用图方法指导（CLI 不支持 C，仅提供方法论）
  调用：Skill(skill="grepai-trace-graph")
  提供：目标函数名、追踪深度、输出格式
```

**重要**：子 skill 的 SKILL.md 提供 HOW（具体方法），本路由器提供 WHEN（何时调用）。永远不要将子 skill 的指令内联到工作流中。

## DeepWiki 使用指南

官方 DeepWiki MCP server 提供 3 个工具，核心仓库为 `swrd/pg14`：

### 工具 1: ask_question（首选 — 精准问答）

```
mcp__deepwiki__ask_question(
    repoName="swrd/pg14",
    question="具体的 PostgreSQL 问题"
)
```

**适用场景**：
- 交叉验证分析结论的正确性（最精准）
- 快速获取某个子系统的架构概述
- 补充本地源码中不明显的设计意图说明
- 问答工作流中的验证循环（详见 workflows/pgfaq-answer.md）

**提问技巧**：
- 聚焦具体问题，避免宽泛提示
- 包含足够上下文让 AI 理解问题范围
- 示例：`"Explain the VFD mechanism in fd.c, including LRU ring structure and FileAccess flow"`

### 工具 2: read_wiki_contents（页面浏览）

```
mcp__deepwiki__read_wiki_contents(repoName="swrd/pg14")
```

获取仓库的完整文档内容。适合浏览整体知识结构。

### 工具 3: read_wiki_structure（结构索引）

```
mcp__deepwiki__read_wiki_structure(repoName="swrd/pg14")
```

获取仓库文档的主题列表（目录），用于定位感兴趣的专题页面。

### 使用优先级

```
1. ask_question  — 精准验证具体论点（首选）
2. read_wiki_structure → 定位 → read_wiki_contents — 系统性浏览
```

### 限制
- DeepWiki 结果不替代本地源码验证
- DeepWiki 返回的信息必须与本地代码对照确认
- 如有冲突，以本地源码和 doc/src/sgml 文档为准
- swrd/pg14 的索引可能不覆盖所有子系统（未索引时 ask_question 仍可基于代码回答）

## 工作流索引

| 文件 | 说明 | 步骤数 |
|------|------|--------|
| `workflows/pgfaq-answer.md` | **问答工作流**：7 步结构化回答 + DeepWiki 验证循环 | 7 步 |
| `workflows/component-deep-dive.md` | 主工作流：6步深度分析，整合所有工具 | 6 步，3 轨道 |
| `workflows/struct-layout-analysis.md` | 结构体内存布局：定位→计算→SVG→文档 | 6 步 |
| `workflows/call-graph-trace.md` | 调用图追踪：定位→追踪（双层）→SVG→文档 | 6 步 |
| `workflows/static-analysis-run.md` | 静态分析：预检→编译库→分析→报告 | 条件性 |

## 引用文件索引

| 文件 | 说明 |
|------|------|
| `references/source-file-map.md` | PostgreSQL 组件→源文件路径映射表 + SGML 文档路径 |
| `references/tool-selection-guide.md` | 场景→工具选择决策树 + ast_grep 模式速查 |
| `references/svg-conventions.md` | SVG 生成规范（流程图、架构图、内存布局图、调用关系图等） |
| `references/install-guide.md` | 安装指南：MCP、外部工具、子 skill 的依赖清单和安装说明 |

## 快速命令

```
# PostgreSQL 问答
为什么 PostgreSQL 使用 MVCC 而不是锁来实现并发控制？
PostgreSQL 的 WAL 机制是如何保证数据不丢失的？
shared_buffers 参数应该设置多大？

# 组件深度分析
分析 PostgreSQL 缓冲区管理器，包括 BufferDesc 结构体内存布局和 ReadBuffer 调用图

# 结构体内存布局
分析 BufferDesc 结构体的内存布局，计算各字段偏移和总大小

# 调用图生成
追踪 ExecProcNode 的完整调用链，生成调用关系图

# 静态分析
对 src/backend/storage/buffer/ 目录做静态代码分析

# 算法分析
解释 PostgreSQL B-tree 索引的页面分裂算法
```

---

**版本**：3.1
**更新日期**：2026-04-23
**适用 PostgreSQL 版本**：14.4
**增强集成**：struct-offset-analyzer, grepai-trace-graph, static-analysis, ast_grep, **DeepWiki 官方 MCP (ask_question + read_wiki_structure + read_wiki_contents)**
