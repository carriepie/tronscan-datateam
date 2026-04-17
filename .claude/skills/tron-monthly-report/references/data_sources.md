# 数据源说明

## 月报数据源（Google Sheet）

- **Spreadsheet ID**: `1IYHnhOpHT4hSB6aiK3fc3jt2LJ3C54gMlN07JMVjNi0`
- **认证凭证**: `/Users/admin/.config/gcp-keys/tronscan-google-sa.json`
- **读取方式**: 脚本自动通过 Service Account 拉取

### 月报读取的 Sheet 及用途

| Sheet 名称 | 用途 |
|-----------|------|
| `app_tron_monthly_overview_report_mi` | 总收入、交易数、价格等全网概览 |
| `app_txn_revenue_by_project_scenario_mi` | 按项目类型（CEX/DEX/PAYMENT等）分类收入 |
| `app_txn_cex_revenue_mi` | 交易所存款/取款/归集收入明细 |
| `app_tkn_usdt_resource_consume_stat_mi` | USDT 能量/带宽消耗统计 |
| `app_txn_usdt_stat_mi` | USDT 交易笔数（有/无燃烧） |
| `app_top_acc_resource_consumption_mi` | TOP 账户地址资源消耗 |
| `app_top_contract_resource_consumption_mi` | TOP 合约地址资源消耗 |
| `upload_tronscan_data_acc_agg` | 地址数（总量/活跃/新增） |
| `upload_multichain-fee辅助数据` | 多链手续费对比（Ethereum/BSC/Solana等） |
| `稳定币市值辅助数据` | TRON 上稳定币月末市值 |
| `app_txn_revenue_by_token_mi` | 按通证分类的收入 |
| `app_energy_supply_category_stat_di` | 能量供应类型统计 |
| `月季报数据` | 补充字段（流通量、持有者数、质押量等 EOM 值） |

### 月报输出

- **Google Doc ID**: `1BIevmdjQM2BQD3RJDixne7kdnhfFtz1HVsxa9rDCBfY`
- **本地备份**: `/Users/admin/Downloads/{CUR_MONTH}月报_debug.docx`

---

## 季报数据源（本地 Excel）

- **文件路径**: `/Users/admin/data-analysis-tron/月季报数据 (2).xlsx`
- **读取方式**: `openpyxl` 直接读取本地文件（数据需提前手动更新到 Excel）

### 季报输出

- **本地文件**: 由脚本中 `OUTPUT_PATH` 配置决定，默认 `/Users/admin/Downloads/YYYY-Qx 季报.docx`

---

## 项目类型（CATS）

月报和季报均按以下项目类型分类统计收入：

`CEX`, `ENERGY_RENTAL`, `PAYMENT`, `DEX`, `GAMBLE`, `CROSSCHAIN`, `LENDING`, `STAKING`, `GAME`, `AI`, `SCAM`, `ARBITRAGE_BOT`, `ORACLE`, `ILLICIT`
