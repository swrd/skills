# PostgreSQL 核心问答工作流

回答关于 PostgreSQL 核心数据库行为、内部机制、SQL、配置、管理、性能、存储、规划器、执行器、复制等问题的结构化流程。

## 触发条件

- "为什么 PostgreSQL ..."
- "PostgreSQL 的 XXX 是怎么工作的"
- "XXX 参数应该怎么设置"
- "XXX 和 YYY 有什么区别"
- "如何排查 XXX 问题"
- 任何关于 PostgreSQL 核心行为的 "是什么/为什么/怎么做" 问题

## 作用域检查

在开始研究前，确认问题属于 PostgreSQL 核心范围：

**接受**：查询处理、存储引擎、事务/锁、MVCC、WAL、缓冲区管理、索引实现、查询优化、执行器、进程架构、内存管理、配置参数、SQL 行为等。

**拒绝**：第三方扩展（非 contrib）、客户端驱动、云服务、独立工具（PgBouncer、Patroni 等）、分支产品。如超出范围，简要说明并建议用户查阅相关项目文档。

## 工作流

### Step 1: 确认环境

验证当前工作目录是 PostgreSQL 源码检出：

```
检查项:
1. 预期路径是否存在（src/backend/, src/include/ 等）
2. doc/src/sgml 是否存在（本地文档目录）
3. 可选: git rev-parse --show-toplevel 确认根目录
```

如果当前目录不是 PostgreSQL 源码树，请用户提供正确路径。

### Step 2: 分类问题

在开始研究前，明确问题的性质和可能涉及的代码区域：

```
1. 问题类型分类:
   - 行为解释型: "为什么 XXX 会这样"
   - 机制原理型: "XXX 是怎么实现的"
   - 配置调优型: "XXX 参数怎么设"
   - 问题排查型: "如何诊断 XXX"
   - 对比分析型: "XXX 和 YYY 的区别"

2. 识别可能的文档区域（doc/src/sgml/）:
   - 通用: doc/src/sgml/*.sgml
   - 配置: doc/src/sgml/config.sgml
   - 索引: doc/src/sgml/indexam.sgml, doc/src/sgml/indices.sgml
   - 事务: doc/src/sgml/mvcc.sgml
   - 存储: doc/src/sgml/storage.sgml
   - WAL: doc/src/sgml/wal.sgml
   - 性能: doc/src/sgml/performance.sgml
   - 并行: doc/src/sgml/parallel.sgml
   - SQL: doc/src/sgml/sql.sgml, doc/src/sgml/queries.sgml

3. 识别可能的源码区域（src/backend/, src/include/）:
   参考 references/source-file-map.md 定位具体文件
```

### Step 3: 搜索本地文档和源码

**优先搜索本地资源，这是权威来源。**

#### 3.1 搜索 SGML 文档

```
Grep: 在 doc/src/sgml/ 中搜索关键词
  - 使用有针对性的搜索词
  - 读取最相关的文档段落
  - 捕获精确的文件路径和上下文
```

#### 3.2 搜索源代码

```
按优先级搜索:
1. 头文件 (src/include/): 理解数据结构和 API
2. 核心实现 (src/backend/): 分析具体逻辑
3. README 文件: 获取组件设计概述

使用 Grep 进行针对性搜索，Read 读取必要的最小片段。
优先使用本地 PG 文档和代码，而非记忆。
对重要声明，记录精确的文件路径和行号。
```

#### 3.3 增强：AST 模式搜索

对于涉及特定代码模式的问题，使用 `ast_grep_search`：

```
# 查找相关结构体
ast_grep_search: "struct $NAME { $$$ }", language="c"

# 查找相关函数
ast_grep_search: "$RET $FUNC($$$ARGS) { $$$BODY }", language="c"

# 查找特定调用模式
ast_grep_search: "$FUNC($$$ARGS)", language="c"
```

#### 3.4 I/O 路径追踪（机制原理型问题强制）

当问题涉及**数据流向、资源分配路径、缓存机制**时，必须追踪调用链到分支决策点：

```
追踪规则:
1. 从上层入口函数开始（如 ReadBufferExtended、heap_insert）
2. 逐层 Read 到包含条件分支的函数
3. 记录分支条件（宏/条件判断）和各分支目标
4. 标注分支点的文件:行号

必须追踪的主题:
- 缓冲区/缓存路径: shared vs local 的分支判断
- WAL 路径: 需要写 WAL vs 跳过 WAL 的条件
- 锁路径: LWLock vs 无锁 vs 自旋锁的选择
- 存储 I/O: smgr 层分发逻辑
- 内存分配: 不同分配器的选择条件
```

### Step 4: 查询 DeepWiki 辅助

使用 DeepWiki MCP 获取 swrd/pg14 仓库的知识作为交叉参考。

```
首选: ask_question 精准问答
   mcp__deepwiki__ask_question(
       repoName="swrd/pg14",
       question="具体的 PostgreSQL 问题，包含足够上下文"
   )

备选: 浏览文档结构
   mcp__deepwiki__read_wiki_structure(repoName="swrd/pg14")  → 定位主题
   mcp__deepwiki__read_wiki_contents(repoName="swrd/pg14")   → 阅读内容

注意:
- 向 DeepWiki 提出聚焦的问题，而非宽泛的通用提示
- ask_question 是首选工具，返回精准的 AI 驱动回答
- 将 DeepWiki 视为辅助验证工具，交叉对照本地代码和文档
- DeepWiki 不作为唯一信息来源
```

### Step 5: 综合回答

#### 5.1 源码引用强制规则

**回答中每个行为声明的引用要求：**

| 声明类型 | 必须引用级别 | 说明 |
|---------|------------|------|
| 运行时行为 | **L1**: `src/path/file.c:行号` | 直接引用实现代码 |
| 数据流向/路径 | **L1**: 分支决策点代码 | 追踪到 if/switch 分支 |
| 设计意图 | **L2**: README 或 SGML 文档 | 可接受文档引用 |
| 补充说明 | **L3**: DeepWiki | 需 L1/L2 确认 |

**禁止**：无源码引用的行为声明。无法找到源码支撑时必须标注为推测。

#### 5.2 撰写草稿

基于本地搜索（Step 3）和 DeepWiki 辅助（Step 4）的结果，按以下结构撰写回答：

```markdown
# <问题标题>

## 结论

<直接回答问题，2-5 句话给出明确结论>

## 原理与证据

<基于 PostgreSQL 文档和源码的详细解释>
- 包含代码/文档引用，标注本地路径和行号
- 区分已确认的事实和推论
- 避免未经检查的源码树或文档支持的建议、传言和版本声明
- **涉及数据流向的声明必须包含分支决策点的源码引用**

### 关键分支路径（如适用）

<当回答涉及"经过X路径"/"使用Y机制"时，必须展示分支决策逻辑>
```
调用链: FuncA -> FuncB -> FuncC
                                   ↳ if (条件) → 路径X  [src/path/file.c:行号]
                                   ↳ else      → 路径Y  [src/path/file.c:行号]
```

## 实践建议

<相关的命令、SQL 示例、配置说明、诊断步骤>
- 仅当与问题相关时才包含此节

## 边界与版本说明

<适用范围、注意事项、源码树/版本限制、不确定性>
- 明确标注哪些结论有源码支撑，哪些是基于推断

## 参考来源

- `doc/src/sgml/...` (本地文档)
- `src/backend/...` (源码)
- `src/include/...` (头文件)
- DeepWiki MCP: `swrd/pg14` 交叉验证
```

#### 5.3 综合规则

- 使用中文撰写（除非用户要求其他语言）
- 回答必须实用且有源码依据
- 包含代码/文档引用，使用本地路径 + 行号格式
- 区分已确认事实和推断
- 避免无支撑的建议、传言和版本声明

### Step 6: 验证和修订（DeepWiki 验证循环）

**这是确保回答正确性的关键步骤。**

```
验证循环:

1. 正面验证 — 将草稿的核心论点通过 ask_question 发送给 DeepWiki:
   mcp__deepwiki__ask_question(
       repoName="swrd/pg14",
       question="针对草稿中的具体论点提问，例如 'VFD 的 FileAccess 函数是否有三种路径分支？'"
   )

2. 对抗性验证（v3.2 新增）— 构造可能否定结论的问题:
   mcp__deepwiki__ask_question(
       repoName="swrd/pg14",
       question="用否定/疑问句式验证关键声明:
         例1: 声明'临时表使用shared buffers'
              → 问 'Do temp tables use local buffers instead?'
         例2: 声明'数据经过X路径'
              → 问 'Is there an alternative path through Y?'"
   )

3. 对 DeepWiki 返回的每一条修正:
   - 重新检查本地源码和文档进行确认
   - 如果修正有据可依 → 修订回答
   - 如果修正与本地源码冲突 → 以本地源码为准，记录差异

4. 负面源码验证（v3.2 新增）:
   - 对每个关键行为声明，Grep 搜索可能推翻该声明的代码
   - 例: 声明"使用shared buffers" → Grep "LocalBuffer|SmgrIsTemp|local_buf"
   - 如发现矛盾 → 回到 Step 3 重新追踪，修正结论

5. 重复验证循环:
   - 直到没有重大的正确性问题
   - 或将剩余的不确定性在"边界与版本说明"中明确记录

验证要点:
- 关键数据结构定义和字段说明是否与源码一致
- 函数调用关系和执行流程是否正确
- 配置参数的默认值和行为是否准确
- 版本特性说明是否适用于当前源码版本（14.4）
- I/O 路径和资源分配是否追踪到了分支决策点
```

### Step 7: 保存最终回答

```
1. 确保输出目录存在:
   C:/Users/用户名/Desktop/markdown/postgresql/

2. 生成文件名:
   - 从问题主题派生，使用简洁的文件系统安全名称
   - 示例: mvcc-mechanism.md, wal-guarantee.md, shared-buffers-tuning.md

3. 保存路径:
   C:/Users/用户名/Desktop/markdown/postgresql/<topic>.md

4. 文件内容:
   - 包含完整的 Step 5 答案格式
   - 包含简短的参考来源列表（本地文档/源码 + DeepWiki 检查）

5. 告知用户保存的文件路径
```

## 研究纪律（强制）

| 规则 | 说明 |
|------|------|
| 本地优先 | 本地 PG 文档和源码是权威来源 |
| 禁止凭记忆 | 可通过代码/文档验证的内容，不从记忆回答 |
| DeepWiki 不替代本地 | DeepWiki 作为辅助验证，不替代本地证据 |
| 不修改源码 | 回答问题时禁止修改 PostgreSQL 源文件 |
| 不浏览网页 | 除非用户明确要求且仍在 PG 核心范围内 |
| 区分事实与推断 | 回答中明确标注哪些是确认的事实、哪些是推断 |
| 版本敏感 | 注意当前源码版本（14.4），不要引用更高版本特性 |
| **行为声明必须有源码引用** | **v3.2 新增：任何运行时行为声明必须附带 L1（src/path:line）引用** |
| **追踪到分支决策点** | **v3.2 新增：I/O 路径、资源分配类问题必须追踪到 if/switch 分支** |
| **负面验证** | **v3.2 新增：对关键结论主动搜索可能推翻该结论的代码路径** |

## 常见问题类型速查

| 问题类型 | 首先搜索 | 然后搜索 | DeepWiki 验证重点 | 负面验证搜索 |
|---------|---------|---------|-----------------|-------------|
| "为什么 XXX 这么慢" | `doc/src/sgml/performance.sgml` | 相关源码模块 README | 性调优建议的准确性 | 替代执行路径 |
| "XXX 参数什么意思" | `doc/src/sgml/config.sgml` | `src/backend/utils/misc/guc.c` | 参数默认值和适用场景 | 参数的运行时条件分支 |
| "XXX 报错怎么处理" | `doc/src/sgml/errcodes.sgml` | 错误处理相关源码 | 错误恢复流程 | 其他可能的错误路径 |
| "XXX 和 YYY 区别" | 两者相关的 sgml 文档 | 两者的头文件和实现 | 设计意图和权衡 | 共享代码路径（可能并非完全不同） |
| "XXX 内部怎么实现" | 组件 README | `src/backend/` + `src/include/` | 架构概述和关键流程 | 替代实现、条件分支 |
| "如何配置 XXX" | `doc/src/sgml/config.sgml` | GUC 参数定义源码 | 推荐值的适用条件 | 配置生效的运行时条件 |
