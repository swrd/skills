# 安装指南

postgres-analysis skill 的完整依赖清单和安装说明。

## 最小安装（核心功能可用）

### 1. oh-my-claudecode MCP（必需）

提供 `ast_grep_search` / `ast_grep_replace`，是 skill 的核心代码智能工具。

```bash
# 安装 oh-my-claudecode
omc setup
```

### 2. DeepWiki MCP（必需）

提供仓库知识查询，用于交叉验证分析结论。使用官方远程 MCP server，提供 3 个工具：
- `ask_question` — 精准 AI 问答（首选）
- `read_wiki_structure` — 文档主题索引
- `read_wiki_contents` — 完整文档内容

在 Claude Code settings 中添加 DeepWiki MCP server：
```json
{
  "mcpServers": {
    "deepwiki": {
      "type": "url",
      "url": "https://mcp.deepwiki.com/mcp",
      "description": "DeepWiki official MCP server"
    }
  }
}
```

> skill 中配置的分析仓库为 `swrd/pg14`（PostgreSQL 14 注释增强版）。

## 推荐安装（增强体验）

### 3. Context7 MCP（可选）

提供外部库文档查询能力。

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@anthropic/context7-mcp"]
    }
  }
}
```

### 4. struct-offset-analyzer skill（推荐）

C 结构体内存布局偏移计算，用于 `struct-layout-analysis` 工作流。

安装到 `~/.claude/skills/struct-offset-analyzer/SKILL.md` 或项目 `.claude/skills/` 下。

### 5. clang-tidy（推荐）

C/C++ 静态代码分析工具，用于 `static-analysis-run` 工作流。

```bash
# macOS
brew install llvm

# Ubuntu/Debian
sudo apt install clang-tidy

# Windows
# 安装 LLVM: https://releases.llvm.org/
```

## 可选安装

### 6. static-analysis skill（可选）

静态分析指导 skill，配合 clang-tidy 使用。

### 7. grepai-trace-graph skill（可选）

调用图生成方法论。注意：grepai CLI 仅支持 TypeScript/TSX，C 代码分析使用 ast_grep + Grep 手动构建。

### 8. cppcheck（可选）

独立的 C/C++ 静态分析工具，无需编译数据库。

```bash
# macOS
brew install cppcheck

# Ubuntu/Debian
sudo apt install cppcheck
```

## 依赖矩阵

| 工具/MCP | 必需? | 影响的工作流 | 缺失降级方案 |
|---------|-------|------------|------------|
| oh-my-claudecode (ast_grep) | **必需** | 所有工作流 | 无 |
| DeepWiki MCP | **必需** | pgfaq-answer, component-deep-dive | 跳过交叉验证 |
| Context7 MCP | 可选 | 查询外部库文档时 | Web 搜索 |
| struct-offset-analyzer | 推荐 | struct-layout-analysis | 手动计算偏移 |
| clang-tidy | 推荐 | static-analysis-run | ast_grep 模式匹配 |
| static-analysis skill | 可选 | static-analysis-run | 直接使用 clang-tidy |
| grepai-trace-graph skill | 可选 | call-graph-trace | ast_grep + Grep 手动构建 |
| cppcheck | 可选 | static-analysis-run | ast_grep 模式匹配 |

## PostgreSQL 源码要求

本 skill 需要在 PostgreSQL 源码目录中使用。确认以下路径存在：

```
src/backend/       — 后端核心源码
src/include/       — 头文件
doc/src/sgml/      — 官方 SGML 文档（问答工作流必需）
```

适用版本：PostgreSQL 14.x（以 14.4 为主）。
