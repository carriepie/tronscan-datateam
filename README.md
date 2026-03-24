# tronscan-datateam

TronScan 数据团队内部工具共享仓库，目前存放 Claude Code skills。

---

## 目录结构

```
.claude/
└── skills/
    └── tron-weekly-report/   # TRON 链周报自动生成
        └── SKILL.md
```

---

## 快速开始（新成员）

### 1. 克隆仓库

```bash
git clone https://github.com/carriepie/tronscan-datateam.git
cd tronscan-datateam
```

### 2. 安装 skills 到本地

```bash
cp -r .claude/skills/tron-weekly-report ~/.claude/skills/
```

### 3. 配置 Trino MCP server

打开 `~/.claude/settings.json`，确保有以下配置（没有就加进去）：

```json
{
  "mcpServers": {
    "tronscan-bigdata": {
      "type": "sse",
      "url": "http://35.171.104.64:8000/sse"
    }
  }
}
```

### 4. 验证

打开 Claude Code，输入：

```
/tron-weekly-report
```

或直接说「生成上周周报」，正常触发即成功。

---

## Skills 使用说明

### tron-weekly-report

**触发方式：**

```
/tron-weekly-report
```

或自然语言：「生成 W13 周报」「生成上周周报」

**功能：** 自动查询 Trino 数仓，生成包含以下内容的完整周报，直接在对话框输出：

1. 核心快照（WoW 环比）
2. TRX 价格与销毁趋势
3. 交易量与用户
4. USDT 专项指标
5. 收入来源分类
6. 能量供给侧分析
7. 交易分类能量消耗
8. CEX 专题（Top10 收入 + 大额 tx 明细）
9. 本周异动总结
10. 下周观察点

**依赖：** 需要连接内网 Trino（`tronscan-bigdata` MCP server）

---

## 协作流程

### 成员更新 skill 后同步给团队（以tron-weekly-report为例）

```bash
cd ~/tronscan-datateam
cp ~/.claude/skills/tron-weekly-report/SKILL.md .claude/skills/tron-weekly-report/
git add .
git commit -m "Update tron-weekly-report skill"
git push
```

### 同事拉取最新 skill

```bash
cd ~/tronscan-datateam
git pull
cp -r .claude/skills/tron-weekly-report ~/.claude/skills/
```

### 同事新增或修改 skill 后上传

```bash
# 上传前先拉取，避免冲突
git pull

# 把本地改好的 skill 复制到仓库目录
cp -r ~/.claude/skills/你的skill名 ~/tronscan-datateam/.claude/skills/

cd ~/tronscan-datateam
git add .
git commit -m "Add/Update 你的skill名"
git push
```

