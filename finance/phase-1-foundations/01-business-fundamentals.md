# Business Fundamentals

## Company

A **company** (or firm, corporation) is a legal entity that exists separately from its owners. It can own assets, sign contracts, sue, be sued, and pay taxes.

**Key properties:**
- Separate legal personality (exists independently of founders)
- Perpetual existence (continues even if founders leave)
- Limited liability (owners can lose only what they invested)
- Can raise capital by selling ownership (equity) or borrowing (debt)

---

## Private Company

Shares are NOT traded on any public exchange. Ownership is restricted to founders, employees, and select investors.

**Characteristics:**
- No obligation to publish financials
- Less regulatory overhead
- Funded by: founder savings, angel investors, venture capital
- More control for founders (fewer shareholders to answer to)

**Examples:** Stripe, SpaceX, ByteDance, Databricks, Swiggy

---

## Public Company

Shares are listed on a stock exchange and can be bought/sold by anyone.

**Characteristics:**
- Must publish quarterly and annual financial reports
- Highly regulated (SEC in US, SEBI in India)
- Thousands or millions of shareholders
- Stock price reflects public market valuation
- Can raise capital by issuing more shares

**Examples:** Apple, Microsoft, Google, Reliance, TCS, HDFC Bank

**Comparison**

| Attribute | Private | Public |
|---|---|---|
| Ownership | Restricted | Open to anyone |
| Financial data | Private | Public, regulated |
| Regulation | Light | Heavy |
| Liquidity for owners | Low (hard to sell) | High (sell on exchange) |
| Decision speed | Fast (few owners) | Slow (board, shareholders) |

---

## Startup

A young company designed for rapid growth. Technology-focused, high risk, high reward.

**Characteristics:**
- Usually private
- Funded by venture capital in exchange for equity
- Prioritizes growth over profitability
- High failure rate (>90%)
- Goal: IPO or acquisition by a larger company

**Lifecycle:**
```
Idea → Seed → Series A → Series B → Series C → IPO or Acquisition
                  (each round = more funding, higher valuation)
```

---

## Unicorn

A private startup valued at $1 billion or more. The term was coined because such companies were once rare.

**Examples:**
- OpenAI ($80B+)
- Stripe ($50B+)
- Databricks ($43B)
- Canva ($26B)

---

## Shareholder

Someone who owns one or more shares of a company. A partial owner.

**Rights depend on share type:**

| Right | Common Stock | Preferred Stock |
|---|---|---|
| Vote on board/decisions | Yes | Usually no |
| Receive dividends | Yes (if declared) | Yes (usually higher) |
| Get paid first in bankruptcy | Last | Before common |
| Convert to common | No | Yes |

**Shareholder ≠ Stakeholder.** Shareholders OWN the company. Stakeholders are AFFECTED by the company.

---

## Stakeholder

Any person or entity affected by a company's actions. A superset of shareholders.

**Stakeholder groups:**

```
Employees (jobs, salaries, culture)
Customers (products, prices, support)
Suppliers (contracts, payments)
Community (environment, jobs, taxes)
Government (taxes, regulation)
Shareholders (ownership, returns)
Creditors (loan repayments)
```

---

## Programming Analogy

```
Company = Class
  - Private company = internal/private class
  - Public company = exported/public class

Shares = Object references
Shareholders = Objects holding references
Stakeholders = All objects that interact with this instance

IPO = Publishing a package to npm/PyPI
  - Before: only internal team uses it
  - After: anyone can install (buy) it

Startup = Early-stage open-source project
  - High risk, high growth
  - Funded by grants/VC (like investor funding)
  - May become mainstream (IPO) or be acquired (acqui-hire)
```

---

## Common Mistakes

- **Assuming all companies publish data.** Private companies do not. Only public companies file regulated reports.
- **Confusing shareholders with stakeholders.** Shareholders own the company. Stakeholders include everyone affected.
- **Thinking all startups will IPO.** Most fail. A small percentage succeed.
- **Ignoring stakeholder conflicts.** Maximizing shareholder value can harm employees, customers, or the environment.

---

## Interview Notes

- **Data modeling for companies:** Schema must distinguish public vs. private, company type, share structure
- **System design for financial platforms:** Most features only work for public companies (data availability)
- **Startup interviews:** "Why do you want to work at a startup vs. public company?" — understand the tradeoffs
- **LLM agents for finance:** Agent must know whether a company is public (data available) or private (no data)

---

## Revision Summary

| Concept | Key Point |
|---|---|
| Company | Legal entity separate from owners |
| Private | Shares not traded publicly |
| Public | Shares traded on exchange, data available |
| Startup | High-growth private company |
| Unicorn | Startup valued at $1B+ |
| Shareholder | Owner of shares |
| Stakeholder | Anyone affected by company |

---

← [00-finance-vs-economics-vs-accounting](00-finance-vs-economics-vs-accounting.md) • [↑ Phase 1](README.md) • [↑ Finance](../README.md) • [02-stock-market-basics](02-stock-market-basics.md) →
