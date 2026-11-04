# Python 接口层规范

变更日期：2026-11-04。原因：按 Python 单服务边界迁移协议约束、补齐契约与适用性。本文为协议层唯一权威；边界/依赖见 [01](01-architecture.md)，语言写法见 [02](02-coding-style.md)。原文见 [归档](../docs/history/2026-11-04-03-interface-layer-before.md)，逐条映射见 [裁决](../docs/编码与接口规范映射与裁决.md)。与旧文本冲突以本次新要求为准。

仅约束实际启用的入口，不要求同时实现 HTTP、RPC、SSE、管理、查询、回调或 A2A。外部存活探测必须满足 ARCH-39，但不必然 HTTP。下列证据均待实现，本轮不授权开发。

## 条款与必备证据

| ID | 级别 | 协议规则及条件 | 必备证据（待实现） |
| --- | --- | --- | --- |
| API-01 | 必须 | 见 ARCH-05/06；入口只做协议转换、结构校验、可信身份提取并调用 engine 用例，领域规则和事务不在入口。 | tests/architecture/test_dependencies.py；tests/api/test_usecase_dispatch.py。 |
| API-02 | 必须 | 已批准入口以一份权威 Schema/Spec 定义字段、单位、必填、默认、错误、授权、分页和幂等；不预建未启用协议。 | 入口契约清单及 tests/api/test_contracts.py。 |
| API-03 | 必须 | 见 ARCH-08/33；已批准 API 契约用 Pydantic 校验后转 domain 类型，不直接暴露领域/数据库对象；响应字段允许集合。不能推导所有适配器都必须 Pydantic。 | tests/api/test_boundary_conversion.py；响应字段泄露测试。 |
| API-04 | 推荐 | 简单请求/结果直接转换；重复复杂转换才提取函数，存储转换留其内部边界。 | 转换调用点评审与 tests/api/test_boundary_conversion.py。 |
| API-05 | 必须 | 见 ARCH-33；协议明确请求大小、类型、格式、长度、范围、允许字段；动态排序/URL 另有允许列表，领域不变量仍由领域保证。 | tests/security/test_input_validation.py；请求大小/越界/未知字段用例。 |
| API-06 | 必须 | 身份只来自可信认证上下文，不信任请求自报 actor/tenant；已启用查询、订阅、取消和下载分别校验资源归属。 | tests/security/test_authorization.py：伪造身份/越权资源拒绝。 |
| API-07 | 必须 | 见 ARCH-18；认证不代替资源授权，难猜 ID/隐藏按钮不算授权；无可信身份与权限配置时管理入口默认关闭。 | tests/security/test_authorization.py：未配置/未认证/已认证越权分别拒绝。 |
| API-08 | 必须 | 见 ARCH-29/44；边界错误为 ALERT_* 稳定语义字符串，与内部异常分离，不要求五位编码；遵循下列错误响应基线。 | tests/api/test_errors.py；tests/security/test_error_redaction.py。 |
| API-09 | 必须 | 普通 JSON 使用明确契约，不强制统一 Result；SSE/文件/第三方协议采用各自格式，不机械包 JSON。 | tests/api/test_response_shapes.py：成功/错误及协议类型。 |
| API-10 | 条件必须 | 有增长列表时定义上限、稳定排序/唯一打破平局键、空/末页、并发可见性；keyset 是否要快照/水位由具体契约决定，不无条件承诺不重不漏。 | tests/api/test_pagination.py：同值排序、并发变更、游标失效。 |
| API-11 | 条件必须 | 已有外部稳定客户端时，破坏字段/类型/必填/语义须迁移方案；新增枚举/默认行为也评估兼容，不强制语义化版本号。 | tests/api/test_compatibility.py；旧/新契约差异及迁移记录。 |
| API-12 | 条件必须 | 可重试写入口定义幂等身份、范围、有效期、重复/在途响应及同键不同参数冲突；外部副作用未知见 ARCH-17/42，不能盲重试。 | tests/api/test_idempotency.py；并发重复、冲突及未知结果。 |
| API-13 | 条件必须 | 有超过连接/网关预算的任务才异步化，定义期限、任务ID和查询契约，不设统一10秒，不为本条预建任务平台。 | tests/api/test_long_tasks.py；预算表与任务查询契约。 |
| API-14 | 条件必须 | 有取消/恢复能力才适用；区分请求受理、实际停止与外部动作已发生，pause/resume 仅公开验证过的能力。 | tests/api/test_cancellation.py；副作用已发生场景。 |
| API-15 | 条件必须 | 启用 SSE 才定义事件类型/ID/顺序、重连游标、保留窗口、重放/慢消费者；重连不得重复执行业务动作。 | tests/api/test_sse_reconnect.py；外部调用计数。 |
| API-16 | 条件必须 | 启用 SSE 时，头提交后使用流内错误/约定断流，不尝试改已发送响应；正常/失败/取消均释放资源。 | tests/api/test_sse_errors.py；连接与任务资源计数。 |
| API-17 | 条件必须 | 启用 SSE 时明确断开订阅是否取消任务，执行与订阅解耦，不因多订阅重复执行。 | tests/api/test_sse_subscription.py；多订阅执行次数。 |
| API-18 | 条件必须 | 启用 A2A 才锁协议/SDK/传输版本并验证发现、授权、任务/流/错误；Card 不含凭据和内部敏感信息。 | tests/api/test_a2a_contract.py；版本与发现信息脱敏。 |
| API-19 | 条件必须 | 启用消息/回调入口才验证来源、Schema、身份及重复投递，定义重试、顺序和失败处理，调用 engine；Webhook 验签与重放基线见下文。 | tests/api/test_inbound_callbacks.py；坏签名、过窗、重复、乱序及拒绝用例。 |
| API-20 | 必须 | 入口行为变化验证结构、认证/授权、响应/错误和失败路径；按适用条件加兼容/幂等/异步/流，不能只有成功用例。 | 验收—契约—测试映射，pytest JUnit 与 tests/reports/artifacts/<用例ID>.log。 |

## API-08/07/19 的协议基线（启用时生效）

- 错误 JSON 字段固定：`error_code: str`（ALERT_*）、`message: str`（可行动且脱敏）、`trace_id: str`（服务端生成/校验）、`retryable: bool`。不把“允许重试”当安全幂等或送达证明。HTTP 使用真实状态：校验400/422、未认证401、无权限403、不存在404、冲突409、限流429、内部500、依赖暂不可用503；具体错误映射在入口契约逐项列出，不全部200；已提交流遵循 API-16。
- 管理入口如启用 HTTP：`Authorization: Bearer <token>` 经 TLS 传递，密钥不进仓库/URL/日志，认证后另校验动作与资源权限；配置缺失默认不注册管理路由。非 HTTP 入口另列等价可信身份载体，不把本基线误用为所有入口强制 HTTP。静默/ack 的操作者、原因、到期和审计见 ARCH-18，P0 不因接口默认值绕过要求。证据：`tests/security/test_authorization.py` 的关闭、坏令牌、越权、过期和审计断言。
- 如果选 HTTP 健康探测，路径固定 `GET /healthz`，返回仅 `status: healthy|degraded`、`components: {collector/evaluator/delivery: healthy|degraded}`；healthy→200，任一退化→503。不返回 DSN、内部地址、路径、错误堆栈；组件未启用不编造健康值，契约排除该键。该路径不代替独立的外部停服探测。证据：`tests/api/test_health.py`、`tests/security/test_error_redaction.py`。未选 HTTP 无需实现本路径。
- 入站 Webhook 如启用自有签名协议：`X-Alert-Timestamp`（UTC Unix秒）、`X-Alert-Nonce`、`X-Alert-Signature`（HMAC-SHA256 十六进制），签名原文为 `timestamp + "\n" + nonce + "\n" + raw_body` 字节；常量时间比较。验签前限制大小，验签失败不执行业务；允许时钟偏差最大300秒，nonce 至少保留600秒并原子去重；密钥轮换/来源绑定在 MVP 明确。第三方协议则遵循其已核验签名规范并登记差异，不擅改供应商协议。测试冻结时钟覆盖边界±300秒、过窗、改包、重放及轮换；去重与业务幂等分开。证据：`tests/api/test_inbound_callbacks.py`。这些是启用条件约束，不是当前批准回调功能。

## API-01～20 全量适用性登记（规范层，不替代 MVP 确认）

“条款适用”与“本 MVP 必须实现某接口”分开。尚无已确认 MVP-01；下表“待方案确认”不是不适用或已通过，确认方案时必须逐行替换并附实际契约/证据。

| 条款ID | 适用/条件适用/不适用 | 原因 | 重新适用条件 | 本 MVP 必须实现某接口 | 证据（待实现） |
| --- | --- | --- | --- | --- | --- |
| API-01 | 适用 | 通用入口规范，约束不等于要求新增入口 | 启用任一相应外部入口 | 待方案确认；本轮不要求新增接口 | tests/architecture/test_dependencies.py；tests/api/test_usecase_dispatch.py。 |
| API-02 | 适用 | 通用入口规范，约束不等于要求新增入口 | 启用任一相应外部入口 | 待方案确认；本轮不要求新增接口 | 入口契约清单及 tests/api/test_contracts.py。 |
| API-03 | 适用 | 通用入口规范，约束不等于要求新增入口 | 启用任一相应外部入口 | 待方案确认；本轮不要求新增接口 | tests/api/test_boundary_conversion.py；响应字段泄露测试。 |
| API-04 | 适用 | 通用入口规范，约束不等于要求新增入口 | 存在重复复杂转换时评估 | 待方案确认；本轮不要求新增接口 | 转换调用点评审与 tests/api/test_boundary_conversion.py。 |
| API-05 | 适用 | 通用入口规范，约束不等于要求新增入口 | 启用任一相应外部入口 | 待方案确认；本轮不要求新增接口 | tests/security/test_input_validation.py；请求大小/越界/未知字段用例。 |
| API-06 | 适用 | 通用入口规范，约束不等于要求新增入口 | 启用任一相应外部入口 | 待方案确认；本轮不要求新增接口 | tests/security/test_authorization.py：伪造身份/越权资源拒绝。 |
| API-07 | 适用 | 通用入口规范，约束不等于要求新增入口 | 启用任一相应外部入口 | 待方案确认；本轮不要求新增接口 | tests/security/test_authorization.py：未配置/未认证/已认证越权分别拒绝。 |
| API-08 | 适用 | 通用入口规范，约束不等于要求新增入口 | 启用任一相应外部入口 | 待方案确认；本轮不要求新增接口 | tests/api/test_errors.py；tests/security/test_error_redaction.py。 |
| API-09 | 适用 | 通用入口规范，约束不等于要求新增入口 | 启用任一相应外部入口 | 待方案确认；本轮不要求新增接口 | tests/api/test_response_shapes.py：成功/错误及协议类型。 |
| API-10 | 条件适用 | 能力未批准，不提前实现 | 启用增长列表查询 | 待方案确认；本轮不要求新增接口 | tests/api/test_pagination.py：同值排序、并发变更、游标失效。 |
| API-11 | 条件适用 | 能力未批准，不提前实现 | 存在稳定外部客户端 | 待方案确认；本轮不要求新增接口 | tests/api/test_compatibility.py；旧/新契约差异及迁移记录。 |
| API-12 | 条件适用 | 能力未批准，不提前实现 | 启用可重试写入口 | 待方案确认；本轮不要求新增接口 | tests/api/test_idempotency.py；并发重复、冲突及未知结果。 |
| API-13 | 条件适用 | 能力未批准，不提前实现 | 启用长任务入口 | 待方案确认；本轮不要求新增接口 | tests/api/test_long_tasks.py；预算表与任务查询契约。 |
| API-14 | 条件适用 | 能力未批准，不提前实现 | 启用取消/恢复 | 待方案确认；本轮不要求新增接口 | tests/api/test_cancellation.py；副作用已发生场景。 |
| API-15 | 条件适用 | 能力未批准，不提前实现 | 启用SSE | 待方案确认；本轮不要求新增接口 | tests/api/test_sse_reconnect.py；外部调用计数。 |
| API-16 | 条件适用 | 能力未批准，不提前实现 | 启用SSE | 待方案确认；本轮不要求新增接口 | tests/api/test_sse_errors.py；连接与任务资源计数。 |
| API-17 | 条件适用 | 能力未批准，不提前实现 | 启用SSE | 待方案确认；本轮不要求新增接口 | tests/api/test_sse_subscription.py；多订阅执行次数。 |
| API-18 | 条件适用 | 能力未批准，不提前实现 | 启用A2A | 待方案确认；本轮不要求新增接口 | tests/api/test_a2a_contract.py；版本与发现信息脱敏。 |
| API-19 | 条件适用 | 能力未批准，不提前实现 | 启用消息/回调入口 | 待方案确认；本轮不要求新增接口 | tests/api/test_inbound_callbacks.py；坏签名、过窗、重复、乱序及拒绝用例。 |
| API-20 | 适用 | 通用入口规范，约束不等于要求新增入口 | 启用任一相应外部入口 | 待方案确认；本轮不要求新增接口 | 验收—契约—测试映射，pytest JUnit 与 tests/reports/artifacts/<用例ID>.log。 |

## 关联与待迁移

[架构](01-architecture.md)、[领域层（Python首版待复核）](04-domain-layer.md)、[基础设施层（Python首版待复核）](05-infrastructure-layer.md)、[日常工作（待迁移，DAILY-25 六类边界）](06-daily-work.md)。迁移顺序见裁决记录。未迁移的 Java 专属语法/工具不直接约束 Python；通用安全约束不因此取消。

参考：[HTTP 语义](https://www.rfc-editor.org/rfc/rfc9110)、[Python hmac](https://docs.python.org/3.12/library/hmac.html)、[Pydantic](https://docs.pydantic.dev/latest/)、[keyset 分页](https://use-the-index-luke.com/no-offset)。路径、错误字段、签名窗和鉴权基线是本项目自定，不声称来自上述来源的强制结论。
