---
name: "backend-tdd-red-tests"
description: "根据 AI-Native PRD、BDD/Gherkin、OpenAPI 契约和数据库模型生成后端 pytest 红灯测试，只写测试不写实现。用于新增接口、加参数、修漏洞、权限边界、状态流转、数据库副作用、队列或外部调用副作用的 TDD 红灯阶段。"
---

# 后端 TDD 红灯测试生成 Skill

根据 AI-Native PRD、BDD/Gherkin、OpenAPI 契约、数据库模型设计生成后端 pytest 红灯测试。

核心判断：这个 skill 只负责写测试，不写业务实现。测试必须先失败，证明它真的能卡住缺口，然后才允许进入最少代码变绿阶段。

## 适用场景

- 用户要求根据 PRD / BDD / OpenAPI 生成后端测试
- 用户要进入 TDD 红灯阶段
- 用户要为新增接口、加参数、修漏洞、权限边界、状态流转补 pytest
- 用户要防止 AI 为了变绿而硬编码响应、伪造副作用或跳过真实业务

## 输入来源

必须读取：

- PRD：`backend/docs/NativePRD/<模块名>/<功能名称>.md`
- BDD：`backend/docs/功能测试场景/<模块名>/<功能名称>.feature`
- OpenAPI：`backend/docs/API规范/用户端.openapi.json`
- 数据库模型设计或现有 Model：`backend/app/**/model/*.py`
- 现有测试目录：`backend/docs/NativePRD/<模块名>/test_<模块名>.py` 或 `backend/tests/`
- 现有测试 fixtures：client、db_session、auth_headers、factory、mock queue 等

缺少 PRD：不能判断业务副作用和实体关系。
缺少 BDD：不能判断场景覆盖和状态码。
缺少 OpenAPI：不能判断 request/response 契约。
缺少 DB Model：不能断言数据库副作用。

## 输出目标

必须输出或写入：

1. pytest 测试文件路径
2. 场景到测试函数映射表
3. 红灯测试代码
4. 需要的 fixtures / factories / mocks 清单
5. 运行命令
6. 预期红灯原因

不允许输出 FastAPI 实现代码、SQLAlchemy 实现代码或业务 service 代码。

## 执行顺序

1. 读取 BDD，逐个 Scenario 建立测试用例。
2. 读取 OpenAPI，确定 endpoint、method、request payload、response schema、状态码。
3. 读取 PRD，确定业务副作用、实体关系、权限规则、状态流转、幂等/并发/外部依赖。
4. 读取 DB Model 或模型设计，确定要断言的数据库字段、状态和关联记录。
5. 阅读现有测试，复用项目 fixtures、factory、鉴权写法和 async 风格。
6. 输出场景到测试函数映射表。
7. 生成 pytest 红灯测试。
8. 测试必须覆盖 API 返回值和关键副作用。
9. 给出最小运行命令，例如 `pytest backend/docs/NativePRD/项目管理/test_项目管理.py -k xxx`。
10. 明确预期红灯原因，例如字段不存在、路由未接收、数据库未保存、权限未拦截。

## 测试必须断言什么

### 1. API 契约

- status_code
- response JSON 字段
- 错误码 / 错误 message
- response schema 中的关键字段

### 2. 数据库副作用

- 是否真实插入 / 更新 / 删除记录
- 字段值是否和请求一致
- 外键关系是否正确
- 状态字段是否正确流转
- 默认记录是否被创建，例如主画布、成员关系、任务记录

### 3. 业务副作用

- 队列任务是否触发
- 外部服务是否被调用
- 通知 / 日志 / 审计记录是否写入
- 配额是否扣减
- 幂等请求是否没有重复创建

### 4. 安全和边界

- 未登录 401
- 越权 403
- 参数非法 422
- 状态非法 409 / 422 / 404，按 OpenAPI 和 BDD 决定
- 重复提交 / 幂等
- 并发冲突
- 外部依赖失败

## 防 AI 恶意合规钉子

### 钉子一：断言副作用

测试不能只看 API 返回值。所有会写库、触发队列、调用外部服务、扣额度、创建默认数据的行为，都必须断言副作用。

### 钉子二：动态参数轰炸

测试数据不能永远是固定字符串。使用 `uuid.uuid4()`、`pytest.mark.parametrize`、边界值和多组输入，防止实现代码针对固定样例写 if-else。

### 钉子三：负向场景必须覆盖

每个 Feature 至少覆盖：

- Happy Path
- 参数非法
- 鉴权失败
- 越权访问
- 状态非法

如果 PRD / BDD 出现幂等、并发、队列、外部依赖，也必须生成对应测试。

### 钉子四：测试先红

生成测试后必须给出运行命令和预期失败点。红灯不是问题，是 TDD 入口。

## 场景到测试映射表

必须输出：

| BDD Scenario | 测试函数 | 类型 | 断言 API | 断言 DB | 断言副作用 | 预期红灯原因 |
|---|---|---|---|---|---|---|
| 用户创建项目成功 | test_create_project_success | Happy Path | 是 | 是 | 主画布/成员 | 路由未实现 |

## pytest 写法规则

- 沿用项目现有 async / sync 测试风格。
- 优先复用现有 fixtures，不发明新测试框架。
- 测试函数名必须表达业务场景。
- Payload 字段必须来自 OpenAPI。
- 断言字段必须能追溯到 PRD、BDD 或 OpenAPI。
- 不要 mock 被测业务本身。
- 可以 mock 外部不可控依赖，例如 LLM、支付、对象存储，但必须断言调用参数。
- 不要为了让测试通过而修改测试目标。

## 示例骨架

```python
import uuid

import pytest
from sqlalchemy import select


@pytest.mark.asyncio
async def test_create_project_success_creates_member_and_main_canvas(
    client,
    db_session,
    auth_headers,
):
    project_name = f"测试项目_{uuid.uuid4()}"

    response = await client.post(
        "/api/v1/projects",
        json={
            "name": project_name,
            "cover_image": "https://example.com/cover.png",
        },
        headers=auth_headers,
    )

    assert response.status_code == 201
    body = response.json()
    assert body["project_id"]
    assert body["main_canvas_id"]
    assert body["name"] == project_name

    project = await get_project_by_id(db_session, body["project_id"])
    assert project is not None
    assert project.name == project_name
    assert project.cover_image == "https://example.com/cover.png"

    member = await get_project_member(db_session, body["project_id"], auth_headers.user_id)
    assert member is not None
    assert member.role == "manager"

    canvas = await get_canvas_by_id(db_session, body["main_canvas_id"])
    assert canvas is not None
    assert canvas.project_id == project.project_id
    assert canvas.is_main is True
```

## 典型调用

```text
请根据以下文件生成后端 TDD 红灯测试：

PRD：backend/docs/NativePRD/项目管理/创建项目.md
BDD：backend/docs/NativePRD/项目管理/bdd/创建项目.feature
OpenAPI：backend/docs/API规范/工作台项目管理.openapi.json
DB Model：backend/app/canvas/project/model/project.py
测试落点：backend/docs/NativePRD/项目管理/test_项目管理.py

要求：
1. 只生成 pytest 测试，不写业务实现。
2. 每个 BDD Scenario 至少对应一个测试函数。
3. Happy Path 必须断言 API 返回、数据库写入和关键副作用。
4. 异常场景必须覆盖参数、鉴权、越权、状态边界。
5. 使用动态数据，避免固定样例。
6. 输出运行命令和预期红灯原因。
```

## 检查清单

- [ ] 是否读取 PRD / BDD / OpenAPI / DB Model / 现有测试
- [ ] 是否每个 Scenario 都映射到测试函数
- [ ] 是否断言 API 返回值
- [ ] 是否断言数据库副作用
- [ ] 是否断言队列 / 外部调用 / 配额 / 默认数据等关键副作用
- [ ] 是否覆盖负向场景
- [ ] 是否使用动态参数
- [ ] 是否给出运行命令
- [ ] 是否说明预期红灯原因
- [ ] 是否没有写业务实现代码
