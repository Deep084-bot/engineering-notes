# Market Capitalization

## Definition

**Market Capitalization (Market Cap)** = Current Stock Price × Total Number of Outstanding Shares

```
Market Cap = Price × Shares Outstanding

Apple example:
  Price: $150
  Outstanding Shares: 15.5 billion
  Market Cap: $150 × 15.5B = $2.3 Trillion
```

Market cap represents the total market value of a company's equity. It's what it would cost to buy **every single share** at the current price.

---

## Classification

| Category | Market Cap Range | Examples |
|---|---|---|
| **Large Cap** | > $10 Billion | Apple, Microsoft, Reliance, TCS, HDFC Bank |
| **Mid Cap** | $2B – $10B | Zomato, Tata Power, Etsy, Dropbox |
| **Small Cap** | < $2B | Regional banks, small biotech companies |
| **Mega Cap** | > $200 Billion | Apple, Microsoft, NVIDIA, Saudi Aramco |

**Note:** Thresholds vary slightly by market and source, but the ranges above are typical.

---

## Why Market Cap Matters

**Risk profile:**

| Attribute | Large Cap | Mid Cap | Small Cap |
|---|---|---|---|
| Volatility | Low | Medium | High |
| Liquidity | High | Medium | Low |
| Risk | Lower | Medium | Higher |
| Growth potential | Limited | Good | High (if successful) |
| Analyst coverage | Many analysts | Some | Few or none |
| Data availability | Excellent | Good | Limited |

**Common usage:**
- Portfolio diversification (hold all three caps)
- Risk assessment (small caps are riskier)
- Index inclusion (most indices only include large caps)
- Investment style (value vs. growth across caps)

---

## Free Float

**Free Float** = shares available for public trading. Excludes shares held by promoters, founders, governments, or strategic investors.

```
Total Shares: 1,000,000
Promoter Holdings: 600,000 (60%)
──────────────────────────────────
Free Float: 400,000 (40%)
Free Float Market Cap: Price × 400,000
```

**Why free float matters:**

| Scenario | Implication |
|---|---|
| Low free float (10-20%) | Price can swing wildly; one big order moves the market |
| High free float (70-80%) | More stable; large orders absorbed easily |
| Low free float + big buyer | Price can spike dramatically |
| Low free float + big seller | Price can crash |

**Programming Analogy:**

```
Free Float = Available capacity in an auto-scaling group

Total EC2 instances = 100
Reserved for critical jobs = 60
Available in auto-scaling group = 40 (free float)

Traffic spike hits those 40 → instances scale (price moves)
More available capacity → smoother traffic handling (stable price)
```

---

## Index Weighting

Most major indices use **free-float market cap** (not total market cap) for weighting.

```
Index weight = Company's Free-Float Market Cap / Total Free-Float MCap of Index

Reliance: ₹15L Cr free-float MCap
NIFTY 50: ₹200L Cr total free-float MCap
Reliance weight in NIFTY 50 = 15/200 = 7.5%
```

This means only freely tradable shares count toward index influence. Promoter-held shares are excluded from index weight calculations.

---

## Market Cap vs. Enterprise Value

**Market Cap** only reflects equity value (stock). It ignores debt and cash.

**Enterprise Value (EV)** reflects the total value of the company:

```
EV = Market Cap + Total Debt - Cash & Cash Equivalents
```

**Why the distinction matters:**

```
Company A: Market Cap = $10B, Debt = $5B, Cash = $1B
  → EV = $10B + $5B - $1B = $14B

Company B: Market Cap = $10B, Debt = $0B, Cash = $5B
  → EV = $10B + $0B - $5B = $5B

Same market cap, very different enterprise value.
```

Enterprise Value is used when evaluating acquisition targets (buyer takes on the debt) and in valuation multiples (EV/EBITDA).

---

## Common Mistakes

- **Ignoring free float.** A company with $1B market cap but 90% promoter holding effectively has only $100M of tradable stock. One large trade can move the price 10%.
- **Thinking all large caps are safe.** Large cap = large company, not safe company. Enron was large cap before it went to zero.
- **Using market cap alone to value a company.** Market cap ignores debt. Two companies with the same market cap can have very different total values.
- **Confusing market cap with enterprise value.** Market cap = what equity holders own. Enterprise value = total company value (equity + debt - cash).

---

## Programming Analogy

```
Market Cap = Total value of all pointers to an object

Each share = one reference (pointer)
Price = value per reference
Shares outstanding = total reference count
Market cap = total reachable object value

Free float = references NOT held by the original authors (promoters)

Enterprise Value = total cost to acquire the object
  = buy all references (market cap) + assume all debt - take cash
```

---

## Real World Examples

| Company | Price (approx) | Shares | Market Cap | Free Float |
|---|---|---|---|---|
| Apple | $150 | 15.5B | $2.3T | ~99% |
| Berkshire Hathaway | $600,000 (BRK.A) | 1.4M (Class A) | ~$900B | Low (Buffett holds a lot) |
| Reliance Industries | ₹2,500 | 675 Cr | ~₹17L Cr (~$200B) | ~55% |
| TCS | ₹3,500 | 365 Cr | ~₹12L Cr (~$145B) | ~27% (Tata Sons holds majority) |

---

## Interview Notes

- **FinTech: "Design a stock screener"** — market cap is the primary filter
- **Data Engineering: "Track market cap changes daily from streaming data"** — requires handling stock splits, buybacks, share issuance
- **Quant: "Why do most indices use free-float market cap?"** — avoids overweighting companies with large promoter holdings that aren't actually tradable
- **Valuation: "What's the difference between market cap and enterprise value?"** — debt and cash adjustment

---

## Revision Summary

| Concept | Formula |
|---|---|
| Market Cap | Price × Shares Outstanding |
| Free Float | Total Shares - Restricted Shares |
| Free Float MCap | Price × Free Float Shares |
| Enterprise Value | Market Cap + Debt - Cash |

- **Large Cap** (>$10B): stable, liquid, lower risk
- **Mid Cap** ($2B–$10B): moderate risk/reward
- **Small Cap** (<$2B): volatile, higher risk/reward
- **Free float** determines actual liquidity and volatility
- **Indices** use free-float market cap for weighting

---

← [04-how-trading-works](04-how-trading-works.md) • [↑ Phase 1](README.md) • [↑ Finance](../README.md) • [06-market-indices](06-market-indices.md) →
