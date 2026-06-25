# 绘丹青 V2 画布系统 — AI 编码守则

你在为绘丹青 V2 画布系统编写任何代码时，必须无条件遵守以下规则。

---

## 1. 文档先行（继承自 `api文档.md`）

已在 workspace rule 中定义：**任何改动必须先更新场景文档和 OpenAPI 规范，再写代码。** 本条不再展开，但你每次动手前必须确认自己已读过 `api文档.md`，并已按该规则输出或更新了对应文档。

V2 画布相关文档位置：

| 层级 | 路径 |
|------|------|
| AI-Native PRD | `backend/docs/NativePRD/<模块名>/<功能名>.md` |
| BDD Gherkin | `backend/docs/NativePRD/<模块名>/bdd/<场景名>.feature` |
| API 契约 | `backend/docs/API规范/工作台项目管理.openapi.json`（V2 工作台 API）<br>`backend/docs/API规范/用户端.openapi.json`（用户端 API） |

---

## 2. BDD 场景驱动的 TDD 循环

### 2.1 测试文件位置

测试文件直接放在对应模块目录下，与 PRD、BDD 同层：

```
backend/docs/NativePRD/<模块名>/test_<模块名>.py
```

现有示例：
```
backend/docs/NativePRD/项目管理/test_项目管理.py
```

### 2.2 RGBC 循环

| 阶段 | 要求 |
|------|------|
| **红灯 (Red)** | 先在对应模块的 `test_<模块名>.py` 中编写场景函数。确保运行新场景测试时失败（接口尚未实现或返回不符合预期）。 |
| **绿灯 (Green)** | 编写**最少业务代码**让红灯变绿。严格按 API → Service → CRUD 的调用链实现，不跳层。 |
| **蓝灯 (Blue)** | 回归验证。修改后必须运行该模块的全量测试，确保已有场景未被撞烂。 |
| **提交 (Commit)** | 绿灯 + 蓝灯全过后，按 `git-commit-message.md` 规范提交。 |

### 2.3 严禁欺骗测试

- 禁止在 API 层直接 `return {"code": 200, "data": {...}}` 硬编码响应体。
- 必须走完整的 `API → Service → CRUD → DB` 调用链，写入真实数据库并读出。
- 状态断言必须检验返回体中的业务字段（如 `data.title`、`data.canvases`），不能只断言 `code == 200`。

### 2.4 场景函数编写规范

```python
class Test创建项目:
    """BDD: 创建项目.feature"""

    async def test_仅填名称创建项目(
        self, async_client: AsyncClient, auth_headers: dict, db_session,
    ):
        """正常路径 - 仅填名称创建项目"""
        title = f"创建测试_{uuid.uuid4().hex[:8]}"
        resp = await async_client.post(
            f"{PREFIX}/projects",
            json={"title": title},
            headers=auth_headers,
        )
        assert resp.status_code == 200, f"HTTP {resp.status_code}: {resp.text[:300]}"
        body = resp.json()
        assert body["code"] == 200, f"code={body.get('code')}, msg={body.get('msg')}"
        data = body["data"]
        assert isinstance(data, dict), "data 必须是对象"
        assert data.get("id"), "data.id 不能为空"
        assert data.get("title") == title

        # 断言数据库副作用（防 AI 硬编码）
        project = db_session.get(StudioProject, int(data["id"]))
        assert project is not None, "project 应写入数据库"
        assert project.title == title
```

---

## 3. 后端分层架构

### 3.1 目录结构规范

每个 V2 画布的功能模块在 `backend/app/studio/<模块名>/` 下统一组织：

```
<模块名>/
├── api/v1/<模块名>.py     # FastAPI 路由（仅参数校验 + 调用 service + 返回响应）
├── service/<模块名>_service.py  # 业务逻辑（编排 DAO + 权限校验 + 跨模块调用）
├── crud/<模块名>_crud.py        # 纯数据访问（一个表对应一个 DAO 类）
├── model/<模块名>.py            # SQLAlchemy ORM 模型
└── schema/<模块名>.py           # Pydantic 请求/响应 schema
```

**调用方向是单向的**：API → Service → CRUD / Model。Schema 被 API 和 Service 同时引用。严禁 API 直接调 CRUD。

### 3.2 路由注册

各模块的 `api/v1/<模块名>.py` 中定义 `router = APIRouter()`，由顶层 `backend/app/studio/router.py` 通过 `include_router` 汇聚，统一挂载在 `/api/v1/studio`。

### 3.3 依赖注入

| 依赖 | 用途 |
|------|------|
| `DependsJwtAuth` | 认证（解析 Bearer Token，注入当前用户） |
| `CurrentSession` | 注入 `AsyncSession`（自动提交） |
| `CurrentSessionTransaction` | 注入 `AsyncSession`（手动事务控制） |

---

## 4. 数据库模型卡点

### 4.1 基类与通用字段

所有 V2 模型统一继承 `Base`（`from backend.common.model import Base`），`Base` 已内置：
- `DateTimeMixin` → `created_time` + `updated_time`（`TimeZone` 类型）
- `MappedAsDataclass` → 支持 `mapped_column(init=False)`

### 4.2 必含字段

每个模型必须显式定义：

```python
class StudioXxx(Base):
    __tablename__ = "drama_xxx"           # 显式表名，前缀 drama_
    __table_args__ = {'extend_existing': True}

    id: Mapped[id_key] = mapped_column(init=False)  # 主键，统一用 id_key
    owner_id: Mapped[int] = mapped_column(BigInteger, index=True, comment='所有者用户 ID')
```

### 4.3 类型映射

| 场景 | 使用 |
|------|------|
| 主键 + 用户 ID 类整型 | `id_key`（自动选择自增/雪花） |
| 带时区时间 | `TimeZone` |
| 长文本 | `UniversalText` |
| 不确定结构（节点、连线、快照内容等） | SQLAlchemy 原生 `JSON` 类型 → Pydantic `dict[str, Any]` 二次校验 |
| 标签/列表 | `JSON` + `default_factory=list` → Pydantic `list[str]` |

### 4.4 唯一约束

需跨字段约束时显式声明：

```python
__table_args__ = (
    UniqueConstraint('project_id', 'user_id', name='uq_drama_xxx'),
    {'extend_existing': True},
)
```

---

## 5. Schema 规范

### 5.1 基类

所有 Pydantic schema 继承 `SchemaBase`（`from backend.common.schema import SchemaBase`）。

### 5.2 ID 类型

API 层接收 `str` 类型的 ID（前端 JS 精度限制），Schema 中使用：

```python
from backend.app.studio.common_schema import SnowflakeStr
id: SnowflakeStr = Field(..., description='画布 ID')
```

Service 层调用 `parse_snowflake_id()` 将 `str` 转为 `int`。

### 5.3 三类 Schema

| 类型 | 特征 | 示例命名 |
|------|------|---------|
| 请求参数 | `Field(...)` 含校验，可有 `@model_validator` | `CreateStudioXxxParam` |
| 响应摘要（列表项） | `from_attributes=True`，字段精简 | `StudioXxxSummary` |
| 响应详情 | `from_attributes=True`，含关联嵌套对象 | `StudioXxxDetail` |

### 5.4 枚举

使用 `StrEnum`（`use_enum_values=True` 自动序列化为字符串值），各模块枚举定义在 `enums.py` 或 `<模块>/enums/`。

### 5.5 统一响应

所有接口用 `response_base.success(data=...)` / `response_base.fail(code=..., msg=...)` 包装响应。

---

## 6. 前端编码约束

### 6.1 技术栈

- Vue 3 组合式 API（`<script setup lang="ts">`）
- TypeScript 强类型
- Tailwind CSS 样式

### 6.2 异步状态机

所有异步请求必须绑定四态状态机，并在 UI 上可感知：

| 状态 | UI 表现 |
|------|--------|
| `idle` | 按钮可点击、无指示器 |
| `processing` | 按钮显示 Loading 动画并禁用 |
| `completed` | 短暂显示成功反馈后恢复 |
| `failed` | 保留错误信息，提供重试按钮 |

---

## 7. 变更规模控制

- **一次只改一个场景模块**：比如"给项目列表加筛选功能"是一个场景，"加审批拒绝原因"是另一个场景，不要混在一次改动中。
- 一个场景通常涉及 2-4 个文件（API + Service + Schema + 对应的测试文件），这是正常的。不要压缩到一个文件，也避免一个场景横跨 5 个以上文件。

---

## 8. 提交纪律

绿色后必须提醒用户："当前场景已变绿，建议 `git commit`"。提交信息遵循 `git-commit-message.md` 规范：

```
<type>(<scope>): <中文 subject>
```

常见 V2 scope：`workbench`, `canvas`, `project`, `collaboration`, `template`, `script-workflow`, `intelligent-parse`。
