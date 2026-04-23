# 结构体内存布局分析工作流

分析 PostgreSQL C 结构体的内存布局，包括字段偏移、对齐、填充和总大小。

## 触发条件

- "分析 XXX 结构体的内存布局"
- "计算 XXX 的字段偏移"
- "XXX 结构体占多少字节"
- "画出 XXX 的内存布局图"

## 工作流

### Step 1: 定位目标结构体

1. 使用 `Grep` 搜索结构体定义：
   ```
   Grep: "typedef struct.*StructName" 或 "struct StructName {"
   path: src/include/ (优先搜索头文件)
   ```

2. 使用 `ast_grep_search` 精确定位（可选）：
   ```
   pattern: "struct $NAME { $$$FIELDS }"
   language: "c"
   ```

3. 记录：
   - 源文件路径和行号
   - 条件编译宏（`#ifdef` 等）
   - 嵌套的头文件依赖

### Step 2: 调用 struct-offset-analyzer skill

通过 Skill 工具委托计算：

```
Skill: skill="struct-offset-analyzer"
参数：提供结构体名称和源文件位置
```

该 skill 提供：
- 类型大小和对齐规则表
- 偏移量计算方法
- 填充字节标注
- 十六进制偏移输出

### Step 3: 补充类型信息

struct-offset-analyzer 可能需要补充的 PostgreSQL 特有类型：

| PG 类型 | 底层类型 | 大小 (64-bit) |
|---------|---------|--------------|
| `Datum` | `uintptr_t` | 8 字节 |
| `Oid` | `unsigned int` | 4 字节 |
| `BlockNumber` | `uint32` | 4 字节 |
| `RelFileNode` | struct (3×Oid) | 12 字节 (+padding) |
| `TransactionId` | `uint32` | 4 字节 |
| `CommandId` | `uint32` | 4 字节 |
| `ItemId` | 指针 | 8 字节 |
| `Page` | 指针 | 8 字节 |
| ` LWLock` | 取决于配置 | 需查源码 |
| `pg_atomic_uint32` | 平台相关 | 需查源码 |
| `pg_atomic_uint64` | 平台相关 | 需查源码 |

对于不确定的 PG 特有类型：
1. `Grep` 搜索 `include/` 中的 typedef 定义
2. `Read` 读取定义确认底层类型
3. 检查条件编译（不同平台可能不同大小）

### Step 4: 生成内存布局 SVG

按照 `references/svg-conventions.md` 中的"结构体内存布局图"规范生成。

关键要素：
- 垂直堆叠的水平矩形条
- 数据字段：实色填充，按模块着色
- Padding：虚线描边 + 浅灰填充
- 左侧：十六进制偏移标注
- 右侧：大小 + 类型标注
- 头部：结构体名称、总大小、源文件引用

保存路径：
```
C:/Users/swrd/Desktop/markdown/postgresql/<主题>/struct-<名称>.svg
```

### Step 5: 生成文档

输出 markdown 包含：

```markdown
## <结构体名> 内存布局

**定义位置**: `src/include/xxx.h:行号`
**总大小**: N 字节 (64-bit)
**最大对齐**: M 字节

### 偏移表

| 偏移 | 字段名 | 类型 | 大小 | 说明 |
|------|--------|------|------|------|
| 0x00 | field1 | int | 4B | ... |
| 0x04 | (padding) | — | 4B | 对齐到 8 字节 |
| 0x08 | field2 | int64 | 8B | ... |

![内存布局图](struct-<名称>.svg)
```

保存路径：
```
C:/Users/swrd/Desktop/markdown/postgresql/<主题>/struct-<名称>.md
```

### Step 6: 质量检查

- [ ] 偏移计算已验证（手动复查关键字段）
- [ ] PG 特有类型的底层类型已确认
- [ ] 条件编译对布局的影响已标注
- [ ] SVG 偏移标注与计算值一致
- [ ] 文档已保存到正确路径
