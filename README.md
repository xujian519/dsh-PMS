# dsh-PMS · 专利管理系统设计文档

面向中国专利代理 / 律所业务的期限、费用、账单系统的**约束层**文档：先定"谁有权说最终话"，再谈实现。

> **宪法 v0.3.0 已批准（2026-10-20）**；其余文档为草案 / 待审。批准程序见 [`CONSTITUTION.md`](CONSTITUTION.md) 第 29 条；口头指示无效。

| 文件 | 内容 | 状态 |
|---|---|---|
| [`CONSTITUTION.md`](CONSTITUTION.md) | 项目宪法 v0.3.0：序言 + 8 章 48 条（含 10 条「之一/之二」款）+ 3 附则。权威层级、规则、数据、组合、效率工具、变更与门禁、对客账单、完整性 | **已批准** |
| [`docs/DOMAIN-MODEL.md`](docs/DOMAIN-MODEL.md) | 领域模型 v0.1：事实层 DDL、`Determination` 三态、规则签名、目录清单、任务分解 T1–T8 | 待审（需按第 15 条换 SQLite 方言） |
| [`docs/DECISION-reminder-strategy.md`](docs/DECISION-reminder-strategy.md) | 期限提醒策略决策稿（落成第 11 条之三） | 已批准 |
| [`docs/AMENDMENT-dsh-plugin.md`](docs/AMENDMENT-dsh-plugin.md) | 修正案：交付形态改为 DSH 插件后宪法如何改 | 已采纳（并入 v0.3.0） |

## 三条根本命题

1. **事实只在登记过的载体里，判定只在可复算的规则里。**
2. **组合只由宿主发生，绝不下沉到核心。**
3. **LLM 只提效，不裁决。**

## 边界

- 本仓库只有文档，不含实现代码。
- 宪法约束的不只是代码，也包括写代码的人（见文末）。
