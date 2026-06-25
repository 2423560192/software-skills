---
name: "bdd-breakdown"
description: "将 PRD/需求拆解为 BDD Gherkin 场景。当用户要求写 BDD、拆需求、写 .feature 文件时触发。BDD 文件存放在 PRD 对应模块的 bdd/ 子目录下。"
---

# BDD Breakdown

## 职责

将 PRD 文档或 Feature 描述拆解为符合 Gherkin 语法的 BDD 场景（`.feature` 文件），覆盖正常路径、异常路径和权限边界。

## BDD 文件存放位置

**BDD `.feature` 文件必须放在 PRD 对应模块的 `bdd/` 子目录下**，与 PRD 文档保持在同一模块目录中。

### 路径规则

```
backend/docs/NativePRD/<模块名>/bdd/<Feature名>.feature
```

**示例：**

| PRD 文件 | BDD 文件 |
|----------|----------|
| `backend/docs/NativePRD/项目管理/创建项目.md` | `backend/docs/NativePRD/项目管理/bdd/创建项目.feature` |
| `backend/docs/NativePRD/画布/保存快照.md` | `backend/docs/NativePRD/画布/bdd/保存快照.feature` |

### 目录结构示例

```
backend/docs/NativePRD/
├── 项目管理/
│   ├── README.md
│   ├── 创建项目.md
│   ├── 删除项目.md
│   └── bdd/
│       ├── 创建项目.feature
│       ├── 删除项目.feature
│       └── 更新项目.feature
├── 画布/
│   ├── README.md
│   ├── 保存快照.md
│   └── bdd/
│       ├── 保存快照.feature
│       └── 查询画布详情.feature
└── index.md
```

## BDD 文件格式规范

每个 `.feature` 文件必须包含：

```gherkin
# BDD: <Feature 名称>
# 来源: backend/docs/NativePRD/<模块名>/<Feature名>.md

Feature: <Feature 名称>
  As a <角色>
  I want to <功能>
  So that I can <业务价值>

  Background: 用户已登录
    Given 用户 "creator_user" 已登录且持有有效 JWT

  Scenario: 正常路径 - <描述>
    Given <前置条件>
    When <操作>
    Then <预期结果>

  Scenario: 异常路径 - <描述>
    When <操作>
    Then 系统返回 code=4xx
    And 响应 msg 中包含校验提示

  Scenario: 权限边界 - <描述>
    Given <未登录或无权限>
    When <操作>
    Then 系统返回 code=401/403
```

### 必须覆盖的场景类型

1. **正常路径**：最少 1 个，覆盖核心业务流程
2. **异常路径**：参数校验失败、空值、超长等
3. **权限边界**：未登录、无权限用户操作
4. **业务约束**：如唯一性校验、级联删除等

## 来源标注

每个 `.feature` 文件第一行必须用注释标注来源 PRD 文件的相对路径：

```
# 来源: backend/docs/NativePRD/<模块名>/<Feature名>.md
```

## BDD 文件命名规范

- 文件名与 PRD Feature 同名，使用中文
- 扩展名为 `.feature`
- 使用短横线连接多个词（如有英文）
