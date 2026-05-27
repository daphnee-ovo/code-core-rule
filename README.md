# README.md
> 一些自用的 coding agent 约束规则
> core file: core_rule.md

> 注：破坏性操作防护（如 rm、.env 修改、数据库写入等）
> 通过独立的 hooks 机制实现，见 [hooks 配置文档](./hooks/README.md)。
> 本规则不重复覆盖已由 hooks 强制保障的行为边界。
