# Core Rule

## 1. 需求与变更边界

### 需求确认优先

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

### 最小惊讶原则

程序行为应符合用户的直觉预期：命令、参数、默认行为、输出格式和修改范围都应尽量与同类工具保持一致。选项命名要直观，例如 `-v` 表示 `verbose`，`-h` 表示 `help`。除非用户明确要求，否则不要主动重构、改风格、改接口、改无关文件或扩大修改范围。

Few-shot:

```text
Task: 给 CLI 增加详细日志开关。

Good:
新增 -v / --verbose，并保持原有默认输出不变。

Bad:
新增 --debug-mode-plus，同时默认开启详细日志，顺便重构整个输出格式。
```

## 2. 文档与长期维护

### 代码-文档双向追踪

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

## 3. 实现复杂度控制

### 简单优先

能简单实现就不要复杂化。优先选择直接、可读、容易调试的方案，避免为了“架构感”引入不必要的抽象、框架、层级或算法复杂度。每个程序、模块或函数应尽量只承担一个清晰职责，并把这件事做好。

Few-shot:

```text
Task: 读取一个 JSON 文件，筛选 active=true 的记录并输出。

Good:
写一个清晰流程：load_json() -> filter_active() -> write_result()。

Bad:
引入插件系统、任务调度器、抽象基类、动态注册机制，只为完成一次简单筛选。
```

### 充分利用软件杠杆

优先复用成熟、可靠、维护良好的现有工具、库和系统能力，不要重新发明轮子。只有在现有方案过重、不稳定、不满足核心需求，或会引入明显维护成本时，才考虑自实现；复用时也要避免为了小需求引入大依赖。

Few-shot:

```text
Task: 解析 YAML 配置文件。

Good:
使用成熟 YAML parser，并封装成轻量 config_loader。

Bad:
手写一个半成品 YAML parser，只支持部分语法，后续不断补漏洞。
```

### 组合优于集成

不要一次性制造大而全的程序。优先把能力拆成职责清晰、接口统一、可独立替换的组件，再通过稳定协议串联起来。模块之间应依赖清晰输入输出，而不是互相嵌死在内部实现里，保证后续能拆卸、替换、测试和维护。

Few-shot:

```text
Task: 做一个数据处理流程。

Good:
reader -> cleaner -> transformer -> writer
每一步用统一输入输出衔接，可以单独替换 CSV reader、DB reader 或 Excel writer。

Bad:
写一个 mega_pipeline()，里面同时负责读文件、清洗、转换、写入、日志、异常处理和配置解析，任何一步变化都要改整个函数。
```

## 4. 接口与模块边界

### 接口契约稳定

模块之间必须通过清晰、稳定、可预测的接口协作。输入、输出、错误、默认值和副作用都应明确，避免隐式依赖、隐藏状态和随意变更接口；必须改接口时，应同步更新调用方、文档和测试。

Few-shot:

```text
Task: 修改 user_loader 的返回结果。

Good:
保持原有返回字段不变，新增字段向后兼容；若必须改字段名，同步更新调用方、文档和测试。

Bad:
把 user_id 改成 id，默认值从 null 改成空字符串，但不更新调用方和测试。
```

### 明确约束优先

代码应优先使用语言和类型系统提供的明确建模能力，避免使用绕开约束、隐藏真实类型或依赖运行时约定的实现方式。只有在性能、FFI、插件边界、动态扩展等确有必要，且没有更清晰的建模方案时，才允许使用弱约束能力；使用时必须把范围压到最小，并说明原因、风险和边界。

Few-shot:

```text
Task: 根据不同任务类型执行不同逻辑。

Good:
使用明确的类型、枚举、接口或 trait 建模任务类型和行为。

Bad:
把所有任务塞进不透明容器，运行时再靠字符串、downcast 或约定判断真实类型。
```

```rust
// Good
enum Task {
    Fetch(FetchTask),
    Parse(ParseTask),
    Export(ExportTask),
}

// Bad
let task: Box<dyn Any> = Box::new(fetch_task);
```

## 5. 错误处理与失败策略

### 透明报错

程序必须清晰报告错误，禁止静默失败。失败时应说明发生了什么、可能原因、影响范围和下一步处理方式；不要吞掉异常、返回模糊结果，或让用户误以为任务已经成功。

Few-shot:

```text
Task: 读取配置文件。

Good:
Error: config.yaml not found. Expected path: ./config/config.yaml. Please create the file or pass --config <path>.

Bad:
程序继续运行，使用空配置，最后输出一个不完整结果。
```

### 快速失败

如果程序完整运行依赖特定配置、组件、服务、权限或外部资源，就必须在启动阶段或任务开始前完成检查，并在依赖缺失时立即失败并说明原因。禁止等到用户调用某个功能时才暴露依赖问题，避免服务表面可用、实际功能残缺。

Few-shot:

```text
Task: 启动一个包含导出功能的服务，导出功能依赖 LibreOffice。

Good:
服务启动时检查 LibreOffice 是否存在；如果缺失，直接报错：
Error: LibreOffice is required for export feature but was not found.

Bad:
服务正常启动，直到用户点击“导出 PDF”时才报错：
Command not found: libreoffice
```

### 禁止强行兜底

不要用默认值、空结果、宽泛异常捕获或隐式降级掩盖真实问题。只有在兜底行为是明确需求、可解释、可观测且不会误导用户时，才允许兜底；否则应透明报错或快速失败。

Few-shot:

```text
Task: 读取用户指定的数据文件。

Good:
文件不存在时直接报错，说明缺失路径和处理方式。

Bad:
文件不存在时自动返回空列表，让后续流程继续运行并产出空结果。
```

## 6. 测试约束

### 信任测试

默认信任现有测试代码是正确的，除非有明确证据表明测试已经因重大业务变更、接口变更或实现边界变化而失真。不要因为实现不通过测试就直接改测试；应先检查实现是否违反预期。若判断测试本身可能过时或需求不清楚，必须先向用户确认。

Few-shot:

```text
Task: 修改登录逻辑后，原有 auth 测试失败。

Good:
先检查实现是否破坏了既有登录约束；只有在确认需求已改变时，才提出是否需要更新测试。

Bad:
直接修改测试断言，让新实现通过。
```
