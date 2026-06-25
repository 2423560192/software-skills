---
alwaysApply: true
scene: git_message
---

# Git Commit Message 生成规则

## 1. 提交信息格式

必须遵循 **Conventional Commits** 规范，格式如下：

```
<type>(<scope>): <subject>

<body>

<footer>
```

- `type`（必填）：提交类型，见下方类型定义
- `scope`（可选）：影响范围，如 `video`, `auth`, `api`, `ui` 等
- `subject`（必填）：简短描述，使用中文，不超过 50 个字符
- `body`（可选）：详细描述，说明修改原因和实现方式
- `footer`（可选）：关联 Issue 或破坏性变更说明

## 2. 提交类型 (type)

根据项目 `do_release.py` 的发布脚本分类，使用以下类型：

| 类型 | 用途 | 对应 Release 分类 |
|------|------|------------------|
| `feat` | 新增功能、特性 | 🚀 新增功能 (Features) |
| `fix` | 修复 Bug、问题 | 🐛 问题修复 (Bug Fixes) |
| `refactor` | 代码重构（不改变功能） | 🔧 代码重构 (Refactor) |
| `perf` | 性能优化 | ⚡ 性能优化 (Performance) |
| `docs` | 文档更新 | 📝 文档更新 (Documentation) |
| `style` | 代码格式、样式调整（不影响功能） | 🎨 代码样式 (Style) |
| `test` | 测试相关 | ✅ 测试 (Tests) |
| `chore` | 构建工具、依赖更新、杂项 | 🔨 构建/工具 (Chore) |
| `ci` | CI/CD、持续集成配置 | 🔧 持续集成/发布 (CI/Release) |
| `build` | 构建系统相关 | 🔧 持续集成/发布 (CI/Release) |

## 3. 提交信息语言

- **Subject 使用中文**，简洁明了
- 技术词汇优先使用中文，参考项目翻译映射：
  - add → 添加
  - implement → 实现
  - fix → 修复
  - update → 更新
  - remove/delete → 移除/删除
  - refactor → 重构
  - improve/enhance → 优化/增强
  - ensure → 确保
  - bump → 升级

## 4. 编写规范

### Subject 规则
1. 使用祈使句，描述「做了什么」而非「做了」
2. 首字母不大写，末尾不加句号
3. 长度控制在 50 字符以内

**示例：**
- ✅ `feat(video): 添加有声视频生成支持`
- ✅ `fix(auth): 修复 Token 过期未刷新问题`
- ✅ `refactor(api): 重构视频生成请求体构建逻辑`
- ❌ `feat: Added video support` （使用英文）
- ❌ `fix: 修复了一个bug。` （末尾有句号）

### Scope 规则
- 使用模块/功能名称作为 scope
- 常见 scope：`video`, `image`, `text`, `auth`, `api`, `ui`, `db`, `config`, `script`
- 如果影响多个模块，可省略 scope

### Body 规则
- 说明「为什么修改」和「怎么修改」
- 每行不超过 72 个字符
- 使用列表说明多项变更

### Footer 规则
- 关联 Issue：`Closes #123`, `Fixes #456`
- 破坏性变更：`BREAKING CHANGE: 描述`

## 5. 完整示例

```
feat(video): 添加 Seedance 有声视频生成支持

- 在 video_generate.jsonc 中添加 extra 字段映射
- SeedanceArkContentBuilder 支持读取 generate_audio 参数
- video_models.jsonc 路由映射添加 generate_audio 字段

Closes #2168543721228861440
```

```
fix(api): 修复视频生成请求体丢失 extra 参数

build_standard_request 时未映射 extra 字段，导致 generate_audio
等扩展参数被丢弃。已在 video_generate.jsonc fields 中添加映射。
```

```
chore(release): 发布 v1.2.0

- 升级 VERSION 文件
- 更新 CHANGELOG.md
```

## 6. 特殊场景

### 版本发布提交
版本发布使用专门的提交格式：
```
chore(release): release v{version}
```

### 多人协作提交
如果是 pair programming，可在 footer 添加：
```
Co-authored-by: 姓名 <email@example.com>
```

### Revert 提交
```
revert: feat(video): 添加有声视频生成支持

This reverts commit abc123.
```
