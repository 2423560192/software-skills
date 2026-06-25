---
name: "python-uv-run"
description: "确保通过 uv run 执行 Python 程序。当涉及 Python 脚本、测试、模块、CLI 或 Python 相关命令时调用。"
---

# Python uv run

当任务涉及工作区中的任何 Python 程序、脚本、模块、测试、CLI、管理命令或基于 Python 的工具时，均通过 `uv run` 执行。

## 必需行为

- 使用 `uv run python ...` 替代 `python ...`。
- 使用 `uv run pytest ...` 替代 `pytest ...`。
- 使用 `uv run ruff ...` 替代 `ruff ...`。
- 使用 `uv run mypy ...` 替代 `mypy ...`。
- 使用 `uv run <python-tool> ...` 执行基于 Python 的工具和控制台脚本。
- 执行模块时，使用 `uv run python -m <module> ...`。
- 执行以 `.py` 结尾的项目脚本时，使用 `uv run python path/to/script.py ...`。

## 示例

```bash
uv run python backend/scripts/test_asset_api.py
uv run python -m backend.some_module
uv run pytest
uv run pytest tests/test_example.py
uv run ruff check .
uv run mypy .
```

## 适用范围

除非用户明确要求为特定命令使用其他运行器，否则本规则适用于工作区中所有与 Python 相关的执行。
