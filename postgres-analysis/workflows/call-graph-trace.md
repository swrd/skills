# 调用图追踪工作流

生成 PostgreSQL 函数调用依赖图，支持自动和手动两种模式。

## 触发条件

- "追踪 XXX 函数的调用链"
- "生成 XXX 的调用图"
- "谁调用了 XXX"
- "XXX 调用了哪些函数"
- "分析 XXX 的影响范围"

## 工作流

### Step 1: 确定追踪参数

向用户确认（或从上下文推断）：

1. **目标函数**：函数名或子系统入口点
2. **追踪方向**：
   - **下游 (callees)**：XXX 调用了谁（默认）
   - **上游 (callers)**：谁调用了 XXX
   - **双向**：完整调用图
3. **追踪深度**：1-5 层（默认 3）
4. **作用域**：
   - 单文件（如 `backend/storage/buffer/`）
   - 单模块（如 `storage/`）
   - 跨模块（全代码库）

### Step 2: 执行追踪（ast_grep + Grep 手动构建）

> **注意**：grepai CLI 仅支持 TypeScript/TSX，不适用于 PostgreSQL C 代码。
> 本工作流使用 ast_grep_search + Grep 作为主要追踪手段。

**追踪下游（callees）— XXX 调用了谁：**

1. `Read` 读取目标函数的完整实现
2. 在函数体内识别所有直接函数调用：
   ```
   ast_grep_search: pattern="$FUNC($$$ARGS)", language="c"
   path: 目标函数所在文件
   ```
3. 过滤掉宏调用和标准库函数
4. 对每个发现的被调用函数，递归执行 Step 2（直到达到深度限制）

**追踪上游（callers）— 谁调用了 XXX：**

1. `Grep` 搜索调用点：
   ```
   Grep: "FunctionName("
   path: 限定作用域目录
   output_mode: content
   ```
2. 对每个匹配结果，`Read` 验证是否为真正的函数调用（排除注释、字符串、声明）
3. 记录调用者函数名和文件位置

**构建调用树：**

```
TargetFunction (src/backend/xxx/file.c:行号)
├── CalledFunc1 (src/backend/yyy/file.c:行号)
│   ├── SubFunc1 (src/backend/zzz/file.c:行号)
│   └── SubFunc2 (src/backend/zzz/file.c:行号)
├── CalledFunc2 (src/backend/yyy/file.c:行号)
└── CalledFunc3 (src/backend/www/file.c:行号) [函数指针调用]
```

### Step 3: 识别特殊情况

PostgreSQL 代码中需要特殊处理的调用模式：

| 模式 | 识别方法 | 标注方式 |
|------|---------|---------|
| 函数指针调用 | 搜索 `->func()` 或 `(*func)()` | 标注 `[间接调用]` |
| 回调注册 | 搜索 `RegisterXxxCallback(func)` | 标注 `[回调]` |
| exec 表调用 | 搜索 `ExecProcNode` / dispatch table | 标注 `[虚函数表]` |
| 宏调用 | `Grep` 搜索大写标识符 | 标注 `[宏]` |
| elog/ereport | 搜索 `ereport(ERROR,` | 标注 `[异常跳转]` |
| longjmp | 搜索 `PG_RE_THROW` / `siglongjmp` | 标注 `[非局部跳转]` |
| 信号处理 | 搜索 `pqsignal` | 标注 `[信号处理]` |

### Step 4: 生成调用关系 SVG

按照 `references/svg-conventions.md` 中的"调用关系图"规范生成。

关键要素：
- 分层布局：按调用深度分层
- 节点：圆角矩形，按调用深度着色
- 边：实线箭头（直接调用）/ 虚线箭头（间接调用）
- 按模块着色节点（参考 source-file-map 中的模块分类）

保存路径：
```
C:/Users/swrd/Desktop/markdown/postgresql/<主题>/callgraph-<函数名>.svg
```

### Step 5: 生成文档

输出 markdown 包含：

```markdown
## <函数名> 调用关系分析

**定义位置**: `src/backend/xxx/file.c:行号`
**追踪方向**: 上游/下游
**追踪深度**: N 层
**总节点数**: M 个函数

### 调用树
（树形文本表示）

### 特殊调用模式
- `[间接调用]` XXX 通过函数指针调用 YYY
- `[虚函数表]` Executor 通过 dispatch table 调用节点函数

### 关键路径
（最深的调用链分析）

![调用关系图](callgraph-<函数名>.svg)
```

保存路径：
```
C:/Users/swrd/Desktop/markdown/postgresql/<主题>/callgraph-<函数名>.md
```

### Step 6: 质量检查

- [ ] 函数指针调用已标注为 `[间接调用]`
- [ ] 宏调用与函数调用已区分
- [ ] 每个调用点已通过 Read 验证
- [ ] 调用树深度未超过指定限制
- [ ] SVG 节点无重叠，连线无穿越
- [ ] 文档已保存到正确路径
