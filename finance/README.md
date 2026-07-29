# Finance

> Engineering reference notes for software engineers learning finance.
> No MBA jargon. No textbooks. First principles, programming analogies, and systems thinking.

## Why

Finance is the second operating system of the modern world. Every software engineer building in FinTech, trading, AI, or e-commerce eventually needs it.

These notes teach finance the way engineers learn: concepts → analogies → examples → systems.

## Prerequisites

- **Finance**: None. Zero knowledge assumed.
- **Programming**: Strong background assumed.

## Repository Philosophy

```
Every concept →
  First principles definition +
  Programming analogy +
  Real example +
  Systems perspective
```

- No history lessons
- No motivational text
- No fluff
- Bullets over paragraphs
- Tables over prose
- Diagrams over descriptions

## Learning Path

The handbook is organized into **phases**. Each phase builds on the previous.

| # | Phase | Covers |
|---|---|---|
| 1 | [Foundations](phase-1-foundations/) | Core concepts: markets, trading, participants, data |
| 2 | Financial Statements | Balance sheet, income statement, cash flow |
| 3 | Fundamental Analysis | Ratios, valuation, DCF, comparable analysis |
| 4 | Technical Analysis | Charts, patterns, indicators (RSI, MACD, moving averages) |
| 5 | Financial Metrics & Business Terminology | KPIs, unit economics, SaaS metrics |
| 6 | Quantitative Finance | Math, statistics, derivatives, pricing models |
| 7 | AI in Finance | ML models, NLP, LLMs for financial analysis |
| 8 | Portfolio & Risk Management | Diversification, asset allocation, risk metrics |

## Folder Structure

```
finance/
├── README.md                  ← You are here
├── glossary.md                ← All terms across all phases
├── learning-roadmap.md        ← Dependency graph, mind map
│
├── phase-1-foundations/       ← Complete
├── phase-2-financial-statements/
├── phase-3-fundamental-analysis/
├── phase-4-technical-analysis/
├── phase-5-financial-metrics/
├── phase-6-quantitative-finance/
├── phase-7-ai-in-finance/
└── phase-8-portfolio-risk-management/
```

Each phase folder contains:
- `README.md` — phase overview, prerequisites, outcomes
- Numbered topic files — one concept per file
- Navigation links between files

## Reading Order

### Phase 1

```
00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11 → 12
```

### Across Phases

```
Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7 → Phase 8
```

Phase 1 is self-contained. All later phases depend on Phase 1.

## How to Use These Notes

- **First pass**: Read each phase in order. Start with Phase 1.
- **Reference**: Jump to any file when you need a concept explained. Use the glossary for term lookup.
- **Interview prep**: Focus on the **Interview Notes** sections in each file.
- **Deep dive**: Follow the recommended books and resources below.

## Conventions

- **Bold** for key terms
- `Code` for math, formulas, ticker symbols
- Tables for comparisons
- Mermaid for workflows and architectures
- Bullets over paragraphs

## Navigation

Every file includes navigation at the bottom:

```
← Previous • ↑ Phase README • ↑ Finance README • Next →
```

Use this to move through the material linearly or jump to index pages.

## Recommended Books

| Title | Author | Best For |
|---|---|---|
| **The Intelligent Investor** | Benjamin Graham | Value investing philosophy |
| **A Random Walk Down Wall Street** | Burton Malkiel | Market efficiency, passive investing |
| **The Little Book That Beats the Market** | Joel Greenblatt | Simple quantitative strategy |
| **Fooled by Randomness** | Nassim Taleb | Probability, luck, and markets |
| **Options, Futures, and Other Derivatives** | John Hull | Derivatives pricing (advanced) |
| **Python for Finance** | Yves Hilpisch | Applying programming to finance |
| **Flash Boys** | Michael Lewis | HFT, market structure |

## Recommended Websites

| Site | Use |
|---|---|
| [Investopedia](https://www.investopedia.com/) | Quick definition lookup |
| [SEC EDGAR](https://www.sec.gov/edgar) | US company filings (primary source) |
| [Yahoo Finance](https://finance.yahoo.com/) | Quick price/symbol lookup |
| [TradingView](https://www.tradingview.com/) | Charts and technical analysis |
| [Macrotrends](https://www.macrotrends.net/) | Historical data, charts |
| [NSE India](https://www.nseindia.com/) | Indian market data |
| [BSE India](https://www.bseindia.com/) | Indian market data |

## Recommended YouTube Channels

| Channel | Focus |
|---|---|
| [Patrick Boyle](https://www.youtube.com/@patrickboyle) | Finance stories, hedge funds, scandals |
| [Plain Bagel](https://www.youtube.com/@ThePlainBagel) | Explains finance concepts clearly |
| [Ben Felix](https://www.youtube.com/@BenFelixCSI) | Evidence-based investing |
| [Rational Reminder](https://www.youtube.com/@RationalReminder) | Deep finance discussions |
| [Aswath Damodaran](https://www.youtube.com/@DamodaranOnline) | Valuation (the best in the world) |
| [Cameron Stewart](https://www.youtube.com/@CameronStewartCFA) | CFA-level content, practical |
| [3Blue1Brown](https://www.youtube.com/@3blue1brown) | Math intuition (useful for quant) |

## Related

- [Operating Systems](/os/)
- [Computer Networks](/networks/)
- [System Design](/system-design/)
- [Machine Learning](/ml/)
