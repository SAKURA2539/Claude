# 测试指南

> 测试跟随源码：禁止为实现测试修改源码，禁止为通过测试而做出匪夷所思的操作——测试为验证源码而生，应严格与源码一致。

## 原则：tests/ 镜像源码包，每个功能下分三类

- `tests/{package}/` 的目录树与源码包 `{package}/` 同构——源码 `{package}/<包>/...` 的测试落在 `tests/{package}/<包>/...` 下，路径一一对应。
- 每个功能模块目录下按测试类型分三个子目录：
  - `unit/` —— 单元测试
  - `integration/` —— 集成测试
  - `e2e/` —— 端到端测试
- 测试一律绝对 import（`from {package}.*`），与测试文件自身路径解耦；挪动测试目录时，凡基于 `__file__` 计算的路径偏移（如 `parents[N]`）须按层级差同步调整。

> 三类之下各架构层（domain / application / adapter）的细分对应见下文「目录映射：指导层级 → 测试目录」。

---

# Config Strategy: Use Conftest

conftest.py 为测试提供可复用的依赖环境（fixtures），核心原则：

## 原则 1：fixture scope 选择

pytest fixture 有两种常用 scope：

- **`session`**：整个测试跑下来只创建一次，所有人共用。适合**只读不写**的东西，比如 `settings`（读完配置文件就完了，不会改它）。
- **`function`**：每个测试函数各自创建一份，用完就扔。适合**会写数据**的东西，比如往 Qdrant 里 upsert 了测试数据——如果所有人共用一根连接，上一个测试写的数据没清，下一个测试就会读到脏数据，断言全乱了。

一句话：**会写共享数据的 fixture 用 function scope，配合 teardown 写完擦干净。**

## 原则 2：写入外部依赖的 fixture 必须 setup → yield → teardown

任何向 Real 外部依赖（Qdrant、DB 容器等）写入数据的 fixture，**必须**在 teardown 阶段清理写入数据，否则导致测试间数据污染。

```python
@pytest.fixture
async def test_point(qdrant_client, collection_name):
    # setup: upsert 测试点
    point_id = str(uuid4())
    qdrant_client.upsert(collection_name, points=[...])
    yield (point_id, payload)
    # teardown: 删除测试点
    qdrant_client.delete(collection_name, points_selector=[point_id])
```

## 原则 3：handler 级 conftest 的抽取时机

当同一 handler 的多个测试模块共享同一组 Real adapter 或测试数据 fixtures 时，将共享部分抽取到 handler 级 `conftest.py`。文件数量是果，共享资源才是因。

# 测试替身定义

## Real

直接使用生产级的真实实现，不做任何替代。适用于：

- **领域对象**：值对象、实体、领域服务 —— 它们是业务规则的载体，不依赖外部 IO，天然可测
- **确定性基建**：DB 容器、本地 Redis —— 行为确定、可重复、低成本

特点：无替代，完全按生产逻辑执行。

## Fake

用内存实现或其他方式替代真实的外部依赖，但保留了完整的业务行为语义。适用于：

- **数据库存储端口**：使用内存结构实现 Repository 接口，行为等价于真实 DB 但无需 IO
- **文件系统端口**：使用内存 buffer 替代真实文件读写

特点：

- 有状态（可读写，可验证写入结果）
- 行为完整（CRUD 语义与真实实现一致）
- 轻量无 IO（纯内存，测试秒级完成）

## Mock

用模拟框架创建的替身，不保留任何业务行为，仅按预设返回。适用于：

- **不可控的外部服务**：非确定性、高延迟、有成本
- **需要验证调用行为的场景**：断言"某个方法被调用了 N 次"、"调用参数符合预期"

特点：

- 无状态（每次调用按预设返回）
- 无真实行为（不执行业务逻辑）
- 可验证调用

---

# 六边形架构测试策略映射表

| 架构层级                                      | 关注点                                           | 测试类型    | 依赖处理方式 (Real / Fake / Mock)                                                                                                                                                               |
| --------------------------------------------- | ------------------------------------------------ | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **领域层 (Domain)**                     | 核心业务规则、状态流转                           | 单元测试    | **全 Real**。仅实例化真实对象（值对象、实体、领域服务），绝对禁止 Mock / Fake。                                                                                                           |
| **应用层 (Application)**                | 业务用例编排、事务边界                           | 单元测试    | **Fake 为主，Mock 为辅**。领域对象为 Real；DB 存储端口使用 Fake（内存实现）；外部不可控的通知服务使用 Mock。                                                                              |
| **适配器层 — 出站错误翻译 (Adapters)** | Adapter 内部异常翻译逻辑（技术异常 → 应用异常） | 单元测试    | **Real + Mock**。被测 Adapter 类用 Real；注入的基建客户端（QdrantClient 等）用 Mock，精确控制异常输入。                                                                                   |
| **适配器层 — 出站数据契约 (Adapters)** | 系统与外部的数据/协议转换                        | 集成测试    | **基建全 Real**，AI / 不可控服务 Mock / 录制。DB/Redis 等确定性基建直连 Real 测试容器；LLM 等非确定性高成本 API 用 VCR 录制回放或 Mock HTTP 客户端（但 LLM 输出约束特化代码用真实 LLM）。 |
| **适配器层 — 入站 (Adapters)**         | 框架/路由 → Use Case 的集成                     | 集成测试    | **Real + Mock**。框架和路由层用 Real，注入后端的 Use Case 用 Mock 隔离。                                                                                                                  |
| **端到端层 (EndTOEnd)**                 | 用户核心链路全贯通验证                           | 端到端 测试 | **全 Real**。贯穿全层，从真实 API 接口一直打到真实数据库，仅对第三方不可控且无法沙盒化的网关做全局 Mock。                                                                                 |

---

# 目录映射：指导层级 → 测试目录

| 指导层级                 | 测试类型 | 对应测试目录                                                                                  |
| ------------------------ | -------- | --------------------------------------------------------------------------------------------- |
| 领域层                   | 单元测试 | `unit/domain/`                                                                              |
| 应用层                   | 单元测试 | `unit/application/usecases/`、`unit/application/models/`、`unit/application/transfers/` |
| 适配器层 — 出站错误翻译 | 单元测试 | `unit/adapter/`                                                                             |
| 适配器层 — 出站数据契约 | 集成测试 | `integration/application/usecases/`                                                         |
| 适配器层 — 入站         | 集成测试 | ❌ 缺失                                                                                       |
| 端到端                   | E2E 测试 | `e2e/application/usecase/`                                                                  |

> **注意：** `integration/application/usecases/` 虽然路径含 `application`，但它的依赖策略是「基建全 Real + AI Mock」，对应指导中**适配器层出站集成测试**。它验证 Use Case + Real Adapter 的联合数据契约，非应用层单元测试的 Fake 策略。

---

# handler 级共享测试模块：`_fake.py` / `_data.py` / `_help.py`

`tests/.../{module}/` 下有三个 handler 级共享模块（`_` 前缀 = 测试基建，非测试文件）：

## `_fake.py` — 测试替身

**职责：** 所有 test double 的唯一来源。包含 Fake class（dict 内存实现）、`:memory:` Qdrant 工厂、mock factory。

**原则：** dict 和 `:memory:` Qdrant（`QdrantClient(location=":memory:")`）都是 Fake——dict 用于简单状态，`:memory:` Qdrant 用于向量搜索。Mock 仅用于不可控的外部服务端口。

## `_data.py` — 纯工厂

**职责：** 纯工厂函数，每次返回新实例。

**硬约束：**

- 无默认参数
- 无预置常量
- 无隐藏数据（函数体内不能有硬编码的测试值，除非该数据是"格式规范"而非"测试数据"）
- 只做构造不做决策

**Why Factory 而非 Object Mother：** 测试数据变化大、与断言紧耦合。Factory 消除 boilerplate 但调用方控制关键值。

## `_help.py` — 无状态工具

**职责：** 断言函数和共享常量。无副作用、无 pytest 依赖。

## 三模块与 conftest 的关系

```
conftest.py (handler 级 / unit 级)
  ├── 从 _fake 导入 Fake class/工厂 → 包装为 @pytest.fixture + yield → teardown
  ├── 从 _data 导入工厂函数 → 在 fixture 中调用，按需传参
  └── 从 _help 导入断言/常量 → 测试函数直接使用
```

**conftest 只做 fixture 生命周期管理**（setup → yield → teardown），不包含 class 定义或工厂逻辑。

---

# 测试文件组织约定

## 原则 1：基础测试与源码同构

一个 `test_xxx.py` 对应一个源码 `xxx.py`，各层（adapter / usecase / domain）同构。只对实际测到的源文件建测试文件，不测的不建空文件占位。性能、集成等非基础测试单独成文件，不混进单元测试目录。

判断是否违反同构：测试文件 import 的被测对象是否来自多个不同源文件——是则应拆分。

## 原则 2：三层结构

| 层   | 载体              | 分隔依据                     | 分隔符                        |
| ---- | ----------------- | ---------------------------- | ----------------------------- |
| 功能 | 文件              | 源码文件                     | 无（文件名 + docstring 承担） |
| 外层 | 文件内 class 之间 | 被测源码符号（函数/方法/类） | 列首粗线                      |
| 内层 | class 内方法之间  | 测试意图                     | 缩进细线                      |

- 外层粗线对应**被测源码符号**，不是测试类本身：测同一符号的多个测试类聚在同一外层块下。
- 粗细线靠缩进区分：列首为外层，缩进为内层。
- 分隔符只在该容器有 ≥2 个兄弟项时添加。

## 原则 3：类名 = 被测单元

测试类名 = `Test` + 被测源码符号（PascalCase），**保留 `Test` 前缀**（pytest 默认 `python_classes=Test*`，前缀因此必需，不依赖仓库另行配置）。去掉的是**测试维度后缀**（Edge / Boundary / Happy / BasicFlow / ErrorBoundary 等场景词）——测试维度由类内分组承载。重命名测试类前 grep 全仓确认无外部引用。

## 原则 4：类内按测试意图分组排序

固定顺序：Happy Path → Error Propagation → Edge Cases → Fallback & Resilience → State & Idempotency。

缺的类别不写空占位块；每个分组（含首组、单类别类）都加类别头。

## 分类裁决

- **边界输入触发的抛错**（由边界输入引发的异常）→ Edge Cases。判断看输入是否在边界，不看是否用了 raises 断言。
- **类型化领域/基础设施异常的传播**（抛出交由调用方决策）→ Error Propagation。
- **异常被捕获并走备选分支**（调用方无感知，返回安全值）→ Fallback。与 Error Propagation 的区别：前者断言抛出，后者断言走了备选路径。
- 空 / None / 零值 / 空集合 / 未知输入 → Edge Cases。
- 幂等性、操作前后状态、跨场景字段一致 → State & Idempotency。

## 约束

- 重组 / 拆分只动文件结构、方法顺序、注释分隔符、类名；不改方法体、docstring、装饰器、import 语义。
- 改完跑全量单元测试：方法数须与改前一致（零增删），且全绿。

---
