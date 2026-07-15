# 使用说明 · UZI-Skill

> 个股深度分析引擎 · A 股 / 港股 / 美股 · 22 维数据 × 评审团 × 机构级估值方法
>
> 本文档是**实操手册**（安装 / CLI / 命令 / 排错）。项目总览与设计理念见 [README.md](README.md)。

---

## 1. 环境要求

| 项 | 要求 |
|---|---|
| Python | 3.10+（推荐 3.12） |
| Git | 任意版本 |
| 操作系统 | macOS / Linux / Windows（Windows 需手动设置 UTF-8） |
| 网络 | 需访问 akshare / yfinance / baostock 数据源；国内建议配镜像 |
| 浏览器 | 可选；Playwright Chromium 用于报告截图 |

> 💡 本机为 macOS 12.7 / Intel i7，建议直接用预编译 wheel，不要源码编译（akshare / pandas 均有 wheel）。

---

## 2. 安装

### 方式 A · 一键脚本（推荐）

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/wbh604/UZI-Skill/main/setup.sh)
```

脚本会自动：检测 Python → 克隆仓库 → 尝试默认 pypi，失败则切换清华/阿里/中科大镜像 → 设置 hooks 可执行权限。脚本本质是执行 `pip install -r requirements.txt`，无全局副作用。

### 方式 B · 手动

```bash
git clone https://github.com/wbh604/UZI-Skill.git
cd UZI-Skill
pip install -r requirements.txt          # 国内挂了就加 -i https://pypi.tuna.tsinghua.edu.cn/simple
python -m playwright install chromium    # 可选，用于截图分享图
```

### 方式 C · uv venv（本机推荐）

本机已用 [uv](https://docs.astral.sh/uv/) 建好 `.venv`（Python 3.12，国内镜像）。仓库无 `pyproject.toml`，属于 venv 模式，用 `uv pip` 而非 `uv sync`：

```bash
uv venv --python 3.12                    # 建 venv（仅首次；本机已建好可跳过）
uv pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
uv run python -m playwright install chromium   # 可选，截图用
```

之后所有 `python run.py ...` 命令，二选一：
- **未激活 venv**：`uv run python run.py ...`（uv 自动用 `.venv`）
- **已激活**：先 `source .venv/bin/activate`，再直接 `python run.py ...`

### 方式 D · 作为 Claude Code plugin

```
/plugin marketplace add wbh604/UZI-Skill
/plugin install stock-deep-analyzer@uzi-skill
```

Codex / Cursor / Gemini / Hermes 等其他 agent 的装法见 README "30 秒上手"章节。

### 可选环境变量

复制 `.env.example` 为 `.env` 按需填写：

```bash
cp .env.example .env
```

| 变量 | 作用 |
|---|---|
| `MX_APIKEY` | 东财妙想 Skills Hub API key（免费，https://dl.dfcfs.com/m/itc4）。设了可做中文名纠错 + 行情快照，稳定性高于直爬 |
| `STOCK_NO_CACHE=1` | 禁用所有 API 缓存，强制重拉 |
| `UZI_NO_AUTO_OPEN=1` | 分析完不自动开浏览器 |

---

## 3. 三档深度（核心概念）

`--depth` 决定下游全部子系统行为：fetcher 集合、评委数量、机构方法数、定性维度是否让 agent 深度介入。

| 档 | 命令示例 | 耗时 | 评委 | 机构方法 | 适用场景 |
|---|---|---|---|---|---|
| `lite` | `python run.py 002217 --depth lite --no-browser` | 30–60s | 10 | 1 | 速判、批量过票 |
| `medium` | `python run.py 600519 --depth medium`（默认） | 2–4min | 51 | 17 | 日常分析默认档 |
| `deep` | `python run.py AAPL --depth deep` | 5–8min | 51 + Bull/Bear 辩论 | 18 | 首次覆盖 / IC memo |

优先级：`--depth` 参数 > `UZI_DEPTH` 环境变量 > `UZI_LITE=1`（向后兼容，等价 lite）> 默认 `medium`。

部分 slash 命令会隐式锁定档位：`/quick-scan` → lite，`/ic-memo` `/initiate` → deep，`/analyze-stock` → medium。

---

## 4. CLI 直跑

`run.py` 是唯一入口，任何 agent / 服务器都能直接跑。

### 基础用法

```bash
python run.py 贵州茅台              # A 股，中文名自动纠错
python run.py 600519.SH             # A 股，上交所代码
python run.py 002273.SZ             # A 股，深交所代码
python run.py AAPL                  # 美股
python run.py 00700.HK             # 港股
```

### 常用组合

```bash
# 不开浏览器（服务器 / CI / Codex 环境必加）
python run.py 600519 --no-browser --depth lite

# 生成公网链接（Cloudflare Tunnel，无需公网 IP / 端口转发）
python run.py 600519 --remote

# 2–4 只股票横向对决
python run.py --versus 600519.SH 000858.SZ --depth lite

# CSV 组合健康度检查
python run.py --portfolio my_holdings.csv

# 锁定单一流派视角（A-I，报告带 SCHOOL LOCK banner）
python run.py 002217 --school F     # F = A 股游资派

# 把产出拷到指定目录（SaaS 集成，配合 --no-browser）
python run.py 600519 --output-dir ./out --no-browser
```

### 完整参数表

| 参数 | 说明 |
|---|---|
| `ticker`（位置参数） | 股票代码或中文名，默认 `002273.SZ` |
| `--depth {lite,medium,deep}` | 思考深度，见第 3 节 |
| `--no-browser` | 不自动开浏览器 |
| `--remote` | 分析后启 HTTP 服务 + Cloudflare Tunnel 出公网链接 |
| `--port PORT` | HTTP 端口，默认 8976 |
| `--force-name CODE` | 强制覆盖识别出的代码 |
| `--no-resume` | 不复用 cache，强制重跑 |
| `--enable-xueqiu-login` | 启用雪球登录态（抓实盘组合维度，opt-in） |
| `--school {A..I}` | 锁定单流派视角，其他派评委 skip |
| `--versus TICKER [TICKER...]` | 2–4 只横向对比 |
| `--portfolio CSV` | 组合批量分析 |
| `--output-dir DIR` | 产出拷贝目录，生成 `index.html` + `report.meta.json` |

### 运行产出

- **HTML 报告**：自包含，离线可看，Bloomberg 风格
- **分享竖图**：1080×1920，发朋友圈
- **群战报**：1920×1080
- **一句话摘要**：复制即发
- `--remote` 额外输出一个 `https://xxx.trycloudflare.com` 公网链接

---

## 5. Slash 命令（agent 内使用）

装好 plugin 后，在 Claude Code 等 agent 里直接说命令。命令定义在 `commands/` 目录。

### 高频 4 条

```
/stock-deep-analyzer:analyze-stock 贵州茅台    # 完整分析（medium，5-8min）
/stock-deep-analyzer:quick-scan 002217         # 30 秒速判（lite）
/stock-deep-analyzer:scan-trap 002217          # 杀猪盘排查
/stock-deep-analyzer:dcf 600519                 # DCF 估值专项
```

### 按场景分类

| 场景 | 命令 |
|---|---|
| 速判 | `quick-scan` |
| 完整分析 | `analyze-stock`、`panel-only`（只跑评审团） |
| 估值建模 | `dcf`、`comps`、`lbo`、`segmental-model`、`model-update` |
| 决策文档 | `initiate`（首次覆盖）、`ic-memo`（投委会备忘录）、`dd`（尽调清单）、`thesis`（逻辑追踪） |
| 事件 / 财报 | `earnings`、`earnings-preview`、`catalysts` |
| AI 维度 | `ai-readiness` |
| 组合 | `returns`（归因）、`rebalance`（再平衡）、`screen`（量化筛选） |
| 风控 | `scan-trap` |

> 自然语言也能触发：说"分析 600519""这只票安全吗""DCF 一下"会自动路由到对应 skill。

---

## 6. Skills 架构（开发者向）

| Skill | 入口 | 触发关键词 |
|---|---|---|
| `deep-analysis` | `skills/deep-analysis/SKILL.md` | 深度分析 / 估值 / DCF / 首次覆盖 |
| `investor-panel` | `skills/investor-panel/` | 评审团 / 大佬怎么看 |
| `lhb-analyzer` | `skills/lhb-analyzer/` | 龙虎榜 / 游资 / 营业部 |
| `trap-detector` | `skills/trap-detector/` | 杀猪盘 / 安全吗 / 群里说 |

核心引擎：`skills/deep-analysis/scripts/run_real_test.py`。`run.py` 是其薄封装 + CLI 入口。深度路径（两段式 stage1 → agent 介入 → stage2）见 `AGENTS.md` 与 `skills/deep-analysis/SKILL.md`。

---

## 7. 常见问题排查

| 现象 | 原因 / 处理 |
|---|---|
| `baostock login() 挂` | 版本太老。requirements 已锁 `baostock>=0.9.1`，`pip install -U baostock` |
| A 股中文名识别错 | 设 `MX_APIKEY`（.env）走官方 NLP 纠错；或用 `--force-name` 直接指定代码 |
| 报告截图不出 | 没装 Chromium：`python -m playwright install chromium` |
| 服务器上自动开浏览器报错 | 加 `--no-browser` |
| 想要公网访问但无公网 IP | 用 `--remote`（Cloudflare Tunnel） |
| 国内 pip 卡住 | `pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple` |
| 数据没刷新 | 加 `--no-resume` 或设 `STOCK_NO_CACHE=1` |
| `agent_analysis.json` 缺失 | v2.10.4 起自动降级 warning，仍出 HTML 报告，不影响使用 |

---

## 8. 相关文档

- [README.md](README.md) — 项目总览、设计理念、66 评审团全名单、22 种方法详解
- [AGENTS.md](AGENTS.md) — agent 完整指令（深度路径两段式流程）
- [INSTALL-HERMES.md](INSTALL-HERMES.md) — Hermes agent 专用安装（绕过 Skills Guard）
- [docs/DATA-PROVIDERS.md](docs/DATA-PROVIDERS.md) — 数据源说明
- [RELEASE-NOTES.md](RELEASE-NOTES.md) — 完整更新日志
