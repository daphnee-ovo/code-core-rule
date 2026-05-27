# Core Rule

## 代码-文档双向追踪

代码与文档必须双向可追踪：代码文件头需要用内部树形图说明当前文件的类、函数、方法或核心流程，必要时加简短批注；能从命名直接看懂的节点不额外解释，避免注释噪音。代码文件头用 Markdown 链接指向关联文档，关联文档也反向链接到相关代码文件或代码目录。每次修改代码或文档时，都必须同步检查另一侧是否仍然准确，避免代码结构、行为逻辑、接口说明和文档索引脱节。

Few-shot:

```python
# File: permission_guard.py
#
# Internal Framework:
# permission_guard.py
# ├── PermissionMode
# ├── PermissionDecision          # allow / deny result
# ├── PermissionGuard
# │   ├── check_tool_call()
# │   ├── check_shell_command()
# │   └── resolve_permission_mode()
# └── build_permission_guard()
#
# Related Docs:
# - [Permission Rules](../../docs/permission.md)
# - [Core Rule](../../core_rule.md#代码-文档双向追踪)
```

```md
# Permission Rules

Related Code:

- [permission_guard.py](../src/agent/permission/permission_guard.py)
- [permission module](../src/agent/permission/)
```

## 需求确认优先

需求不清楚时必须先问清楚再开工。凡是会影响实现方向、修改范围、安全风险、执行成本或验收结果的不确定点，都不能擅自假设。

Few-shot:

```text
User: 给项目加一个权限系统。

Agent:
需要先确认：权限管 shell command、tool call，还是两者都管？默认模式是什么？拒绝后是中断还是重新规划？
```

反例：

```text
User: 给项目加一个权限系统。

Agent:
已默认实现三档权限，并接管所有 shell command。
```

## 简单优先

能简单实现就不要复杂化。优先选择直接、可读、容易调试的方案，避免为了“架构感”引入不必要的抽象、框架、层级或算法复杂度。每个程序、模块或函数应尽量只承担一个清晰职责，并把这件事做好。

Few-shot:

```text
Task: 读取一个 JSON 文件，筛选 active=true 的记录并输出。

Good:
写一个清晰流程：load_json() -> filter_active() -> write_result()。

Bad:
引入插件系统、任务调度器、抽象基类、动态注册机制，只为完成一次简单筛选。
```
