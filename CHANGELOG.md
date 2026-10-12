# Changelog

## v2.1.0 — 2026-10-12

十四层架构 · 40+ 个端点 · 14 个数据源（v2.0.3 为十三层 · 30+ · 11）。
数据源按机构重新计数：东财多个接口算一家，原有 10 家 + 新增 BIS / 欧洲央行 / 美联储 / 纽约联储 4 家。
美国财政部 FiscalData 与原有收益率曲线同属财政部，不另计。

### 修复：Yahoo K 线（`stock_kline_yahoo`）

| 问题 | 修前 | 修后 |
|---|---|---|
| `range_="max"` 拿到的不是日线 | Yahoo 对 `range=max` 忽略日线参数：AAPL 返回 169 根**季线**、0700.HK 270 根**月线**，函数照样当日线返回 | 改用起止时间请求：AAPL 1980-12-12 起 **11,549** 根日线、0700.HK **5,511** 根 |
| 日期随使用者电脑时区漂移 | 用本机时区换算：港股日 K 时间戳在 01:30 UTC，在纽约看会显示成前一天 | 按 Yahoo 返回的交易所时区算日期；新加坡 / 纽约 / 奥克兰三个时区结果一致 |
| 周 / 月 / 季线不可靠 | Yahoo 原生周 / 月 / 季线：末尾多一根只含最新一天的 bar；节假日开头的那周拿上周收盘当开盘（丰田 7203.T 2022-01-03 那周记 2105.5，01-04 实际开 2158.0，东京 / 香港 / 伦敦都有）；按起止日期截取时周线多算或少算一天；季线分组跟着起始日的月份漂移 | 改由日线按交易所当地日历合并：日期标签取周一 / 每月 1 日 / 每季第一天（与 Yahoo 原生一致），开 = 首日开、高低 = 区间极值、收 = 末日收、量 = 合计。AAPL 5 年周 / 月 / 季线与原生逐根一致，只在上述几处、以及 `range_` 的第一根（补齐成完整周期）不同 |
| 缺失 bar 记成 0 | 停牌或缺数的 bar 价格写成 0 | 直接跳过，不用 0 冒充价格 |
| 小数被截成 2 位 | 港股仙股（1373.HK 0.645）、汇率（JPY=X 158.246）精度丢失 | 保留 7 位有效数字（同时去掉 Yahoo 单精度存储的噪声，如 7786.3599… → 7786.36） |
| 月线日期带时分 | `"m" in interval` 把 `1mo` 当成分钟线，日期格式成 `YYYY-MM-DD HH:MM` | 只有分钟 / 小时线带时分 |

新增：`start` / `end`（YYYY-MM-DD，含当天，按交易所当地日期）按起止日期取，只给 `end` 时从最早一根取到 `end`；
周 / 月 / 季线每根都是完整周期（`start` 所在周期从第一天算、`end` 所在周期算到最后一天），`range_` 同样补齐第一个周期
（`range_="2y"` 月线 = 25 根）；分钟 / 小时线在 Yahoo 窗口内也能按起止日期取（分钟线 60 天，1h / 4h 730 天；
1m 30 天且一次最多取约 7 天，只给 `end` 通常会超过 7 天被拒，请同时给 `start`；2m 实测只有最近约 30 个交易日），`start` 或 `end` 整天落在窗口外直接报错。
支持 Yahoo 的 `4h` 周期（之前传 `4h` 会被当成日线，同一天的几根并成一根）。返回 `adj_close`（含分红的复权收盘）。代码不存在、日期格式错或日期不存在（2019-02-30）、
`start` 晚于 `end`、`start` 晚于明天、分钟线超出窗口都抛 `ValueError`（原为 `HTTPError`）；区间内没有交易（节假日、上市前）返回空列表。亚太市场日线的时间戳落在 UTC 前一天，起止日期一律按交易所当地日期筛，东京 / 悉尼 / 奥克兰不会多出一天。
Windows 缺 `tzdata` 时退回固定偏移，最新一根按真实成交时间换算，不会多出一根「明天」；README 快速开始补了 `pip install tzdata`。

### 新增：§2.3 其他市场 K 线（同一函数，换代码后缀）

「Yahoo Finance 代码格式」表补齐日本 `.T`、台湾 `.TW/.TWO`、韩国 `.KS/.KQ`、印度 `.NS`、
伦敦 / 法兰克福 / 巴黎 / 阿姆斯特丹 / 瑞士、新加坡 / 澳洲 / 加拿大 / 巴西，以及 14 个主要指数、外汇 `=X`、期货 `=F` 的写法。不新增代码。

实测全历史日线：丰田 7203.T 1999 起 6,851 根、台积电 2330.TW 2000 起、三星 005930.KS 2000 起、
信实 RELIANCE.NS 1996 起、SAP.DE 1998 起、必和必拓 BHP.AX 1988 起、日经 ^N225 1965 起 15,183 根、标普 500 ^GSPC 1927 起 24,811 根。
注意：伦敦报价单位是便士（SHEL.L 3793.5 GBp）；孟买 `.BO` 只返回 1 根，用 `.NS`；台股成交量比证交所官方少约一成。

### 新增：Layer 13 全球利率 / 汇率 / 央行（S 级，8 个入口）

| 入口 | 来源 | 内容 |
|---|---|---|
| `policy_rates()` | BIS 国际清算银行 | 政策利率，一次取全（48 个经济体，38 个仍在更新） |
| `ecb_fx()` | 欧洲央行 | 欧元参考汇率，默认 20 种常用货币 |
| `ecb_key_rates()` | 欧洲央行 | 存款便利 / 主要再融资 / 边际贷款利率（每行是一次调息，给 `start` 时第一行是当天仍生效的利率） |
| `ecb_yield_curve()` | 欧洲央行 | 欧元区国债收益率曲线（AAA 口径或全部国家） |
| `ecb_series()` | 欧洲央行 | 通用入口，任意数据集 + 序列代码 |
| `nyfed_rates()` | 纽约联储 | SOFR / EFFR / OBFR / TGCR / BGCR / SOFR 平均与指数 |
| `fed_data()` | 美联储 | H.10 美元对 23 种货币、H.6 M1/M2 |
| `fiscal_data()` | 美国财政部 FiscalData | 国债拍卖结果与日程、联邦债务、国债平均利率、财政部记账汇率 |

处理过的坑：
- BIS 部分国家周末记空值（菲律宾），只取 1 个观测会整国丢失 → 多取 10 个再截
- BIS 各国更新有滞后（澳洲停在 09-24 的 4.35，澳洲央行 09-30 已调到 4.60），返回值一律带日期
- ECB 不指定币种时会连带返回已停发货币（RUB、HRK 等），故给默认清单
- 美联储 H.10 报价方向不统一（澳元 / 欧元 / 新西兰元 / 英镑是「1 外币 = 多少美元」），每个序列标出 `quote`
- 美联储下载接口按日期区间取数不稳定，只提供「最近 N 个观测」（1–5000）
- FiscalData 的「未来拍卖」数据集含历史行，自动只保留今天及以后；数值保持官方字符串，`"null"` 转 `None`
- 国库券（Bill）的中标利率在 `high_investment_rate` / `high_discnt_rate`，`high_yield` 恒为空，默认字段已补上
- ECB 关键利率只在调息日有记录：按日期合并三个利率并向前填充（2019-09-18 只调了存款便利）；区间内没调息时返回当时生效的那次，而不是「没有数据」
- 纽约联储「最近 N 条」上限 999，超过直接提示改用起止日期
- 日期格式、日期不存在（2019-02-30、2024-13）、`start` 晚于 `end`、非英文字母的经济体 / 币种代码，本地先报 `ValueError`，不再等官方回 400 或 404（季度与月份混写时先把季度换成月份再比先后）；`start` / `end` 粒度可以不同（`start="2024-12-20", end="2024"`）、季度数据可写 `2024-Q1`、也可直接传 `datetime.date`

### 合规分级调整

复读条款原文（2026-10-11）后新增 **D 级 — 条款禁止未经许可的自动抓取**：

- **Yahoo Finance**（原 C 级）：条款禁止 *"using any automated means… for any purpose without our express, prior permission"*
- **Nasdaq**（原 C 级·条款未核实）：条款（Last Updated: May 11, 2026）禁止 *"any process, whether automated or manual, to capture data or content from the Service… for any reason"*

两者代码保留：Yahoo 是港股 K 线和其他市场 K 线的唯一来源，Nasdaq 是财报日历唯一免 key 的来源。
标注为「仅供个人学习研究，风险自负，不得商用或再分发」，并说明它与已排除的 HKEX CCASS 条款性质相同、保留只因没有替代。

新增 4 家 S 级来源的条款原文（BIS / ECB / 美联储 / 纽约联储），以及纽约联储要求对外展示参考利率时附带的声明原文。

### 其他

- `official_get()` 遇到 HTTP 400 时带上服务端报错原文（去掉 HTML 标签，如 FiscalData 过滤字段写错）

### 测试

- Python 3.9.6 实测：Layer 13 八个入口正常取数 + 出错路径（代码不存在 → `DataNotAvailable`、参数格式错 → `ValueError`、过滤字段错 → `RuntimeError`）
- Yahoo K 线：三个时区日期一致、全历史日线根数、周 / 月 / 季线与日线合并结果逐根比对（AAPL / 7203.T / BHP.AX / AIR.NZ / 0700.HK）、起止日期（含东京 / 悉尼 / 奥克兰、跨夏令时、港股假日）、仙股与汇率精度、无 `zoneinfo` 时的退回路径与有 `zoneinfo` 结果逐根一致
- 独立复审（Opus 5.5）一轮：2 条应修 + 9 条小问题，全部修复或核实（纽约联储声明里的方括号是原文所有）
- 修复复核一轮：3 条应修（周线带 `end` 多一天、季线分组随 `start` 漂移、Layer 13 混合粒度 / 季度写法 / 日期对象被误拒）+ 5 条小问题（无数据时报错、分钟线放宽越出窗口、新西兰误判「未来」、ECB 关键利率 `start` 正好是调息日时多给上一次、不存在的日期没在本地拦），全部修复并实测；修复后再复核一轮：0 条应修，4 条小问题（季度与月份混写时先后没比对、1m 实际可取天数、只给 `end` 超出分钟线窗口时静默返回空、对比结论漏写第一根）与范围外的 `4h` 问题一并修复；这批修复的复核又抓到 1 条（分钟线终点余量缩成 1 天后期货晚盘被截掉），改为 30 小时后实测补齐
- 回归：新浪 / 腾讯行情、新浪与 Yahoo K 线、东财三表、Yahoo 关键指标、SEC Filing、FINRA、CBOE、EDGAR frames、Treasury、CFTC、Nasdaq 均正常

## v2.0.3 — 2026-07-26

### 修复：EDGAR Frames 对资产负债表科目一直返回 404

`market_frame()` 一律拼 `CY{year}Q{q}` / `CY{year}`，但 SEC Frames 对**时点(instant)概念**
要求周期带 `I` 后缀，且不提供纯年度周期。`XBRL_TAGS` 里有 4 个属于此类，此前**全部取不到数**：

| 标签 | 修前 | 修后 |
|---|---|---|
| 总资产 `Assets` | `CY2025Q1` → 404 | `CY2025Q1I` → 5643 家 |
| 股东权益 `StockholdersEquity` | 404 | 5465 家 |
| 现金及等价物 `CashAndCashEquivalentsAtCarryingValue` | 404 | 4506 家 |
| 长期负债 `LongTermDebtNoncurrent` | 404 | 1532 家 |

年度请求同样受影响（`Assets/CY2024` → 404），现落到 `CY2024Q4I`。

新增 `_INSTANT_TAGS` 集合做区分；期间(duration)概念行为不变。

`market_frame()` 还支持传**任意原始 XBRL 标签**（`Liabilities` / `InventoryNet` /
`AssetsCurrent` …），这些不在内置别名表里、同样是时点概念。因此改为**自动回退**：
先按 `_INSTANT_TAGS` 猜一种周期形式，收到 404 再换另一种重试，无需穷举所有时点概念。
已知类型时可显式传 `instant=True/False` 省掉一次探测请求。
验证：13 个标签 × 季度/年度共 26 项请求全部返回有效数据（修前 8 项为 404）。

### 加固：OCC 合约代码允许调整后根符号

`_OSI` 的 root 由 `[A-Z]+` 放宽为 `[A-Z][A-Z0-9]*`。拆股/分拆等公司行为会产生带数字的
调整后合约根（如 `NVDA1`、`BRKB1`），原正则会直接拒绝、`options_chain_cboe()` 静默漏掉这些合约。
后续均为定宽组，正则回溯可正确对齐，标准合约解析结果不变。

验证：NVDA/AAPL/TSLA/T 共 14,424 份真实合约解析成功率 100%；合成的调整后代码亦能正确切分。
（说明：CBOE 按标的返回的链中当前只含标准根，未能实地复现漏解析，此项为防御性加固。）

## v2.0.2 — 2026-07-26

### 修复：腾讯行情字段下标错误 — 两个函数此前 100% 崩溃

感谢 [@HoRiZonn0](https://github.com/HoRiZonn0)（issue #2）提供完整字段对照。逐条实测复现后确认其报告**全部正确**，并发现比报告更严重的后果：

- **`us_stock_quote_tencent` 每次调用必抛 `ValueError`** —— `high_52w` 读 `fields[35]`，而下标 35 实为币种字符串 `"USD"`，`float("USD")` 直接崩。
- **`hk_stock_quote_tencent` 同样必崩** —— `volume` 读 `fields[6]`，港股该字段带小数位（`"22959603.0"`），`int()` 解析失败。此项非 issue 所报，为本次复测新发现。

即：**这两个函数在 v2.0.1 及更早版本中从未真正可用过。**

修正的下标（美股 71 字段 / 港股 78 字段，布局不同，此前误共用一套）：

| 字段 | 原值 | 修正为 | 实测依据 |
|---|---|---|---|
| 英文名 | 美 27 / 港 2 | 美港统一 **46** | `Apple Inc.` / `TENCENT` |
| PE | 美 53 / 港 39 | 美港统一 **39** | 40.32 / 15.87（原 53 实为总市值） |
| PB | 统一 56 | 美 **51** / 港 **58** | 45.93 / 3.14（原 56 美股为 `"GP"`、港股为 0） |
| 52 周高 / 低 | 35 / 36 | **48 / 49** | 334.99·200.72 / 677.7·411.0 |
| 市值 | 44 标注为总市值 | **44=流通 / 45=总** | AAPL 48881 vs 48911 亿，与总股本 `fields[62]` 反算一致 |
| 币种 | 未提供 | 美 **35** / 港 **75** | `USD` / `HKD` |

其他改动：

- 新增 `currency` 字段；美股新增 `eps`（下标 47，`price ÷ eps = pe` 自洽验证）；港股 `fields[2]` 正名为 `code`
- `volume` 统一改用 `int(float(x))`，消除小数位崩溃这一类隐患
- 长度守卫由笼统的 `< 50` 收紧为按实际读取深度判定（美股 52 / 港股 76）
- §1.2 新增**腾讯行情字段对照表**，美港分列、标注「亿本币」单位，并附自行复现命令

验证：`usAAPL` / `usNVDA` / `hk00700` / `hk09988` 四例全部取数正常，数值与腾讯页面一致。

## v2.0.1 — 2026-07-24

### 修复：错误处理把「没数据」和「出故障」混为一谈

v2.0.0 的 Layer 9/10 用 `except Exception: continue` 做日期回退，会**吞掉一切异常**。
后果是 `SEC_CONTACT` 没配置时，用户看到的是"未找到近 7 个工作日的 EDGAR 每日索引"，
而不是"请先配置 SEC_CONTACT"——v2.0.0 特意加的 fail-fast 提示因此永远不会显示。

本版引入明确的异常语义：

- **新增 `DataNotAvailable(RuntimeError)`**：该日/该资源确实没有数据（非交易日、文件尚未发布、标的无期权），调用方可安全回退
- **`RuntimeError`**：配置错误、限流、网络故障 —— 必须冒泡，不得伪装成"没数据"
- **`ValueError`**：参数错误（如把港股代码传给仅支持美股的层）

三处日期回退循环改为只捕获 `DataNotAvailable`；`short_volume_all` / `daily_filings` /
`options_chain_cboe` 的"找不到数据"改抛 `DataNotAvailable`，与调用方的捕获类型对齐。

### 修复：403 的三种含义现在能精确区分

**根因**：SEC Archives 与 FINRA CDN 都托管在 S3 上，而 S3 在调用方没有 `ListBucket`
权限时，对**不存在的对象返回 `403 AccessDenied`（XML）而非 `404 NoSuchKey`**。
两个源实测行为一致。这意味着"当日文件尚未发布"和"被拒绝"共用同一个状态码。

现按响应内容**正向识别**（而非用排除法，否则限流/封禁会被当成"没数据"）：

| 情况 | HTTP | Content-Type | 判定 |
|---|---|---|---|
| 对象不存在 | 403 | `application/xml` + `<Code>AccessDenied</Code>` | `DataNotAvailable`，可回退 |
| SEC UA 未声明 | 403 | `text/html` + `Undeclared Automated Tool` | `RuntimeError`，提示 UA 格式 |
| 限流 / 封禁 / 其他 | 403 | 其他 | `RuntimeError`，冒泡 |

`official_get()` 的 `missing_codes` 参数因此移除——判据改为内容驱动，调用方无需传参。

### 修复：0DTE 日期在冬令时会算错一天

`_et_today()` 原先硬编码 UTC-4（EDT）。冬令时美东是 UTC-5，导致 UTC 04:00–05:00
这一小时被算成次日，`filter_expiry(chain, "0DTE")` 与 `dte_max` 会选错到期日。

改用 `zoneinfo.ZoneInfo("America/New_York")`；无 tzdata 的环境（部分 Windows）
回退到自算 DST，切换时刻按纽约当地 2:00 换算为 UTC（开始 07:00 UTC / 结束 06:00 UTC），
而非笼统用 00:00 UTC。

### 说明

十三层架构 / 30+ 端点 / 11 数据源不变；本版为错误处理与时区正确性补丁。
问题由 Codex CLI 独立审计发现，三轮审计—修复—复审后确认无回归。


## v2.0.0 — 2026-07-24

### 新增：官方源优先架构（八层 → 十三层，18 → 30+ 端点，5 → 11 数据源）

新增层的主力数据全部取自**美国政府 / 自律组织 / 交易所**的公开端点：

- **Layer 6.1 · CBOE 官方期权链**（新增，Yahoo 期权降为 6.2 后备）
  - 全链含 `iv / delta / gamma / vega / theta / rho`——**Yahoo 期权链没有希腊字母**
  - `filter_expiry(chain, "0DTE")` 取当日到期合约；`unusual_activity()` 用 **vol/OI > 1**（当日成交超存量持仓＝新建仓）识别 flow
  - `chain_summary()` 输出 put/call 量比与持仓比、**成交量加权 IV**、**净 delta 敞口**
  - 实测：NVDA 全链 3,908 / 0DTE 168；TSLA 全链 6,200 / 0DTE 326
- **Layer 9 · FINRA Reg SHO 做空层**：全市场每日空头成交量，单文件覆盖 **12,112 只**；含个股时序与占比排行
- **Layer 10 · EDGAR 申报事件流**：每日索引（实测单日 **Form 4 内部人 547 / 8-K 370 / 13F-HR 261 / 144 118**）+ 全文检索（2001 至今所有申报正文）
- **Layer 11 · EDGAR frames 全市场横截面**：任意 XBRL 标签一次拿全市场＝**免费 screener**（净利润 CY2025Q1 覆盖 5,309 家，研发费用覆盖 1,842 家）
- **Layer 12 · 宏观/日历**：美债收益率曲线（1M~30Y）、CFTC COT 持仓报告、Nasdaq 财报日历

### 新增：合规分级（每个源标注级别 + 条款原文）

逐家实读各源条款后分级，**"官方"不等于"可自由使用"**：

- **S 级**（可商用可再分发）：SEC EDGAR / Treasury / CFTC
- **B 级**（商用需自行确认）：FINRA
- **C 级**（需事先授权或条款未核实）：CBOE / Nasdaq / Yahoo / 东财 / 新浪 / 腾讯
- **⛔ 已排除**：HKEX CCASS——其条款明文禁止 robot/bot/spider/scraper 且适用于"whether or not for gain"，**故本工具不提供该抓取代码**

README 与 SKILL.md 均给出条款原文摘录。**本工具只分发代码，不分发任何市场数据。**

### 新增：统一 HTTP 出口（限流 + UA 声明 + 友好错误）

- `_RateLimiter` **线程安全**节流器（用锁，避免并发下被击穿）
- 按源限速：**SEC 官方硬上限 10 req/s，此处设 8 req/s** 留余量；FINRA/CBOE 4/s、Nasdaq 2/s
- `SEC_CONTACT` 未改为真实邮箱时**直接报错**并说明原因（SEC 要求声明 User-Agent，否则返回 Undeclared Automated Tool）
- `assert_us_ticker()` 拦截把港股代码传给仅支持美股的层
- HTTP 403/404/429 给出针对性提示

### 其他

- 顶部新增**端点路由速查表**（按需定位，不必通读全文）
- 架构图、数据源优先级表、数据源汇总表同步更新，汇总表增加"合规级"列
- 原 Layer 1–8（含技术指标层）编号与内容不变，**向后兼容**

### 注意

- ⚠️ 使用前必须把 `SEC_CONTACT` 改成你自己的真实姓名与邮箱
- ⚠️ 若自行改用 `urllib` 实现，**必须手动 `gzip.decompress`**（`requests` 会自动解压，`urllib` 不会）——否则 SEC 返回内容解析失败、表现为"数据莫名变少"


## v1.0.1 — 2026-06-20

### 修复（PR #1 @APTX4869-maker + 连带 bug）
- **5 个函数漏传 `params` 导致始终拿不到数据（PR #1）**：`stock_quote_eastmoney` / `stock_kline_yahoo` / `fund_flow_daily` / `stock_search` / `market_stock_list` 都构造了 `params` dict，但 `requests.get(url, ...)` 时漏了 `params=params`，请求实际是裸 URL → 服务端返回空（东财 push2 返回 `rc:102`/`data:null`）。一次性补齐 5 处。**实测确认**：漏传时 `data=None`，补齐后 `market_stock_list` 返回 `total=5990`。致谢 @APTX4869-maker 的排查与验证。
- **连带 bug：`market_stock_list` 在 `diff` 为 dict 时崩溃（本次一并修）**：补齐 params 后该函数能拿到响应，但东财 push2 的 `diff` 字段**有时是 list、有时是按序号为键的 dict**（如 `{"0":{...},"1":{...}}`）。旧代码 `for item in diff` 遇 dict 会拿到字符串键 → `AttributeError: 'str' object has no attribute 'get'`。新增 `if isinstance(diff, dict): diff = list(diff.values())` 归一化。**实测确认**：当前返回的 `diff` 正是 dict 结构，归一化后正常遍历。

### 说明
- 八层架构 / 18 端点 / 5 数据源不变；纯 bug 修复补丁。这 5 个端点此前**始终返回空数据**，升级强烈建议。

## v1.0 — 2026-05-20

### 首次开源发布

- **八层数据架构**：行情 / K线 / 技术指标 / 基本面 / 资金面 / 期权 / SEC Filing / 工具
- **18 个端点**覆盖美股 + 港股全品类数据
- **5 个数据源**：东财（push2 + push2his + datacenter + search）、Yahoo Finance（crumb 自动管理）、新浪财经、腾讯财经、SEC EDGAR
- 全部零鉴权（Yahoo crumb 自动获取，SEC 仅需 User-Agent）
- 仅依赖 `requests`，零第三方数据封装
- 内嵌完整 Python 代码，AI 编程助手直接可用
- 2026-05-20 全部端点实测验证

### 端点清单

| 层 | 端点 | 数据源 |
|----|------|--------|
| 行情 | 美股/港股实时报价 × 3 | 新浪 + 腾讯 + 东财push2 |
| K线 | 日/周/月/分钟 × 2 | 新浪 + Yahoo chart |
| 技术指标 | MA/EMA + MACD + RSI + KDJ + 布林带 × 1 | 纯Python计算（基于K线OHLCV） |
| 基本面 | 财报三表 + GMAININDICATOR + Yahoo 23模块 + SEC XBRL | 东财datacenter + Yahoo + EDGAR |
| 资金面 | 日级资金流 × 1 | 东财push2his |
| 期权 | 期权链 × 1 | Yahoo |
| SEC Filing | Filing列表 + XBRL × 2 | EDGAR |
| 工具 | 搜索 + 全市场列表 + 新闻 + CIK映射 × 4 | 东财search + 东财push2 + Yahoo + SEC |

### 实测发现与修正

- **东财 push2his K线不覆盖美股/港股**：kline/get 端点对美股(105.AAPL)/港股(116.00700) secid 返回空数据，仅资金流(fflow/daykline/get)正常。K线层改为新浪(美股) + Yahoo chart(美股+港股)
- **Yahoo 新闻需要 cookie**：v1/finance/search 裸请求返回 400，需先访问 fc.yahoo.com 获取 cookie
- **东财 push2 实时行情已验证**：push2.eastmoney.com/api/qt/stock/get 对美股(105.AAPL)和港股(116.00700)均返回完整数据
- **东财 GMAININDICATOR 已验证**：美股(RPT_USF10)返回49字段，港股(RPT_HKF10)返回75字段
- **东财 push2 全市场列表已验证**：美股(m:105)返回5925只，港股(m:116)返回18000+只
