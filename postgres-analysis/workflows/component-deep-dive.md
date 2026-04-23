# 组件深度分析工作流（增强版）

PostgreSQL 子系统的完整深度分析，整合所有可用工具。这是主要的工作流。

## 触发条件

- "分析 PostgreSQL 的 XXX 子系统"
- "解释 XXX 的工作原理"
- "追踪查询从解析到执行的完整流程"
- "分析特定函数的实现"

## 工作流

### Step 1: 理解分析目标

向用户确认以下信息（如果未明确提供）：

1. **分析目标**：具体组件或功能
   - 参考 `references/source-file-map.md` 定位
   - 常见：内存管理、事务系统、查询执行器、优化器、缓冲区管理、存储管理器等

2. **深度级别**：
   - 概览：架构 + 核心数据结构
   - 标准：+ 关键函数 + 流程分析
   - 深度：+ 所有数据结构 + 完整调用链 + 算法细节

3. **关注重点**（可多选）：
   - 数据结构（含内存布局）
   - 函数调用链（含调用图）
   - 算法/逻辑
   - 架构设计
   - 代码质量（含静态分析）

4. **增强分析选项**（新增）：
   - 是否需要结构体内存布局分析？ → 分发到 `struct-layout-analysis`
   - 是否需要完整调用图？ → 分发到 `call-graph-trace`
   - 是否需要静态代码分析？ → 分发到 `static-analysis-run`

### Step 2: 源代码探索

#### 2.1 定位相关源文件

读取 `references/source-file-map.md`，确定目标组件的：
- 源文件目录
- 核心头文件
- README 文件（如存在）

#### 2.2 读取关键文件（按优先级）

```
优先级 1: README 文件
  → 获取高层架构概述和设计决策

优先级 2: 头文件
  → 理解数据结构定义和 API 接口
  → 使用 ast_grep_search 快速发现关键类型定义

优先级 3: 核心实现文件
  → 分析关键算法和逻辑
```

#### 2.3 增强的代码定位

结合多种工具高效定位代码：

| 任务 | 首选工具 | 降级方案 |
|------|---------|---------|
| 查找结构体定义 | `ast_grep_search` `"struct $NAME { $$$ }"` | `Grep "struct NAME {"` |
| 查找函数签名 | `ast_grep_search` `"$RET $FUNC($$$ARGS)"` | `Grep "^FuncName"` |
| 查找宏定义 | `Grep "#define MACRO"` | — |
| 查找所有引用 | `Grep "SymbolName"` 全目录 | — |

### Step 3: 代码分析（三轨道）

根据用户选择的关注重点，执行一条或多条分析轨道。

#### Track A: 数据结构分析

1. 列出所有关键结构体，标注定义位置
2. 分析字段用途和关系
3. **增强**：对核心结构体，调用 `struct-offset-analyzer` skill 计算内存布局

   ```
   Skill: skill="struct-offset-analyzer"
   ```

4. **增强**：使用 `ast_grep_search` 查找结构体的所有使用点

   ```
   ast_grep_search: "$STRUCT *$VAR" 或 "$STRUCT $VAR"
   language: "c"
   ```

5. 输出：结构体字段表 + 内存偏移表 + SVG 内存布局图

#### Track B: 函数调用链分析

1. 识别核心函数，标注定义位置
2. **增强**：使用 `call-graph-trace` 工作流生成完整调用图

   如果仅需简单追踪：
   ```
   ast_grep_search: "$FUNC($$$ARGS)" → 查找 callees
   Grep: "FuncName(" → 查找 callers
   ```

   如果需要完整调用图：
   → 分发到 `call-graph-trace` 工作流

3. **增强**：识别特殊调用模式（函数指针、回调、异常跳转等）
4. 输出：调用树文本 + SVG 调用关系图

#### Track C: 算法/逻辑分析

1. Read 核心函数实现
2. 分析算法步骤和决策逻辑
3. **增强**：使用 `ast_grep_search` 进行模式化分析

   ```
   查找条件分支: "if ($COND) { $$$BODY }"
   查找循环结构: "for ($INIT; $COND; $UPDATE) { $$$BODY }"
   查找错误处理: "ereport(ERROR, $$$ARGS)"
   ```

4. **增强**：对复杂逻辑，可派遣 `systems-programming:c-pro` agent 进行专家分析

   ```
   Agent: subagent_type="systems-programming:c-pro"
   ```

5. 输出：算法描述 + 复杂度分析 + 流程图

### Step 4: 可视化设计

根据分析内容选择图表类型。SVG 规范详见 `references/svg-conventions.md`。

| 内容类型 | 图表类型 | 说明 |
|----------|---------|------|
| 数据结构层次 | 分层图 | 现有 |
| 函数调用流程 | 流程图 | 现有 |
| 算法决策逻辑 | 决策树 | 现有 |
| 内存布局 | 内存布局图 | **新增** — 参考 struct-layout-analysis |
| 调用关系 | 调用关系图 | **新增** — 参考 call-graph-trace |
| 架构关系 | 架构图 | 现有 |
| 状态转换 | 状态机图 | 现有 |

SVG 保存路径：
```
C:/Users/swrd/Desktop/markdown/postgresql/<主题>/
  ├── diagram-architecture.svg
  ├── diagram-flow.svg
  ├── struct-<名称>.svg     (如有结构体分析)
  └── callgraph-<函数>.svg  (如有调用图)
```

### Step 5: 文档生成

#### 文档结构

```markdown
# [分析主题]

## 概述
[简要说明组件作用和在系统中的位置]

## 架构概述
[高层架构描述]

## 核心数据结构
### 结构体1: [名称]
- **定义位置**: `src/path/to/file.h:行号`
- **设计意图**: [说明]
- **字段说明**: [表格]

### 内存布局（如适用）
![内存布局](struct-<名称>.svg)

## 关键函数/机制
### [函数名]
- **源文件**: `src/path/to/file.c:行号`
- **功能**: [说明]
- **调用关系**: caller -> callee

### 调用关系图（如适用）
![调用图](callgraph-<函数>.svg)

## 算法/逻辑分析（如适用）
[详细描述]

## 静态分析结果（如适用）
[检查结果摘要]

## 技术细节
[设计权衡、性能考虑等]

## 参考资料
- PostgreSQL 源代码版本：14.4
- 相关 README: [路径]
```

保存路径：
```
C:/Users/swrd/Desktop/markdown/postgresql/<主题>.md
或
C:/Users/swrd/Desktop/markdown/postgresql/<主题>/index.md
```

### Step 6: DeepWiki 交叉验证

**将分析结论与 DeepWiki (swrd/pg14) 进行交叉验证。**

```
验证流程:

1. 通过 ask_question 将关键论点发送给 DeepWiki:
   mcp__deepwiki__ask_question(
       repoName="swrd/pg14",
       question="针对分析中的具体论点提问，例如 'BufferDesc 结构体的 freeNext 字段用途是什么？'"
   )

2. 对比验证:
   - 架构描述是否与 DeepWiki 一致
   - 关键流程是否与 DeepWiki 描述吻合
   - 数据结构关系是否正确

3. 处理差异:
   - DeepWiki 修正有据 → 更新分析内容
   - DeepWiki 与本地源码冲突 → 以本地源码为准，记录差异
   - 重复验证直到无重大正确性问题
```

### Step 7: 质量检查

**原有检查项：**
- [ ] 所有数据结构标注了源文件位置和行号
- [ ] 代码片段严格按源代码展示
- [ ] SVG 使用内联样式，无 `<style>` 块
- [ ] 图表元素在 viewBox 内，无截断
- [ ] 技术数值已验证源码

**新增检查项：**
- [ ] 结构体偏移计算已验证（如执行了 struct 分析）
- [ ] 调用图准确性已抽查验证（如生成了调用图）
- [ ] 函数指针调用已标注为间接调用
- [ ] 工具可用性限制已在文档中标注（如适用）
- [ ] 文档和 SVG 已保存到正确路径
- [ ] DeepWiki 交叉验证已完成（关键结论已对比）
