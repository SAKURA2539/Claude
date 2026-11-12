# 日志指南

> 日志是可观测性（logs / metrics / tracing）的核心维度。本指南聚焦结构化日志规范，后续可扩展可观测性的其他维度。

- **结构化日志带语义 key**：决策点用 `% 占位 + 语义 key`（stage=/decision=/similarity_score=/reason= 等），同概念全模块用同 key；量大中间结果取前 N 条预览。
- **链路上下文贯穿**：handler 类用 `@log_context(handler=...)`，流程内 `with log_context(tool=...)` 叠加业务标识，跨方法日志可串联。
- **失败携带 context**：error/warning 带可定位业务上下文（实体 ID，如 item_id/collection_id）+ `exc_info=True`；降级日志说清"降级到什么、为什么"。
