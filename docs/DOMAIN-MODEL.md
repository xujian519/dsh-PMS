# 领域模型 v0.1（事实层 + 规则签名）

**状态** 待审 ｜ **依据** 宪法 v0.2.1（第 1、3、6 条之一、11 条之一/之二/之三、17 条） ｜ **日期** 2026-10-20

本文件只定两件事：**什么算事实**、**规则长什么样**。不含 UI、不含接口协议、不含部署。

---

## 一、三条从宪法直接推来的建模约束

1. **事实不引用派生物的代理键**。`deadline_ledger` 是派生层（可删除重建，第 17 条），任何事实表都不得用它的 `uuid` 做外键——否则重建一次，全部响应记录的外键就断了。事实之间用业务键相连：`(case_id, obligation, trigger_action_id)`。
2. **派生结论必须带规则版本**（第 12 条），空值即违宪。
3. **不确定要有类型**。规则不返回 `T`，返回 `Determination<T>`；没有 `default` 分支，编译器帮忙堵住第 3 条。

## 二、枢纽类型：Determination

整个系统只有三种输出状态，没有第四种：

```ts
// domain/shared/determination.ts
export type ExplainStep = {
  ruleId: string
  ruleVersion: string
  input: Record<string, unknown>
  output: unknown
  note?: string
}

export type Determination<T> =
  | { status: 'ok'; value: T; explain: ExplainStep[] }
  | { status: 'undetermined'; missing: string[]; explain: ExplainStep[] }
  | { status: 'conflict'; candidates: Array<{ value: T; source: string }>; note: string }
```

- `ok` 必须带解释链，否则无法回答"为什么是这个日期"（第 1 条检验）。
- `undetermined` 必须带缺失字段名，供 UI 直接展示"缺什么"。
- `conflict` 只允许出现在"官文载明日期与规则推算不一致"这类场景，交人裁决（第 6 条）。
- **没有 `fallback`、没有 `defaultValue`、没有 `Partial<T>` 兜底**。想加一个，就是违宪。

## 三、事实层 DDL

### 3.1 未确认输入

```sql
-- 第 13 条：外部输入先落此表，确认后才生成事实
CREATE TABLE raw_ingest (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  source      text NOT NULL CHECK (source IN ('manual','llm','cpc_export','email')),
  payload     jsonb NOT NULL,
  page_ref    text,                            -- 原文页图引用（LLM 抽取必填）
  captured_by text NOT NULL,
  captured_at timestamptz NOT NULL DEFAULT now(),
  confirmed   boolean NOT NULL DEFAULT false,
  CONSTRAINT llm_needs_page_ref
    CHECK (source <> 'llm' OR page_ref IS NOT NULL)
);
```

### 3.2 主体与案卷

```sql
CREATE TABLE client (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name        text NOT NULL,
  short_name  text,
  created_at  timestamptz NOT NULL DEFAULT now()
);

-- 费减的粒度是主体，不是客户（第 11 条之二）
CREATE TABLE applicant_entity (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name        text NOT NULL,
  id_type     text NOT NULL CHECK (id_type IN ('uscc','id_card','passport','other')),
  id_number   text NOT NULL,
  client_id   uuid REFERENCES client(id),
  created_at  timestamptz NOT NULL DEFAULT now(),
  UNIQUE (id_type, id_number)
);

CREATE TABLE patent_case (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  application_no  text UNIQUE,                 -- 原样存储，不改写、不补零
  case_state      text NOT NULL
                  CHECK (case_state IN ('draft','filed','granted','closed','abandoned')),
  case_type       text NOT NULL
                  CHECK (case_type IN ('invention','utility','design')),
  title           text NOT NULL,
  filing_date     date,                        -- 有优先权时另存优先权日，不混用
  priority_date   date,
  client_id       uuid NOT NULL REFERENCES client(id),
  created_at      timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT appno_required_after_draft
    CHECK (case_state = 'draft' OR application_no IS NOT NULL),
  CONSTRAINT appno_format
    CHECK (application_no IS NULL OR application_no ~ '^[0-9]{13}$')
);

CREATE TABLE case_relation (
  from_case_id uuid NOT NULL REFERENCES patent_case(id),
  to_case_id   uuid NOT NULL REFERENCES patent_case(id),
  relation     text NOT NULL CHECK (relation IN
                 ('family','divisional_of','parent_of','priority_from','pct_national')),
  PRIMARY KEY (from_case_id, to_case_id, relation),
  CHECK (from_case_id <> to_case_id)
);
```

`patent_case.application_no` 只在 `draft` 状态可空；一旦 `filed`，无号即无法与官文对齐，任何期限都无从附丽（第 6 条之一第 3 点）。

### 3.3 官文与送达

```sql
CREATE TYPE delivery_method AS ENUM ('electronic','mail','direct','publication');

CREATE TABLE office_action (
  id                 uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id            uuid NOT NULL REFERENCES patent_case(id),
  doc_type           text NOT NULL,
  dispatch_date      date NOT NULL,            -- 发文日
  dispatch_serial    text NOT NULL,            -- 发文序号，去重键
  delivery_method    delivery_method NOT NULL, -- 必填，缺失即不可判定
  actual_received_on date,                     -- 有证据的实际收到日
  handed_over_on     date,                     -- 直接送交的交付日
  published_on       date,                     -- 公告日
  specified_period   interval,                 -- 官文原文指定期限，没有就留空
  raw_ingest_id      uuid REFERENCES raw_ingest(id),
  confirmed_at       timestamptz,
  confirmed_by       text,
  created_at         timestamptz NOT NULL DEFAULT now(),
  UNIQUE (case_id, dispatch_serial),
  CONSTRAINT delivery_fields_match_method CHECK (
    (delivery_method = 'direct'      AND handed_over_on IS NOT NULL) OR
    (delivery_method = 'publication' AND published_on   IS NOT NULL) OR
    (delivery_method IN ('electronic','mail'))
  )
);
```

那个 `CHECK` 是第 15 条的落点：**送达方式与必填日期在数据库层互相咬合**，插件和 LLM 网关两条写入旁路都无法绕过。

### 3.4 费减资格

```sql
CREATE TABLE fee_reduction_eligibility (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_id      uuid NOT NULL REFERENCES applicant_entity(id),
  year           int  NOT NULL,
  status         text NOT NULL CHECK (status IN ('approved','rejected','pending')),
  reduction_rate numeric(3,2) NOT NULL,        -- 0.85 或 0.70（减缴比例）
  approved_on    date,
  valid_until    date NOT NULL,                -- 备案当年 12-31
  created_at     timestamptz NOT NULL DEFAULT now(),
  UNIQUE (entity_id, year),
  CONSTRAINT reduction_rate_valid CHECK (reduction_rate IN (0.85, 0.70)),
  CONSTRAINT valid_until_is_year_end CHECK (valid_until = make_date(year, 12, 31))
);
```

**注意方向**：这里存的是**减缴比例** 0.85 / 0.70，计算时 `应缴 = 官费 × (1 − reduction_rate)`，即 15% / 30%。这两个数字在业内经常被写反，故在类型层固定名字 `reduction_rate`，另起 `payable_ratio()` 函数只在规则内调用。

### 3.5 事实事件与留痕

```sql
CREATE TABLE audit_event (
  id          bigserial PRIMARY KEY,
  occurred_at timestamptz NOT NULL DEFAULT now(),
  actor       text NOT NULL,
  action      text NOT NULL,
  object_type text NOT NULL,
  object_key  text NOT NULL,
  before      jsonb,
  after       jsonb,
  basis       text
);
-- 角色创建后（0003_roles.sql）追加：REVOKE UPDATE, DELETE ON audit_event FROM app_core;

-- 提醒的"响应"是事实；注意不引用 deadline_ledger.id
CREATE TABLE deadline_response (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id           uuid NOT NULL REFERENCES patent_case(id),
  obligation        text NOT NULL,
  trigger_action_id uuid REFERENCES office_action(id),
  action            text NOT NULL CHECK (action IN
                      ('acknowledged','filed','paid','deferred','waived')),
  actor             text NOT NULL,
  occurred_at       timestamptz NOT NULL,
  note              text
);
```

## 四、派生层 DDL

```sql
CREATE TABLE deadline_ledger (
  id                uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  case_id           uuid NOT NULL REFERENCES patent_case(id),
  obligation        text NOT NULL,
  trigger_action_id uuid REFERENCES office_action(id),
  statutory_due     date NOT NULL,             -- 法定期限，规则算出，不可改
  internal_due      date NOT NULL,             -- 内部期限 = 法定 − 提前量
  buffer_days       int  NOT NULL,
  rule_id           text NOT NULL,
  rule_version      text NOT NULL,             -- 第 12 条：空值即违宪
  input_hash        text NOT NULL,
  state             text NOT NULL CHECK (state IN
                      ('active','done','conflict','undetermined','lapsed')),
  computed_at       timestamptz NOT NULL DEFAULT now()
);

-- 第 6 条：结论唯一。NULL 不参与 UNIQUE 比较，若直接写 UNIQUE(...trigger_action_id)，
-- 无官文触发的义务（如年费）可被重复插入多行。故用 COALESCE 建唯一索引。
CREATE UNIQUE INDEX deadline_ledger_unique
  ON deadline_ledger (
    case_id, obligation,
    COALESCE(trigger_action_id, '00000000-0000-0000-0000-000000000000'::uuid)
  );
```

`reminder_queue`、`case_state_current`、`fee_ledger` 同为派生表，结构从略——它们全部满足：可由一条命令删除并重建，校验和一致（第 17 条）。

## 五、规则签名

规则包目录 `rules/cn/`，全部纯函数，世界通过入参进入。

```ts
// ① 送达日（第 11 条之一）
resolveDeliveryDate(input: {
  dispatchDate: IsoDate
  method: 'electronic' | 'mail' | 'direct' | 'publication'
  actualReceivedOn?: IsoDate
  handedOverOn?: IsoDate
  publishedOn?: IsoDate
}): Determination<IsoDate>

// ② 期限起算（第 11 条之一四步链，第 4 条纯函数）
computeDeadline(input: {
  startDate: IsoDate                          // 送达日
  period: { years?: number; months?: number; days?: number }
  holidays: HolidayCalendar                   // 参数注入，不读外部
}): Determination<IsoDate>
```

四步链在代码里的对应关系，一一对齐：

```ts
const delivered = resolveDeliveryDate(...)        // ① 按方式定送达日
if (delivered.status !== 'ok') return delivered
const start = addDays(delivered.value, 1)         // ② 当日不计入，自次日起算
const end   = addPeriodByMonth(start, period)     // ③ 按月取相应日，无相应日取月末
return shiftToWorkday(end, holidays)              // ④ 遇法定休假日顺延
```

```ts
// ③ 年费滞纳金（第 11 条示例）
annuitySurcharge(input: {
  fullAnnuity: Money
  dueDate: IsoDate
  paidDate?: IsoDate
  reduction: Determination<{ reductionRate: 0 | 0.85 | 0.70 }>
}): Determination<{ amount: Money; band: 0 | 1 | 2 | 3 | 4 | 5 | 6; beyond: boolean }>

// ④ 费减（第 11 条之二）
resolveFeeReduction(input: {
  entityIds: string[]
  paymentYear: number
  records: EligibilityRecord[]                // 该年度备案记录，缺失即 undetermined
}): Determination<{
  reductionRate: 0 | 0.85 | 0.70              // 0 = 明确无资格（查到 rejected 记录）
  payableRatio: 1 | 0.15 | 0.30
  basis: string
}>
```

`reductionRate: 0` 与 `undetermined` 是两件事，不能合并：前者是"查过了，没有资格"，后者是"根本没查到这个年度的备案记录"。合并二者会让"该减未减"和"不该减却减了"两类错误无法区分——这正是第 3 条要防的。

`HolidayCalendar` 是版本化事实（国务院逐年发布），按年加载后传参，规则内不得调用 `new Date()`。

## 六、目录与文件清单

```
src/
  domain/                        # 纯 TS：无框架、无 IO、无全局状态（第 19 条）
    shared/                      determination.ts  money.ts  iso-date.ts
    case/                        case.ts  case-relation.ts
    deadline/                    obligation.ts  buffer-policy.ts
    fee/                         fee-kind.ts
    billing/                     invoice.ts
  rules/                         # 唯一允许出现日期/金额运算的地方（第 7 条）
    cn/
      delivery/                  delivery-date.rule.ts
      deadline/                  deadline.rule.ts  workday-shift.rule.ts
      fee/                       annuity-surcharge.rule.ts  reduction.rule.ts
      holidays/                  2025.ts  2026.ts  2027.ts
      index.ts                   # 规则注册表：id / version / citation / effectiveFrom
  application/                   # 用例：事务、审计、规则调用（第 20 条）
    ingest-office-action.ts      compute-deadlines.ts  issue-invoice.ts
  infra/
    db/migrations/               0001_facts.sql  0002_derived.sql  0003_roles.sql
    db/repositories/
  plugins/                       # Cordis 外围，可卸载（第 21 条）
    notice-ingest/               # 官文录入
    reminder/                    # 提醒与逾期清单
    billing-export/
  interface/
```

## 七、任务分解

| # | 任务 | 依赖 | 验收（对应门禁） |
|---|---|---|---|
| T1 | `Determination` + 日期算术 + 节假日日历参数化 | — | 单测覆盖月末、闰年、跨年 |
| T2 | 送达日规则 | T1 | 三个用例：方式缺失→`undetermined`；electronic→发文日；mail→+15 日 |
| T3 | 期限起算规则 | T1、T2 | 四步链逐步断言；节假日顺延用例 |
| T4 | 事实表迁移 0001 + 角色 0003 | — | 空库迁移通过；`llm_gateway` 写核心表失败 |
| T5 | 台账派生 + 重建命令 | T3、T4 | drop→rebuild→checksum 一致 |
| T6 | 费减与滞纳金规则 | T1、T4 | 跨年度未续备 → 不减缴或 `undetermined` |
| T7 | 提醒派生 + 逾期清单 | T5 | `CONFLICT` 不进队列；改参数重建不丢响应 |
| T8 | 账单（第七章） | T6 | `issued` 不可 UPDATE；行可回溯 |

T1–T3 可并行于 T4，两条线在 T5 汇合。

## 八、待你确认的三处

1. **`application_no` 只收 13 位**。1993 年前的老申请号（8 位）不在 v1 范围，若要收录需另定规则。
2. **优先权日与申请日分列两字段**，因为期限计算里 `申请日` 在细则第十二条下有两种含义（专利法第二十八/四十二条除外时指优先权日）。这个歧义我不在字段名上消除，而在规则里显式选择，避免日后误用。
3. **`deadline_response` 不设外键指向台账**。这是刻意为之（见第一节），但代价是"某条响应属于哪次计算"无法用外键强制，改用 `(case_id, obligation, trigger_action_id)` 三元组匹配。建议后续把这条上升为宪法第 17 条的补充款。
