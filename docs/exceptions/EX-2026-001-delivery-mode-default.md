# EX-2026-001 · 省略送达方式即按电子送达

**状态** 待批准
**违反条款** 第 3 条（缺失即不确定）、第 11 条之一（送达方式为必填事实）、第 13 条之二（工具入参不补事实）
**发现日** 2026-10-20（宪法 v0.3.0 修正案评审时核对 `patent-deadline` 契约发现）
**发现人** 实现评审（DSH 会话）

## 事实

`@deepseek-ai/dsh-patent-deadline` 的库 API `resolveDeliveryDate` / `resolveDeliveryMode`：**省略送达方式时按电子送达处理**（「an omitted mode means electronic delivery」，见该包 README「Period and delivery arithmetic」）。

宪法要求：送达方式缺失时，送达日必须是 `UNDETERMINED` 并附缺失项 `["delivery_method"]`；默认送达方式正是第 3 条点名的三个危险默认值之一（默认无费减、默认电子送达、默认自然日）。

**待核实**：模型面工具 `patent_deadlines` 是否强制调用方显式传入送达方式（本次只核对了库 API 契约，未核工具 schema 与运行时行为）。

## 理由

修复落在同域插件实现仓库，不在本仓库；且不是改一行默认值就完——工作台的「登记官文」表单、期限重算路径、测试都要一起改，否则会出现"工具要求必填、表单仍预选电子送达"的半修状态。

先登记，避免"已经知道但装作不知道"（附则 B：不得装作已合规）。

## 影响面

- **规则**：送达日四分支（electronic / mail / direct / publication）的入口判定
- **工具**：`patent_deadlines`（模型面）、`patentDeadline` 服务（工作台消费）
- **下游**：工作台官文登记表单、期限看板、既有已算出的期限（是否需要重算）

## 偿还计划

1. `resolveDeliveryMode` 省略 → 返回 `UNDETERMINED`，缺失项 `["delivery_method"]`；
2. 工具 schema 将送达方式标为 required；如需"案卷级默认送达方式"，把它做成**一条经人工确认的事实**（第 25 条交互），而不是函数 `default`；
3. 工作台表单不得预选电子送达；
4. 补用例：`{发文日, 方式 NULL}` → `UNDETERMINED`（第 11 条之一原有用例），并纳入第 32 条门禁第 11 项"删参测试"。

负责人：待定 ｜ 预估：0.5 人日（含测试）

## 到期日

**2026-11-19**（建议 30 天）。到期未偿还自动升级为阻塞项（第 31 条）；本破例不得续期超过两次。

## 批准

**待签**——须由第 29 条批准人 `xujian519@gmail.com` 书面批准（git commit 或书面邮件），口头批准无效。
