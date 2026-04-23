# 静态分析工作流

对 PostgreSQL 代码运行静态分析工具，检测潜在问题。

## 触发条件

- "对 XXX 做静态分析"
- "检查 XXX 的代码质量"
- "运行 clang-tidy / cppcheck"
- "扫描 XXX 的潜在 bug"

## 前置条件检查

**此工作流是条件性的** — 依赖外部工具安装状态。

```
检查步骤:
1. which clang-tidy    → 是否安装？
2. which cppcheck      → 是否安装？
3. compile_commands.json 是否存在？
```

### 如果工具未安装

告知用户当前状态并提供降级方案：

```
clang-tidy / cppcheck 未安装。可选方案：

方案 A（推荐）：使用 ast_grep 模式匹配进行代码审查
  - 搜索已知的反模式和风险模式
  - 无需安装任何工具，立即可用

方案 B：安装工具后重试
  - clang-tidy: 包含在 LLVM/Clang 安装中
  - cppcheck: 独立安装，无需编译数据库
  - compile_commands.json: 使用 Bear 或 compiledb 生成

方案 C：跳过静态分析，继续其他工作流
```

如用户选择方案 A，执行下方 "降级：模式匹配审查" 部分。
如用户选择方案 B，提供安装指导后停止。

## 工作流（工具已安装时）

### Step 1: 生成编译数据库

PostgreSQL 使用 Autoconf/MSVC，非 CMake，生成 `compile_commands.json` 需额外步骤：

**Linux/macOS (Bear):**
```bash
./configure
bear -- make -j$(nproc)
```

**Linux/macOS (compiledb):**
```bash
pip install compiledb
compiledb -n make -j$(nproc)
```

**Windows (MSVC):**
需要手动生成或使用 Visual Studio 的编译数据库功能。

### Step 2: 调用 static-analysis skill

```
Skill: skill="static-analysis"
参数：提供目标文件/目录和编译数据库路径
```

推荐的 clang-tidy 检查类别（PostgreSQL 适用）：

| 类别 | 说明 | PG 适用度 |
|------|------|----------|
| `bugprone-*` | 常见 bug 模式 | 高 — 检测空指针、越界等 |
| `clang-analyzer-*` | 静态分析器检查 | 高 — 深度路径分析 |
| `performance-*` | 性能优化建议 | 中 — 可选优化 |
| `cert-*` | 安全编码标准 | 中 — 缓冲区安全 |

**应禁用的检查**（PostgreSQL 代码风格会触发大量误报）：
- `modernize-*`：PG 使用 C 风格编码，不适用 C++ 现代化
- `readability-*`：PG 有自己的代码风格，与 clang-tidy 不一致
- `google-*`：非 Google 代码风格

### Step 3: 运行分析

```bash
# 单文件分析
clang-tidy -p compile_commands.json src/backend/storage/buffer/bufmgr.c \
  --checks='bugprone-*,clang-analyzer-*'

# 全项目分析（谨慎，耗时长）
run-clang-tidy -p compile_commands.json -checks='bugprone-*' src/backend/
```

### Step 4: 分类和报告

按严重程度分类结果：

| 级别 | 说明 | 处理方式 |
|------|------|---------|
| **Error** | 确认的 bug | 必须在报告中突出显示 |
| **Warning** | 可疑代码 | 需要人工审查 |
| **Info** | 建议 | 记录但不需要立即处理 |

### Step 5: 生成文档

```markdown
## 静态分析报告

**分析范围**: src/backend/xxx/
**工具**: clang-tidy (version X.X)
**检查类别**: bugprone-*, clang-analyzer-*

### 摘要
- Error: N 个
- Warning: M 个
- Info: K 个

### 发现列表

#### [E001] bugprone-null-pointer (High)
**文件**: `src/backend/storage/buffer/bufmgr.c:234`
**说明**: ...
**建议**: ...
```

## 降级：模式匹配审查（工具未安装时）

使用 `ast_grep_search` 和 `Grep` 搜索已知的 C 代码反模式：

### 检查清单

```
1. 空指针解引用风险
   ast_grep_search: "*$PTR" → 检查前置 NULL 判断

2. 缓冲区溢出风险
   Grep: "memcpy\|memmove\|strcpy\|strcat\|sprintf"
   → 检查目标缓冲区大小是否足够

3. 资源泄漏
   Grep: "palloc\|malloc\|popen" → 检查配对的 pfree/close
   Grep: "LockAcquire\|LWLockAcquire" → 检查配对的 Release

4. 锁使用规范
   Grep: "LockAcquire" → 检查是否在 PG_ENSURE_ERROR_CLEANUP 中
   Grep: "LWLockAcquire" → 检查是否配对 LWLockRelease

5. 内存上下文规范
   Grep: "palloc\|palloc0" → 检查是否在正确的内存上下文中
   Grep: "MemoryContextSwitchTo" → 检查是否恢复了原上下文

6. 错误处理
   Grep: "ereport\|elog" → 检查 ERROR 级别是否导致正确清理
   Grep: "PG_TRY\|PG_CATCH" → 检查 PG_END_TRY 是否存在
```

### 报告格式

同上方 Step 5 的文档格式，但标注"模式匹配审查（未使用 clang-tidy）"。
