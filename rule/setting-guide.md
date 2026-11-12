# settings.py 配置建模规范

各项目 `{package}/settings.py` 是全项目唯一的配置聚合点。本规范规定**配置类的分层、继承、注释与空行约定**，与 [architecture-guide.md](architecture-guide.md) 的「目的 / 手段」切分配套：settings 同样区分基建配置（手段）与业务配置（目的）。

## 核心原则：基建 / 业务分层 + 组合优于继承

> **基建 settings 承载手段，业务 settings 承载目的；业务以组合（内嵌子类）复用基建，来源 settings 继承基建基类以喂养基建 Client。**

| 类别             | 承载                                            | 变化频率   | 例                                                                                      |
| ---------------- | ----------------------------------------------- | ---------- | --------------------------------------------------------------------------------------- |
| **基建类** | 通用连接/超时/重试/传输/集群/向量库（来源无关） | 换来源不变 | `HttpClientSettings`、`GrpcClientSettings`、`QdrantSettings`、`ClusterSettings` |
| **业务类** | 各功能模块配置                                  | 跟随业务变 | `{Feature}Settings`，如聚类、分析、数据集配置 |

## 文件分区（定义顺序即依赖顺序）

文件按 5 个大区排列，每区以三行 `# ====` 横幅分隔。**定义顺序必须满足继承 / 引用依赖**——基类先于子类、被引用者先于引用者：

| 顺序 | 大区（横幅标题）  | 内容                           | 依赖要求                                              |
| ---- | ----------------- | ------------------------------ | ----------------------------------------------------- |
| 1    | Infra settings    | 基建基类 + 来源骨架            | 全部基类先定义                                        |
| 2    | Settings helpers  | `_apply_*` 优先级 / 连接传播 | 需`ModelSettings` 在前；需在聚合根 `Settings` 之前 |
| 3    | Business settings | 各功能配置类                   | 需对应基建类在前                                      |
| 4    | Aggregation root  | `Settings`                   | 需全部业务类在前                                      |
| 5    | Load helpers      | `load_settings` 等           | 可用 forward ref`"Settings"`                        |

> 顺序不是审美，是 Python 的硬约束：业务类内嵌 `class ClusterSettings(ClusterSettings)` 要求全局 `ClusterSettings` 已定义；`_apply_xxx` 传播器要求定义在聚合根 `Settings` 的 `model_validator` 之前。

## 继承范式

### 1. 来源 settings 继承基建基类

来源 settings **is-a** 基建 settings，可直接喂养基建 Client（`Client(settings)`），无需翻译层：

| 来源 settings            | 继承                   | 来源额外补的字段                                    |
| ------------------------ | ---------------------- | --------------------------------------------------- |
| HTTP 来源 `{Source}ClientSettings` | `HttpClientSettings` | `base_url` / `token` + 校验 |
| gRPC 来源 `{Source}Settings`       | `GrpcClientSettings` | `enabled` + 订阅/规则骨架  |

> 新建来源 settings **必须**继承对应基建基类（HTTP 来源继承 `HttpClientSettings`，gRPC 来源继承 `GrpcClientSettings`）。

### 2. 多继承混入 ModelHttpConnection

Remote 连接需要扁平 HTTP 端点字段时，多继承混入：

```
class ConnectSettings(_XxxConnectSettings, ModelHttpConnection): ...
```

`ModelHttpConnection` 自身继承 `HttpClientSettings`，混入后 Control/Connect 基类对仍各自独立。

### 3. 业务内嵌子类组合基建（组合优于继承）

业务配置通过**内嵌同名子类**继承全局基建类，按需添加业务字段——业务类**不**直接 is-a 整个基建类：

- 业务类（`{Feature}Settings`）内嵌与全局基建类**同名**的子类（如 `class ClusterSettings(ClusterSettings)`、`class QdrantSettings(QdrantSettings)`），各自继承同名全局类，按需添加业务字段。

> 内嵌类的基类名在类体执行时**解析到全局**（此时自身名尚未绑定），故 `class ClusterSettings(ClusterSettings)` 的内层 `ClusterSettings` 指全局基建类，不是自身。这是有意为之。

### 4.  内嵌类继承的字段遮蔽陷阱

父类声明的字段类型在**类定义时冻结**。业务内嵌子类继承父内嵌类（如 `Subscriber`）并添加字段后，父类那行继承字段（如 `subscriber: Subscriber`）**不会**自动把类型切到新子类——必须在业务子类中**重新声明**该字段，绑定到已填充的内嵌类，并以注释标明 why。

范例：业务子类继承父内嵌类并新增字段后，重声明父类中被冻结的字段（绑定到新内嵌类），并附「pydantic fixes field type at class-definition time」注释。

## ModelSettings 的 control / connect 拆分

每类模型（Chat / Embedding / Rerank）→ `Local` + `Remote`，各有 `ControlSettings`（参数/采样）+ `ConnectSettings`（连接）：

| 基类对                  | 职责                           | 子类做什么                                            |
| ----------------------- | ------------------------------ | ----------------------------------------------------- |
| `_XxxControlSettings` | 参数基类（可有 required 字段） | 提供 deployment 默认（`batch_size`/`max_length`） |
| `_XxxConnectSettings` | 连接基类（常为空`pass`）     | Local 补路径 / Remote 多继承`ModelHttpConnection`   |

Local 的 `Remote` 字段类型为 `Remote | None = None`，按需启用。

## 注释规范

| 场景         | 写法                                             | 原则                                            |
| ------------ | ------------------------------------------------ | ----------------------------------------------- |
| docstring    | 单行优先（≤~100 字符）                          | 信息优先，删冗余词；一行放不下且关键才多行      |
| 空骨架类     | `pass` + 单行 docstring 标 base/reserved/empty | —                                              |
| 大区分隔     | 三行`# ====` 横幅（标题居中行）                | Infra / Helpers / Business / Aggregation / Load |
| 文件级子组   | `# --- Group ---`（两端各 3 短横线）           | 紧贴其下一组内聚紧密的类                        |
| 类内字段分组 | `    # -- Group --`（缩进，两端各 2 短横线）   | 区别于子组的`---`                             |
| 约束性注释   | 保留 MUST / 不变量语义                           | 如`touch_interval MUST < nsqd --msg-timeout`  |

注释一律用英文（见 [note-guide.md](note-guide.md)）。

## 空行规范

**空行密度表达内聚层级**，不可被格式化工具统一抹平——这是有意设计，不是疏漏：

| 位置                     | 空行数         | 判据 / 例                                                                |
| ------------------------ | -------------- | ------------------------------------------------------------------------ |
| 跨大区 / 顶层独立类之间  | **两行** | 类之间无内聚关系：`HttpClientSettings`↔`GrpcClientSettings`         |
| 同一子组内紧密相关类之间 | **一行** | 同属一个内聚子组：`_ChatControlSettings`↔`_ChatConnectSettings`↔… |
| 类内字段分组之间         | **一行** | `# -- Group --` 之间                                                   |

**判据**：这一组类是否同属一个内聚子组？是 → 一行；否（跨组 / 独立）→ 两行。文件级子组注释 `# --- ... ---` 前留一行空行、后紧贴其类。

## 命名与校验约定

- 类：`{Feature}Settings` / `{Source}ClientSettings`；基建基类 `HttpClientSettings` / `GrpcClientSettings` / `ModelHttpConnection`。
- 字段：snake_case；可变默认用 `Field(default_factory=...)`，不用裸 `[]`/`{}`。
- 单字段校验：`_validate_xxx`（`@field_validator`，多为 `mode="after"`）。
- 跨字段默认传播：`_apply_xxx`（`@model_validator(mode="after")`）。
- `BaseSettings.model_dump` 已统一重写为 `by_alias=True, exclude_unset=True, exclude_none=True`——子类勿覆盖此语义。

## Settings 聚合根

字段按 `# -- Infra --` / `# -- Business --` / `# -- Health check --` 分组；`model_validator` 把全局默认（连接、调度优先级）传播到各功能 chat/embedding 配置。

## 禁止事项

| 禁止                                                                 | 正确做法                                |
| -------------------------------------------------------------------- | --------------------------------------- |
| 业务类直接 is-a 整个基建类（如 `class {Feature}Settings(ClusterSettings)`） | 内嵌子类组合基建          |
| 新建来源 settings 不继承基建基类                                     | 继承`Http/GrpcClientSettings`         |
| 内嵌子类继承父内嵌类后不重声明冻结字段                               | 重声明字段绑定到已填充内嵌类 + 注释 why |
| docstring 多行展开噪声                                               | 单行优先                                |
| 分组注释用`&` 打包概念                                             | 如实命共性                              |
| 用 find-and-replace 改类/字段名                                      | 用 GitNexus`rename`（见 CLAUDE.md）   |
| 格式化工具抹平空行差异（子组一行 / 跨组两行）                        | 按内聚层级保留                          |

## 改动 Checklist

- [ ] 继承：基建基类先定义；来源 settings 继承基建；业务用内嵌子类组合。
- [ ] 内嵌继承：冻结字段已重声明并注释 why。
- [ ] 定义顺序：满足全部依赖（被引用者先于引用者）。
- [ ] 注释：docstring 单行；分组注释如实命名、无 `&`；约束注释保留 MUST。
- [ ] 空行：子组内一行、跨组两行，未被抹平。
- [ ] 校验：`_validate_*` / `_apply_*` 命名一致；可变默认用 `default_factory`。
- [ ] 改动前已对所编辑类跑 `impact`；提交前跑 `detect_changes`（见 CLAUDE.md）。
