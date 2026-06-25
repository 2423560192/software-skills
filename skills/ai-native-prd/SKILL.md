---
name: "ai-native-prd"
description: "把原始想法、聊天记录或松散需求整理成按模块落盘的 AI-Native PRD，输出模块目录、Feature 文件、实体与属性、行为动作链、场景矩阵、拦截铁网、待确认问题和不做范围，供后续 BDD 和 OpenAPI 生成使用。当用户要求先写 PRD、规范需求、按模块拆需求、整理原始想法、把口头需求变成结构化需求，或在加参数/修漏洞前先对齐需求时触发。"
---

# AI-Native PRD 规范

把原始想法、聊天记录、会议纪要或松散需求整理成“按模块落盘”的 AI-Native PRD。PRD 是后续 BDD / OpenAPI / TDD 的上游事实来源，不是一篇大散文。

核心原则：先拆模块，再拆 Feature；模块是目录，Feature 是 PRD 文件。每次生成 PRD 时，必须写入或规划到 `backend/docs/NativePRD/<模块名>/` 的不同模块目录。

---

## 适用场景

- 用户给出聊天记录、会议纪要、口述需求、粗糙 PRD
- 用户要求先写 PRD，再拆 BDD / OpenAPI
- 用户希望 PRD 先按模块拆开，再分别落到 `backend/docs/NativePRD/` 的不同目录
- 用户要新增参数、改流程、修漏洞，想先把需求和场景写稳
- 用户给出一组功能，例如项目管理、成员管理、画布管理，需要拆成可独立交付的 Feature

---

## 输出目标

一次 PRD 生成必须产出两层内容：

1. 模块层：识别模块边界，并为每个模块规划目录。
2. Feature 层：每个 Feature 单独一份 PRD 文件，文件内预埋 BDD-ready 场景矩阵。

每个 Feature PRD 必须包含：

- 功能一句话
- 业务实体与属性
- 行为动作链
- 场景矩阵（BDD 预埋）
- 拦截铁网
- 待确认问题
- 不做范围
- 文件落点

---

## 粒度规则

标准粒度：一个可独立交付的 Feature 一份 PRD 文件；一个业务模块是目录，不是一份大 PRD。

判断一个动作是否需要单独 PRD，看它是否满足任意一条：

- 有独立入口或独立接口
- 有独立 BDD 场景
- 有独立权限、参数校验或状态流转
- 有独立数据库写入或副作用
- 可以单独测试、单独上线、单独 Commit

项目 CRUD 通常不要塞进一份 PRD，建议拆成：

```text
backend/docs/NativePRD/项目管理/
  README.md
  创建项目.md
  更新项目.md
  删除项目.md
  查询项目列表.md
  查询项目详情.md
```

拆分理由：

- 创建项目通常有创建者、成员角色、默认主画布等副作用
- 更新项目通常涉及字段校验、管理权限和部分更新
- 删除项目通常涉及软删除、级联影响、越权拦截和状态限制
- 查询列表通常涉及分页、筛选、排序和只看有权限的数据
- 查询详情通常涉及资源可见性、成员权限和不存在 / 已删除状态

例外：如果只是后台字典表这类低风险纯 CRUD，且四个动作共享同一套权限和实体，可以先写成一份 `字典项CRUD.md`，但场景矩阵里仍然要分开列 create / update / delete / list / detail。只要后续某个动作逻辑变复杂，就立即拆成独立 PRD。

---

## 执行顺序

1. 读取原始需求，先识别模块，而不是直接写单篇 PRD。
2. 输出模块清单，给每个模块分配中文名称作为目录名。
3. 为每个模块规划目录：`backend/docs/NativePRD/<模块名>/`。
4. 为每个模块写 `README.md`，说明模块职责、包含的 Feature、上下游依赖。
5. 为每个 Feature 写单独 PRD 文件：`backend/docs/NativePRD/<模块名>/<功能名称>.md`。
6. 在每份 Feature PRD 中写实体与字段。
7. 写行为动作链，明确角色、触发、输入、写入、副作用、响应、状态变化。
8. 写场景矩阵，覆盖正常、异常、权限、状态、幂等、并发、外部依赖。
9. 写拦截铁网，明确状态码、错误码、前端表现。
10. 标待确认问题，不脑补不确定规则。
11. 标不做范围，避免 AI 顺手扩展。
12. 不要直接写 BDD、OpenAPI 或代码。

如果用户指定当前项目路径，默认应直接创建或覆盖对应 PRD 文件；如果只是在讨论模板，则只输出文件规划和 PRD 内容。

---

## 模块化落盘规则

推荐目录结构：

```text
backend/docs/NativePRD/
  index.md
  <模块名>/
    README.md
    <功能名称>.md
```

示例：

```text
backend/docs/NativePRD/
  index.md
  项目管理/
    README.md
    创建项目.md
    更新项目封面.md
  项目成员/
    README.md
    邀请成员.md
    修改成员角色.md
  画布/
    README.md
    创建主画布.md
```

落盘规则：

- `backend/docs/NativePRD/index.md`：全局 PRD 索引，列出所有模块和 Feature。
- `backend/docs/NativePRD/<模块名>/README.md`：模块总览，写模块职责、实体、依赖和 Feature 列表。
- `backend/docs/NativePRD/<模块名>/<功能名称>.md`：单个 Feature 的 AI-Native PRD。
- 一个模块下可以有多个 Feature 文件。
- 一个 Feature 不要跨多个模块写；如果确实跨模块，写清上游依赖和下游影响。
- 模块目录名和 Feature 文件名统一使用中文，与项目其他中文文档命名风格一致。
- 如果只生成一个模块，也必须使用模块目录，不允许把 PRD 直接放在 `backend/docs/NativePRD/` 根目录。

---

## PRD 文件标准结构

```markdown
# Feature: [功能名称]

文件落点：backend/docs/NativePRD/<模块名>/<功能名称>.md
所属模块：[模块名]
上游依赖：[无 / 依赖的模块与数据]
下游输出：[会被哪些模块使用]

## 0. 功能一句话

## 1. 业务实体与属性

## 2. 行为动作链

## 2.5 场景矩阵（BDD 预埋）

## 3. 拦截铁网

## 4. 待确认问题

## 5. 不做范围
```

---

## 模块 README 标准结构

```markdown
# Module: [模块名]

## 1. 模块职责

## 2. 核心实体

## 3. Feature 列表

| Feature | 文件 | 优先级 | 上游依赖 | 下游影响 |
|---|---|---|---|---|
| [功能名] | ./[功能名称].md | P0 | [依赖] | [影响] |

## 4. 模块边界

## 5. 模块依赖
```

---

## 规范细则

### 1. 业务实体与属性

必须写清：

- 字段名
- 类型（string / int / boolean / JSONB / datetime / enum / uuid）
- 必填 / 可选 / nullable
- 长度 / 格式 / 枚举 / 默认值
- 关系（外键、所属父实体）/ 索引 / 唯一性
- 备注（中文说明，解释字段业务含义）
- 是否进入 OpenAPI requestBody / responseBody

示例：

| 实体 | 字段 | 类型 | 必填 | 约束 | 备注 |
|---|---|---|---|---|---|
| Project | title | string | 是 | 1-255 字符 | 项目名称 |
| Project | description | string | 否 | UniversalText | 项目简介，可为空 |
| Project | owner_id | int | 是 | 外键 -> User, INDEX | 项目所有者用户 ID |
|-|-|-|-|-|-|
| Canvas | title | string(255) | 自动 | 默认 "主画布" | 根画布名称 |
| Canvas | project_id | int | 自动 | FK -> Project, INDEX | 所属项目 ID |

### 2. 行为动作链

必须写清：

- 角色
- 触发动作
- 前置条件
- 输入
- 请求或操作
- 校验
- 写入
- 副作用
- 响应
- 状态变化

按以下链路展开，每步写清楚：

```text
角色：谁发起操作
  -> 触发动作：用户点了什么 / 调了什么
    -> 输入：请求参数、请求体结构
      -> 校验：参数验证、权限验证、状态验证
        -> 写入：修改了哪些表、哪些字段
          -> 副作用：触发了什么异步任务、通知、日志
            -> 响应：返回给前端的数据结构
              -> 状态变化：操作前后实体状态的变化
```

示例：

```text
角色：项目所有者(manager)
  -> 触发：点击“邀请成员”
    -> 输入：{ user_id: "123", role: "editor" }
      -> 校验：user_id 是否存在、是否已是成员、role 是否合法
        -> 写入：INSERT INTO drama_project_member(project_id, user_id, role)
          -> 副作用：发送通知给被邀请者
            -> 响应：{ code: 200, data: { member_id, user_id, role } }
              -> 状态变化：项目 member_count +1
```

### 3. 场景矩阵（BDD 预埋）

PRD 中必须先定义不同场景，但不要直接写完整 Gherkin。场景矩阵是 BDD 的上游结构。

| 场景 | 类型 | 角色 | 前置条件 | 触发 | 输入 | 副作用 | 期望结果 | 状态码 | 错误码 | BDD 文件 |
|---|---|---|---|---|---|---|---|---|---|---|
| 创建成功 | Happy Path | 已登录用户 | Token 有效 | 提交表单 | 合法参数 | 写库 | 返回 ID | 201 | - | 创建项目.feature |

至少覆盖：

- Happy Path
- 参数非法
- 鉴权失败
- 越权访问
- 状态非法
- 幂等 / 重复提交
- 并发冲突
- 外部依赖失败

### 4. 拦截铁网（8 类）

必须写清处理策略，推荐格式：

| 类型 | 场景 | 拦截条件 | 后端响应 | 错误码 | 前端表现 |
|---|---|---|---|---|---|
| 参数 | 项目名超长 | name > 255 | 422 | ERROR_PROJECT_NAME_INVALID | 表单提示 |
| 鉴权 | 未登录 | Token 缺失 | 401 | - | 跳转登录 |
| 越权 | viewer 编辑 | role != manager/editor | 403 | - | 按钮置灰 |
| 状态 | 已删除再编辑 | deleted_at != null | 404 | - | 提示不存在 |
| 幂等 | 重复提交 | 同 user + 同 project | 400 | ERROR_DUPLICATE | 提示已存在 |
| 配额 | 超限创建 | count >= max | 429 | ERROR_QUOTA_EXCEEDED | 提示上限 |
| 并发 | 两人同改节点 | version 冲突 | 409 | ERROR_VERSION_CONFLICT | 提示刷新 |
| 外部依赖 | AI 超时 | 第三方无响应 | 503 | ERROR_UPSTREAM_TIMEOUT | 提示重试 |

每一项必须给出明确处理策略：拒绝 / 排队 / 降级 / 重试 / 忽略。

### 5. 待确认问题

遇到需求文档中含糊不清的逻辑时，必须在对应位置标注：

```text
[待确认: 允许空内容的剧本吗？如果允许，AI 角色提取如何处理？]
[待确认: 编辑分享链接时 require_login=false 与 permission=edit 冲突，具体 error 文案是什么？]
```

### 6. 不做范围

明确排除：

- 用户提过但明确说“以后再做”的功能
- 容易想当然加上的功能，例如自动保存、撤销重做、版本对比
- 不属于当前 PRD 范围的技术实现细节，例如加缓存、换消息队列

---

## 典型调用

```text
请把下面的原始想法整理成 AI-Native PRD。

要求：
1. 先按模块拆分。
2. 每个模块落到 backend/docs/NativePRD/<模块名>/。
3. 每个 Feature 单独一份 PRD 文件。
4. 每份 PRD 必须包含实体与属性、行为动作链、场景矩阵、拦截铁网、待确认问题和不做范围。
5. 场景矩阵要定义 Happy Path、参数非法、鉴权、越权、状态、幂等、并发、外部依赖失败等场景。
6. 不要直接写 BDD、OpenAPI 或代码。

原始想法：
[粘贴需求]
```

---

## 输出时必须附带的文件清单

每次完成 PRD 规范化后，必须输出计划写入或已经写入的文件清单：

```text
backend/docs/NativePRD/index.md
backend/docs/NativePRD/<模块名>/README.md
backend/docs/NativePRD/<模块名>/<功能名称>.md
```

如果只生成一个模块，也必须使用模块目录，不允许把 PRD 直接放在 `backend/docs/NativePRD/` 根目录。

---

## 下游接力

PRD 完成后，交给 **需求文档拆解为 BDD 模块 Skill** (`bdd-breakdown`)，继续把每个模块 PRD 里的“场景矩阵”翻成 Gherkin。

完整流水线：

```text
原始想法 -> AI-Native PRD -> BDD Gherkin -> OpenAPI -> TDD -> 代码 -> Commit
   ↑              ↑              ↑
 本 Skill     bdd-breakdown  已有流程
```

下游约定：

```
backend/docs/NativePRD/<模块名>/<功能名称>.md
  -> backend/docs/NativePRD/<模块名>/bdd/<功能名称>.feature
  -> backend/docs/API规范/用户端.openapi.json
```

---

## 与本项目文件路径的映射

| 环节 | 输出位置 |
|---|---|
| AI-Native PRD 索引 | `backend/docs/NativePRD/index.md` |
| 模块 README | `backend/docs/NativePRD/<模块名>/README.md` |
| Feature PRD | `backend/docs/NativePRD/<模块名>/<功能名称>.md` |
| BDD Gherkin | `backend/docs/NativePRD/<模块名>/bdd/<功能名称>.feature` |
| 场景文档 | `backend/docs/功能测试场景/工作台/<模块>/<模块>场景.md` |
| OpenAPI | `backend/docs/API规范/用户端.openapi.json` |
