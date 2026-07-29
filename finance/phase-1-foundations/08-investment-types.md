# Investment Types

## Overview

Different types of investments (asset classes) have different risk, return, and liquidity characteristics.

**Risk-Return Spectrum:**

```
Low Risk                  →                  High Risk
────────────────────────────────────────────────────────
Bonds → Gold → Stocks → REITs → Crypto

Lower potential returns                     Higher potential returns
Lower volatility                           Higher volatility
```

---

## Stocks (Equities)

Ownership in a company. The most common investment.

| Attribute | Value |
|---|---|
| Returns (long-term) | ~10% CAGR (historical) |
| Risk | High |
| Volatility | High (15-30% annualized) |
| Liquidity | High (large cap) |
| Dividends | Some pay, some don't |

**Why people invest:**
- Long-term wealth creation
- Dividend income
- Ownership rights and voting
- Liquidity (easy to buy/sell)

**Examples:** Apple, Microsoft, Reliance, TCS.

---

## ETFs (Exchange Traded Funds)

A basket of securities that trades on an exchange like a single stock.

**How they work:**
- Buy one share of SPY → own a slice of all 500 S&P 500 companies
- Trades throughout the day like a stock
- Low fees (expense ratios 0.03%–0.30%)
- Automatically rebalances

**Popular ETFs:**

| ETF | Ticker | What It Holds | Expense Ratio |
|---|---|---|---|
| SPDR S&P 500 | SPY | All 500 S&P 500 stocks | 0.0945% |
| Invesco QQQ | QQQ | NASDAQ 100 stocks | 0.20% |
| Vanguard Total Stock Market | VTI | All US stocks | 0.03% |
| NIFTY 50 ETF | NIFTYBEES | NIFTY 50 stocks | ~0.05% |

**Programming Analogy:**

```
ETF = Wrapper class around an array of assets

class SPY:
    def __init__(self):
        self.holdings = get_sp500_constituents()
    
    def buy(self, shares):
        # buys proportional slice of all 500 stocks
        for stock in self.holdings:
            purchase_proportional(stock, shares)
    
    def rebalance(self):
        # match the index composition (cron job)
        current = get_current_composition()
        target = get_index_composition()
        diff = target - current
        execute_trades(diff)
```

---

## Mutual Funds

Professionally managed pooled investments. Investors buy units of the fund at NAV (Net Asset Value).

**How they work:**
1. Money from thousands of investors is pooled
2. Fund manager decides what to buy/sell
3. NAV calculated daily (total assets / total units)
4. Investors can buy/sell at the end-of-day NAV

**Comparison: ETFs vs. Mutual Funds**

| Feature | ETF | Mutual Fund |
|---|---|---|
| Trade when | Intraday (like a stock) | End of day (NAV) |
| Minimum investment | Price of 1 share | Often $500+ |
| Fees | Low (0.03–0.30%) | Higher (1–2%) |
| Active management | Mostly passive (index) | Both active and passive |
| Tax efficiency | Higher | Lower |
| Transparency | Daily holdings | Quarterly holdings |

**Types:**

| Type | What They Hold | Risk |
|---|---|---|
| Equity fund | Stocks | High |
| Debt fund | Bonds | Low |
| Hybrid fund | Stocks + Bonds | Medium |
| Index fund | Tracks an index | Market |
| Liquid fund | Short-term debt | Very low |

---

## Bonds

Loans made by an investor to a borrower (company or government).

**How they work:**

```
You lend $1,000 to the government at 5% interest for 10 years.

Year 1-10: You receive $50/year (coupon payment)
Year 10: You get your $1,000 back (principal)
Total return: $500 interest + $1,000 principal = $1,500
```

**Key terms:**

| Term | Definition |
|---|---|
| Face Value (Par) | Amount borrowed ($1,000) |
| Coupon Rate | Interest rate (5%) |
| Coupon Payment | Interest paid ($50/year) |
| Maturity | When principal is repaid (10 years) |
| Yield | Effective return based on current price |

**Types by issuer:**

| Type | Issuer | Risk | Return |
|---|---|---|---|
| Government | National government | Very low | 2-7% |
| Corporate | Company | Medium | 4-10% |
| Municipal | Local government | Low | Tax-free interest |
| High Yield ("Junk") | Risky company | High | 8-15% |

**Bond prices and interest rates:**

```
Rates go UP  → Existing bond prices go DOWN
Rates go DOWN → Existing bond prices go UP

Why: If new bonds pay 6%, your 5% bond is less valuable.
```

**Programming Analogy:**

```
Bond = Fixed-term contract with scheduled callbacks

class Bond:
    def __init__(self, principal, coupon_rate, maturity):
        self.principal = principal  # initial investment
        self.coupon_rate = coupon_rate  # interest rate
        self.coupon = principal * coupon_rate  # periodic payment
        self.maturity = maturity  # end date
    
    def get_annual_payment(self):
        return self.coupon  # callback
    
    def on_maturity(self):
        return self.principal  # return investment
```

---

## Gold

A physical commodity used as a store of value and hedge against inflation.

**Properties:**
- Low correlation with stocks (good diversifier)
- Rises during: inflation, market crashes, geopolitical tension, weak dollar
- No cash flow (unlike stocks/bonds — no dividends, no interest)
- Storage costs (if physical)

**Performance context:**

| Environment | Gold Performance |
|---|---|
| High inflation | Up |
| Market crash | Up (safe haven) |
| Strong dollar | Down |
| Rising interest rates | Down (opportunity cost) |
| Stable economy | Flat or down |

**How to invest:**
- Physical gold (bars, coins)
- Gold ETFs (GLD, IAU)
- Gold mining stocks
- Digital gold (sovereign gold bonds in India)

---

## REITs (Real Estate Investment Trusts)

Companies that own and operate income-generating real estate.

**How they work:**

```
REIT owns: office buildings, data centers, apartments, warehouses
Tenants pay rent → REIT collects income
REIT must distribute ≥ 90% of taxable income as dividends
Investors buy/sell REIT shares on exchanges
```

**Types:**

| Type | Properties | Example |
|---|---|---|
| Equity REIT | Owns and operates | Realty Income (retail), Equinix (data centers) |
| Mortgage REIT | Lends to real estate | Annaly Capital |
| Hybrid | Both | Various |

**vs. Direct Real Estate:**

| Attribute | REIT | Direct Real Estate |
|---|---|---|
| Liquidity | High (trade like stock) | Low (months to sell) |
| Minimum investment | Price of 1 share | $10,000+ |
| Diversification | Dozens/hundreds of properties | Usually 1 property |
| Management | Professional management | You manage (or pay a manager) |
| Dividends | 90% of income distributed | All income kept |

---

## Crypto (Cryptocurrency)

Digital assets using blockchain technology. No central authority.

**Characteristics:**
- 24/7 trading (no market close)
- Extremely high volatility (30-80% swings common)
- Minimal to no regulation
- Poor data quality (wash trading, manipulation)
- No fundamental valuation (speculative pricing)
- Correlated with tech stocks (increasingly)

**Major assets:**

| Asset | Type | Purpose |
|---|---|---|
| Bitcoin (BTC) | Store of value | "Digital gold" |
| Ethereum (ETH) | Smart contract platform | DApps, DeFi |
| Stablecoins (USDT, USDC) | Pegged to USD | Trading, payments |
| Solana (SOL) | High-speed blockchain | DeFi, gaming |

**Risks specific to crypto:**
- Regulatory uncertainty
- Exchange hacks and fraud
- Extreme volatility
- Wash trading (fake volume)
- No investor protection

---

## Programming Analogy

```
Asset Classes = Data Types in a type system

Stock = User-defined class (can do anything, has state)
ETF  = Collection / List (holds multiple items, iterable)
Bond = NamedTuple (fixed fields, predictable)
Gold = Singleton (one global instance, stores value)
REIT = Stream (constant income, stateful)
Crypto = Dynamic type (unpredictable behavior, no compile-time checks)
```

---

## Common Mistakes

- **Thinking all diversification is good.** Owning 10 tech stocks is NOT diversification. You need uncorrelated asset classes.
- **Ignoring correlation.** When stocks crash, sometimes gold and bonds also crash (2008: everything correlated). True diversification is hard.
- **Chasing past returns.** The best performing asset class last year is often the worst this year. Mean reversion is powerful.
- **Treating crypto like stocks.** Different risk profile, different behavior, different regulation. Apply different analysis.
- **Forgetting about fees.** A 2% annual fee on a mutual fund eats 33% of your returns over 20 years.

---

## Interview Notes

- **FinTech: "Design a multi-asset portfolio tracker"** — must handle different data sources, pricing schedules (NAV daily, stocks real-time)
- **System Design: "Design a robo-advisor"** — asset allocation, rebalancing, tax harvesting
- **ML: "Predict correlation between asset classes"** — time-varying correlations, regime changes

---

## Revision Summary

| Asset | Return | Risk | Liquidity | Best For |
|---|---|---|---|---|
| Stocks | High | High | High | Long-term growth |
| ETFs | Market | Market | High | Diversified exposure |
| Mutual Funds | Varies | Varies | Daily | Professional management |
| Bonds | Low-Med | Low-Med | Medium | Income, stability |
| Gold | Low-Med | Medium | High | Inflation hedge |
| REITs | Medium | Medium | High | Real estate income |
| Crypto | Very High | Very High | High | Speculation |

- Higher return potential = higher risk
- Correlation between assets determines true diversification
- Consider fees, liquidity, and tax treatment
- No single asset class is "best" — they serve different purposes

---

← [07-market-cycles](07-market-cycles.md) • [↑ Phase 1](README.md) • [↑ Finance](../README.md) • [09-price-movement](09-price-movement.md) →
