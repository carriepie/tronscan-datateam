---
name: tron-weekly-report
description: 生成 TRON 链完整周报分析。用户说「生成周报」「weekly report」「W{n}周报」时触发。通过 Trino MCP 查询真实数据，直接在对话框输出文本分析，不写文件。
user-invocable: true
---

## 任务说明

生成 TRON 链上一周的完整数据周报，覆盖收入、交易、用户、USDT、能量、分类、CEX 专题、异动。

**输出方式**：直接在对话框打字输出 Markdown 文本，不创建任何文件。

---

## 第一步：确认周期

用户若未指定，默认取上一个完整自然周（周一至周日）。

- 当前 W 号 = 本年第几周
- 当周：`Wn`，日期 `YYYY-MM-DD ~ YYYY-MM-DD`
- 上周（环比）：`Wn-1`，日期 `YYYY-MM-DD ~ YYYY-MM-DD`

---

## 第二步：并行查询数据（所有查询用 mcp__tronscan-bigdata__trino_query，schema = hive.tron）

以下查询**全部并行执行**，分区字段统一用 `dt`：

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

### Q4 Top10 CEX 每日收入（按交易所汇总）
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

### Q5 收入分类（按场景，过滤低于 1 万笔的噪声）
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

### Q8 CEX 大额 USDT 转账明细（≥500万，tx 粒度）
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

---

## 第三步：计算环比

- **工作日均**：当周 Mon-Fri 均值 vs 上周 Mon-Fri 均值，计算 WoW %
- **周末跌幅**：当周 Sat+Sun 均值 vs 当周 Mon-Fri 均值，计算 %
- **异动识别**：任一指标单日 WoW 偏差 > 15% 或周内最高/最低偏差 > 20% 时标记为异动

---

## 第四步：按以下结构直接输出文本

```
# TRON W{n} 周报（YYYY-MM-DD ~ YYYY-MM-DD）

## 一、核心快照（W{n} vs W{n-1} 环比）
表格：指标 | W{n-1}工作日均 | W{n}工作日均 | WoW | W{n}周末均 | 周末跌幅
覆盖：总交易量、USDT交易笔数、活跃用户、USDT活跃用户、日均交易额、TRX销毁、TRX价格

## 二、TRX 价格与销毁日趋势
表格：日期 | TRX价格 | 日销毁TRX | 销毁USD价值 | 产出TRX | 净通缩/膨胀
点评异动（含价格与销毁的背离或共振）

## 三、交易量与用户
表格：日期 | 总交易笔数 | USDT笔数 | 活跃用户 | USDT用户 | 日均交易额 | USDT成交额 | 人均频次
标注本周峰值日和低谷日，点评驱动因素

## 四、USDT 专项指标
表格：日期 | USDT笔数 | 活跃地址 | TRX总销毁 | 质押能量TRX | 燃烧能量TRX | 人均燃烧/笔
分析质押 vs 燃烧占比变化

## 五、收入来源分类（一级分类占比 + 重点项目日趋势）
5.1 一级分类占比表格（COMMON_TRANSFER / CEX / ENERGY_RENTAL / PAYMENT / DEX / SCAM / OTHERS）
5.2 GasFree 支付日趋势（如有异动）
5.3 DEX（SUN.io 等）日趋势（如有异动）
5.4 SCAM 粉尘攻击量

## 六、能量供给侧分析
6.1 能量供给结构（取本周最后一天快照）：来源 | 冻结TRX | 能量B | 份额
6.2 本周能量供给异动（WoW 变化 > 5% 的实体单独列出，标注增减量和百分比）

## 七、交易分类能量消耗（USDT vs TRX vs 其他）
表格 + 说明 USDT 占比

## 八、CEX 专题（固定栏目）

### 8.1 Top10 CEX W{n} 收入汇总表
交易所 | W{n}总收入USD | 日均收入 | 总交易笔数 | 工作日峰值用户 | 周末跌幅

### 8.2 Top3 CEX 日收入趋势表
日期 | Binance | OKX | Bybit | 备注

### 8.3 大额 USDT 转账明细（≥$5M，含 tx hash）
日期 | 时间 | 金额USDT | From地址 | From标签 | To地址 | To标签 | Tx Hash（前16位...后8位）

### 8.4 CEX 本周异动（能量燃烧突变、collect爆量、大额归集规律等）

## 九、本周异动总结
表格：编号 | 类型 | 描述 | 影响

## 十、下周观察点
编号列表，每条说明观察指标和阈值
```

---

## 输出规范

- 全部用 Markdown 表格 + 文字，直接在对话输出，不写文件
- 数字保留合适精度（USD 用 $X.XXM / $X,XXX 格式，TRX 用 XM / XK 格式）
- 异动标记：▲ 上升、▼ 下降、⚠️ 异常、**加粗** 强调
- 时间戳用 UTC+8 显示
- Tx Hash 截断显示：前16位...后8位
- 写作风格：数据陈述紧凑，不加多余修饰，不用"断崖式""暴跌"等套话，总结段不逐条复述
