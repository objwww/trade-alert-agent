# Python 领域层规范

变更日期：2026-11-04。原因：按用户“开始域和基础设施规范，Codex 先落盘”要求，迁移历史技术栈表述并对齐已生效架构、编码和接口裁决。状态：首版落稿，待独立一致性复核；不是实现验收或开发授权。新要求与历史文本冲突以本版为准。

原文见 [归档](../docs/history/2026-11-04-04-domain-layer-before.md)，逐条处置、级别变化和条件见 [裁决与全量适用性表](../docs/领域与基础设施规范映射与裁决.md)。边界/依赖仅以 [01](01-architecture.md) 为权威，语言写法见 [02](02-coding-style.md)，协议契约见 [03](03-interface-layer.md)；本章仅补充领域层语义与证据，不复制架构全文。

所有证据路径均为待实现要求，沿用 CODE-21 测试布局，不表示文件存在或测试通过。条件不成立须登记“不适用＋原因＋重新适用条件”；未实现不是不适用。工具不能证明的语义须行为测试及评审，不虚报机器覆盖。

## 条款与必备证据

| ID | 级别 | Python 规则与适用条件 | 必备证据（待实现） |
| --- | --- | --- | --- |
| DOMAIN-01 | 必须 | 统一术语，状态、单位与规则语义以 MVP 契约为准；告警分组、发生代次、接收目标、通知事件分别定义，不以 fingerprint 代替事件身份。 | tests/domain/test_terminology.py；MVP 术语与字段表。 |
| DOMAIN-02 | 必须 | 见 ARCH-40；围绕共同不变量定义最小状态边界，不预设五个聚合或一聚合一仓储，不把投递历史及技术上下文塞入聚合。 | MVP 边界/不变量表；tests/domain/test_invariants.py。 |
| DOMAIN-03 | 必须 | 状态通过业务行为改变并保证不变量；不依赖入口代检，不提供绕过规则的任意 setter；内部集合不得暴露可写引用。 | tests/domain/test_invariants.py：非法变化拒绝、外部集合修改不影响内部状态。 |
| DOMAIN-04 | 必须 | 见 ARCH-16/36；业务告警状态与投递状态分开。MVP 列合法迁移、重复、终态、再次发生与取消语义；未知不等于未发出，ack 不等于恢复或送达。 | tests/domain/test_state_transitions.py；四个恢复用例引用 ARCH-16/36。 |
| DOMAIN-05 | 推荐 | 见 ARCH-13；事务围绕一致性要求而非形式设计，同库跨边界原子更新记录理由；不得把外部调用放入事务。 | MVP 事务边界表；tests/storage/test_transactions.py。 |
| DOMAIN-06 | 必须 | 值对象构造/受控工厂校验空值、范围、单位、精度、币种；金额写法见 CODE-09，时间见 CODE-10，不另定舍入规则。 | tests/domain/test_value_objects.py：非法输入构造失败；金额入口 tests/coding/test_no_float_money.py。 |
| DOMAIN-07 | 必须 | 见 ARCH-08、CODE-12；值对象使用 Python 不可变表示，嵌套集合冻结或防御复制；相等基于业务值，只有需作键/集合成员时要求可哈希，不强制冻结生命周期实体。 | tests/domain/test_invariants.py：嵌套修改、值相等和按需哈希断言。 |
| DOMAIN-08 | 条件必须 | 存在计费/配额时见 ARCH-15，区分额度、已用、预留与未知；拒绝新超额动作但如实记录实际超额及待核对值。未启用不建预算子系统。 | tests/domain/test_budget.py：预留、超额、未知；未启用登记原因与重新适用条件。 |
| DOMAIN-09 | 条件必须 | 存在授权/风险业务判断时，规则覆盖身份、资源、参数、敏感性及副作用；见 ARCH-18/22，不信任模型标签，“只读”也须授权。 | tests/security/test_authorization.py：敏感读取与身份伪造；MVP 风险规则表。 |
| DOMAIN-10 | 必须 | 边界见 ARCH-04/08/28；领域仅允许标准库纯计算依赖；架构测试明确拒绝 fastapi、pydantic、sqlalchemy、httpx、aiosqlite 及其子模块，并按实际依赖清单扩展，不将该清单视为穷举。 | tests/architecture/test_dependencies.py：domain 零本项目其他层导入与框架禁入。 |
| DOMAIN-11 | 必须 | 领域服务只承载不自然属于单实体的业务规则，保持确定性且不执行 I/O。原文允许领域服务执行外部 I/O 的例外不保留；外部交互由 engine 端口编排。 | tests/domain/test_rule_determinism.py；tests/architecture/test_dependencies.py；服务调用评审。 |
| DOMAIN-12 | 必须 | 端口归属见 ARCH-07：真正领域语义契约可属 domain；采集、投递、持久化等用例端口按使用方属 engine。领域可声明契约，不自行调用 I/O 实现。 | MVP 端口—使用方表；tests/architecture/test_dependencies.py。 |
| DOMAIN-13 | 禁止 | 见 ARCH-04/25；领域执行 SQL、HTTP、模型/工具调用或用全局容器取实现；时间规则显式接收 aware UTC 时间参数，不读取不可控系统时间。 | tests/domain/test_rule_determinism.py；tests/architecture/test_dependencies.py；tests/engine/test_clock.py。 |
| DOMAIN-14 | 条件必须 | 采用领域事件时，事件表示已发生事实，使用不可变快照而非可变实体引用；无需消费/可靠交付可直接返回行为结果，不按字段变化机械造事件。 | tests/domain/test_events.py：事实字段、嵌套不可变与实体后续修改隔离。 |
| DOMAIN-15 | 条件必须 | 跨上下文/外部集成事件明确事件ID、类型、版本、主体及实际需要的身份上下文；不暴露内部对象。告警通知事件身份见 ARCH-42，由应用及存储协作可靠交付。 | tests/domain/test_events.py；tests/notifiers/test_event_contract.py；MVP 集成契约。 |
| DOMAIN-16 | 必须 | 区分领域事件、审计、状态日志、事件溯源；只有从事件重建状态才称事件溯源。状态重放不得重新发送通知或执行外部写动作。 | MVP 事件分类表；采用重放时 tests/recovery/test_replay.py 断言外部写调用为0。 |
| DOMAIN-17 | 必须 | 见 ARCH-45；领域测试使用 pytest 纯单元执行、不装配 app，无需启动 Web/数据库/SDK，覆盖合法非法迁移、边界、重复、集合封装及异常；金额/预算依各条款条件登记。 | tests/domain/test_state_transitions.py；tests/domain/test_invariants.py；验收—用例映射。 |
| DOMAIN-18 | 必须 | 隔离门禁见 ARCH-20/28；不可变与不变量用行为测试证明，不以 frozen、mypy 通过或覆盖率代替深不可变和业务正确性。 | tests/architecture/test_dependencies.py；tests/domain/test_invariants.py；pytest JUnit 与评审记录。 |

## 验收与关联

首个 MVP 逐行覆盖本章条款，列出适用性、原因、重新适用条件、具体契约和实际测试节点。参数、状态集合、超时/容量/重试次数在方案中量化，未确认不派开发。四入口仍为 ruff format --check、ruff check、mypy、pytest；架构检查由 pytest 承接。

不预设 Alert/Rule/Silence/Route/OnCall 聚合，不因规范引入事件溯源或复杂 DDD。告警规则按需落 domain/rules/。

关联：[基础设施](05-infrastructure-layer.md)、[日常验证（待迁移，DAILY-25 六类边界）](06-daily-work.md)。本轮不创建 src/、tests/、MVP 工程骨架或生产代码。

## 参考与归因

本章是项目自定的 Python 映射，不宣称来源逐条强制相同实现；本轮未新增外部源码/工具实测，链接为参考入口，非核验通过证明。沿用先前已记录依据，具体选型在 MVP 独立核验。

- [Python dataclasses](https://docs.python.org/3.12/library/dataclasses.html)：语言机制参考；深不可变仍需行为测试。
- [美团 DDD 实践](https://tech.meituan.com/2024/05/09/DDD-Practice-Trading-System.html)：边界设计参考，不指定聚合数。
