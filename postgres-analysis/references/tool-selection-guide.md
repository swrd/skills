# 工具选择决策指南

本文件指导在不同分析场景下选择最合适的工具。

## 工具可用性矩阵

| 工具 | 当前状态 | 调用方式 | 适用语言 |
|------|---------|---------|---------|
| `ast_grep_search` | **可用** | MCP 工具直接调用 | C |
| `ast_grep_replace` | **可用** | MCP 工具直接调用 | C |
| `struct-offset-analyzer` skill | **可用** | `Skill` 工具: `skill="struct-offset-analyzer"` | C |
| `static-analysis` skill | **可用** | `Skill` 工具: `skill="static-analysis"` | C/C++ |
| `grepai-trace-graph` skill | **可用** | `Skill` 工具: `skill="grepai-trace-graph"` | 通用 |
| `systems-programming:c-pro` agent | **可用** | `Agent` 工具: `subagent_type="systems-programming:c-pro"` | C |
| clang-tidy | **可用** (LLVM 22.1.3) | Bash: `clang-tidy -p compile_commands.json` | C/C++ |
| grepai CLI | **不适用** | 仅支持 TypeScript/TSX，PG 分析使用 ast_grep + Grep 替代 | — |
| LSP (clangd) | **未安装** | 降级到 Grep + Read | — |
| **DeepWiki MCP** | **可用（官方）** | `mcp__deepwiki__ask_question` 问答 + `read_wiki_structure` 索引 + `read_wiki_contents` 内容 | 通用 |
| **Context7 MCP** | **可用** | `mcp__context7__resolve-library-id` + `mcp__context7__query-docs` | 通用 |

## 场景→工具映射

### 源码定位

| 场景 | 首选工具 | AST 模式 / Grep 模式 | 降级方案 |
|------|---------|---------------------|---------|
| 查找结构体定义 | `ast_grep_search` | `"struct $NAME { $$$ }"` (language: "c") | `Grep "struct NAME {"` |
| 查找函数定义 | `ast_grep_search` | `"$RET $NAME($$$ARGS) { $$$BODY }"` (language: "c") | `Grep "^FunctionName("` |
| 查找宏定义 | `Grep` | `"#define MACRO_NAME"` | — |
| 查找 typedef | `ast_grep_search` | `"typedef $TYPE $NAME"` (language: "c") | `Grep "typedef.*NAME"` |
| 查找枚举定义 | `Grep` | `"enum NAME {"` | — |

### 调用链追踪

| 场景 | 首选工具 | 说明 | 降级方案 |
|------|---------|------|---------|
| 完整调用图（grepai CLI 不支持 C） | `ast_grep_search` + `Grep` | 详见 call-graph-trace 工作流 | 手动 Read 验证 |
| 查找函数的所有调用者 | `Grep` | `"FunctionName("` 全目录搜索 | — |
| 查找函数内调用了谁 | `ast_grep_search` | 在函数范围内搜索 `"$FUNC($$$ARGS)"` | Read 函数体 + Grep |
| 查找函数指针调用 | `Grep` + `Read` | 搜索赋值和使用模式，需人工判断 | — |

### 数据结构分析

| 场景 | 首选工具 | 说明 |
|------|---------|------|
| 结构体字段列表 | `Read` 头文件 | 直接读取定义 |
| 结构体内存布局 / 偏移计算 | `struct-offset-analyzer` skill | 通过 Skill 工具调用 |
| 查找结构体被谁引用 | `Grep` | `"StructName"` 搜索 |
| 结构体间的关系（嵌入/指针） | `ast_grep_search` | `"$STRUCT *$VAR"` 或 `"$STRUCT $VAR"` |

### 代码模式分析

| 场景 | 首选工具 | AST 模式 |
|------|---------|---------|
| 查找锁获取/释放模式 | `ast_grep_search` | `"LockAcquire($$$ARGS)"` |
| 查找内存上下文使用 | `ast_grep_search` | `"MemoryContextAlloc($$$ARGS)"` |
| 查找错误处理模式 | `Grep` | `"PG_TRY"` / `"PG_CATCH"` / `"ereport"` |
| 查找条件编译块 | `Grep` | `"#ifdef"` / `"#ifndef"` / `"#if"` |
| 查找回调/函数指针赋值 | `ast_grep_search` | `"$VAR = $FUNC"` |
| 查找 palloc 调用 | `ast_grep_search` | `"palloc($$$ARGS)"` |

### 静态分析

| 场景 | 首选工具 | 说明 | 降级方案 |
|------|---------|------|---------|
| 代码质量检查（工具已安装） | `static-analysis` skill | clang-tidy / cppcheck | — |
| 代码模式审查（工具未安装） | `ast_grep_search` | 搜索已知的反模式 | 手动 Read 审查 |
| 缓冲区溢出风险 | `ast_grep_search` | `"memcpy($DST, $SRC, $SIZE)"` 检查 | — |
| 空指针风险 | `ast_grep_search` | `"NULL"` + 上下文分析 | — |

### 文档查阅（新增）

| 场景 | 首选工具 | 说明 | 降级方案 |
|------|---------|------|---------|
| PG 官方文档查询 | `Grep` in `doc/src/sgml/` | 本地 SGML 文档是权威来源 | — |
| 交叉验证结论 | `DeepWiki MCP` | `mcp__deepwiki__ask_question(repoName="swrd/pg14", question="...")` | 跳过验证 |
| 获取高层架构概述 | `DeepWiki MCP` | `mcp__deepwiki__ask_question(repoName="swrd/pg14", question="...")` | 读取本地 README |
| 补充设计意图说明 | `DeepWiki MCP` | 查询特定主题页面 | 基于 README 和注释推断 |
| 外部库文档查询 | `Context7 MCP` | 用于查询非 PG 的外部依赖库文档 | Web 搜索 |

### 跨文件分析

| 场景 | 首选工具 | 说明 |
|------|---------|------|
| 全局符号搜索 | `Grep` | 按名称搜索函数/变量 |
| 按模块搜索 | `Grep` (path 参数) | 限定搜索目录范围 |
| 头文件引用追踪 | `Grep` | `"#include.*header.h"` |
| 类型定义追踪 | `ast_grep_search` | `"typedef $OLD $NEW"` |

## ast_grep_search 常用模式速查

```
# C 语言模式（language 参数始终为 "c"）

# 查找结构体定义
"struct $NAME { $$$FIELDS }"

# 查找函数定义
"$RET $FUNC($$$ARGS) { $$$BODY }"

# 查找函数调用
"$FUNC($$$ARGS)"

# 查找赋值
"$VAR = $EXPR"

# 查找 if 语句
"if ($COND) { $$$BODY }"

# 查找 return 语句
"return $EXPR"

# 查找类型声明
"$TYPE *$VAR"

# 查找 for 循环
"for ($INIT; $COND; $UPDATE) { $$$BODY }"
```

## 降级策略总览

```
Tier 0: 本地文档（doc/src/sgml/） — 权威来源，优先搜索
  ↓ 未找到相关文档时
Tier 1: AST 工具（ast_grep_search） — 最精确，当前环境可用
  ↓ 不可用时
Tier 2: 文本搜索（Grep） — 较快但可能误匹配
  ↓ 不确定时
Tier 3: 直接阅读（Read） — 最慢但最准确，用于验证
  ↓ 需要交叉验证时
Tier 4: DeepWiki MCP（swrd/pg14） — 辅助验证，不替代本地源码
```
