# Python 编码规范

变更日期：2026-11-04。原因：按最终裁决完成 Java→Python 迁移并纠正证据载体。本文为语言级唯一权威；架构边界只引用 [01](01-architecture.md)，协议见 [03](03-interface-layer.md)。与旧文本冲突以本次新要求为准。

原文：[归档](../docs/history/2026-11-04-02-coding-style-before.md)；逐条处置、级别变化和评审例外见 [映射与裁决](../docs/编码与接口规范映射与裁决.md)。所有测试路径是待实现要求，不是已有测试或通过证明。本次不建工程骨架、不授权开发。

## 条款与必备证据

| ID | 级别 | Python 规则 | 必备证据（待实现） |
| --- | --- | --- | --- |
| CODE-01 | 必须 | 使用 Ruff 格式化与检查，CI 不自动修改；配置见本页附表。版本遵循 ARCH-19/20，规则存在性与正反探针在 MVP 锁版。 | pyproject.toml、uv.lock；tests/coding/test_tool_config.py；四入口原始输出。 |
| CODE-02 | 必须 | 模块/包 snake_case，类 UpperCamelCase，函数/变量 snake_case，常量 UPPER_SNAKE_CASE；标准库、第三方、本项目分组导入，不以星号导入隐藏依赖。 | Ruff N/I/F 正反探针与命名评审记录。 |
| CODE-03 | 必须 | 布尔值名称明确；三态业务使用有明确成员语义的 Enum，不混淆 None 与 False；bool ／ None 的类型合法性不代表满足业务三态契约。 | tests/coding/test_three_state_enum.py：断言枚举成员集合与各成员分支，禁用 bool 承载未知态，None/False 不混用；不以类型检查通过代替。 |
| CODE-04 | 推荐 | 采用中性 handler/用例函数命名，沿用确有语义的角色名；不强制 Controller/Router/UseCase 后缀，不为后缀建类。 | 评审清单（非机器门禁，不新增 CI 入口）；可选自写 AST 仅辅助评审。 |
| CODE-05 | 优先 | 见 ARCH-02；Python 写法优先标准库与直接函数组合，不为未来隔离预建工厂/基类。 | 方案依赖清单与引入理由。 |
| CODE-06 | 必须 | 职责明确，固定业务值具名；复杂度 C901 阈值 max-complexity = 15，超过阈值仅提示，不作为正确性判据；行数按 DAILY-29：文件/类>500行、函数>50行触发评审，pytest承接扫描；不声称Ruff原生支持行数；参数数目仍为评审提示。 | pyproject.toml 阈值；复杂度 15/16 正反探针及独立 --exit-zero 输出；评审处置记录。 |
| CODE-07 | 推荐 | 简单对象直接构造；重复复杂转换才提取函数，不预建同构模型体系。 | 转换调用点与重复度评审。 |
| CODE-08 | 必须 | 源码与测试完整类型注解，mypy strict = true，不设 tests 总体放宽；公开接口/端口以 Google docstring 说明输入输出、None、失败、不变量，类型以注解为准；Protocol 按端口需要使用。 | mypy 实际源码和测试扫描输出；端口签名/docstring 评审；Google D 码配置探针。 |
| CODE-09 | 必须 | 涉金额用 Decimal、币种与字符串序列化；测试字符串造数，禁 Decimal(float)、float() 出站及 float 混算；quantize 显式舍入，按商城源码核验。不涉金额登记原因与重新适用条件。 | tests/coding/test_no_float_money.py 为统一金额回归入口：字面量、变量、别名、跨函数、混算、出站、字符串序列化、币种和舍入；类型约束、行为测试与审查共同补足，AST/RUF032 不是充分证明。 |
| CODE-10 | 必须 | 见 ARCH-25；业务时间 datetime.now(timezone.utc) 或等价 aware UTC，注入 now: Callable[[], datetime]；超时/耗时用注入单调钟；禁 utcnow()，禁 time.time() 计耗时。 | tests/coding/test_time_contract.py；tests/engine/test_clock.py，时钟跳变/时区无关断言。 |
| CODE-11 | 必须 | 明确 None、空集合、未找到与失败；按契约返回空集合，失败抛明确异常，不以 None 伪造成功，不把 Optional 直接充当协议。 | tests/coding/test_empty_semantics.py：空结果/未找到/失败分别断言。 |
| CODE-12 | 必须 | 值比较用 ==，身份比较用 is；见 ARCH-08。需要不可变值对象时优先 frozen/slots dataclass，嵌套集合用 tuple 等或防御复制；非一律冻结。仅需作键/集合元素的对象测哈希一致性，禁为绕检查滥用 unsafe_hash。 | tests/domain/test_invariants.py：嵌套不可变、相等、需哈希对象的 hash 一致性。 |
| CODE-13 | 必须 | 拥有的资源用 with/async with 或明确 try/finally 关闭，不关闭借用资源；清理失败不得覆盖原始业务异常，且清理失败可观察，不统一要求 __cause__ 形态。 | tests/coding/test_resource_cleanup.py：业务和清理同时失败、借用资源不关闭。 |
| CODE-14 | 禁止 | 空 except、吞异常、print 排障或伪造成功；禁 bare except，边界按类别处理；转换异常显式 raise ... from e 为项目可读性要求，不声称不用 from 就必然丢隐式 __context__。 | tests/coding/test_exception_chain.py；Ruff E722/S110/B904 正反探针和边界审查。 |
| CODE-15 | 必须 | 见 ARCH-29/44；内部用异常，接口边界按 API-08 转稳定错误，不每层重复打印堆栈；生产 debug 关闭。 | tests/api/test_errors.py；tests/security/test_error_redaction.py；无入口登记范围。 |
| CODE-16 | 必须 | 见 ARCH-30；stdlib logging + JSON formatter，使用参数化日志及 extra=，禁日志 f-string/print；昂贵参数先 isEnabledFor 检查，参数化仅延迟格式化；有值才带 trace_id/rule_id/order_sn，避开 LogRecord 保留键。 | tests/observability/test_logging.py：脱敏、换行、extra 冲突、昂贵参数不执行；G004/T201 探针。 |
| CODE-17 | 必须 | 依赖显式构造注入，不用全局可变单例隐藏依赖；main.py 仅装配见 ARCH-05；密钥与配置见 ARCH-32，出站地址见 ARCH-34，不重复规定另一安全边界。 | 装配评审；tests/security/test_secret_config.py；.env.example；适用时 SSRF 测试。 |
| CODE-18 | 必须 | 见 ARCH-14/27/31；asyncio 任务明确所有者、并发、容量、超时、取消和回收；优先 TaskGroup，取消传播不得吞 CancelledError。满载按事件等级持久化/拒绝/聚合/升级，不统一拒绝，P0 见架构规则。 | tests/engine/test_execution_limits.py；tests/engine/test_backpressure.py；tests/notifiers/test_severity_policy.py。 |
| CODE-19 | 必须 | 见 ARCH-27；阻塞调用受控隔离，不在事件循环直接阻塞；contextvars 设置后按 token reset，线程退出不冒充底层调用已停止。 | tests/engine/test_context_isolation.py；tests/engine/test_blocking_io.py：并发串线、取消及退出。 |
| CODE-20 | 必须 | 见 ARCH-31/42；避免 N+1、重复外调和无界循环，设置终止/总期限/额度；确定无外部副作用的失败可按契约重试，unknown 不按普通失败重试，依架构确认/安全重试/补偿。 | tests/engine/test_retry_budget.py；tests/recovery/test_delivery_recovery.py；调用计数与状态断言。 |
| CODE-21 | 必须 | 验收段统一归本条：执行四入口及行为匹配验证，不为覆盖率/格式造无意义代码；测试独立、可重复、自动断言；纯文档不强制新增单测。边界六类见 DAILY-25，测试布局见附表。 | 验收—用例映射；tests/reports/artifacts/<用例ID>.log；pytest JUnit；评审说明实际执行与未覆盖项。 |
| CODE-22 | 禁止 | 函数使用共享可变默认参数；使用 None 哨兵或 dataclass field(default_factory=...)，每次调用独立。 | B006 正反探针；tests/coding/test_mutable_defaults.py。 |
| CODE-23 | 必须 | noqa/type: ignore 指定规则码/错误码、理由与复审条件；第三方无类型例外限具体模块，不全局 ignore_missing_imports，不以豁免逃避安全正确性。 | PGH003/PGH004 探针；tests/coding/test_suppression_policy.py 与豁免评审清单。 |

## 配置附表（约束已定，配置与版本待 MVP 探针锁版）

- 四个 CI 入口仅：`ruff format --check`、`ruff check`、`mypy`、`pytest`。pytest 承接架构测试，不新增 lint-imports 入口。扫描范围必须含实际 src 与 tests，不以空包通过代替。
- Python `target-version = "py312"`、`line-length = 100`；mypy `strict = true`，测试函数/fixture/辅助函数完整注解，测试返回 `None`。具体补丁版本及依赖版本按 ARCH-19。
- Ruff 明确规则集基线：`E4,E7,E9,F,I,N,B006,B904,RUF006,RUF032,DTZ003,T201,G004,ASYNC251,SIM105,S110,PGH003,PGH004,D`。不整组启用 TRY/PLR/ALL；上述来自既有报告的实测命中与本轮探针不等于全配置兼容通过，MVP 须逐码核验并锁版。
- C901 不进阻断 select，配置 `[tool.ruff.lint.mccabe] max-complexity = 15`；同一 ruff check 入口的独立提示通道执行 `ruff check --select C901 --exit-zero src tests`。Ruff 没有 warning 严重级别配置；报告必须保留提示和处置，不把 exit-zero 当零问题。不得为此新增第五入口。
- `[tool.ruff.lint.pydocstyle] convention = "google"`。Google 自动排除 `D203,D204,D213,D215,D400,D401,D404,D406,D407,D408,D409,D413`。互斥对中 D203 排除/D211 保留，D213 排除/D212 保留；不再显式开启被排除项。类型以注解为准。测试缺 docstring 的例外只按明确文件/规则登记，不放宽类型检查。
- CODE-03 业务三态、CODE-04 语义后缀、CODE-09 金额完整性、CODE-10 耗时语义不得虚报静态工具覆盖。RUF032 检查 Decimal 浮点构造字面量，不覆盖所有浮点使用；mypy 可检查明确 Decimal 的混算但不能独立证明全链路金额安全。

## 测试证据目录（从首个 MVP 起固定，不创建空目录）

```text
tests/
  conftest.py
  architecture/  # 依赖契约及四类样例
  coding/        # Python 写法与金额、时间回归
  domain/        # 规则与不变量单元测试
  engine/        # 编排、取消、预算、时钟
  api/           # 已启用入口契约和组件测试
  collectors/    # 采集契约与 source_down
  notifiers/     # 配对、分组、分级
  storage/       # 持久化契约
  recovery/      # 崩溃点及重启恢复
  config/        # 配置与版本
  security/      # 授权、输入、脱敏、出站
  observability/ # 日志和时延
  faults/        # 仅告警侧端口故障注入
  e2e/           # 商城只读或获授权测试账号
  reports/
    artifacts/   # <用例ID>.log/.png
```

这是证据主题布局，非要求每个目录预建。单元/契约/组件/端到端/故障注入在方案中标明，跨主题引用实际测试节点而不复制实现；架构正文证据路径沿用。JUnit 固定为 `tests/reports/junit.xml`；验收报告落 `mvp/MVP-XX-测试报告.md`。DAILY-25 的输入、权限、依赖、状态、资源、终止六类作为边界清单引用；06 已按用户新要求迁移首版，待独立复核。

## 来源

项目自定：严格类型、行宽100、py312、复杂度15、日志选型与证据目录；不冒称外部组织统一规则。参考 [Python dataclasses](https://docs.python.org/3.12/library/dataclasses.html)、[asyncio](https://docs.python.org/3.12/library/asyncio-task.html)、[time](https://docs.python.org/3.12/library/time.html)、[Google 风格](https://google.github.io/styleguide/pyguide.html)、[Ruff 配置](https://docs.astral.sh/ruff/settings/)、[mypy 配置](https://mypy.readthedocs.io/en/stable/config_file.html)。来源归因与限制见 [调研处置稿](../docs/国内大厂代码规范调研.md)。

2026-11-04补充：CODE-21 单测覆盖率分母、98%硬门槛及禁止伪测试见 DAILY-14；CODE-06行数提示见 DAILY-29。只引用唯一权威，不重复维护阈值策略。
