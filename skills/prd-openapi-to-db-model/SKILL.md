---
name: "prd-openapi-to-db-model"
description: "根据 AI-Native PRD、OpenAPI 契约和现有 SQLAlchemy/Alembic 风格共同生成数据库模型设计，输出字段映射表、SQLAlchemy Model、Pydantic 二次校验建议和 Alembic 迁移要求。当用户要求生成数据库模型、设计 PostgreSQL 表结构、补 SQLAlchemy Model、生成 Alembic 迁移、判断 JSONB/外键/索引/唯一约束时触发。"
---

# PRD 与 OpenAPI 生成数据库模型 Skill

把 AI-Native PRD 和 OpenAPI 契约合并成数据库模型设计，输出 SQLAlchemy Model 及其完整模块代码（model / schema / crud / service / api）。

核心判断：数据库模型不能只看 OpenAPI。OpenAPI 锁接口字段，PRD 锁实体边界、业务关系、索引理由、状态流转和副作用。现有代码锁本项目的 ORM 风格和迁移习惯。

---

## 适用场景

- 用户要求根据 PRD / OpenAPI 生成数据库模型
- 用户要求设计 PostgreSQL 表结构、SQLAlchemy Model、Alembic 迁移
- 用户要求把接口字段落库
- 用户要新增字段、加参数、修漏洞，需要补模型和迁移
- 用户要判断 JSONB、外键、索引、唯一约束、软删除、状态字段怎么设计
- 用户要求先输出设计文档，再按框架规范输出代码

---

## 输入来源

必须优先读取：

- **PRD**：`backend/docs/NativePRD/<模块名>/<功能名称>.md`
- **OpenAPI**：`backend/docs/API规范/用户端.openapi.json`
- **现有模型代码**：例如 `backend/app/**/model/*.py`
- **现有迁移目录**：例如 `backend/alembic/versions/`

如果缺少 PRD，只能生成接口字段映射，不能可靠判断实体关系、索引和副作用。

如果缺少 OpenAPI，只能生成数据库草案，不能确认接口 request/response 字段和校验边界。

如果缺少现有模型代码，必须先声明假设，避免创造不符合项目风格的 ORM 写法。

---

## 分工原则

| 来源 | 负责内容 |
|---|---|
| PRD | 实体边界、业务关系、字段含义、状态流转、副作用、索引理由、唯一性、软删除、审计需求 |
| OpenAPI | request/response 字段、类型、nullable、format、enum、必填、错误响应、接口层校验 |
| 现有模型代码 | SQLAlchemy 版本、Declarative Base、UUID/时间戳写法、命名规范、关系写法、软删除习惯 |
| 现有迁移 | Alembic revision 风格、默认值写法、索引命名、外键策略、可回滚习惯 |

---

## 第一步：询问模块落盘位置

开始之前，必须先问用户：

> 生成的模型代码放在哪个目录下？例如 `backend/app/canvas/project/`？

规则：

- 不要假设路径，必须让用户指定。
- 用户可能指定一个已存在的目录（追加新模型），也可能指定一个新目录。
- 路径确定后，在此路径下创建标准的 5 层目录结构。
- 输出文件路径跟随用户指定的位置，设计文档始终放在 `backend/docs/NativePRD/<模块名>/db-model-design.md`。

### 标准目录结构

当路径确定后（例如 `backend/app/studio/project/`），创建：

```
<模块路径>/
├── model/
│   └── <实体名>.py          # SQLAlchemy ORM
├── schema/
│   └── <实体名>.py          # Pydantic 请求/响应 Schema
├── crud/
│   └── <实体名>_crud.py     # 数据访问层 (DAO)
├── service/
│   └── <实体名>_service.py  # 业务逻辑层
├── api/
│   └── v1/
│       └── <实体名>.py      # FastAPI 路由
└── __init__.py              # 各层共用（如无则省略）
```

---

## 第二步：调研项目框架规范

在生成任何代码之前，必须先阅读现有项目代码，提取以下规范，输出调研摘要：

### 必须阅读的文件清单

| 文件 | 学习目标 |
|------|---------|
| `backend/common/model.py` | Base 类、id_key、DateTimeMixin、UniversalText 等 |
| `backend/common/schema.py` | SchemaBase 类、配置惯例 |
| `backend/app/<某模块>/model/*.py` | ORM 风格（字段写法、关系写法、表名命名、注释） |
| `backend/app/<某模块>/schema/*.py` | Schema 风格（from_attributes、SnowflakeStr、Field） |
| `backend/app/<某模块>/crud/*_crud.py` | DAO 模式（类结构、方法签名、select/insert/update/delete） |
| `backend/app/<某模块>/service/*_service.py` | Service 模式（@dataclass、依赖注入、事务控制） |
| `backend/app/<某模块>/api/v1/*.py` | API 模式（路由注册、依赖注入、响应包装） |
| `backend/alembic/versions/` 最近 1-2 个迁移 | 迁移风格（upgrade/downgrade、索引命名、FK 策略） |

### 调研输出摘要格式

```markdown
## 项目框架调研摘要

### ORM 基类
- Base: from backend.common.model import Base
- id_key: from backend.common.model import id_key → Mapped[id_key]
- 时间戳: Base.DateTimeMixin 自动提供 created_time/updated_time
- 长文本: UniversalText
- 时区: TimeZone

### Schema 基类
- SchemaBase: from backend.common.schema import SchemaBase
- ID 类型: SnowflakeStr = Annotated[str, BeforeValidator(str)]
- 枚举: StrEnum + use_enum_values=True

### DAO 模式
- 每个表一个 DAO 类
- 方法直接接受 db: AsyncSession + 参数，返回 ORM 对象
- ...

### API 模式
- router = APIRouter()
- DependsJwtAuth 做认证
- CurrentSession / CurrentSessionTransaction 注入 db
- response_base.success() / fail() 包装响应

### 迁移风格
- revision 命名: xxxx_描述.py
- upgrade/downgrade 完整
- 索引命名: ix_表名_列名
- FK 命名: fk_表名_列名
```

---

## 第三步：设计文档

输出到 `backend/docs/NativePRD/<模块名>/db-model-design.md`，包含：

1. 数据库表设计说明（每个表的设计决策理由）
2. 字段映射表（PRD → OpenAPI → DB Column → 类型 → nullable → 索引/约束 → 是否落库 → 依据）
3. SQLAlchemy Model 代码（预览）
4. Pydantic 二次校验建议
5. Alembic 迁移要求
6. 风险和待确认问题

---

## 第四步：生成代码文件

在设计文档确认后，按用户指定的路径生成实际代码文件。

### 生成规则

1. **Model 文件**（`<路径>/model/<实体名>.py`）
   - 继承项目现有 Base
   - 使用项目现有 id_key、UniversalText、TimeZone 等
   - __tablename__ 与 PRD 一致
   - __table_args__ = {'extend_existing': True}
   - 每个字段写 comment（中文）
   - FK 带 ondelete

2. **Schema 文件**（`<路径>/schema/<实体名>.py`）
   - 继承项目现有 SchemaBase
   - 使用 SnowflakeStr 做 ID 类型
   - 分三类：CreateParam / Summary / Detail
   - Detail 带 from_attributes=True

3. **CRUD 文件**（`<路径>/crud/<实体名>_crud.py`）
   - DAO 类，方法接受 db: AsyncSession + 参数
   - 覆盖：list / get / create / update / delete

4. **Service 文件**（`<路径>/service/<实体名>_service.py`）
   - @dataclass Service 类
   - 编排 DAO + 权限校验 + 跨模块调用

5. **API 文件**（`<路径>/api/v1/<实体名>.py`）
   - router = APIRouter()
   - DependsJwtAuth 做认证
   - CurrentSession 注入 db
   - response_base.success() / fail() 包装

---

## 第五步：Alembic 迁移

在模型文件创建后，生成迁移要求或迁移脚本。必须满足：

- upgrade / downgrade 都完整
- 所有索引、外键、唯一约束、默认值和模型一致
- 新增非空字段必须考虑历史数据填充值
- 删除字段必须说明数据丢失风险
- 重命名字段必须说明兼容和回滚方案
- 不允许写无法回滚的迁移

---

## 典型交互流程

```text
用户：帮我根据 PRD 建立项目管理模块的数据库模型。

助手：请问生成的模型代码放在哪个目录下？
例如：backend/app/canvas/project/ 或 backend/app/studio/project/

用户：放在 backend/app/studio/project/

助手：好的，我先调研项目框架规范。

[助手读取现有模型代码，输出调研摘要]

助手：以下是项目框架规范摘要，确认后我开始设计。

[用户确认]

助手：输出设计文档到 backend/docs/NativePRD/项目管理/db-model-design.md
然后按框架规范在 backend/app/studio/project/ 下生成：
  - model/project.py
  - schema/project.py
  - crud/project_crud.py
  - service/project_service.py
  - api/v1/project.py
```

---

## 常见冲突处理

- OpenAPI 字段缺少 PRD 定义：先补 PRD，不直接落库。
- PRD 实体关系未在 OpenAPI 出现：可作为 DB 关系候选，但必须说明不暴露到接口。
- PRD 要 unique，现有数据可能重复：列出迁移风险和清洗方案。
- PRD 要删除数据，项目有软删除习惯：优先软删除并说明原因。
- OpenAPI 是 string format uri，DB 可用 String/Text，URL 格式由 Pydantic 校验。
- JSONB 结构稳定且需要查询：建议拆表，不要偷懒塞 JSONB。

---

## 检查清单

- [ ] 是否先询问用户代码落盘路径
- [ ] 是否调研了项目框架规范（Base/Schema/DAO/API 风格）
- [ ] 是否同时读取 PRD、OpenAPI、现有模型
- [ ] 是否输出字段映射表
- [ ] 是否区分落库字段、计算字段、临时输入字段
- [ ] 是否包含主键、外键、索引、唯一约束
- [ ] 是否对齐项目 SQLAlchemy 风格
- [ ] 是否说明 Pydantic 二次校验
- [ ] 是否包含 Alembic upgrade / downgrade 要求
- [ ] 是否创建了完整的 5 层目录结构（model/schema/crud/service/api）
- [ ] 是否列出迁移风险和待确认问题
