---
name: "backend-tdd-green-implementation"
description: "根据已经生成并验证失败的后端 pytest 红灯测试，结合 PRD、BDD/Gherkin、OpenAPI 契约、数据库模型和失败日志，做最小后端实现并跑通定向测试与全量测试。用于 FastAPI、SQLAlchemy、Alembic、Pydantic、队列、LLM 调用、权限、状态流转、接口加字段、漏洞修复等后端 TDD 绿灯阶段。"
---

# 后端 TDD 绿灯实现与全量测试 Skill

根据已经红灯的 pytest 测试，完成最小后端实现，让测试从红灯变绿，并通过全量测试。

核心判断：这个 skill 不负责重新设计需求，也不负责重新生成测试。它只在红灯测试已经存在时使用，目标是基于 PRD / BDD / OpenAPI / DB Model 写真实业务代码，拒绝硬编码、伪造副作用和为了过测试改测试。

## 项目约定速查（绘丹青 FBA + V2 画布）

本项目的架构和编码规范由以下 workspace rule 定义，绿灯实现阶段必须遵守：

| 规范 | 来源 |
|------|------|
| 分层架构 + 目录结构 + 路由注册 + DI 注入 | `.trae/rules/v2-canvas-coding.md` |
| Alembic 迁移安全红线 | `.trae/rules/alembic-migration.md` |
| Git Commit Message | `.trae/rules/git-commit-message.md` |
| 文档先行（SSOT） | `.trae/rules/api文档.md` |

### 目录结构（当前实际）

工作台模块位于 `backend/app/canvas/<模块名>/`，每个模块严格分层：

```
<模块名>/
├── api/v1/<模块名>.py     # FastAPI 路由（参数校验 + 调用 service + 返回响应）
├── service/<模块名>_service.py  # 业务逻辑（编排 DAO + 权限校验）
├── crud/<模块名>_crud.py        # 纯数据访问（DAO 类）
├── model/<模块名>.py            # SQLAlchemy ORM 模型
└── schema/<模块名>.py           # Pydantic 请求/响应 schema
```

调用方向单向：**API → Service → CRUD / Model**。严禁 API 直接调 CRUD。

### 路由注册

各模块的 `router` 在 `backend/app/canvas/api/router.py` 中通过 `studio_router.include_router()` 汇聚，统一挂载在 `/api/v1/studio`，标签格式 `v2-画布/<序号>-<名称>`。

### 核心基类与类型

| 元素 | 导入来源 | 用途 |
|------|---------|------|
| `Base` | `backend.common.model` | 所有 ORM 模型基类（含 `DateTimeMixin` + `MappedAsDataclass`） |
| `id_key` | `backend.common.model` | 主键类型（自动选择自增/雪花） |
| `UniversalText` | `backend.common.model` | 长文本列类型 |
| `TimeZone` | `backend.common.model` | 带时区 DateTime 列类型 |
| `SchemaBase` | `backend.common.schema` | 所有 Pydantic schema 基类 |
| `SnowflakeStr` | `backend.common.schema` | API 层 ID 类型（`str`，自动转雪花） |
| `ResponseModel` | `backend.common.response.response_schema` | 统一响应体类型 |

### ORM 模型必含字段

```python
class StudioXxx(Base):
    __tablename__ = "drama_xxx"              # 显式表名，前缀 drama_
    __table_args__ = {'extend_existing': True}

    id: Mapped[id_key] = mapped_column(init=False)
    owner_id: Mapped[int] = mapped_column(BigInteger, index=True, comment='所有者用户 ID')
```

### 三层 Schema

| 层级 | 命名模式 | 特征 |
|------|---------|------|
| 请求参数 | `CreateStudioXxxParam` / `UpdateStudioXxxParam` | `Field(...)` 含校验 |
| 响应摘要（列表项） | `StudioXxxSummary` | `from_attributes=True`，字段精简 |
| 响应详情 | `StudioXxxDetail` | `from_attributes=True`，含嵌套对象 |

### 认证与 DI 注入

```python
from backend.common.security.jwt import DependsJwtAuth
from backend.database.db import CurrentSession, CurrentSessionTransaction

AUTH_DEPS = [DependsJwtAuth]

# 读操作用 CurrentSession（自动提交）
# 写操作用 CurrentSessionTransaction（手动事务控制）
```

### 统一响应与异常

```python
# 成功
return response_base.success(data=...)

# 异常
from backend.common.exception import errors
raise errors.NotFoundError(msg='资源不存在')     # → 404
raise errors.ForbiddenError(msg='无权操作')      # → 403
raise errors.RequestError(msg='参数非法')        # → 400
```

### 命名约定

| 层 | 命名模式 | 实例化 |
|----|---------|--------|
| CRUD | `StudioXxxDAO` | `studio_xxx_dao = StudioXxxDAO()` |
| Service | `StudioXxxService` | `studio_xxx_service = StudioXxxService()` |
| API Router | `router = APIRouter()` | 直接使用 |

### CRUD / Service 方法签名

```python
# DAO：所有方法接收 db: AsyncSession，使用 keyword-only 参数
async def create(self, db: AsyncSession, *, title: str, owner_id: int, ...) -> StudioProject:
    project = StudioProject(title=title, owner_id=owner_id, ...)
    db.add(project)
    await db.flush()
    return project

# Service：类方法，接收 db + 当前用户 ID
@classmethod
async def create_project(cls, db: AsyncSession, *, obj: CreateStudioProjectParam, current_user_id: int) -> StudioProjectDetail:
    ...
```

### Alembic 安全红线（必须遵守）

- **禁止删除任何已有迁移文件**
- **禁止修改任何已有迁移文件**（包括 import）
- **禁止执行降级命令**（`alembic downgrade`）
- **禁止删除 alembic_version 表记录**
- **禁止在用户未明确要求时新建迁移文件**
- 只能通过 `alembic revision --autogenerate` 增量追加新文件
- 如果外键依赖导致自动生成的表顺序错误，**只能调整该最新脚本中 `create_table` 的上下顺序**，严禁改 ORM Model 的字段约束

### 测试运行命令

使用 `uv run` 前缀执行 Python：

```bash
uv run pytest backend/docs/NativePRD/<模块名>/test_<模块名>.py -k <关键字> -v
uv run pytest backend/docs/NativePRD/<模块名>/test_<模块名>.py -v
uv run pytest backend/docs/NativePRD/ -v
```

### Git Commit 规范

```
<type>(<scope>): <中文 subject>
```

常见 scope：`workbench`, `canvas`, `project`, `collaboration`, `template`

---

## 适用场景

- 用户已经有 pytest 红灯测试，要求实现后端让测试通过
- 用户要求根据 PRD / BDD / OpenAPI / DB Model 完成 FastAPI 后端
- 用户要求"跑通全部测试""让红灯变绿""根据 TDD 完成后端"
- 用户在加参数、修漏洞、改接口、补权限、补状态流转后，需要做最小代码实现
- 用户担心 AI 为了通过测试硬编码响应、跳过数据库写入、跳过队列或伪造 LLM 结果

## 输入来源

必须优先读取：

- 红灯测试：`backend/docs/NativePRD/<模块名>/test_<模块名>.py`
- 红灯日志：用户提供的 pytest 输出，或本地运行 `uv run pytest` 得到的失败日志
- PRD：`backend/docs/NativePRD/<模块名>/<功能名称>.md`
- BDD：`backend/docs/NativePRD/<模块名>/bdd/<功能名称>.feature`
- OpenAPI：`backend/docs/API规范/用户端.openapi.json` 或 `backend/docs/API规范/<模块名>.openapi.json`
- 数据库模型：`backend/app/canvas/<模块名>/model/<模块名>.py`
- Pydantic schema：`backend/app/canvas/<模块名>/schema/<模块名>.py`
- API 路由 / service / crud：按项目分层读取 `backend/app/canvas/<模块名>/`
- Alembic 迁移目录：`backend/alembic/versions/`
- 测试 fixtures：`backend/tests/conftest.py` 和 `backend/docs/NativePRD/conftest.py`

缺少红灯测试：先使用 [[后端TDD红灯测试生成Skill]]。
缺少 OpenAPI：不能可靠判断 request / response 契约。
缺少 DB Model 或迁移：不能可靠判断真实持久化。
缺少红灯日志：先运行定向 pytest，得到当前失败点。

## 输出目标

必须输出或完成：

1. 红灯失败原因定位
2. 最小实现范围说明
3. 后端代码修改
4. 必要的 SQLAlchemy Model / Pydantic / Alembic / service / route 修改
5. 定向测试命令与结果
6. 全量测试命令与结果
7. 改动摘要、风险点和建议 commit message（遵循 Conventional Commits + 中文 subject 规范）

不允许只输出解释而不实现，除非缺少必要文件、测试环境无法启动，或用户明确要求只分析。

## 执行顺序

### 1. 确认红灯真实存在

先运行最小定向测试，例如：

```bash
uv run pytest backend/docs/NativePRD/项目管理/test_项目管理.py -k test_create_project -v
```

如果测试已经通过，不能盲目改代码。应说明当前不是红灯，并检查是否需要补测试。

如果失败来自测试环境、依赖缺失、数据库连接错误，先区分：

- 环境问题：修测试环境或说明阻塞
- 测试问题：只在测试与 OpenAPI / BDD 明确冲突时才改测试
- 实现缺口：进入绿灯实现

### 2. 建立失败映射表

必须先整理：

| 失败测试 | 对应 Scenario | 失败点 | 缺失实现 | 涉及文件 | 是否需要迁移 |
|---|---|---|---|---|---|
| test_create_project_with_cover_image | 创建项目携带封面图 | response 缺少 cover_image | schema / model / service 未接收保存 | schema/project.py / model/project.py / service/project_service.py | 是 |

这个表决定最小实现范围，防止顺手重构。

### 3. 只改必要边界

按照失败点从外到内修改，严格遵循 **API → Service → CRUD** 调用链：

1. Pydantic request / response schema（继承 `SchemaBase`，ID 字段用 `SnowflakeStr`）
2. FastAPI route 入参、出参和依赖（`DependsJwtAuth`、`CurrentSession` / `CurrentSessionTransaction`）
3. service / crud 的最小业务逻辑（CRUD 用 `db.flush()`，不手动 `commit()`）
4. SQLAlchemy Model 字段、关系、索引（继承 `Base`，表名 `drama_` 前缀）
5. Alembic migration upgrade / downgrade（遵守安全红线，只增量追加）
6. 队列、外部服务、LLM 调用、审计、配额等真实副作用

只修改本次 Scenario 需要的路径。不要重写旧功能，不要重排无关模块，不要顺手格式化大文件。

### 4. 写真实业务，不写假实现

必须满足：

- 真实接收 OpenAPI 定义的请求字段
- 真实执行 Pydantic 校验（Schema 继承 `SchemaBase`）
- 真实写入或读取 SQLAlchemy 数据库（通过 DAO 层，`db.flush()` 后返回）
- 真实维护状态流转、权限判断（`errors.ForbiddenError`）、外键关系和默认记录
- 涉及队列时真实创建任务记录并触发队列入口
- 涉及 LLM 时调用项目提供的解析函数
- 外部依赖可以 mock，但业务代码不能绕过调用链
- 响应必须用 `response_base.success(data=...)` 包装

严禁：

- 根据测试样例硬编码固定 JSON
- 在 API 层直接 `return {"code": 200, "data": {...}}` 硬编码响应体
- 在业务代码中判断测试字符串然后返回特定结果
- 跳过数据库写入，只返回看起来正确的响应
- 跳过队列、通知、审计、配额等副作用
- 删除或弱化测试断言
- 为了过测试把异常吞掉

### 5. 迁移必须可回滚

涉及数据库结构变化时：

- 更新 SQLAlchemy Model（继承 `Base`，`__tablename__ = "drama_xxx"`）
- 只通过 `alembic revision --autogenerate` 增量生成新迁移文件
- **绝对禁止**删除或修改任何已有迁移文件
- `upgrade` 和 `downgrade` 必须完整
- 索引、外键、唯一约束、server_default 必须和模型一致
- 有数据迁移风险时必须列出风险和回滚方案
- 如果外键依赖导致自动生成的表创建顺序错误，**只调整该最新脚本中 `create_table` 的顺序**，不修改 ORM Model

不要写只能前进不能回滚的迁移。

### 6. 红绿循环

每轮只解决一组相关失败：

```text
Red：运行定向测试，确认失败点
Green：写最小真实实现
Verify：重跑同一条定向测试
Expand：跑同文件测试或相关模块测试
Full：跑 NativePRD 下全量 pytest
Clean：只在全部变绿后做小范围清理
```

如果新的失败暴露旧功能回归，先定位是否由本次修改造成。不能为了新测试变绿牺牲旧测试。

## 测试运行要求

所有 Python 命令必须使用 `uv run` 前缀。至少运行：

```bash
uv run pytest backend/docs/NativePRD/<模块名>/test_<模块名>.py -k <关键字> -v
uv run pytest backend/docs/NativePRD/<模块名>/test_<模块名>.py -v
uv run pytest backend/docs/NativePRD/ -v
```

如果项目测试量很大，可以先跑相关模块，再跑全量。最终必须尝试 NativePRD 下全量测试。

如果全量测试无法完成，必须说明：

- 执行了什么命令
- 失败或阻塞原因
- 哪些测试已经通过
- 剩余风险是什么

## 改测试规则

默认不改测试。

只有下面情况可以改：

- 测试与 OpenAPI 明确冲突
- 测试与 BDD / PRD 明确冲突
- 测试依赖不存在的 fixture，且项目已有等价 fixture
- 测试断言字段拼写错误，且契约文件可证明

改测试前必须说明冲突依据。不能因为实现困难而改测试。

## 常见实现路径（项目适配版）

### 接口新增字段

1. `schema/<模块名>.py`：Pydantic request 增字段（继承 `SchemaBase`）
2. `schema/<模块名>.py`：Pydantic response 增字段（`from_attributes=True`）
3. `model/<模块名>.py`：SQLAlchemy Model 增 column（使用 `UniversalText` / `TimeZone` 等类型）
4. Alembic：增量生成 migration，`upgrade` 加列，`downgrade` drop 列
5. `service/<模块名>_service.py`：接收并保存字段
6. `crud/<模块名>_crud.py`：DAO 的 `create`/`update` 方法处理字段
7. Response 用 `response_base.success(data=...)` 返回
8. 跑新增字段测试、相关接口测试、全量测试

### 权限漏洞修复

1. 读取 BDD 越权 Scenario
2. 定位 route dependency（`DependsJwtAuth`）或 service 权限判断
3. 在真实查询对象后判断 `owner_id == current_user_id` 或其他角色判断
4. 返回 `errors.ForbiddenError(msg='无权操作')` 或 `errors.NotFoundError(msg='资源不存在')`
5. 保证授权用户旧路径仍然通过

### 状态流转修复

1. 读取 PRD 状态机和 BDD 非法状态 Scenario
2. 找到状态变更入口（通常在 service 层）
3. 增加允许流转表或显式判断
4. 非法流转返回 `errors.RequestError(msg='...')`
5. 断言数据库状态没有被错误修改

### 队列 / 异步任务

1. API 创建任务记录（通过 DAO 写入数据库）
2. 初始状态写入数据库
3. 调用项目队列入口
4. 测试环境可以 mock 队列发送，但必须断言调用参数
5. 不允许只返回 task_id 不落库

### LLM 解析

1. service 调用项目已有 LLM 封装函数
2. 测试中 mock LLM 返回值
3. 业务代码解析并持久化结果
4. 断言调用参数来自真实请求
5. 不允许在业务代码中硬编码解析 JSON

## 最终回复格式

完成后必须输出：

```text
已完成：
- 修改了哪些文件
- 实现了哪些红灯缺口

验证：
- uv run pytest backend/docs/NativePRD/<模块>/test_<模块>.py -k xxx：通过
- uv run pytest backend/docs/NativePRD/<模块>/test_<模块>.py：通过
- uv run pytest backend/docs/NativePRD/：通过 / 未通过，原因是 ...

风险：
- 是否涉及迁移
- 是否有外部依赖 mock
- 是否有未覆盖边界

建议提交：
- <type>(<scope>): <中文 subject>
```

## 典型调用

```text
请使用后端 TDD 绿灯实现流程完成这个功能。

红灯测试：
backend/docs/NativePRD/项目管理/test_项目管理.py -k test_create_project

参考文件：
PRD：backend/docs/NativePRD/项目管理/创建项目.md
BDD：backend/docs/NativePRD/项目管理/bdd/创建项目.feature
OpenAPI：backend/docs/API规范/工作台项目管理.openapi.json
DB Model：backend/app/canvas/project/model/project.py

要求：
1. 先运行定向 pytest，确认红灯。
2. 不要改测试，除非测试与 OpenAPI 或 BDD 明确冲突。
3. 只做最小后端实现，严格遵循 API → Service → CRUD 调用链。
4. 必须真实写数据库和触发关键副作用。
5. 遵守 Alembic 安全红线，只增量追加迁移。
6. 最后跑相关测试和全量 pytest。
```

## 检查清单

- [ ] 是否确认红灯真实存在
- [ ] 是否建立失败测试到实现缺口的映射
- [ ] 是否读取 PRD / BDD / OpenAPI / DB Model / 现有代码
- [ ] 是否只修改必要文件
- [ ] 是否遵循 API → Service → CRUD 调用链
- [ ] Schema 是否继承 `SchemaBase`，ID 是否用 `SnowflakeStr`
- [ ] ORM Model 是否继承 `Base`，表名是否 `drama_` 前缀
- [ ] 响应是否用 `response_base.success(data=...)` 包装
- [ ] 异常是否用 `errors.NotFoundError` / `errors.ForbiddenError` / `errors.RequestError`
- [ ] 是否真实接收、校验、保存和返回字段
- [ ] CRUD 是否用 `db.flush()` 而非手动 `commit()`
- [ ] 是否断言并实现数据库副作用
- [ ] 是否真实触发队列 / LLM / 审计 / 配额等关键副作用
- [ ] 是否没有硬编码测试样例
- [ ] 是否没有为了过测试弱化测试
- [ ] 是否补齐 Alembic upgrade / downgrade（遵守安全红线）
- [ ] 是否定向测试通过
- [ ] 是否相关测试通过
- [ ] 是否 NativePRD 下全量 pytest 通过
- [ ] 所有 Python 命令是否使用 `uv run` 前缀
- [ ] 是否说明剩余风险和符合 Conventional Commits 的建议 commit message
