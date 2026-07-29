# Financial Data Sources

## Overview

Financial data comes from many sources with different quality, latency, cost, and coverage. Choosing the right source is the most important architectural decision for any FinTech system.

**Data Quality Spectrum:**

```
Low Quality ←──────────────────────────────────→ High Quality

RSS Feeds / Social Media   Yahoo Finance (free)   SEC EDGAR / Exchange Data
NewsAPI (free tier)        Alpha Vantage          NSE/BSE (paid) / XBRL
```

---

## Yahoo Finance

Free, widely-used source for historical and current market data.

| Attribute | Value |
|---|---|
| Cost | Free |
| Data Quality | Medium (delays, gaps, errors) |
| Latency | 15-minute delayed (free tier) |
| Coverage | Global stocks, ETFs, indices, crypto, mutual funds |
| Format | JSON via unofficial API |
| Best For | Prototyping, historical data, learning |

**What it provides:**
- Historical OHLCV (Open, High, Low, Close, Volume)
- Current price and market data
- Financial statements (limited)
- Options data

**Limitations:**
- Unofficial API (no guarantee, can break anytime)
- Rate limits (~2K requests before blocking)
- No streaming (polling only)
- Delayed data (15 min for free tier)
- Can have incorrect data for small caps

---

## SEC EDGAR (US)

The US Securities and Exchange Commission's database of regulatory filings. All US public companies must file here.

| Attribute | Value |
|---|---|
| Cost | Free (government data) |
| Data Quality | Excellent (legally mandated, audited) |
| Latency | Real-time (filed immediately) |
| Coverage | All US public companies |
| Format | HTML, XBRL (structured XML), plain text |
| Best For | Fundamental analysis, US equities |

**Key filings:**

| Filing | Contents | Frequency | Engineer's View |
|---|---|---|---|
| **10-K** | Full annual report (financials, risks, strategy) | Yearly | Comprehensive JSON blob |
| **10-Q** | Quarterly report (revenue, profit, guidance) | Quarterly | Short update |
| **8-K** | Major events (CEO change, bankruptcy, M&A) | As needed | Event stream |
| **13F** | Institutional holdings (what big funds own) | Quarterly | Who owns what |
| **Proxy Statement** | Executive pay, board elections | Yearly | Governance |

**Why SEC EDGAR is the gold standard:**
- Legally mandated — companies must file accurate data
- Machine-readable (XBRL format — structured, tagged data)
- No API key required, no rate limits (within reason)
- Historical data back to 1994 (older in text format)
- Free for everyone

**Access methods:**
- EDGAR search (web interface)
- XBRL API (structured data)
- SEC's JSON API (modern)
- Third-party parsers (many open-source libraries)

---

## NSE / BSE (India)

Exchange-provided data for Indian markets.

| Attribute | NSE | BSE |
|---|---|---|
| Cost | Free (delayed) / Paid (real-time) | Free (delayed) / Paid (real-time) |
| Data Quality | Excellent | Excellent |
| Coverage | ~2,000 NSE-listed stocks | ~5,000 BSE-listed stocks |
| Format | CSV, API | CSV, API |

**Available data:**
- Live indices (NIFTY 50, Bank NIFTY, etc.)
- Stock quotes (15-min delayed free)
- Historical data
- Futures & Options data
- Corporate actions (splits, dividends, buybacks)

**Access methods:**
- NSE website (CSV downloads)
- Broker APIs (Zerodha Kite, Angel One, Upstox)
- Paid data vendors (Bloomberg, Reuters, IQVIA)

**Important note:** Most Indian stocks are listed on both NSE and BSE. NSE has higher volume; BSE has more companies.

---

## Annual Reports

Comprehensive documents published once per year. The most complete source of company information.

**Contents:**
- Audited financial statements (balance sheet, income statement, cash flow)
- Management's discussion and analysis (MD&A)
- Business strategy and outlook
- Risk factors
- Board composition and executive compensation
- Related-party transactions

**Where to find:**
- SEC EDGAR (10-K for US companies)
- NSE/BSE corporate filings (India)
- Company investor relations page
- AnnualReports.com (aggregator)

**Engineer's view:**
- Structured parts: Financial tables (XBRL, machine-readable)
- Unstructured parts: Management commentary (NLP / LLM territory)
- Embedded risks: Text mentions risks that numbers don't capture

---

## Quarterly Reports

Short updates every 3 months. More timely than annual reports but less comprehensive.

**Contents:**
- Revenue, profit, margins
- Segment performance breakdown
- Updated guidance for next quarter/year
- Key operating metrics (users, subscribers, ARPU, same-store sales)
- No audit (unaudited, but reviewed)

**Why quarterly reports matter more:**
- More frequent signals (4× per year vs. 1×)
- Stock moves significantly on quarterly results
- Annual reports mostly confirm what quarterly reports already showed
- Guidance updates are critical (forward-looking)

**Examples of metrics tracked quarterly:**

| Company | Key Metric | Why It Matters |
|---|---|---|
| Apple | iPhone revenue | Core business health |
| Netflix | Subscriber additions | Growth trajectory |
| Tesla | Vehicle deliveries | Production and demand |
| Reliance | Jio subscriber additions | Telecom growth |

---

## Company Filings

All legal documents companies must submit to regulators.

| Jurisdiction | Filing System | Key Documents |
|---|---|---|
| United States | SEC EDGAR | 10-K, 10-Q, 8-K, 13F, DEF 14A |
| India | NSE/BSE Corporate Filings | Quarterly results, shareholding pattern, board meetings |
| United Kingdom | Companies House | Annual accounts, confirmation statements |
| European Union | ESMA / National Regulators | Annual financial reports, insider transactions |

**What they contain beyond financials:**
- Insider trading reports (who bought/sold and when)
- Shareholding pattern (who owns what percentage)
- Pledge disclosures (promoters using shares as loan collateral)
- Board meeting outcomes (dividend declaration, stock split)
- Auditors' reports (any red flags?)

---

## RSS Feeds

Real-time news feeds in XML format.

| Attribute | Value |
|---|---|
| Cost | Free |
| Data Quality | Low (headlines only, no full articles) |
| Latency | Low (near real-time) |
| Format | XML (RSS 2.0 / Atom) |
| Coverage | Thousands of sources (if they still support RSS) |

**Why RSS matters for systems:**
- Machine-readable (XML, no scraping)
- Free and real-time
- Standard format (parseable by any language)
- Low latency (pushed, not polled)

**Limitations:**
- Declining support (many sites removed RSS)
- Headlines only (no article body)
- No metadata (no sentiment, no classification)
- Noise (includes irrelevant stories)

---

## News APIs

Structured news access with metadata and often sentiment analysis.

**Options:**

| API | Pricing | Coverage | Best For |
|---|---|---|---|
| NewsAPI | Free: 100 req/day | 150K+ sources | Prototyping |
| Alpha Vantage | Free: 5 req/min | News + prices | All-in-one |
| Polygon.io | Paid (~$30-200/mo) | Real-time + historical | Production US markets |
| Bloomberg | Very expensive (~$20K+/yr) | Comprehensive | Institutional |
| Reuters | Very expensive | Comprehensive | Institutional |

**What news APIs provide:**
- Headlines + article text
- Source, author, date, category
- Sentiment scores (some)
- Ticker tags (which stocks are mentioned)
- Entity recognition (companies, people, topics)

---

## Data Aggregators

Paid services that collect, normalize, and distribute data from multiple sources.

| Service | Pricing | Coverage | Best For |
|---|---|---|---|
| Bloomberg Terminal | ~$25K/yr | Everything | Professional traders |
| Reuters Eikon | ~$20K/yr | Everything | Professional traders |
| Quandl (Nasdaq Data Link) | Free + paid tiers | Alternative data | Quantitative research |
| Intrinio | $50-200/mo | US + some global | Startups / small teams |
| Polygon.io | $30-200/mo | US equities | Developers |

---

## Programming Analogy

```
Data Sources = External API integrations

Yahoo Finance = Free public API (unreliable, rate-limited, changes without notice)
SEC EDGAR = Government API (well-documented, no auth, reliable, no rate limit)
NSE/BSE = Exchange-specific SDK (needs subscription, regional)
RSS = Pub/Sub webhook feed (real-time, unstructured)
News API = Enterprise NLP pipeline (costs money, structured output)

A financial data pipeline is an ETL system:

EXTRACT:
  - SEC EDGAR (structured XBML) → pull batch
  - Yahoo Finance (JSON) → poll every 5 min
  - RSS (XML) → push stream
  - News API (JSON) → poll every min

TRANSFORM:
  - Normalize schemas into common format
  - Resolve ticker symbols across exchanges
  - Handle missing/corrupt data
  - Adjust for stock splits, dividends

LOAD:
  - Time-series database (prices)
  - Relational database (fundamentals)
  - Document store (filings, reports)
  - Search index (for LLM retrieval)
```

---

## Common Mistakes

- **Relying on free data for production systems.** Yahoo Finance is fine for learning, not for a product serving users.
- **Ignoring rate limits.** News APIs, Yahoo Finance, Alpha Vantage — all have strict limits on free tiers.
- **Not normalizing across sources.** Different sources use different tickers, date formats, and field names for the same data.
- **Assuming real-time data from free sources.** Free data is 15-20 minutes delayed. Real-time costs money.
- **Not handling corporate actions.** Stock splits change historical prices. Dividends cause price drops. Adjustments are necessary.
- **Trusting data without validation.** Free sources have errors. Cross-reference with multiple sources.

---

## Interview Notes

- **System Design: "Design a financial data ingestion platform"** — handle heterogeneity, rate limits, latency, data quality
- **Data Engineering: "Build a pipeline processing 10+ data sources into unified schema"** — normalization, deduplication, error handling
- **Backend: "Design a ticker symbol resolution service"** — mapping AAPL (NASDAQ) vs. AAPL.NS (NSE) vs. 037833100 (CUSIP)
- **ML Pipeline: "How do you feed financial data to an ML model?"** — feature engineering from raw source data, handling survivorship bias, look-ahead bias

---

## Revision Summary

| Source | Cost | Quality | Latency | Best For |
|---|---|---|---|---|
| Yahoo Finance | Free | Medium | 15-min delayed | Prototyping, historical |
| SEC EDGAR | Free | Excellent | Real-time | US fundamental analysis |
| NSE / BSE | Free/Paid | Excellent | Delayed or real-time | Indian market data |
| Annual Reports | Free | High | Yearly | Deep company analysis |
| Quarterly Reports | Free | High | Quarterly | Earnings tracking |
| Company Filings | Free | High | Event-driven | Corporate actions |
| RSS Feeds | Free | Low | Real-time | Real-time news stream |
| News APIs | Free/Paid | Medium | Real-time | Structured news |

- **Free sources** are good for learning and prototyping
- **Regulatory sources** (SEC EDGAR) are the gold standard for fundamental data
- **Exchange data** (NSE/BSE) requires subscription for real-time access
- **Normalization** across sources is the hardest engineering challenge
- **Always validate data** — free sources contain errors

---

← [10-company-performance](10-company-performance.md) • [↑ Phase 1](README.md) • [↑ Finance](../README.md) • [12-ai-in-finance](12-ai-in-finance.md) →
