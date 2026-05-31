# Core Rule

## 0. 安全与执行优先级

### 外部安全优先

外部安全系统的拦截和禁令具有最高执行优先级，包括 hooks、权限系统、沙箱、CI 安全检查、策略引擎等。Agent 不得质疑、绕过、削弱或重新评估其合理性；被拦截后只能停止相关操作，并按安全系统给出的信息向用户说明原因。

Few-shot:

```text
Task: 执行 rm -rf 某个目录。

Hook:
Blocked: destructive command requires explicit approval.

Good:
停止执行，并告知用户该操作被安全系统拦截，需要按规则处理。

Bad:
尝试改写命令、换路径、拆分步骤，绕过 hook 继续删除。
```

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

### 范围升级需确认

执行中途发现实际影响范围显著超出初始预期时，必须暂停并向用户说明情况，不得擅自扩大执行范围。Agent 可以说明根因、影响范围、风险和可选方案，但必须等待用户确认后，才能从小修复升级为大重构、接口改造或跨模块重写。

Few-shot:

```text
Task: 修复 parser 的一个边界 case bug。

Good:
发现根因涉及 parser 整体状态管理后，暂停继续扩大修改，并说明：
A. 最小修复：只修当前 bug，风险较低。
B. 根治修复：重构 parser 模块，影响接口、调用方和测试。
等待用户确认后再继续。

Bad:
发现根因较深后，直接重构整个 parser，并修改调用方、接口和测试。
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

能简单实现就不要复杂化。代码首先是给人读的，其次才是给机器运行的；优先选择直接、可读、容易调试的方案，避免炫技式写法，以及为了“架构感”引入不必要的抽象、框架、层级或算法复杂度。每个程序、模块或函数应尽量只承担一个清晰职责，并把这件事做好。

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

不要一次性制造大而全的程序。新增功能时，优先创建职责清晰的新模块，并通过稳定接口与现有模块组合；不要为了方便把新逻辑堆进旧程序、旧函数或既有主流程。模块之间应依赖清晰输入输出，而不是互相嵌死在内部实现里，保证后续能拆卸、替换、测试和维护。

Few-shot:

```text
Task: 给数据处理流程增加 Excel 导出能力。

Good:
新增 excel_writer，并接入既有 reader -> cleaner -> transformer -> writer 流程；保持原有 reader / cleaner / transformer 边界不变。

Bad:
把 Excel 导出、格式处理、路径判断和异常处理都塞进原来的 transformer，让主流程越来越臃肿。
```

## 4. 接口与模块边界

### 高内聚低耦合原则

模块内部应保持高内聚，围绕单一职责组织代码；模块外部应保持低耦合，通过清晰、稳定、可预测的接口、协议、数据结构或标准协作，而不是依赖彼此的具体实现逻辑。应优先提供可组合、可替换的机制，而不是把策略写死在主流程里；必须改接口时，应同步更新调用方、文档和测试。

Few-shot:

```text
Task: 给 agent 增加新的权限审批模式。

Good:
将权限判断收敛到 permission_policy 模块：
- agent runner 只负责调度工具调用，不关心具体审批策略。
- permission_policy 只负责根据输入返回 PermissionDecision。
- 新增 auto-permission / free-permission 时，只新增或替换策略实现，不改 runner 主流程。
- 策略模块可以单独测试，runner 只测试“接收 decision 后如何执行”。

Bad:
把 on-request、auto-permission、free-permission 的判断逻辑直接写进 agent runner：
- runner 同时负责调度、风险判断、用户确认和工具执行。
- 新增权限模式时必须修改主流程。
- 权限逻辑难以单独测试。
- 一个策略 bug 可能影响整个 agent 执行链路。
```

### 可扩展但不预设终局

不要假设当前设计就是最终答案。设计协议、文件格式、配置结构、数据结构和模块接合部时，应保留合理扩展空间，例如版本号、自描述字段、稳定接口、兼容策略和清晰扩展点。扩展性必须服务于可预见的演进方向，不能为了虚构未来引入过度抽象；需要留下扩展提示时，应在具体接合部说明“如果需要扩展，应从这里接入”，而不是散落空泛 TODO。

Few-shot:

```text
Task: 设计 agent 的 tool result 数据结构。

Good:
{
  "schema_version": "1.0",
  "tool_name": "read_file",
  "status": "success",
  "result": {...},
  "metadata": {...}
}

后续需要增加耗时、权限来源或缓存信息时，可以扩展 metadata 或升级 schema_version，不破坏旧调用方。

Bad:
{
  "output": "..."
}

所有结果都塞进 output 字符串里。后续需要区分 success / error / metadata / permission_info 时，只能靠字符串解析或破坏旧格式。
```

### 配置集中管理

可变参数、路径、名称、阈值、开关、模型名、环境差异项等配置应集中管理，不应散落在业务逻辑中。简单脚本可以把配置统一放在文件开头；中大型项目应使用独立配置文件，例如 `config.toml`、`config.yaml` 或环境变量配置。配置应有清晰命名，并作为单一事实来源，避免同一个含义在多个位置重复定义。不要为了不会变化的普通局部变量强行配置化。

Few-shot:

```text
Task: 多个流程都需要使用默认验证集名称。

Good:
在文件开头或配置文件中集中定义：
val_name = "validation"

业务逻辑只引用 val_name。

Bad:
在训练、评估、导出、日志逻辑中反复写死 "validation"。
后续要改成 "val" 时，需要全局搜索替换，且容易漏改。
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
