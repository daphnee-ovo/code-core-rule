# Core Rule

## 0. Safety and Execution Priority

### External Safety Comes First

Blocks and prohibitions from external safety systems have the highest execution priority, including hooks, permission systems, sandboxes, CI security checks, and policy engines. The agent must not question, bypass, weaken, or re-evaluate them. When blocked, the agent must stop the relevant action and explain the reason based on the safety system's message.

Few-shot:

```text
Task: Run rm -rf on a directory.

Hook:
Blocked: destructive command requires explicit approval.

Good:
Stop execution and tell the user that the operation was blocked by the safety system and must be handled according to the rule.

Bad:
Rewrite the command, change the path, split the operation into smaller steps, and continue deleting to bypass the hook.
```

## 1. Requirement and Change Boundary

### Confirm Requirements First

When requirements are unclear, clarify before starting work. Any uncertainty that affects implementation direction, modification scope, safety risk, execution cost, or acceptance criteria must not be resolved by assumption.

Few-shot:

```text
User: Add a permission system to the project.

Agent:
Need to confirm first: should permissions cover shell commands, tool calls, or both? What is the default mode? After denial, should execution stop or should the agent re-plan?
```

Counterexample:

```text
User: Add a permission system to the project.

Agent:
Implemented three permission modes by default and took over all shell commands.
```

### Confirm Scope Escalation

If the actual impact scope is found during execution to be significantly larger than initially expected, the agent must pause and explain the situation to the user. The agent must not expand execution scope on its own. The agent may explain the root cause, impact scope, risks, and options, but must wait for user confirmation before upgrading a small fix into a major refactor, interface redesign, or cross-module rewrite.

Few-shot:

```text
Task: Fix an edge-case bug in the parser.

Good:
After discovering that the root cause involves the parser's overall state management, pause further expansion and explain:
A. Minimal fix: fix only the current bug with lower risk.
B. Thorough fix: refactor the parser module, affecting interfaces, callers, and tests.
Wait for user confirmation before continuing.

Bad:
After finding a deeper root cause, directly refactor the entire parser and modify callers, interfaces, and tests.
```

### Principle of Least Surprise

Program behavior should match user expectations. Commands, options, defaults, output formats, and modification scope should stay consistent with comparable tools where possible. Option names should be intuitive, such as `-v` for `verbose` and `-h` for `help`. Unless explicitly requested by the user, do not proactively refactor, change style, change interfaces, modify unrelated files, or expand scope.

Few-shot:

```text
Task: Add a verbose logging flag to a CLI.

Good:
Add -v / --verbose and keep the original default output unchanged.

Bad:
Add --debug-mode-plus, enable verbose logging by default, and refactor the entire output format at the same time.
```

## 2. Documentation and Long-Term Maintenance

### Bidirectional Code-Documentation Traceability

Code and documentation must be traceable in both directions. A code file header should describe the internal structure of the current file using a tree of classes, functions, methods, or core flows, with short comments only when useful. Nodes that are obvious from their names should not be over-commented. The code file header should link to related Markdown documentation, and the related documentation should link back to the relevant code file or code directory. Every code or documentation change must check whether the other side remains accurate, preventing code structure, behavior, interface descriptions, and documentation indexes from drifting apart.

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
# - [Core Rule](../../core_rule.md#bidirectional-code-documentation-traceability)
```

```md
# Permission Rules

Related Code:

- [permission_guard.py](../src/agent/permission/permission_guard.py)
- [permission module](../src/agent/permission/)
```

## 3. Implementation Complexity Control

### Prefer Simplicity

If a simple implementation is enough, do not overcomplicate it. Code is read by humans first and executed by machines second. Prefer direct, readable, and easy-to-debug solutions. Avoid clever code, and avoid unnecessary abstractions, frameworks, layers, or algorithmic complexity introduced only for an appearance of architecture. Each program, module, or function should generally have one clear responsibility and do it well.

Few-shot:

```text
Task: Read a JSON file, filter records where active=true, and write the result.

Good:
Use a clear flow: load_json() -> filter_active() -> write_result().

Bad:
Introduce a plugin system, scheduler, abstract base classes, and dynamic registration just to perform one simple filter.
```

### Use Software Leverage

Prefer reusing mature, reliable, well-maintained tools, libraries, and system capabilities. Do not reinvent the wheel. Implement from scratch only when existing solutions are too heavy, unstable, incompatible with core needs, or introduce obvious maintenance cost. Reuse must also avoid pulling in a large dependency for a small need.

Few-shot:

```text
Task: Parse a YAML configuration file.

Good:
Use a mature YAML parser and wrap it in a lightweight config_loader.

Bad:
Handwrite an incomplete YAML parser that supports only part of the syntax and requires ongoing patches.
```

### Composition over Integration

Do not build one large all-in-one program. When adding a new feature, prefer creating a new module with a clear responsibility and composing it with existing modules through stable interfaces. Do not pile new logic into old programs, old functions, or existing main flows merely for convenience. Modules should depend on clear inputs and outputs, not be embedded into each other's internal implementation details, so they remain removable, replaceable, testable, and maintainable.

Few-shot:

```text
Task: Add Excel export to a data processing pipeline.

Good:
Add excel_writer and connect it to the existing reader -> cleaner -> transformer -> writer flow; keep the existing reader / cleaner / transformer boundaries unchanged.

Bad:
Put Excel export, formatting, path decisions, and error handling into the existing transformer, making the main flow increasingly bloated.
```

## 4. Interface and Module Boundary

### High Cohesion, Low Coupling

Modules should be highly cohesive internally, organizing code around a single responsibility. Modules should be loosely coupled externally, collaborating through clear, stable, and predictable interfaces, protocols, data structures, or standards rather than depending on each other's concrete implementation logic. Prefer composable and replaceable mechanisms instead of hard-coding policies into the main flow. When an interface must change, update callers, documentation, and tests together.

Few-shot:

```text
Task: Add a new permission approval mode to an agent.

Good:
Consolidate permission decisions into a permission_policy module:
- The agent runner only schedules tool calls and does not care about the concrete approval policy.
- permission_policy only returns a PermissionDecision based on its inputs.
- When adding auto-permission / free-permission, only add or replace policy implementations; do not modify the runner's main flow.
- The policy module can be tested independently, while the runner only tests how it acts after receiving a decision.

Bad:
Write the logic for on-request, auto-permission, and free-permission directly into the agent runner:
- The runner handles scheduling, risk evaluation, user confirmation, and tool execution at the same time.
- Adding a new permission mode requires modifying the main flow.
- Permission logic is hard to test independently.
- A policy bug may affect the entire agent execution chain.
```

### Extensible, but Do Not Assume Finality

Do not assume the current design is the final answer. When designing protocols, file formats, configuration structures, data structures, and module junctions, leave reasonable room for extension, such as version fields, self-describing fields, stable interfaces, compatibility strategies, and clear extension points. Extensibility must serve foreseeable evolution and must not introduce excessive abstraction for imaginary futures. When an extension hint is needed, place it at the specific junction and explain where future extension should connect, rather than scattering vague TODO comments.

Few-shot:

```text
Task: Design the agent's tool result data structure.

Good:
{
  "schema_version": "1.0",
  "tool_name": "read_file",
  "status": "success",
  "result": {...},
  "metadata": {...}
}

If duration, permission source, or cache information is needed later, metadata can be extended or schema_version can be upgraded without breaking old callers.

Bad:
{
  "output": "..."
}

All results are placed into one output string. Later, distinguishing success / error / metadata / permission_info requires string parsing or breaking the old format.
```

### Prefer Explicit Constraints

Code should prefer explicit modeling capabilities provided by the language and type system. Avoid implementation approaches that bypass constraints, hide real types, or rely on runtime conventions. Weakly constrained mechanisms are allowed only when performance, FFI, plugin boundaries, dynamic extension, or similar needs make them necessary and no clearer modeling approach exists. When used, their scope must be minimized and their rationale, risks, and boundaries must be documented.

Few-shot:

```text
Task: Execute different logic for different task types.

Good:
Use explicit types, enums, interfaces, or traits to model task types and behavior.

Bad:
Put every task into an opaque container, then determine the real type at runtime with strings, downcast, or conventions.
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

## 5. Error Handling and Failure Strategy

### Transparent Errors

Programs must report errors clearly and must not fail silently. When a failure occurs, explain what happened, possible causes, impact scope, and the next handling step. Do not swallow exceptions, return vague results, or make the user believe the task succeeded when it did not.

Few-shot:

```text
Task: Read a configuration file.

Good:
Error: config.yaml not found. Expected path: ./config/config.yaml. Please create the file or pass --config <path>.

Bad:
Continue running with an empty configuration and finally output an incomplete result.
```

### Fail Fast

If complete program operation depends on specific configuration, components, services, permissions, or external resources, check them during startup or before the task begins. If a dependency is missing, fail immediately and explain why. Do not wait until the user invokes the affected feature before exposing the dependency problem, which would make the service appear available while actually being incomplete.

Few-shot:

```text
Task: Start a service with an export feature that depends on LibreOffice.

Good:
Check whether LibreOffice exists during service startup; if missing, fail immediately:
Error: LibreOffice is required for export feature but was not found.

Bad:
Start the service normally, then fail only when the user clicks "Export PDF":
Command not found: libreoffice
```

### No Forced Fallbacks

Do not use default values, empty results, broad exception catches, or implicit degradation to hide real problems. Fallbacks are allowed only when the fallback behavior is an explicit requirement, explainable, observable, and not misleading. Otherwise, report the error transparently or fail fast.

Few-shot:

```text
Task: Read a user-specified data file.

Good:
If the file does not exist, fail directly and explain the missing path and how to handle it.

Bad:
If the file does not exist, automatically return an empty list, let the downstream flow continue, and produce an empty result.
```

## 6. Testing Constraint

### Trust Tests

Trust existing test code by default unless there is clear evidence that the tests have become invalid due to major business changes, interface changes, or implementation boundary changes. Do not change tests merely because the implementation fails them. First check whether the implementation violated expected behavior. If a test may be outdated or the requirement is unclear, confirm with the user first.

Few-shot:

```text
Task: After modifying login logic, existing auth tests fail.

Good:
First check whether the implementation broke existing login constraints; only after confirming that the requirement has changed should the agent ask whether the test should be updated.

Bad:
Directly modify the test assertion so the new implementation passes.
```
