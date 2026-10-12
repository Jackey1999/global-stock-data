<p align="center"><a href="README_zh.md">简体中文</a> | <b>English</b></p>

<h1 align="center">global-stock-data</h1>

<p align="center">
  <b>US Market Data for AI Coding Assistants</b><br>
  Options Greeks · 0DTE Flow · SEC Filings · Short Volume · Fundamentals · Screener · Global Rates & FX
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-blue.svg" alt="License: Apache 2.0"></a>
  <a href="https://www.python.org/"><img src="https://img.shields.io/badge/Python-3.9+-3776AB?logo=python&logoColor=white" alt="Python 3.9+"></a>
  <a href="https://github.com/simonlin1212/global-stock-data/stargazers"><img src="https://img.shields.io/github/stars/simonlin1212/global-stock-data?style=social" alt="GitHub stars"></a>
  <a href="#architecture"><img src="https://img.shields.io/badge/layers-14-2ea44f.svg" alt="Layers"></a>
  <a href="#endpoints"><img src="https://img.shields.io/badge/endpoints-40%2B-2ea44f.svg" alt="Endpoints"></a>
  <a href="#data-sources"><img src="https://img.shields.io/badge/sources-14-2ea44f.svg" alt="Sources"></a>
  <a href="#data-sources"><img src="https://img.shields.io/badge/auth-zero-success.svg" alt="Zero-Auth"></a>
</p>

---

<p align="center">
  <a href="#architecture">Architecture</a> ·
  <a href="#compliance-tiers">Compliance</a> ·
  <a href="#quick-start">Quick Start</a> ·
  <a href="#endpoints">Endpoints</a> ·
  <a href="#data-sources">Data Sources</a> ·
  <a href="#faq">FAQ</a>
</p>

> **Full-stack market data for US equities** — 14 layers · 40+ endpoints · 14 sources · zero-auth · only `requests`.
>
> **Official-source-first (V2.0):** CBOE options with full Greeks + **0DTE** flow, FINRA market-wide short volume, SEC EDGAR filing stream + a free market-wide screener, Treasury / CFTC / earnings calendar. **Every source is labeled with its compliance tier, terms quoted verbatim** — "official" does not mean "free to use".
>
> **New in V2.1:** a global rates & FX layer — policy rates for 38 central banks (BIS), ECB FX / key rates / euro-area yield curve, NY Fed SOFR / EFFR, Fed H.10 FX / H.6 money stock, and US Treasury auctions + federal debt, all tier S. Yahoo K-line now returns true full-history daily bars in exchange-local dates, builds weekly / monthly / quarterly bars from daily data, and works for Japan / Taiwan / Korea / India / Europe and more.

A single self-contained Skill file that turns raw US stock data — scattered across many sources — into a toolkit an AI coding assistant can call directly. You no longer memorize Eastmoney secid prefixes, Yahoo crumb auth flows, or SEC EDGAR CIK mappings. That's all handled.

> *This project distributes **code, not market data**. Data is fetched by the user under each source's own terms. For commercial use, rely only on tier-S sources.*

> Compatible with [Claude Code](https://github.com/anthropics/claude-code) · [Codex](https://github.com/openai/codex) · [OpenClaw](https://github.com/anthropics/openclaw). The Skill file is structured Markdown + embedded Python — any AI coding assistant with context injection can use it.

---


## Architecture

```
US Full-Stack Data · 14-layer architecture · V2.1
│
├── Market Data      Sina + Tencent + Eastmoney push2        Real-time quotes, 25-78 fields
├── K-line           Sina(back to 1984) + Yahoo chart        Daily / weekly / monthly / minute; JP/TW/KR/IN/EU via suffix
├── Technical Ind.   MA/EMA + MACD + RSI + KDJ + Bollinger   Pure Python, zero extra deps
├── Fundamentals     Eastmoney + Yahoo + SEC XBRL            Statements + key metrics + holdings
├── Fund Flow        Eastmoney push2his                      Daily main/large/medium/small flow
├── Options          Yahoo crumb                             Options chain (US only)
├── SEC Filing       EDGAR submissions + XBRL                10-K/10-Q/8-K + 503 GAAP metrics (US)
└── Tools            Eastmoney + Yahoo + SEC CIK             Search + market list + news + ticker↔CIK

━━━ New in V2.0 (official-source-first) ━━━
├── Options·CBOE     cdn.cboe.com official feed   ⭐ full chain + IV + delta/gamma/vega/theta/rho + 0DTE + flow
├── Short Volume     FINRA Reg SHO                ⭐ market-wide daily short volume (12,112 symbols) + series
├── Filing Stream    EDGAR daily index + FTS      ⭐ Form 4 / 8-K / 13F same-day + full-text since 2001
├── Market Screener  EDGAR frames                 ⭐ any XBRL tag across the whole market (1,842~5,309 co.) — free
└── Macro / Calendar Treasury + CFTC + Nasdaq     ⭐ yield curve (1M~30Y) + COT + earnings calendar

━━━ New in V2.1 ━━━
└── Global Rates/FX  BIS + ECB + NY Fed + Fed + Treasury ⭐ 38 central-bank policy rates + EUR/USD FX + SOFR/EFFR + euro curve + auctions
```

---

## Compliance tiers

Source terms differ a lot. **"Official" does not mean "free to use".** These tiers come from reading each source's actual terms of service (2026-07-24; Yahoo / Nasdaq re-checked and 4 new official sources read on 2026-10-11). Quotes are verbatim.

<table>
<thead>
<tr><th nowrap>Tier</th><th nowrap>Commercial</th><th nowrap>Redistribute</th><th>Sources</th><th>Basis (quoted)</th></tr>
</thead>
<tbody>
<tr><td nowrap><strong>S</strong></td><td nowrap>✅</td><td nowrap>✅</td><td>SEC EDGAR / Treasury / CFTC / Federal Reserve Board / NY Fed / BIS / ECB</td><td>EDGAR: <em>&quot;Anyone can access and download this information <strong>for free</strong>&quot;</em>, <em>&quot;We <strong>allow scripted access</strong>&quot;</em> — hard limit <strong>10 req/s</strong>, User-Agent required. Fed Board: <em>&quot;public domain&quot;</em>. NY Fed: <em>&quot;personal or business purposes&quot;</em>, <em>&quot;through an automated process&quot;</em>. BIS: <em>&quot;unrestricted&quot;</em>. ECB: <em>&quot;reused free of charge&quot;</em>. <strong>BIS / ECB / Fed Board require attribution, ECB data must not be modified, and NY Fed reference rates need its notice when shown to others</strong></td></tr>
<tr><td nowrap><strong>B</strong></td><td nowrap>⚠️ verify first</td><td nowrap>❌</td><td>FINRA</td><td>Files are published for download, but terms prohibit <em>&quot;data mining, scraping or harvesting tools (including robots)&quot;</em> and state <em>&quot;<strong>non-commercial use</strong>&quot;</em></td></tr>
<tr><td nowrap><strong>C</strong></td><td nowrap>❌ needs permission</td><td nowrap>❌</td><td>CBOE / Eastmoney / Sina / Tencent</td><td>CBOE requires <em>&quot;<strong>approval in advance</strong>&quot;</em> + <em>&quot;<strong>execution of a license agreement</strong>&quot;</em>; Eastmoney / Sina / Tencent are website front-end endpoints, personal research only</td></tr>
<tr><td nowrap><strong>D</strong></td><td nowrap>❌</td><td nowrap>❌</td><td>Yahoo / Nasdaq</td><td>Terms prohibit automated access without permission. Yahoo: <em>&quot;using any automated means… for any purpose without our express, prior permission&quot;</em>. Nasdaq: <em>&quot;any process, whether automated or manual, to capture data… for any reason&quot;</em>. <strong>Personal study and research only, at your own risk; no commercial use or redistribution</strong></td></tr>
<tr><td nowrap><strong>⛔ Excluded</strong></td><td nowrap>—</td><td nowrap>—</td><td>HKEX (CCASS)</td><td>Terms prohibit <em>&quot;any '<strong>robot</strong>', '<strong>bot</strong>', '<strong>spider</strong>', '<strong>scraper</strong>'...&quot;</em> and apply <em>&quot;<strong>whether or not for gain</strong>&quot;</em> → <strong>this tool ships no scraper for it</strong></td></tr>
</tbody>
</table>

A working HKEX shareholding layer was built, then removed. Shipping code that violates a source's terms would defeat the point of grading sources in the first place. **This project distributes code, not data.** For commercial use, rely only on tier S.

**So why keep tier-D Yahoo and Nasdaq?** Honestly, their terms are the same kind as HKEX's. The difference is that they have no substitute: Yahoo is the only source for HK K-lines and for markets beyond the US and HK, and Nasdaq is the only key-free earnings calendar, while the HKEX shareholding layer was a nice-to-have. So the code stays and the terms are quoted in full (see "数据源合规分级" in SKILL.md). Whether to use them is your call under those terms.

---

## Quick start

**3 steps, 2 minutes.**

```bash
# 1. Create the skill directory
mkdir -p ~/.claude/skills/global-stock-data

# 2. Download SKILL.md
curl -o ~/.claude/skills/global-stock-data/SKILL.md \
  https://raw.githubusercontent.com/simonlin1212/global-stock-data/main/SKILL.md

# 3. Install the dependency
pip install requests
# Windows only: also install tzdata so Yahoo K-line dates use the exchange's time zone
pip install tzdata
```

⚠️ **Before using the SEC layers, set `SEC_CONTACT` in SKILL.md to your real name and email.** SEC requires a declared User-Agent; requests without one are rejected as an *Undeclared Automated Tool*. The code raises a clear error if you forget.

Launch Claude Code and say "check AAPL's financials" — the skill activates on its own. **Codex / OpenClaw users:** paste SKILL.md into your system prompt or project context file; the embedded Python runs as-is.

---

## Endpoints

**Price** — Sina / Tencent / Eastmoney quotes (25-78 fields) · Sina & Yahoo K-line (US back to 1984, daily→minute; Japan / Taiwan / Korea / India / Europe and more via ticker suffix) · MA/EMA/MACD/RSI/KDJ/Bollinger computed locally.

**Value** — Eastmoney three statements + GMAININDICATOR (US 49 / HK 75 fields) · Yahoo quoteSummary (23 modules: financials, analysts, institutional holdings) · SEC EDGAR XBRL (503 GAAP metrics).

**Flow & positioning ⭐** — CBOE options with full Greeks + IV + **0DTE** + unusual-flow detection · FINRA market-wide daily short volume (12,112 symbols) · Eastmoney daily fund flow.

**Filings & events ⭐** — EDGAR daily index (same-day **Form 4 insider / 8-K / 13F**, one day: 547 / 370 / 261) · full-text search since 2001 · 10-K/10-Q/8-K list.

**Screener & macro ⭐** — EDGAR frames (any XBRL tag across the whole market, free — net income CY2025Q1 = 5,309 companies) · Treasury yield curve (1M~30Y) · CFTC COT · Nasdaq earnings calendar.

**Global rates & FX ⭐ (V2.1)** — policy rates for 38 central banks in one call (BIS) · ECB FX / three key rates / euro-area yield curve · NY Fed SOFR / EFFR · Fed H.10 USD vs 23 currencies / H.6 M1·M2 · Treasury auction results and schedule / federal debt / average interest rates.

**Tools** — stock search (CN+EN) · full-market list (US 5925+ / HK 18000+) · news by ticker · ticker↔CIK.

<details>
<summary>Full endpoint tables</summary>

<br>

### Options ⭐ (US only)

| Endpoint | Data |
|----------|------|
| **CBOE official** | Full chain + **IV + delta/gamma/vega/theta/rho**, plus **0DTE filtering** and **unusual-flow** detection (vol/OI > 1 = new positioning) |
| Yahoo (fallback) | Options chain, all expiries — **no Greeks** |

### Short volume ⭐ (US only)

| Endpoint | Data |
|----------|------|
| FINRA Reg SHO | Market-wide **daily short volume** (12,112 symbols), per-symbol series, ratio ranking |

### Filing stream ⭐ (US only)

| Endpoint | Data |
|----------|------|
| EDGAR daily index | Same-day **Form 4 / 8-K / 13F / 144** — one day: 547 / 370 / 261 / 118 |
| EDGAR full-text | Search every filing body back to 2001 |
| EDGAR submissions / XBRL | Filing list + structured financials (503 GAAP metrics) |

### Market screener ⭐ (US only)

| Endpoint | Data |
|----------|------|
| EDGAR frames | One XBRL tag across the whole market in one call — a free screener |

### Macro / calendar ⭐

| Endpoint | Data |
|----------|------|
| Treasury / CFTC / Nasdaq | Yield curve (1M~30Y) · COT · earnings calendar (pre/post-market + EPS est.) |

### Global rates / FX / central banks ⭐ (V2.1)

| Endpoint | Data |
|----------|------|
| BIS `policy_rates()` | Policy rates for 48 economies (38 still updated), all in one call |
| ECB `ecb_fx()` / `ecb_key_rates()` / `ecb_yield_curve()` / `ecb_series()` | EUR reference rates · deposit facility and the other two key rates · euro-area government curve (AAA or all issuers) · any ECB series |
| NY Fed `nyfed_rates()` | SOFR / EFFR / OBFR / TGCR / BGCR with percentiles and volume; SOFR averages and index |
| Fed Board `fed_data()` | H.10 USD vs 23 currencies · H.6 M1/M2 and components |
| Treasury `fiscal_data()` | Auction results (high yield, bid-to-cover) and upcoming schedule · daily federal debt · average interest rates · Treasury reporting FX rates |

### Price / value / flow / tools

| Endpoint | Data |
|----------|------|
| Sina / Tencent / Eastmoney push2 | US/HK real-time quotes, 25-78 fields |
| Sina / Yahoo chart | K-line, daily→minute, US back to 1984; Yahoo gives full daily history (S&P 500 since 1927) and markets beyond US/HK |
| Eastmoney datacenter / GMAININDICATOR | Statements + key metrics (bilingual) |
| Yahoo quoteSummary | 23 modules: financials / analysts / institutional holdings |
| Eastmoney push2his | Daily main/large/medium/small fund flow |
| Eastmoney search / push2 list / Yahoo search / SEC CIK | Search / full-market list / news / ticker↔CIK |

Every source is free, no API key. Yahoo crumb is auto-managed; SEC EDGAR only needs a declared User-Agent.

</details>

---

## Usage examples

Just tell your AI assistant:

| Scenario | Prompt |
|----------|--------|
| 0DTE options flow ⭐ | "Show NVDA's 0DTE options flow — unusual activity" |
| Short volume ⭐ | "TSLA's short volume ratio trend this week" |
| Insider filings ⭐ | "Any Form 4 insider filings today for my watchlist" |
| Full-text filings ⭐ | "Which companies first mentioned 'HBM4' in an 8-K" |
| Market screener ⭐ | "Rank all filers by R&D expense for CY2025Q1" |
| Macro ⭐ | "What's the 10Y-2Y Treasury spread right now" |
| Central banks ⭐ | "Current policy rates of the major central banks" · "SOFR and EFFR over the past week" |
| FX / Treasuries ⭐ | "EUR/JPY this month" · "Next week's Treasury auction schedule" |
| Other markets | "Toyota 7203.T daily K-line for the past year" · "TSMC 2330.TW full history" |
| US / HK quote | "AAPL's price and PE" · "How's Tencent 00700 today" |
| K-line | "Pull TSLA's daily K-line for the past 6 months" |
| Statements / valuation | "Apple's latest income statement" · "BABA's PE/PB/ROE + target" |
| Institutional holdings | "Which institutions hold NVDA, and what %" |
| Technical analysis | "AAPL's MACD and RSI — any golden cross?" |
| Batch compare | "Compare valuations of AAPL MSFT GOOGL" |

---

## Data sources

| Source | Tier | Auth | Coverage |
|--------|------|------|----------|
| **SEC EDGAR** | **S** | real UA | US filings / XBRL / **filing stream** / **full-text search** / **market screener** |
| **US Treasury** | **S** | none | **Yield curve (1M~30Y)** · FiscalData **auctions / federal debt / average rates** |
| **CFTC** | **S** | none | **COT reports** |
| **Federal Reserve Board** | **S** | none | **H.10 FX / H.6 money stock** (attribution) |
| **NY Fed** | **S** | none | **SOFR / EFFR / OBFR / TGCR / BGCR** (notice required when shown to others) |
| **BIS** | **S** | none | **Central-bank policy rates** (attribution) |
| **ECB** | **S** | none | **EUR FX / ECB key rates / euro-area yield curve** (attribution, no modification) |
| **FINRA** | **B** | none | US **market-wide daily short volume** (verify before commercial use) |
| **CBOE** | **C** | none | US **options + Greeks + IV + 0DTE** (needs Cboe's prior approval) |
| Eastmoney (push2 / push2his / datacenter / search) | C | none | US+HK quotes / fund flow / statements / search |
| Sina | C | none | US+HK quotes, US K-line |
| Tencent | C | none | US+HK quotes |
| Yahoo Finance | D | cookie+crumb (auto) | US+HK+other markets, all categories (terms prohibit automated access) |
| Nasdaq | D | none | US **earnings calendar** (terms prohibit automated access) |

**Tiers:** **S** = government and central-bank data, commercial + redistribution OK (some require attribution) · **B** = published files, verify before commercial use · **C** = needs prior approval or terms unverified, personal research only · **D** = terms prohibit automated access without permission; personal study and research only, at your own risk, no commercial use or redistribution. See [Compliance tiers](#compliance-tiers) for the quoted basis. All requests go through direct HTTP with a built-in thread-safe rate limiter (SEC capped at 8 req/s against the official 10).

---

## FAQ

**Q: How does this relate to a-stock-data?**
Sister project. [a-stock-data](https://github.com/simonlin1212/a-stock-data) covers China A-shares; global-stock-data covers US and HK. Install both — they don't conflict.

**Q: Does Yahoo Finance need an API key?**
No. The code fetches cookie + crumb automatically and refreshes on expiry.

**Q: Any limits on SEC EDGAR?**
Yes — a declared User-Agent and 10 req/s. Set `SEC_CONTACT` to your real email; the built-in throttle stays under the limit.

**Q: HK options data?**
No. HK options aren't in Yahoo's coverage and need HKEX's paid feed. The options layer is US only.

**Q: Can I look up stocks outside the US and HK?**
K-lines and Yahoo fundamentals, yes: Japan `.T`, Taiwan `.TW`, Korea `.KS`, India `.NS`, London `.L`, Germany `.DE` and more — see "Yahoo Finance 代码格式" in SKILL.md. Note that Yahoo is a tier-D source. For China A-shares, use the sister project a-stock-data.

**Q: How fresh are the global rates and FX?**
ECB and NY Fed update every business day; Fed H.10 is published each Monday for the prior week; BIS lags by a few days to about three weeks depending on the country. Every row carries its date — check the central bank itself when you need today's exact value.

**Q: Running from a server in mainland China — can it reach Yahoo / SEC?**
They're overseas and may be flaky on a direct connection. Use a proxy, or lean on the Eastmoney / Sina / Tencent sources.

**Q: Can I use it without Claude Code?**
Yes. SKILL.md is Markdown + embedded Python — Codex, OpenClaw, or any AI coding assistant can read it, and you can copy the Python into your own scripts.

---

## Changelog

See [CHANGELOG.md](./CHANGELOG.md).

## Disclaimer

This project provides data-access tools only. It is not investment advice. Investing involves risk.

## License

[Apache License 2.0](./LICENSE) — **Author:** Simon Lin · X [@linsizhen](https://x.com/linsizhen) · Email: [simonlin0423@gmail.com](mailto:simonlin0423@gmail.com)
