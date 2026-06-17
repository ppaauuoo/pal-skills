---
name: momentum-finder
description: >
  Find momentum stocks currently in a dip with 10-20%+ upside potential in
  1 month. Optionally match the pattern of a reference ticker (e.g. "find
  stocks like SNDK"). Covers multi-sector search, pattern matching, catalyst
  identification, and risk summary. Trigger when user asks to "find momentum
  stocks", "find dip stocks", "stocks like [TICKER]", "find me stocks
  similar to [TICKER]", or asks for 10-20% upside plays.
---

# Momentum Finder Skill

Find dip-buying opportunities in momentum stocks — single query or pattern-match to a reference ticker.

## Two Modes

### Mode A — General Scan
User asks: "find momentum stocks in a dip" / "what stocks could go up 10-20% in a month"

### Mode B — Pattern Match
User asks: "find stocks like [TICKER]" / "similar pattern to [TICKER]"

---

## Step 1 — Decode the Reference Ticker (Mode B only)

Before searching for matches, fully characterize the reference stock across 5 dimensions:

| Dimension | What to extract |
|---|---|
| **Rally profile** | Total % gain, timeframe, from what base (spin-off / IPO / 52-week low) |
| **Chart pattern** | Ascending channel / parabolic / base-and-break / cup-and-handle |
| **Narrative re-rating** | What story caused the market to re-price it? (AI infrastructure / supercycle / spin-off unlock) |
| **Fundamental driver** | Revenue growth %, margin expansion, backlog, contracted supply |
| **Dip behavior** | How it dips: RSI extreme → breathe → resume / macro selloff / sector rotation |

Search queries to run:
```
"[TICKER] stock pattern chart [YEAR] technical analysis"
"[TICKER] stock momentum breakout ascending channel [YEAR]"
"[TICKER] fundamental story revenue growth backlog"
```

---

## Step 2 — Scan for Momentum Stocks in Dips

### For General Scan (Mode A)
Run 3 search angles in parallel:
1. `"momentum stocks pullback dip buy opportunity [MONTH YEAR]"`
2. `"high momentum stocks 10-20% upside potential [YEAR] RSI oversold"`
3. `"RSI oversold momentum stocks bounce candidates [MONTH YEAR]"`

### For Pattern Match (Mode B)
Run 2 additional targeted searches:
1. `"stocks similar to [TICKER] [SECTOR] parabolic rally pattern [YEAR]"`
2. `"[TICKER] pattern match [narrative keyword] momentum [YEAR]"` — use the narrative keyword extracted in Step 1 (e.g. "AI storage supercycle", "nuclear power", "data center infrastructure")

---

## Step 3 — Sector Diversification Rule

**Always search across at least 3 different sectors.**

If the reference ticker (Mode B) is in a specific sector, explicitly exclude that sector and search others:

Priority sectors to check:
- **Industrials** — AI power equipment, construction, defense
- **Energy / Nuclear** — AI data center power demand, uranium supercycle
- **Utilities** — Nuclear baseload operators with long-term PPAs
- **Construction / Infrastructure** — Data center builders, grid modernization
- **Software / SaaS** — AI platform re-ratings
- **Healthcare / Biotech** — Catalyst-driven bounces
- **Financials** — Rate-sensitive momentum plays

Search query template for sector sweep:
```
"[sector] stocks parabolic rally AI infrastructure dip buy [YEAR]"
"[sector] stocks ascending channel breakout pullback opportunity [YEAR]"
```

---

## Step 4 — Pattern Match Scoring (Mode B)

For each candidate, score against the reference ticker's 5 dimensions:

| Dimension | Weight | Score 1-5 |
|---|---|---|
| Rally profile similarity | High | % move size, timeframe, base type |
| Chart pattern match | High | Same channel / breakout shape |
| Narrative re-rating match | Highest | Same "not cyclical anymore" story |
| Fundamental driver match | High | Revenue growth, backlog, contracts |
| Dip behavior match | Medium | How it pulls back and resumes |

Only surface candidates scoring ⭐⭐⭐ or higher (3+/5 dimensions matched).

---

## Step 5 — Catalyst Check

For each shortlisted stock, identify:
- **Near-term catalyst** (earnings date, product launch, macro event, index inclusion)
- **Macro tailwind** (AI capex, rate cut expectations, commodity cycle)
- **Technical setup** (support level, EMA convergence, RSI mean reversion zone)
- **Risk factors** (valuation stretch, rate sensitivity, sector rotation risk)

---

## Step 6 — Output Format

Always deliver in this structure:

### 1. Pattern Definition (Mode B only)
3-5 bullet summary of what the reference ticker's pattern IS — so the user understands what they're looking for.

### 2. Candidate Table
| Ticker | Sector | Pattern Match ⭐ | Rally | Current Dip | Upside Est. | Catalyst | Risk |
|---|---|---|---|---|---|---|---|

### 3. Per-Candidate Deep Dive
For each ⭐⭐⭐+ candidate:
- **Why it dipped** (1 line — be specific, not "macro concerns")
- **Why it bounces** (1-2 lines — fundamental + technical reason)
- **The AI/supercycle/re-rating narrative** (what story are investors pricing in?)
- **Backlog/contracts/visibility** (locked-in revenue, not spot-market dependent)
- **Catalyst** + date if known
- **Upside target** with basis (analyst target / technical level / fair value)

### 4. Risk Summary
One paragraph on macro context: what kills these setups (rate moves, commodity reversal, AI spend slowdown, geopolitical events).

### 5. Disclaimer
Always append: *"Not financial advice. Momentum setups carry reversal risk. Use position sizing and stop losses."*

---

## Key Pattern Signals to Always Check

| Signal | Bullish | Bearish |
|---|---|---|
| RSI | 40-60 after pullback from 70-99 | Still above 80 after correction |
| Moving averages | Price at / just above 100-day EMA | Price below 200-day EMA |
| Backlog / contracts | Multi-year locked-in revenue | Spot-market dependent |
| Volume on dip | Declining (healthy pullback) | Spiking (distribution) |
| Analyst revisions | Estimates going UP | Estimates being cut |
| Options activity | Unusual CALL buying | Heavy PUT buying |

---

## Web Search Best Practices

- Always use **3 parallel search angles** — vary phrasing, not just the query
- Include current **month + year** in all queries for recency
- Fetch full content from the 2-3 highest-signal sources
- Cross-reference: technical analysis + fundamentals + analyst targets
- For pattern match: find the source that most clearly describes the reference ticker's chart/narrative, then use those exact keywords to find similar stocks

---

## Example Invocations

> "find me momentum stocks in a dip that could go up 10-20% in a month"
→ Mode A: 3-sector general scan, dip filter, catalyst check, full output

> "find stocks with a similar pattern to SNDK"
→ Mode B: decode SNDK (parabolic ascending channel + AI storage supercycle + contracted backlog), search 3+ non-memory sectors, score + surface top matches

> "same pattern as NVDA but not tech"
→ Mode B: decode NVDA pattern, exclude Technology sector, sweep Industrials / Energy / Infrastructure / Healthcare
