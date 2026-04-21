---
name: tron-weekly-report
description: TRON 周报完整流程。「生成周报」「weekly report」「W{n}周报」触发阶段一+二（Trino报告+Insight）；「校验GSheet」「GSheet校验」触发阶段三（更新日期后的数据核对）。
user-invocable: true
---

## 流程说明

共三个阶段，按顺序执行：

| 阶段 | 触发方式 | 内容 |
|------|---------|------|
| 一 | 说「生成周报」 | Trino 查询 + 输出完整数据报告 |
| 二 | 阶段一完成后自动接续 | Insight 分析（统计异常 + 模式判断） |
| 三 | 你手动更新汇报截止日后说「校验GSheet」 | GSheet vs Trino 数据核对 |

---

# 阶段一：Trino 查询 + 周报输出

## 第一步：确认周期

用户若未指定，默认取上一个完整自然周（周一至周日）。

- 当前 W 号 = 本年第几周
- 当周：`Wn`，日期 `YYYY-MM-DD ~ YYYY-MM-DD`
- 上周（环比）：`Wn-1`，日期 `YYYY-MM-DD ~ YYYY-MM-DD`
- 4周基线起点：`W4_MON`（往前数4个完整周的周一）

---

## 第二步：并行查询数据（全部用 mcp__tronscan-bigdata__trino_query，schema = hive.tron）

以下查询**全部并行执行**：

### Q1 交易量 & 用户宽表
```sql
SELECT dt, txn_cnt_1d, usdt_txn_cnt_1d, trx_txn_cnt_1d,
  active_user_cnt_1d, usdt_txn_active_user_cnt_1d,
  ex_txn_cnt_1d, tronlink_txn_cnt_1d,
  ROUND(txn_amt_usd_1d/1e9, 3) AS txn_vol_b_usd,
  ROUND(usdt_txn_amt_usd_1d/1e9, 3) AS usdt_vol_b_usd,
  txn_with_burn_cnt_1d, txn_with_energy_burn_cnt_1d,
  per_addr_txn_cnt_1d
FROM hive.tron.app_com_txn_transaction_stat_di
WHERE dt BETWEEN '{W_PREV_MON}' AND '{W_CUR_SUN}'
ORDER BY dt
```

### Q2 TRX 流通 / 销毁 / 能量供应
```sql
SELECT dt, ROUND(total_turn_over/1e9,3) AS circulation_b,
  ROUND(total_burn/1e6,3) AS burn_m_trx,
  ROUND(total_burn_in_usd/1e6,3) AS burn_m_usd,
  ROUND(total_produce/1e6,3) AS produce_m_trx,
  ROUND(total_energy_limit/1e9,2) AS energy_supply_b,
  trx_price
FROM hive.tron.app_com_gov_trx_turn_over_di
WHERE dt BETWEEN '{W_PREV_MON}' AND '{W_CUR_SUN}'
ORDER BY dt
```

### Q3 USDT 专项
```sql
SELECT dt, usdt_txn_cnt, user_cnt, user_initiated_cnt,
  ROUND(trx_burned,2) AS trx_burned,
  ROUND(staked_energy_in_trx,2) AS staked_energy_trx,
  ROUND(burned_energy_in_trx,2) AS burned_energy_trx,
  ROUND(avg_energy_burn_trx,4) AS avg_energy_burn_trx,
  usdt_txn_cnt_with_burn, energy_burn_user_cnt
FROM hive.tron.app_txn_usdt_stat_di
WHERE dt BETWEEN '{W_PREV_MON}' AND '{W_CUR_SUN}'
ORDER BY dt
```

### Q4 Top10 CEX 每日收入
```sql
SELECT dt, exchange_name,
  total_txn_cnt, active_user_count,
  ROUND(trx_burnt_cnt,2) AS trx_burnt,
  ROUND(energy_consumed_cnt/1e6,2) AS energy_m,
  ROUND(energy_stake_revenue_usd,2) AS energy_stake_usd,
  ROUND(energy_burn_revenue_usd,2) AS energy_burn_usd,
  ROUND(energy_stake_revenue_usd+energy_burn_revenue_usd+bandwidth_burn_revenue_usd+other_fees_burn_revenue_usd,2) AS total_rev_usd
FROM hive.tron.app_txn_top_10_cex_stat_di
WHERE dt BETWEEN '{W_CUR_MON}' AND '{W_CUR_SUN}'
ORDER BY dt, total_rev_usd DESC
```

### Q5 收入分类
```sql
SELECT dt, txn_category, txn_sub_category, txn_cnt, active_address_cnt,
  ROUND(energy_stake_revenue_usd+energy_burn_revenue_usd+bandwidth_burn_revenue_usd+other_fees_burn_revenue_usd,2) AS total_rev_usd,
  ROUND(energy_stake_revenue_usd,2) AS energy_stake_usd,
  ROUND(energy_burn_revenue_usd,2) AS energy_burn_usd
FROM hive.tron.app_txn_revenue_by_project_scenario_di
WHERE dt BETWEEN '{W_CUR_MON}' AND '{W_CUR_SUN}'
  AND txn_cnt > 10000
ORDER BY dt, total_rev_usd DESC
```

### Q6 能量供给分布
```sql
SELECT dt, energy_supply_type, delegate_energy_entity,
  ROUND(total_frozen_trx/1e6,2) AS frozen_m_trx,
  ROUND(total_energy_gained/1e9,3) AS energy_b
FROM hive.tron.app_energy_supply_category_stat_di
WHERE dt BETWEEN '{W_CUR_MON}' AND '{W_CUR_SUN}'
ORDER BY dt, total_frozen_trx DESC
```

### Q7 交易分类能量消耗
```sql
SELECT dt, category, total_txn_cnt,
  ROUND(energy_consumed_cnt/1e9,3) AS energy_g,
  ROUND(trx_burnt_cnt,2) AS trx_burnt
FROM hive.tron.app_txn_resource_consume_stat_by_txn_category_di
WHERE dt BETWEEN '{W_CUR_MON}' AND '{W_CUR_SUN}'
ORDER BY dt, total_txn_cnt DESC
```

### Q10 CEX 能量策略监测（归集燃烧 vs 委托量）
```sql
SELECT dt, exchange_name, exchange_txn_type,
  SUM(txn_cnt) AS txn_cnt,
  ROUND(SUM(trx_burnt_cnt), 0) AS trx_burnt,
  ROUND(SUM(trx_burnt_for_energy_cnt), 0) AS trx_burnt_for_energy
FROM hive.tron.app_txn_top_cex_stat_di
WHERE dt BETWEEN '{W_PREV_MON}' AND '{W_CUR_SUN}'
  AND exchange_txn_type IN ('collect', 'delegate', 'undelegate')
GROUP BY dt, exchange_name, exchange_txn_type
ORDER BY dt, exchange_name, exchange_txn_type
```

### Q8 CEX 大额 USDT 转账明细（≥500万）
```sql
WITH cex_addr AS (
  SELECT DISTINCT address, belong_exchange_name AS exchange_name
  FROM hive.tron.ods_es_account_exchanges_df
  WHERE dt = '{W_CUR_SUN}'
    AND belong_exchange_name IS NOT NULL AND belong_exchange_name != ''
)
SELECT t.dt, t.txn_hash, t.transfer_time, t.from_address,
  COALESCE(cf.exchange_name,'unknown') AS from_exchange,
  t.to_address,
  COALESCE(ct.exchange_name,'unknown') AS to_exchange,
  ROUND(CAST(t.transfer_divide_by_decimal_amt AS DOUBLE),2) AS usdt_amount
FROM hive.tron.dwd_txn_transfer_di t
LEFT JOIN cex_addr cf ON t.from_address = cf.address
LEFT JOIN cex_addr ct ON t.to_address = ct.address
WHERE t.dt BETWEEN '{W_CUR_MON}' AND '{W_CUR_SUN}'
  AND t.token_symbol = 'USDT'
  AND CAST(t.transfer_divide_by_decimal_amt AS DOUBLE) >= 5000000
  AND (cf.exchange_name IS NOT NULL OR ct.exchange_name IS NOT NULL)
ORDER BY CAST(t.transfer_divide_by_decimal_amt AS DOUBLE) DESC
LIMIT 40
```

### Q9 4 周历史基线（阶段二 Insight 用）
```sql
SELECT dt, txn_cnt_1d, usdt_txn_cnt_1d, active_user_cnt_1d,
  ROUND(txn_amt_usd_1d/1e9, 3) AS txn_vol_b_usd
FROM hive.tron.app_com_txn_transaction_stat_di
WHERE dt BETWEEN '{W4_MON}' AND '{W_PREV_SUN}'
ORDER BY dt
```

```sql
SELECT dt, ROUND(total_burn/1e6,3) AS burn_m_trx, trx_price,
  ROUND(total_energy_limit/1e9,2) AS energy_supply_b
FROM hive.tron.app_com_gov_trx_turn_over_di
WHERE dt BETWEEN '{W4_MON}' AND '{W_PREV_SUN}'
ORDER BY dt
```

```sql
SELECT dt,
  ROUND(total_revenue_usd/1e6,3) AS revenue_m_usd,
  ROUND(total_revenue_trx/1e6,3) AS revenue_m_trx
FROM hive.tron.app_txn_tron_network_revenue_overview_di
WHERE dt BETWEEN '{W4_MON}' AND '{W_PREV_SUN}'
ORDER BY dt
```

---

## 第三步：计算环比

- **工作日均**：当周 Mon-Fri 均值 vs 上周 Mon-Fri 均值，计算 WoW %
- **周末跌幅**：当周 Sat+Sun 均值 vs 当周 Mon-Fri 均值，计算 %
- **异动识别**：任一指标单日 WoW 偏差 > 15% 或周内最高/最低偏差 > 20% 时标记为异动

---

## 第四步：输出周报

```
# TRON W{n} 周报（YYYY-MM-DD ~ YYYY-MM-DD）

## 一、核心快照（W{n} vs W{n-1} 环比）
表格：指标 | W{n-1}工作日均 | W{n}工作日均 | WoW | W{n}周末均 | 周末跌幅
覆盖：总交易量、USDT交易笔数、活跃用户、USDT活跃用户、日均交易额、TRX销毁、TRX价格

## 二、TRX 价格与销毁日趋势
表格：日期 | TRX价格 | 日销毁TRX | 销毁USD价值 | 产出TRX | 净通缩/膨胀

## 三、交易量与用户
表格：日期 | 总交易笔数 | USDT笔数 | 活跃用户 | USDT用户 | 日均交易额 | USDT成交额 | 人均频次

## 四、USDT 专项指标
表格：日期 | USDT笔数 | 活跃地址 | TRX总销毁 | 质押能量TRX | 燃烧能量TRX | 人均燃烧/笔

## 五、收入来源分类
5.1 一级分类占比（COMMON_TRANSFER / CEX / ENERGY_RENTAL / PAYMENT / DEX / SCAM / OTHERS）
    + 若总收入 WoW 有显著变化，标注各类型贡献度% = (类型增量 / 总增量 × 100)
5.2 有异动的子类型日趋势（GasFree / SUN.io / 等）
5.3 SCAM 粉尘攻击量

## 六、能量供给侧分析
6.1 供给结构（本周最后一天快照）：来源 | 冻结TRX | 能量B | 份额
6.2 本周能量供给异动（WoW > 5% 的实体）

## 七、交易分类能量消耗
表格 + USDT 占比说明

## 八、CEX 专题
8.1 Top10 CEX W{n} 收入汇总表（含工作日均收入、总笔数、周末跌幅）
8.2 Top3 CEX 日收入趋势表
8.3 大额 USDT 转账明细（≥$5M，含 tx hash 前16位...后8位）
8.4 CEX 本周异动（基于 Q4 + Q10）
    检测项：
    - 单日 energy_burn_trx 突变（通常走委托能量的交易所突然大量燃烧 = 能量池告急）
    - collect 爆量（单日 collect_txn_cnt > 工作日均 1.5x）
    - 大额冷钱包归集规律（时间窗口、中转地址模式）
    - **能量策略升级**：collect_trx_burnt WoW < -30% 且 delegate_ops WoW > +100%
      → 标注：「该交易所激活自质押能量委托系统，每日归集能量成本从 X TRX 降至 Y TRX，
         归集笔数变化 Z%（真实业务量指标）」

## 九、本周异动总结
表格：编号 | 类型 | 描述 | 驱动实体 | 贡献度% | 结构/周期

## 十、下周观察点
编号列表，每条说明：观察指标 + 阈值 + 若触发说明什么
```

---

# 阶段二：Insight 分析（阶段一完成后自动接续）

## Insight 分析步骤

### 三层钻取（总量 → 类型 → 实体）

**L1 总量**：计算本周工作日均 vs 4周基线均值的偏差%，取 |偏差| Top 3 指标。
若 TRX 价格 WoW > 5%，先做价格剥离（TRX收入 vs USD收入分开看）。

**L2 类型**：对每个 L1 异动指标，按 txn_category（CEX/COMMON_TRANSFER/DEX/PAYMENT/ENERGY_RENTAL/SCAM）或 exchange_txn_type（collect/withdraw/deposit/delegate）拆分，计算各类型贡献度%：
```
贡献度% = (类型增量) / |总增量| × 100
```
找贡献度 > 20% 的类型进入 L3。

**L3 实体**：在 L2 异动类型内，找驱动的具体交易所/项目：
- CEX类型 → 按 exchange_name 逐个算贡献度（Q4 数据）
- 能量异动 → 按 delegate_energy_entity（Q6 数据）
- DEX/PAYMENT → 按 txn_sub_category
- **CEX 能量策略检测**（Q10 数据）：
  collect_trx_burnt WoW < -30% 且 delegate_ops WoW > +100% → 能量策略升级（P9），
  此时真实业务量指标是 collect_txn_cnt，不是 trx_burnt

### 模式匹配

- **P1 价格驱动**：TRX价格↑ + USD收入↑ + TRX收入持平 → USD增长为价格贡献
- **P2 避险需求**：市场波动 + USDT笔数↑ + USDT用户↑ → 链上避险转移
- **P3 质押转向**：TRX价格↑ + self_stake↑ + rental↓ → 自质押替代短租
- **P4 CEX 集中**：单一CEX WoW > 30% + 其他平 → 该交易所专项事件
- **P5 客单价升**：收入↑ + 用户↓ + 交易额↑ → 大额用户主导
- **P6 粉尘扰动**：SCAM↑↑ + 活跃地址虚高 + USDT笔数持平
- **P7 能量迁移**：rental份额↑ + self_stake份额↓ → 自质押性价比下降
- **P8 量收背离**：交易笔数↑ + 收入↓ → 低能耗交易增多
- **P9 CEX 能量策略升级**：collect_trx_burnt↓↓ + delegate_ops↑↑ → 成本优化，非业务萎缩

### Insight 输出格式

```
## Insight — W{n}（{YYYY-MM-DD} ~ {YYYY-MM-DD}）

**本周一句话**
[最重要结论，≤30字，必须含具体数字和驱动主体]

---

**关键发现**

1. **[指标+方向]**：本周[值]，WoW [±X%]。[驱动实体] 贡献了 [Y%] 的增量——[1-2句机制解释，结构/周期判断]

2. **[指标+方向]**：[解释，重点说为什么，不复述数字]

3. **[指标联动或反常项]**：[两指标的关联关系，比单独描述更有价值]

---

**下周信号**
- [指标]：若 [具体条件+阈值]，说明 [含义]
- [指标]：若 [条件]，说明 [含义]
```

**禁止**：每条没有驱动实体；有结论没贡献度%；collect_trx_burnt 下降直接说"归集活动减少"（先查 P9）；收入涨直接说"用量增长"（先查 P1）；超过3条关键发现；用情绪化词。

---

# 阶段三：GSheet 数据校验（你更新汇报截止日后触发）

**触发词**：「校验GSheet」或「GSheet校验」

> 此阶段需要你已完成备份工作并手动更新 `原始数据Raw` 的汇报截止日，再触发。

## 步骤1：读取 GSheet 当前汇报截止日

```python
import os, gspread

gc = gspread.oauth(
    credentials_filename=os.path.expanduser('~/.config/gspread/credentials.json'),
    authorized_user_filename=os.path.expanduser('~/.config/gspread/authorized_user.json')
)
wb = gc.open_by_key('1_2Ymk7EGjHqEera4V0v-4zc1ivHA7_qQpFUh91QuAsw')
ws_raw = wb.get_worksheet_by_id(978892695)

current_date = ws_raw.cell(1, 2).value
print(f"当前汇报截止日: {current_date}")
```

确认截止日与本次周报周期的周日一致后继续。如不一致，提示用户检查。

## 步骤2：读取关键指标（L7D avg）

```python
METRICS = {
    'Total Revenue (USD)':  {'row': 4},
    'from burning USD':     {'row': 5},
    'from staking USD':     {'row': 6},
    'Total Revenue (TRX)':  {'row': 22},
    'Active addresses':     {'row': 29},
    'Total Transactions':   {'row': 108},
    'TRX Burned':           {'row': 79},
    'Staking Rate':         {'row': 70},
}

gsheet_vals = {}
for name, cfg in METRICS.items():
    l7d = ws_raw.cell(cfg['row'], 5).value  # E列 = L7D avg
    wow = ws_raw.cell(cfg['row'], 7).value  # G列 = WoW
    gsheet_vals[name] = {'l7d': l7d, 'wow': wow}
```

## 步骤3：与阶段一的 Trino 数据对比

使用阶段一已查询的 Trino 数据（在当前对话中），计算同期 L7D avg，逐指标对比：

**校验规则**：
- 偏差 < 0.1%：✅ 正常
- 偏差 0.1%～1%：⚠️ 轻微差异，记录
- 偏差 > 1%：❌ 异常，需检查对应底层 tab

## 步骤4：输出校验结果

```
## GSheet 数据校验结果 — W{n}

汇报截止日：{current_date} ✅

| 指标 | GSheet L7D avg | Trino L7D avg | 偏差 | 状态 |
|------|---------------|---------------|------|------|
| Total Revenue (USD) | $X,XXX,XXX | $X,XXX,XXX | 0.0X% | ✅ |
| from burning USD    | ... | ... | ... | ✅ |
| from staking USD    | ... | ... | ... | ✅ |
| Total Transactions  | ... | ... | ... | ✅ |
| Active addresses    | ... | ... | ... | ✅ |
| TRX Burned          | ... | ... | ... | ✅ |

结论：[全部正常 / X项异常，需检查以下底层 tab：...]
```

如有 ❌，指出需检查的具体 tab 名称（如 `app_txn_tron_network_revenue_overview_di`）。

---

## 输出规范（全阶段通用）

- 全部在对话框输出，不写文件
- 数字精度：USD 用 `$X.XXM / $X,XXX`，TRX 用 `XM / XK`
- 异动标记：▲ 上升、▼ 下降、⚠️ 异常、**加粗** 强调
- 时间戳 UTC+8，Tx Hash 截断前16位...后8位
- 写作风格：紧凑，不加修饰，不用套话，不逐条复述
