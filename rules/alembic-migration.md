# Alembic DB Migration Rules

## 🚫 操作安全红线（最高优先级）

以下操作**绝对禁止**，即使 AI 认为"这样更合理"或"只是小改动"，也一律不准做：

- **禁止删除任何 Alembic 迁移文件**：`migrations/versions/` 下的 `.py` 文件一个都不准删，不管是你认为"多余的""重复的""不需要的"，全部不准动。
- **禁止修改任何已存在的迁移文件**：包括但不限于修改 `upgrade()`、`downgrade()`、表结构、字段定义、索引、约束等。哪怕是改一行 `import` 也不行。
- **禁止执行 Alembic 降级命令**：`alembic downgrade`、`alembic downgrade -1`、`alembic downgrade <revision>` 等所有降级操作一律不准跑。
- **禁止删除 Alembic 版本记录**：不准动 `alembic_version` 表，不准 `alembic stamp` 重置版本号。
- **禁止在未经用户明确要求时新建迁移文件**：用户没开口说"帮我生成一个 migration"，就不要 `alembic revision --autogenerate`。
- **禁止执行 `alembic upgrade head` 以外的任何 Alembic CLI 命令**：除非用户**逐字逐句、明确地**要求你执行某个特定 Alembic 命令。

> 违反以上任何一条都可能导致数据库迁移树损坏、CI/CD 部署失败、生产环境回滚困难。**宁可多问用户一句，也不要自作主张动 Alembic。**

## 数据库迁移核心铁律

- **禁止魔改历史迁移**：绝对不允许直接修改、重写或删除 `migrations/versions/` 下任何已经成功提交并运行过的旧迁移脚本文件。所有数据库结构变更必须通过 `alembic revision --autogenerate` 增量追加新文件。
- **修复初始脚本拓扑顺序规范**：当在初始化或本地重建数据库执行 `--autogenerate` 时，如果因表间外键（Foreign Key）依赖导致 Alembic 自动生成的 `upgrade()` 顺序颠倒报错（例如 Canvas 依赖 Project，但 Canvas 被写在了前面），AI 只能调整该最新脚本中 `create_table` 的上下执行顺序。**严禁**去修改、删除或脑补原 ORM Model 里的底层字段约束。
- **拒绝硬编码连接串**：绝对禁止在 `alembic.ini` 或 `migrations/env.py` 中以任何形式硬编码（Hardcode）数据库连接字符串。必须严格且唯一地通过 `core.config.settings.DATABASE_URL`（动态读取 `.env` 环境变量）来为 `config.set_main_option` 赋值。
- **禁止生产环境自动生成**：在线上（Production）或测试（Staging）环境的部署/开发指令中，只允许输出 `alembic upgrade head` 命令。**绝对禁止**在新环境执行 `--autogenerate`，防止产生多源迁移树冲突。

## 报错纠偏标准工作流

1. 当运行迁移报错时，先使用 `pytest` 检查当前最新的 SQLAlchemy ORM Model 是否存在业务字段拼写冲突。
2. 若属于外键依赖顺序错误，在最新的单个迁移脚本内进行拓扑排序修正。
3. 修正后，在当前会话中仅输出执行结果和最终的 `alembic upgrade head` 收尾指令，严禁添加长篇散文解释。
