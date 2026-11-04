# Python 基础设施层规范

变更日期：2026-11-04。原因：按用户“开始域和基础设施规范，Codex 先落盘”要求，迁移历史技术栈表述并对齐已生效架构、编码和接口裁决。状态：首版落稿，待独立一致性复核；不是实现验收或开发授权。新要求与历史文本冲突以本版为准。

原文见 [归档](../docs/history/2026-11-04-05-infrastructure-layer-before.md)，逐条处置、级别变化和条件见 [裁决与全量适用性表](../docs/领域与基础设施规范映射与裁决.md)。边界/依赖仅以 [01](01-architecture.md) 为权威，语言写法见 [02](02-coding-style.md)，协议契约见 [03](03-interface-layer.md)；本章仅补充基础设施层语义与证据，不复制架构全文。

所有证据路径均为待实现要求，沿用 CODE-21 测试布局，不表示文件存在或测试通过。条件不成立须登记“不适用＋原因＋重新适用条件”；未实现不是不适用。工具不能证明的语义须行为测试及评审，不虚报机器覆盖。

## 条款与必备证据

| ID | 级别 | Python 规则与适用条件 | 必备证据（待实现） |
| --- | --- | --- | --- |
| INFRA-01 | 必须 | 职责与依赖见 ARCH-05～09/28；技术适配分别落 collectors/notifiers/storage，不增顶层 adapters。转换技术类型/异常，不自行决定业务规则，入口经 engine。 | tests/architecture/test_dependencies.py；tests/collectors/test_contracts.py；tests/notifiers/test_contracts.py；tests/storage/test_contracts.py。 |
| INFRA-02 | 必须 | 客户端由 main.py 装配、明确所有权；按实际数据源/通道配置具名实例，共享连接不共享请求可变状态；生命周期及锁版本见 CODE-13、ARCH-19。 | tests/engine/test_resource_lifecycle.py；pyproject.toml、uv.lock；装配清单。 |
| INFRA-03 | 必须 | 见 ARCH-17/31/42、CODE-20；逐依赖声明连接、请求/语句、锁等待及整轮期限，有限退避与抖动，不叠加重试。确定无外部副作用的失败按契约重试；unknown 先确认/安全幂等重试/补偿。 | tests/engine/test_retry_budget.py；tests/collectors/test_timeouts.py；tests/notifiers/test_timeouts.py；依赖预算表。 |
| INFRA-04 | 条件必须 | 使用数据库时，参数绑定查询，动态标识符固定映射/白名单；关键唯一性由数据库约束实现并处理冲突，禁止先查再插冒充并发保证。商城只读权限见 ARCH-22。 | tests/storage/test_constraints.py；tests/security/test_input_validation.py；tests/collectors/test_readonly_access.py。 |
| INFRA-05 | 条件必须 | 使用持久化时，存储对象不向端口泄露 ORM/行对象/游标；受控转换为 domain 契约或 engine 读模型，不强制转换框架。 | tests/storage/test_mapping.py；tests/storage/test_contracts.py。 |
| INFRA-06 | 条件必须 | 存储契约声明字段类型、长度、精度、索引及身份隔离；金额/时间引用 CODE-09/10。敏感大字段声明授权、保留与清理策略，不预建多租户。 | MVP 存储字段/索引/保留表；tests/storage/test_schema_contract.py；tests/security/test_data_isolation.py。 |
| INFRA-07 | 条件必须 | 应用过的版本化迁移不回改；新脚本修正，验证空库及支持旧版升级。明确备份恢复、前向修复和不可回滚 DDL；不预设 Alembic/ORM。 | tests/storage/test_migrations.py：空库、旧版、失败；备份恢复演练记录。 |
| INFRA-08 | 条件必须 | 结构/大批数据变更先评估锁、数据量、磁盘及实际部署影响；在线方案取决于所选数据库。清理按保留策略，不能删除待确认/补偿依赖数据或统一永久软删。 | MVP 迁移影响清单；tests/storage/test_retention.py：待办保护、到期清理与容量边界。 |
| INFRA-09 | 条件必须 | 启用缓存才定义事实来源、键空间、授权、过期、失效与一致性窗口；缓存与授权/幂等/预算存储失败策略分开，不统一放行。 | tests/storage/test_cache_policy.py；缓存失败策略表。 |
| INFRA-10 | 条件必须 | 恢复语义唯一见 ARCH-16/27/36；实现持久意图、原身份、结果及安全 checkpoint。当前单实例防双开；多实例领取/fencing 仅批准架构变更后另设计，不预建。 | tests/recovery/test_delivery_recovery.py；tests/engine/test_single_instance.py；崩溃点矩阵。 |
| INFRA-11 | 条件必须 | 外部写记录操作、事件/请求身份、目标/主体、请求摘要、状态、结果引用及期限；按实际能力用唯一约束处理竞争，同键异参冲突。通知身份见 ARCH-42；未知结果核对见 ARCH-36，不用 fingerprint+channel+window。 | tests/storage/test_idempotency.py；tests/recovery/test_delivery_recovery.py：重复、冲突、unknown。 |
| INFRA-12 | 条件必须 | 见 ARCH-17/42；逐通道核验幂等键、查询/对账能力及期限，无确认能力进入可跟踪补偿。数据库锁/执行权不构成通道恰好一次保证，不以幂等键证明送达。 | MVP 通道能力/风险表；tests/notifiers/test_confirmation.py；tests/recovery/test_delivery_recovery.py。 |
| INFRA-13 | 条件必须 | 见 ARCH-13/16；业务状态与待发布事件需要原子一致时同一事务保存状态与事件/发布记录，先持久化再可执行入队。持久待办表可满足，不强制消息中间件或命名 Outbox；提交后内存回调不等于可靠交付。 | tests/storage/test_atomic_event_commit.py；tests/recovery/test_delivery_recovery.py：提交后入队前及外发前崩溃。 |
| INFRA-14 | 条件必须 | 授权与可靠意图见 ARCH-18/36；动作前保存必要授权证据及意图，动作后记录结果，unknown 持久待确认。补偿容量、次数、期限、负责人及升级由 MVP 量化，满载不得静默丢 P0。 | tests/security/test_authorization.py；tests/recovery/test_compensation.py；MVP 补偿参数表。 |
| INFRA-15 | 必须 | LLM根因分析为项目必需，见 ARCH-26；模型输出逐层验证解析、Schema、字段和语义，失败不伪造成功/重做写动作；真实用量来自响应，缺失记未知。 | 启用 ADR；tests/engine/test_model_output.py；tests/engine/test_model_fallback.py。 |
| INFRA-16 | 条件必须 | 启用工具时按身份/任务限制工具集，每次执行前验证参数、资源、当前授权/审批；可信身份不来自模型参数，见 ARCH-18。 | tests/security/test_tool_authorization.py：撤销、伪造身份及逐次授权。 |
| INFRA-17 | 条件必须 | 使用 Python SDK 自动循环/中间件时核验实际版本和调用顺序，覆盖流式、拒绝、取消及多轮，每次副作用均授权。 | SDK 版本/顺序记录；tests/engine/test_sdk_loop.py。 |
| INFRA-18 | 必须 | 启用模型编排时记录 Prompt、参数、工具 Schema、策略/编排版本或摘要，运行绑定可追溯配置；变更/回滚验证兼容，不要求热更新。规则配置仍见 ARCH-41。 | tests/config/test_model_versioning.py；运行版本字段契约。 |
| INFRA-19 | 条件必须 | 启用记忆/RAG 时按实际身份及资源 ACL 控制存取，会话ID/向量元数据不是授权证明；空结果和检索故障区分，不伪造证据。 | tests/security/test_retrieval_acl.py；tests/collectors/test_retrieval_failures.py。 |
| INFRA-20 | 条件必须 | 启用 MCP/外部工具协议时锁协议/SDK，授权服务器、工具和出站目标；远端描述/注解不构成信任，Schema/工具变化重新核验权限与契约。 | tests/security/test_external_tools.py；协议/SDK 锁版和变更记录。 |
| INFRA-21 | 条件必须 | 不可信代码执行需单独 ADR 和授权；匹配威胁模型隔离身份、文件、网络及资源，不带生产凭据。普通子进程或 dry-run 不称安全沙箱。 | 威胁模型/隔离方案；tests/security/test_sandbox.py。 |
| INFRA-22 | 必须 | 不可信数据控制见 ARCH-18/33/34；采集结果、通知回执、检索文档不能授予权限；在适配边界拒绝注入、未授权出站、路径穿越及可执行反序列化。 | tests/security/test_input_validation.py；适用时 tests/security/test_ssrf.py、tests/security/test_external_tools.py。 |
| INFRA-23 | 必须 | 密钥与诊断保密见 ARCH-30/32；技术实现限制敏感报文采样、访问及保留，配置/异常输出均脱敏，仅提交 .env.example。 | tests/security/test_secret_config.py；tests/observability/test_logging.py；配置/保留清单。 |
| INFRA-24 | 必须 | 观测见 ARCH-24/30/39；记录各阶段错误、时延、重试、状态和积压。指标标签有界，订单/事件/用户ID只作受控日志关联，不作无界指标标签；计费以账本为准。 | tests/observability/test_metrics.py；tests/observability/test_latency.py；标签允许表与积压字段契约。 |
| INFRA-25 | 必须 | Python 上下文隔离见 CODE-19；asyncio/contextvars 按 token 清理，线程边界显式传递；恢复任务通过原事件ID和 run_id 关联。外部存活及处理责任见 ARCH-39。 | tests/engine/test_context_isolation.py；tests/observability/test_recovery_correlation.py；外部停服探测记录。 |
| INFRA-26 | 必须 | 验证见 ARCH-23/45；选定数据库的真实约束、迁移、映射、并发及失败路径做组件验证，不能只用 mock。未知、重复、恢复和隔离依用例逐项断言，禁止对商城注入故障。 | tests/storage/test_constraints.py；tests/storage/test_migrations.py；tests/recovery/test_delivery_recovery.py；tests/faults/ 实际节点；MVP 测试报告。 |

## 验收与关联

首个 MVP 逐行覆盖本章条款，列出适用性、原因、重新适用条件、具体契约和实际测试节点。参数、状态集合、超时/容量/重试次数在方案中量化，未确认不派开发。四入口仍为 ruff format --check、ruff check、mypy、pytest；架构检查由 pytest 承接。

数据库、采集方式及驱动仍待源码/环境核验；SQLite WAL 不是消除写锁的保证，不强制 PostgreSQL、Redis、消息中间件、ORM 或迁移工具。通知恢复必须覆盖 ARCH-16/36 的四个既定用例，不重定义其判据。

关联：[领域](04-domain-layer.md)、[日常验证（待迁移，DAILY-25 六类边界）](06-daily-work.md)。本轮不创建 src/、tests/、MVP 工程骨架或生产代码。

## 参考与归因

本章是项目自定的 Python 映射，不宣称来源逐条强制相同实现；本轮未新增外部源码/工具实测，链接为参考入口，非核验通过证明。沿用先前已记录依据，具体选型在 MVP 独立核验。

- [Python asyncio](https://docs.python.org/3.12/library/asyncio-task.html)：取消及任务生命周期参考。
- [SQLite WAL](https://www.sqlite.org/wal.html)：仅为候选数据库特性参考，不视作已选型。
- [Alertmanager 通知实现](https://github.com/prometheus/alertmanager/blob/v0.28.1/notify/notify.go)：通知可靠性参考，不照搬为逐目标配对证明。

2026-11-04变更：INFRA-15/18项目级适用，级别条件必须→必须，因用户明确要求LLM根因分析；版本与数据、预算、评测流程见 DAILY-12/13/31/36～39，决策见[ADR-001](../adr/ADR-001-确定性告警与LLM根因分析.md)。工具/RAG/MCP/沙箱仍按实际能力登记，使用LLM不自动批准工具执行。
