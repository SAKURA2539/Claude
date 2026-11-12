# 错误处理规范

各项目 `handler/` 根下所有功能的错误处理遵循本规范。与 [architecture-guide.md](architecture-guide.md) 的分层约束配套：本文件规定**错误在各层之间如何流动、由谁抛出、由谁处置**。

## 核心原则：三层分工

错误判定有一个边界：

> **动作失败例如模型故障是错误，动作成功例如查询为空非错误**

错误处理只有一句总纲：

> **Adapter 翻译，Use Case 决策，Domain 声明。**

对应六边形「目的 / 手段」的切分（见 [architecture-guide.md](architecture-guide.md) 心智说明），错误天然分两类：

| 错误性质                     | 归属                 | 语义                                                                                           |
| ---------------------------- | -------------------- | ---------------------------------------------------------------------------------------------- |
| **业务违规**（目的层） | `DomainError`      | 业务规则被违反——标准未加载、规则未配置、状态机非法转换。**与技术无关**。               |
| **技术故障**（手段层） | `ApplicationError` | 外部 Provider 挂了——Qdrant 超时、Model 不可用、LMDB 写失败。是「手段」特定的，会随换技术而变。 |

三个动词锁定三层各自的职责：

- **Domain 声明** —— 领域层只负责**定义并声明**领域错误（它是什么）。不 catch、不了解任何技术。
- **Adapter 翻译** —— 适配器把 Provider 的技术异常**翻译**成统一的 `ApplicationError`（换个说法），`from exc` 保留原因链。**不做**「要不要继续」的决策。
- **Use Case 决策** —— 用例拿到翻译后的统一错误，决定**传播还是降级**（怎么办）。

三者是接力，每一层只干自己那一件、不越界：

```
请求进入 Use Case
  ├─ 业务违规 ──→ Use Case 编排中检测到 ──→ raise DomainError          （Domain 声明，Use Case raise）
  └─ 技术故障 ──→ Adapter catch 技术异常 ──→ 翻译为 ApplicationError
                                          ──→ Use Case 收到统一错误 ──→ 决策：传播 or 降级
```

## 分层职责矩阵

| 层                        | 可 raise                               | 可 catch                           | 禁止                                              |
| ------------------------- | -------------------------------------- | ---------------------------------- | ------------------------------------------------- |
| **Domain**          | `DomainError`                        | —                                 | 了解 Adapter/Provider；导入 application           |
| **Adapter**         | `ApplicationError`                   | Provider 技术异常                  | raise`DomainError`；`except Exception` 笼统吞 |
| **Use Case**        | `DomainError` + `ApplicationError` | `ApplicationError`（做决策）     | catch 原始 Provider 异常（如`grpc.RpcError`）   |
| **Driving Adapter** | —                                     | `ApplicationError` → 协议状态码 | 做降级/重试决策                                   |

> Use Case 是唯一「两类错误都能 raise」的层：编排中检测到业务违规 → raise `DomainError`；传播技术故障 → 透传/包装 `ApplicationError`。

## 错误类型与文件命名

每个六边形模块维护**两个** `_error.py`（不是 `errors.py`），均应在对应 `__init__.py` 重导出：

| 文件                      | 基类                                       | 子类命名                      | 语义                                                                          |
| ------------------------- | ------------------------------------------ | ----------------------------- | ----------------------------------------------------------------------------- |
| `domain/_error.py`      | `<Module>DomainError(RuntimeError)`      | `<What>Error`               | 业务规则违反                                                                  |
| `application/_error.py` | `<Module>ApplicationError(RuntimeError)` | `<Operation><Outcome>Error` | 用例级失败：契约违反、外部服务降级（由 adapter 翻译而来）、并发、跨系统不一致 |

范例：`handler/{module}/domain/_error.py` 与 `handler/{module}/application/_error.py`。

> `DomainError` 与 `ApplicationError` 都是**预期错误**——调用方能据此做有意义的决策（降级/重试/提示用户）。**意外错误**（编程缺陷：`IndexError`、`AttributeError`、`None.foo`）不应在 Adapter/Use Case 中被 catch，让其冒泡到 Driving Adapter 兜底。

## Adapter 翻译规则

1. catch **具体** Provider 异常类型，不写 `except Exception`。
2. `raise AppError(...) from exc` —— 保留 `__cause__` 链。
3. try 块**只包裹 Provider 调用**，业务转换逻辑放在 try 外。
4. **永远不在 Adapter 中 raise `DomainError`**。

各 Provider 标准 catch 类型：

| Provider    | catch 类型                                                                             |
| ----------- | -------------------------------------------------------------------------------------- |
| Qdrant      | `(ApiException, TimeoutError, ConnectionError, grpc.RpcError)`                       |
| Model Runtime | `ModelInvocationError`（基类覆盖 `ModelTimeoutError` / `ModelUnavailableError`） |
| LMDB        | `Exception`（其异常类型不稳定，允许放宽）                                            |
| gRPC Client | `(grpc.RpcError, asyncio.TimeoutError, ConnectionError)`                             |
| HTTP Client | `(aiohttp.ClientError, asyncio.TimeoutError)`                                        |

**范例**（Qdrant 向量搜索：窄化 catch + `from exc`，业务转换在 try 外）：

```python
# adapter/vector_repository_adapter.py  {Feature}Repository.search_by_vector
try:
    raw = await self._qdrant.search(
        collection_name=self.collection_name,
        query_vector=query_vector,
        limit=top_k,
        with_payload=True,
    )                                       # ← try 只包 Provider 调用
except (ApiException, TimeoutError, ConnectionError, grpc.RpcError) as exc:
    raise VectorSearchFailedError(          # ← 翻译为统一应用错误
        collection_name=self.collection_name,
        detail=f"search_by_vector top_k={top_k}: {exc}",
    ) from exc                              # ← from exc 保留异常链
return [_qdrant_scored_point_to_scored_label_point(p) for p in raw]   # ← 转换在 try 外
```

换 Provider 时只改这里的 catch 类型，上层零改动——这正是「手段封 adapter」在错误层的体现。

## Use Case 决策模型

Use Case 通过 port 调 adapter，拿到的是 adapter 翻译后的 `ApplicationError`（统一格式，**非原始技术异常**）。此时做一个业务判断：

> 这个调用的返回值，对当前流程是**主信号**还是**辅助信号**？

| 信号类型           | 失败策略                                            | 典型场景                   |
| ------------------ | --------------------------------------------------- | -------------------------- |
| **主信号**   | 传播（不写 try-catch，让`ApplicationError` 冒泡） | 向量搜索、相似项检索 |
| **辅助信号** | 降级为空/默认值 +`log.warning`                    | 结果审计、纠错修正        |                 
| **状态加载** | 传播（无状态则后续无法继续）                        | 本地 profile 加载          |

### 降级范例（辅助信号）

辅助信号常是「锦上添花」：主体阶段已有初判结果（默认值），辅助阶段（如审计 audit、纠错 correct）只是把它修正得更准。失败时保留初判即可，主流程继续。

```python
# usecases/{feature}_classification.py  _stage_audit
try:
    audit_output = await self._audit_port.audit(audit_input)
except AuditServiceError as exc:             # ← catch 翻译后的应用错误，非原始 ModelInvocationError
    logger.warning(
        "audit %s failed (degrading to keep): %s",
        audit_input.entity_id, exc,
    )
    return                                   # ← 降级：保留初判默认值，不 raise
```

**降级三件套**（缺一不可）：catch **具体类型**（非 `except Exception`） + `logger.warning` 带上下文且说清「降级到什么、为什么」 + `return` 回默认值不 raise。

### 传播（主信号）

传播就是**不写 try-catch**——port 调用裸奔，`ApplicationError` 直接冒泡到 Driving Adapter。把上面降级范例的 `except` 块整个去掉，即为传播。

判据不在代码长相，在业务语义：**有无兜底**。失败了下游无路可走 → 传播；失败了有默认值可退 → 降级。

## asyncio.gather 并发模式

并发批量调用 port 时：

- 必须 `return_exceptions=True`。
- 主信号失败用 `isinstance(result, Exception)`（**不是** `BaseException`）检测 → re-raise。
- 辅助信号失败降级为 fallback 值。
- 每个失败单独 `log.warning`，不中断其他并发任务。

```python
# usecases/{feature}_classification.py  并发辅助任务
correction_results = await asyncio.gather(*correction_tasks, return_exceptions=True)
for r in correction_results:
    if isinstance(r, Exception):             # ← Exception 不是 BaseException
        logger.warning("correction task failed (degrading to keep): %s", r)
```

> 用 `isinstance(r, Exception)` 而非 `BaseException`，避免误拦截 `KeyboardInterrupt` / `SystemExit`。

## 禁止事项

| 禁止                                              | 正确做法                                  |
| ------------------------------------------------- | ----------------------------------------- |
| Adapter`except Exception` 笼统吞                | catch 具体 Provider 异常类型              |
| Adapter raise`DomainError`                      | 翻译为`ApplicationError`                |
| Use Case catch 原始 Provider 异常                 | 依赖 Adapter 翻译后的`ApplicationError` |
| 装饰器 / context manager 处理异常                 | 显式 try-catch，保留行号和调用栈          |
| `raise NewError(...)` 不写 `from exc`         | 必须`from exc`，保留 `__cause__`      |
| `except Exception: pass`                        | 至少`log.warning`                       |
| `isinstance(r, BaseException)` 判断 gather 结果 | 用`isinstance(r, Exception)`            |
| Port 接口中定义异常类型                           | 异常定义在`_error.py`                   |

## Checklist

- [ ] `domain/_error.py` + `application/_error.py` 已创建，`__init__.py` 重导出。
- [ ] Adapter：裸奔调用 → 加 try-catch 翻译；`except Exception` → 窄化具体类型。
- [ ] Use Case：逐调用点判断主/辅助信号 → 传播或降级。
- [ ] asyncio.gather：`return_exceptions=True`，主信号 re-raise，辅助降级。
- [ ] 测试：翻译正确性 + 降级行为 + 传播行为 + 异常链完整性。
- [ ] 回归：成功路径行为与修改前完全一致。
