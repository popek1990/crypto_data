---
source_name: Situation Room
source_url: https://situationroom.space/
source_type: Bitcoin and global macro intelligence dashboard
language: English
review_date: 2026-04-25
access_model: Web dashboard; free and paid tiers; no public API documentation identified during review
status: Candidate data source / metric taxonomy reference
---

# Situation Room — Data Inventory

## 1. Short Description

Situation Room is a Bitcoin and global macro intelligence dashboard. It combines Bitcoin market data, Bitcoin network metrics, mining data, Lightning Network metrics, on-chain indicators, macro market data, geopolitical/event intelligence, and AI-generated briefings.

For this knowledge base, treat Situation Room primarily as a **dashboard and metric taxonomy source** unless a documented API or export mechanism is later confirmed.

## 2. Access Notes

| Area | Access Status | Notes |
|---|---:|---|
| Public dashboard | Available | Shows core Bitcoin, macro, map, and briefing sections. Some values are dynamically loaded. |
| Briefing archive | Restricted | Archive access requires sign-in. |
| Advanced dashboards/tools | Tiered | Some tools and deeper analysis are available only on paid tiers. |
| Public API | Not confirmed | No public API documentation was visible from the reviewed pages. |

## 3. Main Data Categories

### 3.1 Bitcoin Market Data

| Data Point | Description |
|---|---|
| BTC price / 24h movement | Live or near-live Bitcoin price and short-term movement. |
| 7-day performance | Bitcoin performance over the last 7 days. |
| 30-day performance | Bitcoin performance over the last 30 days. |
| Market capitalization | Bitcoin market cap. |
| 24h trading volume | Bitcoin trading volume over the last 24 hours. |
| Supply | Bitcoin circulating or available supply metric. |
| ATH | Bitcoin all-time high reference. |
| Distance from ATH | Percentage or value difference from all-time high. |

### 3.2 Bitcoin Network Data

| Data Point | Description |
|---|---|
| Block height | Current Bitcoin blockchain height. |
| Transaction fees — fast | Estimated fee level for faster confirmation. |
| Transaction fees — medium | Estimated fee level for medium-priority confirmation. |
| Transaction fees — slow | Estimated fee level for lower-priority confirmation. |
| Mempool | Current transaction backlog / mempool state. |
| Hashrate | Bitcoin network hashrate. |
| Difficulty | Current Bitcoin mining difficulty. |
| Next halving | Estimated timing or block distance to the next Bitcoin halving. |

### 3.3 Bitcoin Mining Data

| Data Point | Description |
|---|---|
| Block subsidy | Current Bitcoin block subsidy. |
| Blocks today | Number of blocks mined during the current day. |
| Average block time | Average time between blocks. |
| Daily mining revenue | Estimated daily miner revenue. |
| Revenue per TH | Miner revenue per terahash. |
| Difficulty epoch | Current difficulty adjustment epoch. |
| Retarget countdown | Estimated time or blocks until the next difficulty adjustment. |

### 3.4 Lightning Network Data

| Data Point | Description |
|---|---|
| Channels | Total Lightning Network channels. |
| Capacity | Total Lightning Network BTC capacity. |
| Nodes | Number of Lightning Network nodes. |
| Average channel size | Average BTC capacity per channel. |
| Median base fee | Median routing base fee. |
| Median fee rate | Median routing fee rate. |

### 3.5 On-Chain and Sentiment Data

| Data Point | Description |
|---|---|
| Fear & Greed | Market sentiment indicator. |
| MVRV ratio | Market value to realized value ratio. |
| MVRV Z-score | Valuation indicator shown as a 90-day chart. |
| Exchange inflow | BTC moving into exchanges. |
| Exchange outflow | BTC moving out of exchanges. |
| Net flow | Net exchange flow. |
| Exchange balance | BTC balance held on exchanges. |
| Whale transactions >100 BTC | Large Bitcoin transaction activity. |
| Mempool scan | Transaction backlog / pending activity signal. |

### 3.6 Bitcoin Macro Conviction Score

The site includes a composite **Bitcoin Macro Conviction Score**. It is described as a combined indicator using five macro-related signals and updating every 60 seconds.

| Signal | Description |
|---|---|
| Sentiment | Market sentiment component. |
| Momentum | Price or trend momentum component. |
| Valuation | Valuation-related component, likely tied to metrics such as MVRV. |
| Network | Bitcoin network health/activity component. |
| Monetary | Macro/liquidity/monetary conditions component. |

### 3.7 Global Situation Map

The dashboard includes a global map combining event intelligence and country-level macro/social overlays.

#### Event Categories

| Category | Description |
|---|---|
| Bitcoin | Bitcoin-related events or signals. |
| Conflict | War, military, or geopolitical conflict events. |
| Disaster | Natural disaster or crisis events. |
| Economy | Economic events and macro developments. |
| Political | Political events and governance-related developments. |

#### Map Layers / Country Indicators

| Layer | Description |
|---|---|
| GDP per capita | Country-level economic output per person. |
| Debt to GDP | Government debt relative to GDP. |
| Inflation rate | Country-level inflation data. |
| Unemployment | Country-level unemployment data. |
| Economic freedom | Economic freedom score/index. |
| Income inequality / Gini | Income inequality metric. |
| Life expectancy | Population health / longevity indicator. |
| Human Development Index | HDI-style development indicator. |
| Homicide rate | Country-level violence/safety metric. |
| Corruption index | Corruption perception or governance quality metric. |
| Peace index | Country-level peace/stability metric. |
| Press freedom | Press freedom / media freedom metric. |

### 3.8 Macro Market Data

| Asset / Index | Category |
|---|---|
| S&P 500 | US equity index |
| Nasdaq | US technology/growth equity index |
| Dow Jones | US equity index |
| FTSE 100 | UK equity index |
| DAX | German equity index |
| Nikkei 225 | Japanese equity index |
| Hang Seng | Hong Kong equity index |
| VIX | Volatility index |

### 3.9 Commodities, Rates, DXY and FX

| Data Point | Category |
|---|---|
| Gold | Commodity / monetary asset |
| Silver | Commodity / monetary metal |
| Crude oil | Energy commodity |
| Natural gas | Energy commodity |
| Copper | Industrial commodity |
| DXY | US Dollar Index |
| US 10Y Yield | US Treasury yield |
| US 2Y Yield | US Treasury yield |
| EUR/USD | FX pair |
| GBP/USD | FX pair |
| USD/JPY | FX pair |
| USD/CNY | FX pair |
| BTC / Gold oz | Bitcoin value expressed relative to gold ounces |

### 3.10 Central Bank and Macro Watch

| Data Area | Description |
|---|---|
| Central bank rates | Dashboard section for central bank rate monitoring. |
| Macro data | Broader macro indicators are included in the free tier description. |
| Macro Focus dashboard | Paid-tier macro dashboard. |
| Macro Cycle tool | Paid-tier tool referencing ISM PMI tracking and a macro-dominoes framework. |
| Global Liquidity tool | Paid-tier tool referencing a 21-economy M2 aggregate with an 84-day Bitcoin lead. |

### 3.11 Intelligence Briefings and News

| Feature | Description |
|---|---|
| Intelligence briefing | Filterable briefing section covering Bitcoin, conflict, disaster, economy, and political categories. |
| AI briefing | AI-generated intelligence summary. |
| Briefing archive | Archive of past briefings; sign-in required for access. |
| Wire | Headline/news feed section. |
| Weekly newsletter digest | Newsletter available in the free tier. |
| Full 5-agent briefings | Paid-tier briefing system covering Market, Network, Geopolitical, Macro, and Outlook. |
| AI analysis dashboards | Paid-tier AI analysis; higher tiers mention 8 specialist agents and on-demand analysis. |

## 4. Tools and Advanced Features Mentioned

| Tool / Feature | Access Tier Mentioned | Description |
|---|---:|---|
| DCA Signal | General | Weekly composite score, recommended buy signal, and 12-month chart. |
| BTC stacking comparison | General | Compares signal-based stacking vs vanilla DCA. |
| Trading Desk | Members | Autonomous trading terminal. |
| Ops Room | Members | Live member chat via Nostr. |
| Trading pool view | Members | Capital flow topology / trading pool view. |
| On-Chain Deep Dive | Members | Advanced on-chain dashboard. |
| Macro Cycle | Members | ISM PMI tracker and macro-cycle framework. |
| Global Liquidity | Members | 21-economy M2 sum with an 84-day Bitcoin lead. |
| UTXO Cosmography | Tool page visible | Bitcoin UTXO-oriented visualization/tool page listed. |
| TXID Pathing | Tool page visible | Transaction ID pathing / tracing tool page listed. |
| Blockchain | General | Blockchain-focused tool page listed. |
| DCA Exit Strategy | VIP | Signal-timed profit-taking strategy on excess BTC. |
| Combined portfolio chart | VIP | Buy + exit simulation with convergence. |
| Threshold alerts | VIP | Alerts via Nostr DM. |

## 5. Suggested Classification for the GitHub Knowledge Base

| Field | Suggested Value |
|---|---|
| Domain | Bitcoin, macro, on-chain, geopolitical intelligence |
| Best use | Metric discovery, dashboard reference, market-intelligence taxonomy |
| Data granularity | Mixed: live dashboard metrics, daily/weekly briefings, charts, event feeds |
| API availability | Unknown / not documented publicly |
| Reliability check needed | Yes — identify original upstream providers before using in production |
| Paid access risk | Medium to high — several useful features are behind sign-in or subscription tiers |

## 6. Integration Notes

- Do not treat Situation Room as a primary machine-readable data source until an API, export, or documented data-access method is confirmed.
- Use the visible dashboard structure as a checklist for data categories to collect from primary providers.
- For production use, map each metric to upstream sources such as market-data APIs, Bitcoin node/mempool data, Lightning Network explorers, macroeconomic databases, central bank data, and geopolitical/news APIs.
- Track access tier requirements because several advanced datasets and tools appear to be gated.

## 7. Candidate Tags

`bitcoin` `btc` `macro` `on-chain` `mining` `lightning-network` `mempool` `hashrate` `mvrv` `exchange-flows` `fear-and-greed` `global-liquidity` `central-banks` `geopolitics` `market-intelligence` `dashboard` `paid-access`
