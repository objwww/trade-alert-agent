# 架构规范

适用：Gulimall Python 单服务告警 Agent 的系统设计、开发、重构与评审。

版本：2026-11-04；状态：按用户授权更新，已补齐落稿审查 P1 及本轮文档 P2，补充项待独立复验。仅架构章节，非全部工程规范已定稿。

## 变更与适用规则

2026-11-04：因项目明确采用 Python 单服务告警系统，将 Java/Spring 架构条款按原 ID 映射；纳入已裁决 ARCH-22～45，合并重复要求，不扩大首个 MVP。原文归档于 [更新前架构](../docs/history/2026-11-04-architecture-before.md)，逐条处置见 [映射与裁决](../docs/架构规范映射与裁决.md)。新要求与旧表述冲突以本版为准。

“必须／禁止”为适用条件内的硬约束；条件不成立登记“不适用＋原因＋重新适用条件”，不得以未实现假称通过。推荐项可记录理由取舍。所有必备证据是**后续实现验收要求**，不是当前已有测试；路径可在 MVP 方案映射到等价载体，但不得删去证据义务。本次只更新文档，未运行实现测试。

2026-11-04 补齐原因：独立复核发现恢复行为和 unknown 处理仅有测试主题、缺少正向判据，且 ARCH-03 级别变化漏记；本次补齐既有终裁，不重开选型。

## 唯一目录与源码依赖清单

```text
src/<项目包>/
  api/
  engine/
  domain/       # 规则按需放 domain/rules/
  collectors/
  notifiers/
  storage/
  main.py       # 仅装配入口
```

- api → engine → domain；collectors/notifiers/storage → engine 端口或 domain 契约。
- domain 不反向依赖；engine 不 import 适配器实现；适配器之间不直接耦合，协作经端口和用例；main.py 可以引用各层但不得被业务层反向引用。
- 箭头是源码依赖，不代表运行时调用方向；依赖倒置不构成双向导入。标准库纯计算类型允许用于 domain。
- 2026-11-04：domain/rules/ 替代原顶层 alert_rules/，减少职责重复并避开仓库根 rules/；不另增 adapters/。
- FastAPI、SQLite/PostgreSQL、采集方式与 import-linter 锁定版本均未定案；不因调研摘要提前锁定。

## 条款与必备证据

| ID | 级别 | 项目规则与适用条件 | 必备证据（待实现） |
| --- | --- | --- | --- |
| ARCH-01 | 必须 | 修改前阅读目标模块、调用方、契约和数据流；共享路径的问题不得仅修单入口。 | 评审记录列出读取范围、数据流和回归用例。 |
| ARCH-02 | 优先 | 复用已有能力和标准库；不预建无调用需求的接口、通用框架或空模块。 | 方案依赖清单及实际调用关系。 |
| ARCH-03 | 必须 | 采用 Python 模块化单体、单实例、单服务进程及 asyncio 基线；不引入微服务、多 Agent 或多进程来满足规范。 | 部署方案、进程清单；防重复启动测试见 ARCH-27。 |
| ARCH-04 | 必须 | 源码依赖向业务内核收敛；domain 不依赖本项目其他层、HTTP/ORM/Pydantic 等技术框架，不执行 I/O；engine 不依赖适配器实现。 | tests/architecture/test_dependencies.py：合法、直接越界、间接越界、循环四组样例。 |
| ARCH-05 | 必须 | api 经 engine 用例进入系统，不直接操作存储、采集器或通知客户端；main.py 只装配、启动和关闭资源，不放业务逻辑。 | 依赖契约检查与入口组件测试；装配入口评审。 |
| ARCH-06 | 必须 | api 负责协议、结构校验和身份提取；engine 负责用例、授权协调及编排；domain 负责规则与不变量；collectors/notifiers/storage 分别承担采集、投递和持久化适配。 | 模块职责表、边界类型转换及用例测试。 |
| ARCH-07 | 必须 | 端口按使用方和业务语义归属：用例所需采集、投递、持久化端口可放 engine；真正的领域仓储和契约放 domain。不强求全部端口进 domain，也不限制端口总数。 | 端口调用方清单与依赖机器检查；禁止无实际调用的一类一接口。 |
| ARCH-08 | 禁止 | 领域泄露 HTTP、ORM、SDK 或 API 校验模型；边界将外部数据转换为 domain 类型。领域值对象、事件、评估记录优先 frozen dataclass 并校验不变量；不要求所有生命周期对象冻结，不将浅冻结当深不可变。 | tests/domain/test_invariants.py 与边界转换测试；domain 框架导入检查。 |
| ARCH-09 | 条件必须 | 启用外部 SDK 时限定在对应适配目录；入口 SDK 仅作协议适配。未启用的 SDK 不建模块，记录“不适用＋原因＋重新适用条件”。 | SDK 导入契约、适配器契约测试；启用前补充版本与异常转换验证。 |
| ARCH-10 | 必须 | 上下文通过显式契约或事件协作，不访问对方内部状态或存储实现；越界和循环依赖分别验证。 | tests/architecture/test_dependencies.py 的越界、间接路径及循环样例。 |
| ARCH-11 | 推荐 | 仅共享真实复用且语义稳定的最小类型，不建无边界 common 模块。 | 类型使用方与职责评审记录。 |
| ARCH-12 | 必须 | engine 编排用例，domain 判断业务合法性；查询可走应用查询端口，不强制加载聚合、不强制 CQRS 或额外读模型层；禁止返回 api 类型造成反向依赖。 | 查询用例及返回类型检查、依赖门禁。 |
| ARCH-13 | 必须 | 持久化事务短小；外部 HTTP、通知、模型调用及异步等待置于事务外，采用准备、外部执行、结果提交分段流程。 | 事务范围集成测试；慢外部依赖不持有事务的验证记录。 |
| ARCH-14 | 必须 | 无论是否启用 LLM，都定义评估间隔、单轮超时、并发上限、取消、结构化执行记录和资源释放；启用 SDK 内部循环时不得绕过外层授权、次数和预算检查。 | tests/engine/test_execution_limits.py；run_id、started_at、duration_ms、outcome 日志字段及取消测试。 |
| ARCH-15 | 必须 | 本项目LLM根因分析存在计费/配额与资源预算，项目级适用，不得标不适用；执行前预留、执行后结算，并发原子控制，未知用量保守记账。token/成本/次数/超时限额由MVP量化，未定不派开发。 | tests/engine/test_model_budget.py：并发超额、失败、未知、重启结算；预算配置与审计。 |
| ARCH-16 | 必须 | 可恢复工作持久化意图、状态和结果，按调用前崩溃、外部成功本地未提交、重复调度等崩溃点定义恢复。重启后处于待投递状态的告警与通知事件必须继续投递，沿用原通知事件身份，不得构造新身份重发；已确认送达的按通知事件身份去重，不重复外发，禁止仅因重启将 pending 清为 failed。恢复执行仍遵守 ARCH-17/18/42；不能确定是否已送达时进入 unknown 确认/补偿流程，不盲目重发。业务告警状态与通知投递状态分开；pending/sent/failed/unknown 仅为投递状态。 | tests/recovery/test_delivery_recovery.py；崩溃点矩阵、状态转移及重启前 pending、重启后送达用例 recovery-restart-pending（判据见下表）。 |
| ARCH-17 | 必须 | 外部副作用区分成功、确定失败、未知；超时不等于失败，取消不承诺撤销已完成动作；未知写结果不得盲目重试。 | 通道能力表（幂等键/结果查询）、超时未知及补偿升级测试。 |
| ARCH-18 | 必须 | 授权落实在可信执行边界；配置、模型输出、第三方材料不能授予权限；恢复执行重新验证撤销与过期。静默、确认等管理操作鉴权并审计；P0 静默记录操作人、原因、到期时间。 | tests/security/test_authorization.py；actor、action、reason、expires_at、outcome 审计字段。 |
| ARCH-19 | 必须 | 采用 Python 3.12 系列，交付时固定具体补丁版本；pyproject.toml 声明依赖，uv.lock 锁定解析结果，使用对应版本文档；本次不虚构已锁版本。 | 交付环境版本、pyproject.toml、uv.lock 及可复现安装记录。 |
| ARCH-20 | 必须 | 依赖边界机器可验，覆盖实际包及合法、直接越界、间接越界、循环样例；工具不能证明的语义、事务、权限用测试补足。import-linter 为候选，版本及可执行样例在首个 MVP 方案核验。 | pytest 入口执行架构检查；CI 仅 ruff format --check、ruff check、mypy、pytest，不自动改源码。 |
| ARCH-21 | 推荐 | 重大边界及选型记录 ADR；当前模块、部署和信任边界记 ARCHITECTURE.md；状态机、字段、阈值和测试契约记 MVP 方案。 | ADR 的采纳/否决、独立核验依据及需求—验收映射。 |
| ARCH-22 | 必须 | 商城访问默认只读；若选数据库采集，上线前具有 DB 层只读授权、查询超时及频率上限。无只读账号不代表 HTTP 一定可替代；采集方式须核验源码和环境后选定。 | 只读权限拒写证据、超时和频率测试；真实商城写入、下单及压测须另获用户授权。 |
| ARCH-23 | 必须 | 故障仅在告警系统端口边界注入，优先测试替身；运行注入开关默认关闭、鉴权并审计。不对商城注入故障。 | tests/faults/ 的端口注入用例、默认关闭及审计测试、环境隔离声明。 |
| ARCH-24 | 必须 | 采集健康、评估和投递可关联；检测延迟、投递延迟、端到端延迟分别定义起止及测量。不能仅用 notify_ts-eval_ts 代表全链路。 | run_id、rule_id、source_id、notification_event_id 等有值关联；时序测试和延迟证据，未知起点明确标不可测。 |
| ARCH-25 | 必须 | 内部业务时间采用 aware UTC 并注入 now: Callable[[], datetime]；超时和耗时用单调时钟，阈值断言不依赖本地时区。 | tests/engine/test_clock.py：跨时区、业务时钟跳变及超时用例。 |
| ARCH-26 | 必须 | LLM用于根因分析和修复建议；触发、恢复与分级由确定性规则决定，阈值配置化。分析旁路、有界、失败降级且不延迟/丢告警；不执行商城修复。提示词/排查流程为可配置、版本化场景技能，按类型选择组合；具体治理见 DAILY-31/36～39、INFRA-15/18。项目级不得标不适用；分期延期须写交付依赖。 | tests/engine/test_model_fallback.py、test_analysis_dedup.py；tests/config/test_scenario_selection.py；模型基准评测及版本记录。 |
| ARCH-27 | 必须 | 单实例防双开，受控并发；阻塞 I/O 通过受控线程等方式隔离，设置超时、资源上限和退出清理；外部存活监测不能依赖已停止服务自身。 | 双启动、阻塞 I/O、取消、退出关闭和外部停服探测测试；线程取消不冒充底层调用已停止。 |
| ARCH-28 | 必须 | 分层强制合并执行 ARCH-04～10、20；沿用唯一六目录清单，不新增顶层 adapters/。端口归属不迁就某个工具；组合 Layers/Forbidden 等机器契约可接受。 | tests/architecture/test_dependencies.py 四类样例及命名契约：domain 零本项目导入、engine 不导入适配器实现、api 不导入 collectors/notifiers/storage、三适配器互不导入；main.py 独立装配豁免且业务层不得反向导入。候选工具锁定版本及实跑记录。 |
| ARCH-29 | 必须 | 对外提供稳定错误标识，对内定义明确异常；不强制 A/B/C 五位编码、统一 Result 包装或新增无消费者 API；无外部接口时标不适用。 | 如启用接口：异常映射、稳定标识及脱敏测试。 |
| ARCH-30 | 必须 | 采用 stdlib logging 加 JSON formatter，字段经 extra= 传递；关联字段按场景有值才带，不编造 alert_id。凭据和敏感数据脱敏，并定义轮转及保留策略，不机械照搬保留15天。 | tests/observability/test_logging.py；评估/投递字段和脱敏、换行输入、轮转配置检查。 |
| ARCH-31 | 必须 | 所有出站调用有显式单次超时、整轮预算及取消/资源释放策略；数据库按驱动分别界定连接、锁等待、语句执行期限，不能以一个 timeout 参数冒充全部保证。 | 慢 HTTP/DB/通知测试；依赖预算表和退出资源证据。 |
| ARCH-32 | 必须 | 密钥、含凭据 DSN、Webhook 配置不硬编码，不进仓库；配置与密钥分离，仅提交 .env.example；环境变量或受控密钥源加载。 | 配置加载/缺失校验、仓库敏感值检查及日志不泄密证据。 |
| ARCH-33 | 必须 | 边界校验类型、长度、范围和格式；Pydantic 仅用于 api 契约和采集侧外部解析，进入领域前转型。SQL 参数绑定；动态标识符用固定映射或白名单；不加载不可信 pickle/任意可执行规则。 | tests/security/test_input_validation.py；恶意输入、参数化查询、动态标识符及不安全反序列化拒绝测试。 |
| ARCH-34 | 条件必须 | 使用 HTTP/Webhook 等出站目标时按用途授权协议、主机及地址范围；验证重定向与 DNS 地址变化，实际连接遵循同一策略。商城内网地址仅显式授权，不能一刀切禁内网。 | tests/security/test_ssrf.py：未授权目标、重定向、解析变化拒绝及授权内网通过；无此出站则登记不适用。 |
| ARCH-35 | 条件必须 | 熔断按调用频率及失败成本决定，不全局强制。低频采集默认不用通用错误率熔断；连续失败/不可用时长触发 source_down，有界退避仍须满足恢复发现时限。启用熔断须定义样本、冷却、半开、过期数据及恢复。 | tests/collectors/test_source_health.py；MVP 指定阈值及最大探测间隔，不固定3次；失败不得当正常或静默停探。 |
| ARCH-36 | 必须 | 区分规则评估、告警生命周期、通知策略和投递恢复；记录分组、抑制、静默、路由、发送的语义。内部恢复事件始终保留；某接收目标确定未成功收到 firing 时不单独外发恢复，firing 投递结果为 unknown 时不得直接判定为未发出；按通道幂等键、结果查询或对账确认，确认已送达不重发；无法确认时进入补偿队列并记录风险，不得静默归类。ack 不等于送达或业务恢复。 | tests/notifiers/test_notification_pairing.py；抑制后恢复、分目标配对、unknown、重新发生及内部恢复事件用例；pairing-never-delivered-close、delivery-unknown-not-unfired 判据见下表。 |
| ARCH-37 | 必须 | 分组通知含分组维度、时间窗、严重度、实例总数、摘要、截断说明及可访问的详情引用；比例仅在分母有效时给出。分组等待和合并不得突破时延预算或丢 P0。抑制数量入审计，对外披露遵循权限。 | tests/notifiers/test_grouping.py：风暴、截断、恢复、时限、P0 及访问控制；不强制预建详情页面。 |
| ARCH-38 | 条件必须 | 启用规则/路由变更时，隔离 dry-run/影子评估，使用只读样本，不发送、不改正式状态；输出预计事件数、通知数及成本口径。这不是任意 Python 安全沙盒。 | tests/config/test_dry_run.py：正式状态和通知调用零变化，估算报告；未启用变更能力则登记条件。 |
| ARCH-39 | 必须 | 分别观测采集、评估、投递健康，与业务告警至少逻辑隔离；服务存活采用外部探测，明确共享通道失败的风险与升级路径，不假称逻辑隔离等同故障隔离。 | 健康指标/结构化记录、停服外部探测及通知通道故障演练证据。 |
| ARCH-40 | 必须 | 按状态、不变量和一致性要求文档化领域边界；跨边界通过 engine 用例/显式契约，不访问内部实现。不预设五个聚合、不强制一聚合一仓储；无值班需求不建 OnCall。 | 领域状态/不变量表、依赖检查与关键状态迁移测试。 |
| ARCH-41 | 条件必须 | 规则/路由配置版本化、校验后原子生效、可回滚并记录操作审计，禁止无版本热改；不强制语义化版本号，灰度按实际发布能力启用。 | tests/config/test_versioning.py：非法配置不替换当前版本、回滚、重启恢复；固定配置也要有可追溯版本。 |
| ARCH-42 | 必须 | 重试仅用于确认幂等或有明确去重/查询保证的操作；通知身份区分告警发生代次、接收目标、事件类型及序号，同事件重试不换身份。fingerprint 仅作分组，fingerprint+channel+window 不是通用幂等键。本地去重不等于通道幂等。 | tests/recovery/test_delivery_recovery.py：同事件重试、再次发生、恢复、unknown、重启；不支持通道幂等/查询须列重复与漏发风险及补偿。 |
| ARCH-43 | 条件必须 | 按需求配置分级、渠道与静默；P0 不得被风暴限流或无效静默吞掉。逐级升级和值班按实际需求启用，不在首个 MVP 自动预建。 | tests/notifiers/test_severity_policy.py；P0、静默权限/到期；启用升级时补充时序用例。 |
| ARCH-44 | 必须 | 生产禁 debug 暴露；外部响应不含内部路径、SQL、堆栈和凭据。无 HTTP 服务时不为本条创建 HTTP 模块。 | 生产配置检查与错误响应脱敏测试；无入口登记不适用范围。 |
| ARCH-45 | 必须 | domain 纯函数与规则确定性做单测；适配器做契约测试，并按场景补组件、恢复、端到端只读及端口故障注入。验收按“给定输入→可观测输出/条数/字段/时限”，逐项关联测试与证据。 | 测试报告及 tests/reports/artifacts/<用例ID>.log/.png；pytest --junitxml=... 输出报告，单元覆盖率按 DAILY-14 达98%，不新增CI入口。 |

## 恢复与配对的最低验收判据（待实现）

下列为项目裁决的可观测契约，不是已执行测试。通知事件身份至少区分告警发生代次、接收目标、事件类型及事件序号。

| 用例 ID | 给定输入与前置条件 | 可观测现象（判据） | 证据文件（待实现） |
| --- | --- | --- | --- |
| recovery-restart-pending | 重启前持久层存在 1 条携带原身份的 pending 通知事件，且可确认该事件尚未产生外部送达；授权有效、事件可投递、通道正常、无其他积压；恢复周期及通道预算由 MVP 量化。 | 从重启就绪起，在恢复调度周期＋通道响应预算内送达，投递状态为 sent；事件身份不变，duplicate_sent = 0；不得仅因重启清成 failed。 | tests/recovery/test_delivery_recovery.py；tests/reports/artifacts/recovery-restart-pending.log |
| recovery-restart-dedup | 外部已成功送达但本地结果未提交即崩溃，重启前持久层仍为 pending；通道结果查询或对账可确认原事件已送达，且授权有效。 | 按原事件身份确认并将本地状态收敛到 sent；重启后外发调用次数 = 0，duplicate_sent = 0；不得构造新身份重发。确认完成预算由 MVP 量化。 | tests/recovery/test_delivery_recovery.py；tests/reports/artifacts/recovery-restart-dedup.log |
| pairing-never-delivered-close | 某接收目标对该实例及发生代次确定无 firing 成功投递记录（被抑制/从未发出），随后告警恢复；排除 unknown。 | 本目标、本代次恢复处理的通知器调用次数 = 0；never_delivered 决策审计记录 = 1，含 instance_id、generation、reason=never_delivered；内部恢复事件与业务状态仍更新。正对照：同实例同代次另一目标已成功收到 firing、授权有效且恢复可投递，在通道正常时须于 MVP 规定预算内收到 1 条 close 通知。审计计数不限制其他类型日志。 | tests/notifiers/test_notification_pairing.py；tests/reports/artifacts/pairing-never-delivered-close.log |
| delivery-unknown-not-unfired | firing 投递结果为 unknown（超时/无回执）。 | 确认前投递状态保持 unknown，不得仅因未知转为 failed 或另造 not_sent 状态以触发普通重试；先持久化含 notification_event_id、target_id、generation、reason、next_action 的风险/待确认条目。通道支持结果查询/对账则先确认，确认已送达不重发；仅有幂等键时按通道契约复用同一身份安全重试，不把幂等键本身当送达凭据；无法确认则进入补偿队列且不得静默。无幂等/查询保证不得盲目重发。 | tests/notifiers/test_notification_pairing.py；tests/recovery/test_delivery_recovery.py；tests/reports/artifacts/delivery-unknown-not-unfired.log |

上述 duplicate_sent = 0 是指定可控场景的断言，不是无通道保证时的全局恰好一次承诺。补偿队列指持久化可跟踪待办，不要求新建消息中间件；容量、重试次数上限、处理期限、责任与升级路径由 MVP 方案写明；满载不得静默丢 P0。可恢复通知事件必须先持久化原身份和投递意图，再进入可执行调度或通知队列；MVP 须覆盖提交后入队前崩溃及外发前崩溃的恢复窗口，不指定数据库或消息中间件。

## 验收门与范围

- MVP-01 方案书必须包含条款适用性登记表：`条款ID | 适用/条件适用/不适用 | 原因 | 重新适用条件 | 证据文件`，逐项覆盖 ARCH-01～45，包括 ARCH-16/36 及 ARCH-09/15/26/29/34/35/38/41/43/44。缺行不通过；条件适用须说明触发条件和本 MVP 是否触发，不得以未实现替代不适用。上述四条最低验收判据须关联对应条款；确不适用需同样登记，不得静默删除。
- 每个 MVP 方案定义完整状态机、字段、时序、边界异常、启动配置、测试数据和可粘贴命令；用户未确认方案前不派开发任务。
- 告警专项覆盖去重、风暴、恢复、source_down、正常链路零误报、时延及分级不可被静默吞掉。时延预算由评估间隔＋for 窗口＋投递余量推导，分别声明测量起点；本规范不虚构统一数值。
- 测试边界覆盖输入、权限、依赖、状态、资源、终止六类；真实商城只读，不擅自下单、写入、压测或做故障注入。
- 队列和规则匹配基数有上限，定义满载时持久化、拒绝、聚合或升级策略，不得因满载静默丢 P0；自动修复/清理商城不在授权内，未来另行 ADR 和授权。
- 金额使用 Decimal、字符串造数；舍入按商城实际源码核验，不预设统一舍入模式。
- 四个 CI 入口不变；JUnit 是 pytest 的参数，不采用无效的 coverage.py --junitxml 写法。具体工具严格度及测试实现由后续工程规范和 MVP 方案确认。

## 来源与证据等级

来源思想不等于指定本项目实现；工具、类型和测试路径为项目自定。以下链接不是“全部章节审计通过”的声明。

- S1：[COLA](https://github.com/alibaba/COLA)：业务核心与技术解耦；不据此指定 import-linter。
- S2：[腾讯 Python 安全指南](https://github.com/Tencent/secguide/blob/main/Python安全指南.md)：输入、参数化、密钥、网络访问及脱敏安全思想；Pydantic 为项目映射。
- S3：[Kitex 熔断](https://www.cloudwego.io/zh/docs/kitex/tutorials/service-governance/circuitbreaker/)：失败策略候选，不照搬高频 RPC 参数。
- S4：[AntMonitor](https://www.sofastack.tech/blog/ant-intelligent-monitoring/)：变更隔离、自监控与成本评估思想，不照搬集群。
- S5：[美团 DDD](https://tech.meituan.com/2024/05/09/DDD-Practice-Trading-System.html)：边界按业务不变量划分，不据此预定聚合数。
- S6：[百度报警风暴](https://developer.baidu.com/article/detail.html?id=290328)：合并与逐级通告实践，不承诺其覆盖本项目全部管线。
- S7：[Alertmanager 概念](https://prometheus.io/docs/alerting/latest/alertmanager/)及[通知实现 v0.28.1](https://github.com/prometheus/alertmanager/blob/v0.28.1/notify/notify.go)：分组、抑制、静默及恢复通知参照，不宣称逐实例配对实现相同。
- S8：[import-linter Layers](https://github.com/seddonym/import-linter/blob/master/docs/contract_types/layers.md)、[Forbidden](https://github.com/seddonym/import-linter/blob/master/docs/contract_types/forbidden.md)：文档能力已核验；锁定版本与四类样例未执行，仍为候选。
- S9：[Python 3.12 asyncio](https://docs.python.org/3.12/library/asyncio-task.html)、[SQLite WAL](https://www.sqlite.org/wal.html)：WAL 不消除单写者与锁等待，不将同步驱动自动异步化。
- S10：[P3C](https://github.com/alibaba/p3c)：Java→Python 仅借鉴工程意图；黄山版 PDF 本轮未独立核验，不能以其具体数字为强制依据。
- S11：[12-Factor 配置](https://12factor.net/config)：配置与代码分离。
- 调研中的 Stars 为作者提供、未独立核验，不作为历史快照认证或选型依据。京东、华为等未逐项核验的具体原文，仅列为调研线索，不宣称公司级统一共识。

## 2026-11-04 用户需求变更

原因：项目明确需要LLM根因分析与修复建议，以及98%单测门槛。ARCH-15/26由条件必须→必须，原LLM默认关闭口径由[ADR-001](../adr/ADR-001-确定性告警与LLM根因分析.md)覆盖；ARCH-45覆盖率仅参考→按DAILY-14硬门槛。原稿见[本次变更前归档](../docs/history/2026-11-04-architecture-before-llm-revision.md)。本页不复制日常流程全文。
