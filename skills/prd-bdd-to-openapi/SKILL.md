---
name: "prd-bdd-to-openapi"
description: "根据 AI-Native PRD 和 BDD/Gherkin 场景共同生成或更新 OpenAPI 契约。PRD 提供实体字段、类型、约束、默认值、关系和业务含义；BDD 提供接口行为、路径、状态码、错误场景和验收条件。当用户要求根据 PRD/BDD 生成 OpenAPI、更新 openapi.yaml/openapi.json、锁接口契约、补请求体或响应体字段时触发。"
---

# PRD 与 BDD 生成 OpenAPI Skill

把 AI-Native PRD 和 BDD 场景合并成严格 OpenAPI 契约。

核心判断：OpenAPI 不能只靠 BDD 生成。BDD 锁行为，PRD 锁数据模型。只给 BDD，AI 容易缺失字段类型、长度、nullable、默认值、实体关系、响应体结构和错误码细节。

---

## 输入来源

必须同时读取：

- `backend/docs/NativePRD/<模块名>/<功能名称>.md`
- `backend/docs/NativePRD/<模块名>/bdd/<功能名称>.feature`
- 现有 OpenAPI 文件，例如 `backend/docs/API规范/用户端.openapi.json`

如果缺少 PRD，只能生成接口骨架，不能补全字段 Schema。

如果缺少 BDD，只能生成 Schema 草案，不能确定完整状态码、错误场景和行为路径。

---

## 分工原则

| 来源 | 负责内容 |
|---|---|
| PRD | 实体、字段、类型、必填、nullable、长度、格式、枚举、默认值、关系、业务含义、requestBody/responseBody 候选 |
| BDD | 用户场景、路径行为、前置条件、成功/失败状态码、权限边界、状态边界、错误响应、验收断言 |
| 现有 OpenAPI | 已有 path、component 命名、错误响应格式、分页格式、认证方式、版本规范 |

---

## 执行顺序

1. 读取 PRD，提取实体字段表和业务约束。
2. 读取 BDD，提取 Scenario、状态码、错误码、输入输出断言。
3. 读取现有 OpenAPI，复用已有 components、ErrorResponse、分页模型和安全方案。
4. 建立字段映射表：PRD 字段 -> OpenAPI schema 字段 -> request/response 位置。
5. 生成或更新 path / method / operationId / tags。
6. 生成 requestBody，只包含本接口真正接收的字段。
7. 生成 responses，覆盖 BDD 中出现的所有成功和失败状态码。
8. 生成 components.schemas，字段约束必须来自 PRD，行为错误必须来自 BDD。
9. 检查不得新增 PRD 和 BDD 都没有依据的字段。
10. 输出变更摘要和待确认问题。

---

## 生成规则

### 字段规则

- 字段名以 PRD 为准。
- 字段类型、长度、format、nullable、enum、默认值以 PRD 为准。
- requestBody 是否必填，以 PRD 字段约束和 BDD 场景共同决定。
- responseBody 字段必须能从 PRD 实体或 BDD Then 断言追溯。
- OpenAPI 中不得出现 PRD 没有定义、BDD 没有断言的业务字段。

### 行为规则

- path / method 优先来自 BDD 场景或现有 OpenAPI 习惯。
- 状态码以 BDD 为准。
- 错误码以 BDD 或 PRD 拦截铁网为准。
- 如果 PRD 和 BDD 冲突，停止生成并列出冲突，不要自行裁决。

### 复用规则

- 复用现有 `components.schemas.ErrorResponse`。
- 复用现有认证 scheme。
- 复用已有分页、列表、ID、时间戳、用户摘要等通用 schema。
- 不要重复创建含义相同但名字不同的 schema。

---

## 输出格式

必须输出：

1. 读取的 PRD 文件路径
2. 读取的 BDD 文件路径
3. 更新的 OpenAPI 文件路径
4. 字段映射表
5. OpenAPI patch 或完整 YAML/JSON 片段
6. 冲突与待确认问题

字段映射表格式：

| PRD 字段 | 类型/约束 | requestBody | responseBody | 来源依据 |
|---|---|---|---|---|
| cover_image | string, uri, nullable | 是 | 是 | PRD 字段 + BDD 响应断言 |

---

## 典型调用

```text
请根据以下文件更新 OpenAPI：

PRD：backend/docs/NativePRD/项目管理/创建项目.md
BDD：backend/docs/NativePRD/项目管理/bdd/创建项目.feature
OpenAPI：backend/docs/API规范/用户端.openapi.json

要求：
1. PRD 负责字段、类型、约束、nullable、默认值和业务含义。
2. BDD 负责接口行为、状态码、错误场景和验收断言。
3. 只能新增 PRD 或 BDD 有依据的字段。
4. 如果 PRD 与 BDD 冲突，列出冲突并停止。
5. 输出字段映射表和 OpenAPI 更新片段。
```

---

## 常见冲突处理

- PRD 有字段，BDD 没有使用：可以放入 schema 候选，但不要自动放入当前 requestBody，除非该接口确实需要。
- BDD 使用字段，PRD 没有定义：停止，要求先补 PRD。
- BDD 有状态码，PRD 拦截铁网没有错误码：保留状态码，错误码标待确认。
- PRD 说字段 optional，BDD 要求缺失时报 422：冲突，需确认。
- 现有 OpenAPI 与 PRD 字段命名不同：优先保留现有兼容性，并输出迁移风险。

---

## 检查清单

- [ ] 是否同时读取 PRD 和 BDD
- [ ] 字段类型和约束是否来自 PRD
- [ ] 状态码和错误场景是否来自 BDD
- [ ] 是否复用现有 components
- [ ] 是否列出字段映射表
- [ ] 是否没有脑补业务字段
- [ ] PRD / BDD / OpenAPI 冲突是否列出
- [ ] 输出是否能直接用于前端类型生成和后端校验
