# 六边形架构编码规范

本文件为强制编码 rule，约束分层、命名、映射与职责边界。

配套规范（同目录，编辑 handler / 测试时一并加载）：

- [error-guide.md](error-guide.md) — 错误处理：Adapter 翻译 / Use Case 决策 / Domain 声明
- [log-guide.md](log-guide.md) — 日志（可观测性的核心维度）
- [test-guide.md](test-guide.md) — 测试策略：Real / Fake / Mock 映射

## 分层定义

| 层                           | 职责                             |
| ---------------------------- | -------------------------------- |
| **领域层 domain**      | 实体、值对象、领域服务、业务规则 |
| **应用层 application** | 用例编排、事务边界、端口接口定义 |
| **适配器层 adapter**   | Driving、Driven                  |

## 结构布局

### 目录组织

目录第一级按**架构层**（adapter/application/domain），第二级按**功能**（如 feature_a/feature_b）。跨功能共享代码才放架构层顶层、与功能目录平级。

### 文件命名

| 层             | 文件命名                   | 类命名                                                                             |
| -------------- | -------------------------- | ---------------------------------------------------------------------------------- |
| port           | `{feature}_port.py`      | `{Feature}Port(Protocol)`                                                        |
| model          | `{feature}_model.py`     | `{Feature}Input` / `{Feature}Output`                                           |
| adapter        | `{feature}_adapter.py`   | `{Tech/Source}{Feature}Adapter`                                                  |
| usecase        | `{snake_case}.py`        | `{PascalCase}Handler`                                                            |
| transfer       | `{snake_case}_models.py` | `{PascalCase}Request` / `{PascalCase}Response` / `{PascalCase}Code(IntEnum)` |
| domain entity  | `{snake_case}.py`        | `{PascalCase}`                                                                   |
| domain value   | `{snake_case}.py`        | `{PascalCase}`                                                                   |
| domain service | `{snake_case}.py`        | `{PascalCase}`                                                                   |

### 映射规则

1. **port ↔ adapter** — 每个 port 恰好有对应的 adapter 实现，port 定义协议，adapter 绑定不同的技术/来源。一个 port 可暴露多个接口。
2. **port ↔ model** — 每个 port 有对应 model 文件定义入参/出参 DTO。
3. **usecase ↔ transfer** — 每个 usecase 有一个主要 transfer 文件。复杂用例可多 transfer，简单用例可共享。
4. **usecase 依赖注入规范** — Handler 构造签名 `__init__(self, context: HandlerContext, **kwargs)`，端口通过 `kwargs.get("port_field_name")` 注入，工厂函数在 `bootstrap.py` 显式传参。
5. **工厂命名** — adapter 工厂 `create_{name}_adapter(context)` 在 `adapter/__init__.py`；handler 工厂 `create_{name}_handler(context)` 在 `bootstrap.py`。
6. **`adapter/__init__.py` 双重职责** — 除导出适配器类外，还提供工厂辅助函数 `create_{name}_adapter(context)`。

### 目录职责

| 目录                      | 用途                                   | 边界                                                                              | 职责                                                                               |
| ------------------------- | -------------------------------------- | --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `domain/entity`         | 领域实体                               | 零外部依赖，禁导入 adapter/application                                            | 承载身份标识+生命周期+不变量保护                                                   |
| `domain/value`          | 值对象                                 | 零外部依赖，禁导入 adapter/application                                            | 承载不可变领域描述+自校验                                                          |
| `domain/service`        | 服务、业务规则                         | 零外部依赖，禁导入 adapter/application                                            | 承载跨对象、无归属纯领域逻辑                                                       |
| `application/port/`     | 由 adapter 实现、供 usecase 调用的协议 | 只定义协议，禁导入 adapter；IO 由 model 规定                                      | 抽象业务事件与业务目的，而非技术功能。A port identifies a purposeful conversation. |
| `application/model/`    | 承载 port 交互 IO DTO                  | 与技术/来源解耦，禁导入 adapter                                                   |                                                                                    |
| `application/usecase/`  | 编排 port 实现功能                    | 仅持有 domain/port 引用，禁 import 任何技术 client                                |                                                                                    |
| `application/transfer/` | usecase 出入参（driving 端转换暂居此） | 驱动端↔业务层序列化边界，禁导入 usecase                                          |                                                                                    |
| `adapter/`              | 用技术/来源实现 port                   | 唯一持有技术/来源 client（LLM/Qdrant/LMDB/HTTP/gRPC），prompt/schema/SDK 封装于此 |                                                                                    |

## 心智说明

### 层间切分（Principle）

六边形里只有两类东西：**目的** 和 **手段**。

- **目的** = 业务真正在乎的：要处理的对象（用户输入、数据实体）、要达成的目标（研判一个风险、回答一个问题）。不随技术变化。
- **手段** = 为实现目的而用的具体技术：用 LLM 还是规则、用哪个数据源的什么格式、用 HTTP 还是 gRPC。会变、可替换。

**切法**：目的留内部（domain 实体/值对象、app 用例与 port 契约）；手段封 adapter。port 的作用就是**钉住目的**——它只说"我要达成什么"，adapter 决定"用什么达成"。

**为什么**：手段可整体替换，目的不会变。手段封 adapter，换手段时只动 adapter、内部零改动；若手段渗进内部，换手段就得改业务逻辑。

**判据**：

> 它是"业务要处理的对象 / 要达成的目标"（目的），还是"为达成所用的具体技术"（手段）？

**例子**：

|                  | 目的 → 内部                                             | 手段 → adapter                                                                                 |
| ---------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **提示词** | 用户提示词＝输入＝被处理的对象（conversation，领域实体） | 系统提示词＋LLM＋json_schema＝"用 LLM 实现"这组手段。明天不用 LLM，它整体作废；但输入照样要处理 |
| **数据**   | 业务自己建模的实体（领域抽象）                           | 外部数据源的原始格式＋翻译成实体的规则。换数据源，格式与翻译都换                                |
| **能力**   | port 契约＝"我要 识别 / 分类 / 排序"（业务语义）       | adapter 实现＝用 LLM＋schema / 向量检索 / 规则 / 远程调用去实现，对 port 是黑盒                 |

### 领域建模（domain）

- **值对象按概念层级最小内聚**：每个 dataclass 字段恰好归属同一概念层级。判据：下游能否按需引用其中一类**而不被迫带入其余字段**。
- **组合优于继承、优于拍平**：大结构内嵌小值对象；port 输入 DTO 复用 domain values（内嵌而非复制字段），加字段只扩值对象，所有 DTO 自动受益。
- **多终态结果显式建模**：用例有互斥终态（命中/新建/未知）时把终态编码进响应结构。优先单字段判别式（`Literal`/枚举 outcome）从结构禁止非法组合；退而用"门控布尔+子对象"须在 docstring 写明组合契约。反模式：多布尔并存、合法组合靠隐式约定。
- **元数据驱动消除重复 DTO**：多个结构同构 DTO 用元数据表 + `__init_subclass__` 自动生成字段与序列化。仅当 ≥2 个同构 DTO 才上（YAGNI）。
- **决策逻辑归位 model**：与数据强相关的纯决策（阈值比较/序号转换/判定构造）封装进 model 工厂方法或纯函数，handler 只消费结果。

### 用例编排（usecase）

- **主入口即提纲**：analyze/process/handle 只表达业务判定顺序，不掺 I/O 细节。判据：读完能否复述流程而无需跳进辅助方法。
- **阶段前缀切分**：方法名带阶段语义（entry/stage/exit 或同类前缀），前缀即文档；辅助方法职责单一且多路径复用，尤其**唯一写入出口**保证落库 schema 一致。

### 状态检查前置（check）

执行前先检查状态——当前条件（标准有无、规则是否配置、环境是否就绪）支不支持执行。不支持 → guard 分支提前返回（返回值表达，见「失败与状态二分」），不碰处理动作；支持才继续。

**为什么**：把执行停在最早处——条件不满足时执行无意义、浪费资源，还可能把错误前提的结果写库。

### 持久化写入后置（write）

如无特殊情况，处理全部完成才写持久化——写入是流程最后一步（唯一写入出口）。处理途中失败即不写（字段失败 = 不落库）；落库的是最终判定结果（可能含 audit/correction 修正）。

**为什么**：落库的永远是最终结果——处理可能失败或被后续阶段修正，先写会落半成品或需回滚的数据。

### 失败（fail）

「不是所有非正常结果都是失败」。调用结果先分清两种性质：

- **技术故障**（动作失败）：手段执行不了。adapter 翻译为 `ApplicationError` 后 **raise**，由 usecase 决策传播或降级。
- **业务状态不符合**：目的侧状态不满足（无标准、规则未配置）。**用返回值表达**（`None` / 空集），usecase 走 guard 分支，不是错误。

**判据**：有无兜底——有兜底 → 返回值 + 分支；无兜底 → raise。

**推论**：

- 状态不符合以**成功态承载结果**，即动作成功，失败态只留给真故障——调用方区分「正常降级」与「需重试/报警」。
- 同 port 内两套机制：查询失败 raise；查询成功但状态不符返回 `None`（LMDB 读失败 raise；无标准返回 `None`）。

### 数据载体与信息流（data）

- **中间环节传全量领域建模**：port / usecase 是中间环节，须传全量领域信息（领域建模，非裸 str / dict）；"给不给用户看"是出口（driving adapter）职责，只在末端做。
- **类型化建模以处理为前提**：建模是有损抽象——类型化模型只留被建模的字段、丢弃其余，且原始→类型→原始不可逆。故是否类型化建模，取决于下游**消费语义**还是**原样搬运**：
  - **消费语义**（读字段做判断 / 计算 / 筛选 / 转换）→ 采用类型化建模，享类型安全。**无消费则不建模**：类型化跨过"无处理的边界"即空转，收紧类型（宽换窄）同样无人消费；正解是数据退回处理层、类型化内聚于此、不跨边界。
  - **原样搬运**（来源进什么、原样出什么，如 `raw_message` 回灌下一次 LLM 调用）→ 保留原始格式（裸 dict / wire 类型），**勿类型化**：类型化表达不了来源全部（厂商特定格式 / raw REST payload），转回即不可逆丢失。根源注释 why；属"裸 dict 白名单"的往返保真实例，不报。
- **裸 dict 白名单**：裸 dict 不一定违规——若为保留外部来源原始格式（provider 消息、evidence 快照）供序列化回放 / fidelity，是有意设计而非泄漏。判据：根源处是否有注释说明 why——有注释佐证＝有意白名单（不报）；无注释＝仍按裸 dict 信号报，并建议补注释。
- **出口策略不下沉采集口**：出口策略（如 reasoning exposure）不下沉到中间采集契约；下沉即"过早裁剪"——既污染 port 契约，又中途丢信息（下游要用就没了）。
- **bool 开关判去向**：bool 开关判去向——流向用户＝业务开关；流向 adapter 自解析＝手段。

### 适配器内部分层（adapter）

adapter 不直接碰基建 Client（`http_client` / `grpc_client`），只接触「来源 Client」——后者是 adapter 与基建之间唯一的桥。

- **基建（utils/）**：来源无关的纯传输（`HttpClient`、`GrpcClient`）。
- **来源（runtimes/）**：每个来源 = `Settings` + `Client`，Client 基于基建封装该来源（如 `BaseLlmClient` 组合 `HttpClient`）。
- **adapter**：组合来源 Client，翻译成 port 契约（DTO），禁碰基建。

**来源 settings 继承基建 settings**：基建提供连接/超时/重试等通用字段的 settings 基类，来源 settings 继承之即可直接喂养基建 Client（`Client(settings)`，settings is-a 基建 settings），无需翻译层。新建来源 settings 必须继承对应基建 settings 基类。

## 模块解剖

```
Port = 我要做什么
Adapter = 我用什么技术、怎么做到
Usecase = 我先做什么、后做什么
```

各模块目录树、port↔model↔adapter 映射表、Handler 依赖矩阵单独成文，存于 memory 池对应项目子目录（业务记忆，`_` 命名）。
