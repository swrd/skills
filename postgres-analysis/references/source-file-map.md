# PostgreSQL 14.4 源文件映射表

本文件记录 PostgreSQL 各子系统对应的源文件和头文件位置，供工作流快速定位。

## 本地文档 (`doc/src/sgml/`)

PostgreSQL 官方文档的 SGML 源文件，是回答配置、行为、SQL 等问题的权威来源。

| 文档主题 | 文件路径 | 说明 |
|---------|---------|------|
| 总目录 | `doc/src/sgml/postgres.sgml` | 文档入口 |
| SQL 命令参考 | `doc/src/sgml/reference.sgml` | 所有 SQL 命令 |
| 配置参数 | `doc/src/sgml/config.sgml` | 所有 GUC 参数说明 |
| MVCC / 并发 | `doc/src/sgml/mvcc.sgml` | 事务隔离、锁、MVCC |
| 索引 | `doc/src/sgml/indices.sgml` | 索引类型和使用 |
| 索引内部 | `doc/src/sgml/indexam.sgml` | 索引访问方法 |
| WAL | `doc/src/sgml/wal.sgml` | 预写式日志 |
| 存储 | `doc/src/sgml/storage.sgml` | 物理存储布局 |
| 性能 | `doc/src/sgml/performance.sgml` | 性能提示 |
| 并行查询 | `doc/src/sgml/parallel.sgml` | 并行执行 |
| 查询优化 | `doc/src/sgml/planner.sgml` | 规划器概述 |
| 高可用 | `doc/src/sgml/high-availability.sgml` | HA 方案 |
| 备份恢复 | `doc/src/sgml/backup.sgml` | 备份和恢复 |
| 错误代码 | `doc/src/sgml/errcodes.sgml` | 错误代码列表 |
| SQL 语法 | `doc/src/sgml/sql.sgml` | SQL 语言 |
| 查询 | `doc/src/sgml/queries.sgml` | 查询语法 |
| 数据类型 | `doc/src/sgml/datatype.sgml` | 内置数据类型 |
| 函数 | `doc/src/sgml/func.sgml` | 内置函数 |
| 触发器 | `doc/src/sgml/triggers.sgml` | 触发器 |
| 规则系统 | `doc/src/sgml/rules.sgml` | 查询重写规则 |
| PL/pgSQL | `doc/src/sgml/plpgsql.sgml` | PL/pgSQL 语言 |
| 客户端认证 | `doc/src/sgml/client-authentication.sgml` | 认证配置 |
| 监控 | `doc/src/sgml/monitoring.sgml` | 监控和统计 |
| 例程维护 | `doc/src/sgml/maintenance.sgml` | 日常维护（VACUUM 等） |
| 内存 | `doc/src/sgml/runtime.sgml` | 运行时环境（含内存参数） |

## 核心后端 (`src/backend/`)

### 存储与访问层

| 组件 | 源文件目录 | 核心头文件 | README |
|------|-----------|-----------|--------|
| 堆存储 (Heap) | `backend/access/heap/` | `include/access/heapam.h` | — |
| B-tree 索引 | `backend/access/nbtree/` | `include/access/nbtree.h` | `backend/access/nbtree/README` |
| Hash 索引 | `backend/access/hash/` | `include/access/hash.h` | — |
| GIN 索引 | `backend/access/gin/` | `include/access/gin.h` | — |
| GiST 索引 | `backend/access/gist/` | `include/access/gist.h` | — |
| SP-GiST 索引 | `backend/access/spgist/` | `include/access/spgist.h` | — |
| BRIN 索引 | `backend/access/brin/` | `include/access/brin.h` | — |
| 事务管理 | `backend/access/transam/` | `include/access/transam.h`, `xact.h` | `backend/access/transam/README` |
| WAL 日志 | `backend/access/transam/` (xlog*.c) | `include/access/xlog.h`, `xloginsert.h` | — |
| 多版本并发控制 | `backend/access/heap/` (heapam.c), `backend/access/transam/` | `include/access/htup.h`, `tqual.h` | — |

### 存储管理层

| 组件 | 源文件目录 | 核心头文件 | README |
|------|-----------|-----------|--------|
| 缓冲区管理 | `backend/storage/buffer/` | `include/storage/bufmgr.h`, `buf_internals.h` | `backend/storage/buffer/README` |
| 锁管理器 | `backend/storage/lmgr/` | `include/storage/lmgr.h`, `lock.h` | `backend/storage/lmgr/README` |
| 存储管理器接口 | `backend/storage/smgr/` | `include/storage/smgr.h` | — |
| 页面操作 | `backend/storage/page/` | `include/storage/bufpage.h` | — |
| 空闲空间管理 | `backend/storage/freespace/` | `include/storage/freespace.h` | — |
| 文件管理 | `backend/storage/file/` | `include/storage/fd.h` | — |
| 共享内存 | `backend/storage/ipc/` | `include/storage/ipc.h`, `shmem.h` | — |
| LWLock | `backend/storage/lmgr/` | `include/storage/lwlock.h` | — |
| 条件变量 | `backend/storage/lmgr/` | `include/storage/condition_variable.h` | — |
| DM (Deadlock) | `backend/storage/lmgr/` (deadlock.c) | `include/storage/lock.h` | — |

### 查询处理

| 组件 | 源文件目录 | 核心头文件 | README |
|------|-----------|-----------|--------|
| SQL 解析器 | `backend/parser/` | `include/parser/` | — |
| 查询重写 | `backend/rewrite/` | `include/rewrite/rewriteHandler.h` | — |
| 查询优化器 | `backend/optimizer/` | `include/optimizer/` | `backend/optimizer/README` |
| 路径生成 | `backend/optimizer/path/` | `include/optimizer/pathnode.h` | — |
| 计划生成 | `backend/optimizer/plan/` | `include/optimizer/planner.h` | — |
| GEQO 遗传优化 | `backend/optimizer/geqo/` | `include/optimizer/geqo.h` | — |
| 查询执行器 | `backend/executor/` | `include/executor/executor.h` | `backend/executor/README` |
| 表达式求值 | `backend/executor/execExpr*.c` | `include/executor/execexpr.h` | — |
| 流量控制器 | `backend/tcop/` | `include/tcop/` | — |

### 进程管理

| 组件 | 源文件目录 | 核心头文件 | README |
|------|-----------|-----------|--------|
| Postmaster | `backend/postmaster/` | `include/postmaster/postmaster.h` | — |
| 后台写入器 | `backend/postmaster/bgwriter.c` | `include/storage/bgwriter.h` | — |
| WAL 写入器 | `backend/postmaster/walwriter.c` | — | — |
| 自动清理 | `backend/postmaster/autovacuum.c` | `include/postmaster/autovacuum.h` | — |
| 检查点 | `backend/postmaster/checkpointer.c` | `include/storage/checkpoint.h` | — |
| 统计收集 | `backend/postmaster/pgstat.c` | `include/pgstat.h` | — |
| 启动进程 | `backend/postmaster/startup.c` | — | — |

### 工具库

| 组件 | 源文件目录 | 核心头文件 | README |
|------|-----------|-----------|--------|
| 内存管理 | `backend/utils/mmgr/` | `include/utils/memutils.h` | — |
| 抽象数据类型 | `backend/utils/adt/` | `include/utils/builtins.h` | — |
| 系统缓存 | `backend/utils/cache/` | `include/utils/catcache.h`, `relcache.h` | — |
| 函数管理器 | `backend/utils/fmgr/` | `include/fmgr.h` | — |
| 字符编码 | `backend/utils/mb/` | `include/mb/pg_wchar.h` | — |
| 排序 | `backend/utils/sort/` | `include/utils/sortsupport.h` | — |
| 错误处理 | `backend/utils/error/` | `include/utils/elog.h` | — |
| GUC 配置 | `backend/utils/misc/` | `include/utils/guc.h` | — |
| 时间/日期 | `backend/utils/adt/datetime.c` | `include/utils/datetime.h` | — |

### 系统目录

| 组件 | 源文件目录 | 核心头文件 | README |
|------|-----------|-----------|--------|
| 目录管理 | `backend/catalog/` | `include/catalog/` | — |
| 索引编目 | `backend/catalog/index.c` | `include/catalog/index.h` | — |
| 表编目 | `backend/catalog/heap.c` | `include/catalog/heap.h` | — |
| 依赖追踪 | `backend/catalog/pg_depend.c` | `include/catalog/dependency.h` | — |

## 前端组件 (`src/`)

| 组件 | 源文件目录 | 说明 |
|------|-----------|------|
| psql 客户端 | `bin/psql/` | 交互式 SQL shell |
| pg_dump | `bin/pg_dump/` | 数据库备份 |
| pg_restore | `bin/pg_dump/` | 数据库恢复 |
| initdb | `bin/initdb/` | 初始化数据库集群 |
| pg_ctl | `bin/pg_ctl/` | 服务器启停控制 |
| libpq (客户端库) | `interfaces/libpq/` | C 客户端库 |
| PL/pgSQL | `pl/plpgsql/` | 内置过程语言 |

## 测试

| 组件 | 源文件目录 | 说明 |
|------|-----------|------|
| SQL 回归测试 | `test/regress/` | 主测试套件 |
| 隔离测试 | `test/isolation/` | 并发事务测试 |
| 恢复测试 | `test/recovery/` | 恢复和复制测试 |
