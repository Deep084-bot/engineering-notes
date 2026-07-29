# Learning Roadmap

> How the phases connect. Use this to plan your learning path.

## Phase Dependency Graph

```mermaid
graph TD
    P1[Phase 1<br/>Foundations] --> P2[Phase 2<br/>Financial Statements]
    P1 --> P4[Phase 4<br/>Technical Analysis]
    P2 --> P3[Phase 3<br/>Fundamental Analysis]
    P2 --> P5[Phase 5<br/>Financial Metrics]
    P3 --> P6[Phase 6<br/>Quantitative Finance]
    P4 --> P6
    P3 --> P7[Phase 7<br/>AI in Finance]
    P5 --> P7
    P6 --> P7
    P3 --> P8[Phase 8<br/>Portfolio & Risk]
    P5 --> P8
    P6 --> P8
    P7 --> P8
```

## Reading Flow

```mermaid
flowchart LR
    START([Start Here]) --> P1
    
    P1[Phase 1<br/>Foundations]
    
    P1 --> P2[Phase 2<br/>Financial Statements]
    P1 --> P4[Phase 4<br/>Technical Analysis]
    
    P2 --> P3[Phase 3<br/>Fundamental Analysis]
    P2 --> P5[Phase 5<br/>Financial Metrics]
    
    P3 --> P6[Phase 6<br/>Quantitative Finance]
    P5 --> P6
    
    P3 --> P7[Phase 7<br/>AI in Finance]
    P5 --> P7
    P6 --> P7
    
    P3 --> P8[Phase 8<br/>Portfolio & Risk]
    P5 --> P8
    P6 --> P8
    P7 --> P8
    
    P8 --> DONE([Done])
```

## Recommended Paths

### Full Track (everything)

```
P1 → P2 → P3 → P4 → P5 → P6 → P7 → P8
```

### Fundamental Analysis Track

```
P1 → P2 → P3 → P5 → P8
```

### Technical / Quant Track

```
P1 → P2 → P4 → P6 → P8
```

### AI / ML Track

```
P1 → P2 → P3 → P5 → P7
```

## Phase Descriptions

```mermaid
mindmap
  root((Finance))
    P1 Foundations
      Core concepts
      Market structure
      Trading mechanics
      Data sources
    P2 Financial Statements
      Balance Sheet
      Income Statement
      Cash Flow
      Reading reports
    P3 Fundamental Analysis
      Valuation ratios
      DCF modeling
      Comparable analysis
      Moat assessment
    P4 Technical Analysis
      Chart patterns
      Indicators
      Trends
      Volume analysis
    P5 Financial Metrics
      SaaS metrics
      Unit economics
      KPIs
      Business terminology
    P6 Quantitative Finance
      Probability
      Derivatives
      Pricing models
      Risk modeling
    P7 AI in Finance
      ML for markets
      NLP for filings
      LLM agents
      Alternative data
    P8 Portfolio & Risk
      Asset allocation
      Diversification
      Risk metrics
      Rebalancing
```

## Phase Details

| # | Phase | Prerequisites | Outcome |
|---|---|---|---|
| 1 | Foundations | None | Mental model of markets |
| 2 | Financial Statements | Phase 1 | Read financial reports |
| 3 | Fundamental Analysis | Phase 2 | Value a company |
| 4 | Technical Analysis | Phase 1 | Read charts and trends |
| 5 | Financial Metrics | Phase 2 | Understand business KPIs |
| 6 | Quantitative Finance | Phase 3 or 4 | Build pricing models |
| 7 | AI in Finance | Phase 3, 5 | Build financial AI systems |
| 8 | Portfolio & Risk | Phase 3, 5, 6 | Manage investment portfolios |

## How to Use This

1. Start with **Phase 1** regardless of your goal
2. Use the dependency graph to decide what to study next
3. Follow a track that matches your interest (fundamentals, quant, AI)
4. Each phase builds on previous knowledge — don't skip prerequisites
