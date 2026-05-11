# Coinglass API Reference (v4.0)

> **Last sync:** 2026-05-08 z `docs.coinglass.com` (kanon) + `coinglass.com/pricing` (plan tiers) + `coinglass.com/CryptoApi` (overview).

## Spis treści

2. [Konfiguracja API](#2-konfiguracja-api)
3. [Konwencje](#3-konwencje)
4. [REST Endpointy](#4-rest-endpointy)
   - 4.1 [General / Foundation](#41-general--foundation)
   - 4.2 [Futures](#42-futures)
   - 4.3 [Spot](#43-spot)
   - 4.4 [Options](#44-options)
   - 4.5 [ETF](#45-etf)
   - 4.6 [On-chain](#46-on-chain)
   - 4.7 [Hyperliquid](#47-hyperliquid)
   - 4.8 [Indicators](#48-indicators)
   - 4.9 [Economic Data](#49-economic-data)
   - 4.10 [Order Book L2/L3](#410-order-book-l2l3)
   - 4.11 [Anomaly / Alert (Phase 1 priorytet 4)](#411-anomaly--alert)
5. [WebSocket Streams](#5-websocket-streams)
6. [Mapowanie d3pth ecosystem](#6-mapowanie-d3pth-ecosystem)
7. [Operational guidance](#7-operational-guidance)
8. [Changelog źródła](#8-changelog-źródła)
9. [Errata / MS1 bootstrap notes](#9-errata--ms1-bootstrap-notes)

---

## 2. Konfiguracja API

### 2.1 Base URL i wersja

| | |
|---|---|
| **Base URL** | `https://open-api-v4.coinglass.com/` |
| **Wersja** | v4.0 (jedyna wspierana — wszystkie wcześniejsze wersje deprecated) |
| **Format** | REST + JSON |
| **WebSocket** | `wss://open-api-v4.coinglass.com/ws` |
| **Dokumentacja kanon** | [docs.coinglass.com/reference](https://docs.coinglass.com/reference) |
| **Mirror fallback** | [coinglass.readme.io/reference/api-new-version](https://coinglass.readme.io/reference/api-new-version) |

### 1.2 Authentication

Wszystkie endpointy wymagają nagłówka `CG-API-KEY` z kluczem użytkownika.

```http
GET /api/futures/openInterest/history?symbol=BTC&interval=1h HTTP/1.1
Host: open-api-v4.coinglass.com
accept: application/json
CG-API-KEY: <YOUR_API_KEY>
```

Brak klucza lub zły klucz → odrzucone z błędem auth. **Klucza nie commitować** — w repo tylko placeholder w `.env.example`.

Źródło: [docs.coinglass.com/reference/authentication](https://docs.coinglass.com/reference/authentication).

### 2.3 Plan tiers

Stan na **2026-05-08** (źródło: [coinglass.com/pricing](https://www.coinglass.com/pricing)). Pricing zmienny — przed bootstrapem MS1 sprawdzić aktualne ceny.

| Plan | Cena/msc | Cena/rok | RPM | Endpoint count | Commercial use | Uwagi |
|------|----------|----------|-----|----------------|----------------|-------|
| **Hobbyist** | $29 | $278 | 30 | 80+ | NIE | Personal use only |
| **Startup** | **$79** | $790 | **80** | **130+** | NIE | **Wybrany dla d3pth/Hydra** (decyzja Orkiestratora 2026-05-07) |
| **Standard** | $299 | $2,870 | 300 | 150+ | TAK | Pierwsza opcja produkcyjna; L2/L3 OB tier |
| **Professional** | $699 | $6,710 | 1200 | 160+ | TAK | Najwyższy tier publiczny |
| **Enterprise** | by inquiry | by inquiry | custom | custom | TAK | Custom data granularity, history range |

**Decyzja MS1:** Startup ($79). 80 RPM trzeba dzielić między 9 priorytetowych endpointów (`coinglass_service_todo.md` Faza 1) — patrz sekcja 7.1 "Token bucket strategy".

### 2.4 Cache TTL strategy w coinglass-service

Każdy endpoint MA przypisany TTL dla cache w MS1. Cele:
1. Oszczędzić płatne calls do CG (Startup 80 RPM = max 115k req/dzień, ale realnie część tego rezerwujemy na anomaly detection)
2. Akceptowalna freshness dla każdego typu danych

Tabela TTL per kategoria (proponowana, do walidacji przy bootstrapie MS1):

| Kategoria | TTL cache | Uzasadnienie |
|-----------|-----------|--------------|
| supported-coins / supported-exchanges | 24h | Lista zmienia się rzadko |
| coins-markets / pairs-markets bulk | **5 min** | Ranking symboli — refresh co poll Hydry |
| OI history (interval=1h+) | 5 min | Historia nie zmienia closed candle |
| OI history (interval=1m, 5m) | 1 min | Live candles |
| Funding rate history | 15 min | Funding zmienia się co 8h na większości giełd |
| Funding rate live | 1 min | Real-time monitoring |
| Liquidation order (raw stream) | NO CACHE | Stream events — każdy unique |
| Liquidation aggregated history | 5 min | Agregat |
| Liquidation aggregated map | **5 min** | Levels stabilne; refresh dla UI |
| Liquidation max-pain | 5 min | Pojedynczy punkt |
| Long/Short ratio | 5 min | Często konsumowane |
| Taker buy/sell | 5 min | |
| Pair Taker Buy/Sell history | 5 min | |
| Options OI / max-pain | 30s | Wskazana przez CG (cache 30s wszystkie tiery) |
| ETF AUM / flows history | 1h | Daily snapshot wystarczy |
| On-chain exchange-balance | 30 min | Slow-moving |
| Hyperliquid wallet positions | 5 min | |
| Hyperliquid whale alerts | 1 min | Real-time alerty |
| Indicators (Pi Cycle, F&G) | 1h | Daily |
| Economic data | 24h | Calendar |
| Order Book L2/L3 | NO CACHE | Real-time |

---

## 3. Konwencje

### 3.1 Format symbolu

- **Coin-level** (większość endpointów): `BTC`, `ETH`, `SOL` — bez sufiksu
- **Pair-level** (np. `pairs-markets`, `liquidation-order`): `BTCUSDT_BinanceFutures` (format `<symbol>_<exchange>`)

### 3.2 Format exchange ID

Pojedyncze słowa, kapitalizacja zachowana (jak w `supported-exchanges`):

`OKX, Binance, HTX, Bitmex, Bitfinex, Bybit, Deribit, Gate, Kraken, KuCoin, CME, Bitget, dYdX, CoinEx, BingX, Coinbase, Gemini, Crypto.com, Hyperliquid, Bitunix, MEXC, WhiteBIT, Aster, Lighter, EdgeX, Drift, Paradex, Extended, ApeX Omni`

29 giełd — patrz [`/api/futures/supported-exchanges`](https://docs.coinglass.com/reference/supported-exchanges).

### 3.3 Format timestamp

- **Request:** Unix milliseconds (int64), np. `startTime=1746729600000`
- **Response:** Unix milliseconds (int64) w polu `time` lub `t`

### 3.4 Pagination

CG nie używa cursor-based pagination dla większości endpointów. Limity:
- `aggregated-liquidation-history` i podobne time-series: parametr `limit` (max ~1500 punktów)
- `liquidation-order` (raw): parametr `limit` (max kilka tysięcy)

Dla większych zakresów: dziel po `startTime` / `endTime`.

### 3.5 Error response format

Wszystkie błędy zwracają JSON z `code` i `msg`:

```json
{
  "code": "30001",
  "msg": "API key invalid",
  "data": null
}
```

`code = "0"` → success. Pełna lista kodów: [docs.coinglass.com/reference/responses-error-codes](https://docs.coinglass.com/reference/responses-error-codes).

`TODO: requires manual verification` — pełna mapa error codes (30001 auth, 30002 rate limit, 30003 plan-not-allowed, 40001 bad params...) niedostępna w snippetach WebSearch.

### 3.6 Standard HTTP status codes

| Status | Znaczenie | Akcja w MS1 |
|--------|-----------|-------------|
| 200 | OK | Parse data |
| 401 | Auth fail | Alert ops, NIE retry |
| 403 | Plan not allowed | Log + propagate jako "endpoint unavailable" |
| 429 | Rate limit | Token bucket backoff (exponential, max 30s) |
| 500-503 | Upstream | Circuit breaker (5× → open 30s) |

---

## 4. REST Endpointy

### Konwencja oznaczeń

| Symbol | Znaczenie |
|--------|-----------|
| ✅ | MS1 Phase 1 (priorytet wysoki, refresh < 15 min) |
| 🟡 | MS1 Phase 0 (foundation/discovery) |
| 🟢 | MS1 Phase 1 candidate (verify schema first) |
| 🔵 | MS1 Phase 2 (deferred, post-bootstrap) |
| ⚪ | OUT — nie planujemy konsumować w MS1 (link do CG docs jako reference) |
| 📡 | d3pth direct consumer (przez MS1) |
| 🐍 | Hydra direct consumer (przez MS1) |

**Format A (FULL)** — pełny opis dla 🟡 + ✅ + 🟢 + 📡 + 🐍 (~30-40 endpointów).
**Format B (INDEX)** — tabela w sekcji 6 dla pozostałych ~120-130 endpointów.

---

### 4.1 General / Foundation

#### 🟡 `GET /api/futures/supported-coins`

**Status:** MS1 Phase 0 (Foundation — fundament wszystkich późniejszych calls)
**Plan tier:** Hobbyist+
**Cache TTL w MS1:** 24h
**CG docs:** [coins](https://docs.coinglass.com/reference/coins)

**Path params:** brak
**Query params:** brak

**Response (truncated):**
```json
{
  "code": "0",
  "data": ["BTC", "ETH", "SOL", "BNB", "XRP", "ADA", ...]
}
```

**Operational note:** lista ~500+ symboli. Trzymaj w memory cache mikroserwisu. Refresh raz dziennie.
**Cross-link:** używane przez `coinglass-service` przy walidacji symboli na input innych endpointów.

---

#### 🟡 `GET /api/futures/supported-exchanges`

**Status:** MS1 Phase 0 (Foundation)
**Plan tier:** Hobbyist+
**Cache TTL w MS1:** 24h
**CG docs:** [supported-exchanges](https://docs.coinglass.com/reference/supported-exchanges)

**Response (truncated):**
```json
{
  "code": "0",
  "data": ["OKX", "Binance", "HTX", "Bitmex", "Bybit", "Deribit", "Hyperliquid", ...]
}
```

29 giełd potwierdzonych w audycie 2026-05-08 (sekcja 9.1).

---

#### 🟡 `GET /api/futures/supported-exchange-pairs`

**Status:** MS1 Phase 0 (Foundation)
**Plan tier:** Hobbyist+
**Cache TTL w MS1:** 24h
**CG docs:** [instruments](https://docs.coinglass.com/reference/instruments)

**Response:** mapa exchange → lista par. `TODO: requires manual verification` — dokładna struktura (czy `exchange: pairs[]` czy lista flat z `{symbol, exchange}`?) nie potwierdzona w snippetach.

---

#### ⚪ `GET /api/exchange-list`

**Status:** OUT (duplicate funkcjonalności `supported-exchanges`)
**CG docs:** [exchange-list](https://docs.coinglass.com/reference/exchange-list)

---

### 4.2 Futures

Największa kategoria — ~50 endpointów. FULL dokumentacja dla MS1 Phase 1 priorytetowych. INDEX dla reszty w sekcji 6.

---

#### 4.2.1 Markets bulk

##### ✅📡🐍 `GET /api/futures/coins-markets`

**Status:** MS1 Phase 1 (scheduled bulk, polling co ~5 min) — decyzja Sokoła 2026-05-08
**Plan tier:** Hobbyist+ (Startup confirmed)
**Cache TTL w MS1:** 5 min
**CG docs:** [coins-markets](https://docs.coinglass.com/reference/coins-markets)

**Per-coin (symbol-level)** performance metrics — agregat across all exchanges.

**Query params:** brak (zwraca cały universe)
**Response (truncated):**
```json
{
  "code": "0",
  "data": [
    {
      "symbol": "BTC",
      "currentPrice": 95430.5,
      "priceChangePercent24h": 1.23,
      "marketCap": 1890000000000,
      "openInterest": 25500000000,
      "fundingRate": 0.0001,
      "longShortRatio": 1.05
    },
    ...
  ]
}
```

**Operational note:** głowny bulk endpoint dla rankingu symboli (Hydra). Polluj co 5 min przez scheduler MS1.
**Konsumenci:** Hydra (Block C ranking), d3pth (`tab_scanner.py` może w przyszłości używać do crossy CG ↔ Binance).

---

##### ✅📡 `GET /api/futures/pairs-markets`

**Status:** MS1 Phase 1 (cached deep-dive on-demand, NIE bulk polling) — decyzja Sokoła 2026-05-08
**Plan tier:** Hobbyist+ (Startup confirmed)
**Cache TTL w MS1:** 5 min (per pair, lazy load)
**CG docs:** [pairs-markets](https://docs.coinglass.com/reference/pairs-markets)

**Per-trading-pair (exchange-level)** performance metrics — `BTCUSDT_BinanceFutures` granularity.

**Query params:**
| Nazwa | Typ | Wymagany | Opis |
|-------|-----|----------|------|
| `symbol` | string | TAK | np. `BTC` (zwraca wszystkie pary dla tego coin) |

**Response (truncated):**
```json
{
  "code": "0",
  "data": [
    {
      "instrumentId": "BTCUSDT_BinanceFutures",
      "exchange": "Binance",
      "currentPrice": 95430.5,
      "indexPrice": 95425.3,
      "priceChangePercent24h": 1.23,
      "volume24h": 18500000000,
      "openInterest": 8200000000,
      "liquidationLong24h": 12500000,
      "liquidationShort24h": 4300000
    },
    ...
  ]
}
```

**Operational note:** **NIE pollować całego universe** (zjada Startup 80 RPM). Lazy load per-symbol gdy d3pth/Hydra zażąda detail per coin. Cache per `symbol`, TTL 5 min.
**Errata #1 (sekcja 9.1):** to **różny endpoint** od `coins-markets`, nie zastępuje się.
**Konsumenci:** d3pth tab "Futures Pulse" deep-dive per coin.

---

##### 🔵 `GET /api/futures/coins-price-change`

**Status:** Phase 2 (deferred — duplikuje pole `priceChangePercent24h` z `coins-markets`)
**CG docs:** [coins-price-change](https://docs.coinglass.com/reference/coins-price-change)

---

#### 4.2.2 Open Interest

##### ✅📡🐍 `GET /api/futures/openInterest/ohlc-history`

**Status:** MS1 Phase 1 (priorytet 1 z `coinglass_service_todo.md`)
**Plan tier:** Hobbyist+
**Cache TTL w MS1:** 1 min (interval=1m, 5m), 5 min (1h+)
**CG docs:** [oi-ohlc-histroy](https://docs.coinglass.com/reference/oi-ohlc-histroy) (uwaga: literówka w slug `histroy` jest po stronie CG, ZACHOWANA)

**Query params:**
| Nazwa | Typ | Wymagany | Opis |
|-------|-----|----------|------|
| `symbol` | string | TAK | np. `BTC` |
| `exchange` | string | NIE | filtr giełdy |
| `interval` | string | TAK | `1m`, `5m`, `15m`, `1h`, `4h`, `1d`, `1w` |
| `startTime` | int64 | NIE | Unix ms |
| `endTime` | int64 | NIE | Unix ms |
| `limit` | int | NIE | max ~1500 |

**Response (truncated):**
```json
{
  "code": "0",
  "data": [
    { "time": 1746729600000, "open": 25400000000, "high": 25600000000, "low": 25380000000, "close": 25500000000 }
  ]
}
```

**Konsumenci:** Hydra (Block C — `oi_change_24h`, `oi_atr`), d3pth (`tab_futures_pulse.py` overlay).

---

##### ✅🐍 `GET /api/futures/openInterest/exchange-list`

**Status:** MS1 Phase 1
**Plan tier:** Hobbyist+
**Cache TTL w MS1:** 5 min
**CG docs:** [oi-exchange-list](https://docs.coinglass.com/reference/oi-exchange-list)

Per-exchange OI dominance. **Konsumenci:** Hydra (OI dominance ranking).

`TODO: requires manual verification` — dokładny response schema (czy zwraca `[{exchange, oi, dominance%}]`) nie potwierdzony w snippetach.

---

##### 🔵 `GET /api/futures/openInterest/exchange-history-chart`

**Status:** Phase 2 (chart visualization, agregat z `oi-ohlc-history` + `exchange-list`)
**CG docs:** [exchange-open-interest-history](https://docs.coinglass.com/reference/exchange-open-interest-history) — UWAGA: pomimo nazwy slug, dokumentacja może dotyczyć Options (potwierdzić przy implementacji)

`TODO: requires manual verification` — confusion w naming (futures vs options). Sprawdzić oba w docs.coinglass.com przy bootstrapie.

---

#### 4.2.3 Funding Rate

##### ✅🐍 `GET /api/futures/funding-rate/oi-weight-ohlc-history`

**Status:** MS1 Phase 1 (priorytet 1)
**Plan tier:** Hobbyist+
**Cache TTL w MS1:** 15 min
**CG docs:** [oi-weight-ohlc-history](https://docs.coinglass.com/reference/oi-weight-ohlc-history) (UWAGA: ten slug jest dla **funding rate** OI-weighted, nie open interest — confusing naming)

**Query params:**
| Nazwa | Typ | Wymagany | Opis |
|-------|-----|----------|------|
| `symbol` | string | TAK | np. `BTC` |
| `interval` | string | TAK | `1h`, `4h`, `1d` |
| `startTime` | int64 | NIE | |

**Response:** OHLC funding rate ważony OI per giełda. **Konsumenci:** Hydra (`funding_flip_count_7d`, Block C).

---

##### 🟡 `GET /api/futures/fundingRate/exchange-list`

**Status:** MS1 Phase 0 (Foundation discovery)
**Cache TTL w MS1:** 1h
**CG docs:** [fr-exchange-list](https://docs.coinglass.com/reference/fr-exchange-list)

Lista current funding rate per exchange + countdown do next funding. **Konsumenci:** d3pth tab "Futures Pulse Lite" (sekcja Funding).

---

#### 4.2.4 Long/Short Ratio

##### ✅🐍 `GET /api/futures/longShort/global-account-ratio`

**Status:** MS1 Phase 1 (priorytet 1 — Long/Short ratio global)
**Plan tier:** Hobbyist+
**Cache TTL w MS1:** 5 min
**CG docs:** [global-longshort-account-ratio](https://docs.coinglass.com/reference/global-longshort-account-ratio)

Long/short account ratio history dla pary na konkretnej giełdzie.

`TODO: requires manual verification` — dokładny path (`/longShort/...` vs `/long-short-ratio/...`) i query params (per-pair czy per-coin?).

---

##### ✅🐍 `GET /api/futures/longShort/top-account-ratio`

**Status:** MS1 Phase 1 (priorytet 2)
**CG docs:** [top-longshort-account-ratio](https://docs.coinglass.com/reference/top-longshort-account-ratio)

Long/short ratio TOP traders (top 20% by margin balance) — proportion of net long vs net short accounts. Sygnał institutional sentiment.

---

##### ✅🐍 `GET /api/futures/longShort/top-position-ratio`

**Status:** MS1 Phase 1 (priorytet 2)
**CG docs:** [top-longshort-position-ratio](https://docs.coinglass.com/reference/top-longshort-position-ratio)

Long/short position ratio TOP traders — actual position size, nie account count. Większy sygnał niż account-ratio.

---

##### 🔵 `GET /api/futures/net-position`

**Status:** Phase 2
**CG docs:** [net-position](https://docs.coinglass.com/reference/net-position)

Net long/short position aggregate.

---

#### 4.2.5 Liquidation (RODZINA 6 endpointów — kluczowa dla `plan_futures_pulse_coinglass.md`)

##### 🔵 `GET /api/futures/liquidation/order`

**Status:** Phase 1 priorytet 4 (anomaly pipeline) — decyzja Sokoła 2026-05-08
**Plan tier:** Hobbyist+
**Cache TTL w MS1:** NO CACHE (stream events, każdy unique)
**CG docs:** [liquidation-order](https://docs.coinglass.com/reference/liquidation-order)

Raw individual liquidation events. Polluj często (1 min) → buduj anomaly detector "kaskada likwidacji" (`coinglass_service_todo.md` Faza 1 priorytet 4).

**Konsumenci:** MS1 anomaly detector (NIE d3pth Liquidation Levels — to inny use-case).

---

##### 🔵 `GET /api/futures/liquidation/aggregated-history`

**Status:** Phase 1 priorytet 4 (anomaly time-series) — decyzja Sokoła 2026-05-08
**Cache TTL w MS1:** 5 min
**CG docs:** [aggregated-liquidation-history](https://docs.coinglass.com/reference/aggregated-liquidation-history)

Time-series long/short liquidations OHLC across exchanges. Służy do anomaly detection (skok long-liq w short interval = squeeze trigger).

---

##### ✅📡🐍 `GET /api/futures/liquidation/aggregated-map`

**Status:** **MS1 Phase 1 PRIMARY dla "Liquidation Levels"** — decyzja Sokoła 2026-05-08
**Plan tier:** Hobbyist+
**Cache TTL w MS1:** 5 min
**CG docs:** [liquidation-aggregated-map](https://docs.coinglass.com/reference/liquidation-aggregated-map)

**Cross-exchange aggregated liquidation map** — clusters per leverage level (100x, 50x, 25x, 10x, 5x) per kierunek (long/short).

**Query params:**
| Nazwa | Typ | Wymagany | Opis |
|-------|-----|----------|------|
| `symbol` | string | TAK | np. `BTC` |
| `range` | string | NIE | `1d`, `7d`, `30d` (zakres czasowy) |

**Response:** clusters z `price`, `leverage`, `side` (long/short), `density` lub `liquidation_volume`.

`TODO: requires manual verification` — dokładny response schema. Krytyczne dla `find_wall_liq_coincidences()` w `fetchers/coinglass.py` rewrite (`plan_futures_pulse_coinglass.md` Issue #1).

**Kontrakt MS1↔d3pth (proponowany):** MS1 wystawia `GET /api/v1/liquidations/{symbol}` zwracający normalizowane clusters w formacie:
```json
{
  "symbol": "BTC",
  "clusters": [
    { "price_usd": 92000, "side": "long", "leverage": 25, "liquidation_usd": 145000000 },
    ...
  ],
  "max_pain": 94500,
  "data_freshness": "fresh"
}
```

**Konsumenci:** d3pth `fetchers/coinglass.py` (rewrite ze STUB), Hydra (`liq_imbalance_pct`).

---

##### 🟢📡 `GET /api/futures/liquidation/aggregated-heatmap/model1`

**Status:** **MS1 Phase 1 candidate (verify response shape FIRST)** — decyzja Sokoła 2026-05-08
**Plan tier:** Hobbyist+ (potwierdzić — niektóre heatmapy są Standard+)
**Cache TTL w MS1:** 5 min
**CG docs:** [liquidation-aggregate-heatmap](https://docs.coinglass.com/reference/liquidation-aggregate-heatmap)
**Aktualna ścieżka:** `/api/futures/liquidation/aggregated-heatmap/model1` (potwierdzona przez Sokoła 2026-05-08)

Aggregated liquidation levels na heatmap chart. Continuous (price × time) zamiast discrete clusters.

**Decyzja warunkowa (Sokół):** jeśli response schema zawiera `price_buckets[]` z `leverage_levels[]` możliwe do parsowania → wchodzi jako **secondary source dla Liquidation Levels** (uzupełnienie `aggregated-map`). Jeśli schemat czysto wizualny (np. base64 PNG, raw pixel grid) lub niestabilny → relegowany do Phase 2.

`TODO: requires manual verification` — response schema krytyczna dla decyzji Phase 1 vs Phase 2.

---

##### ✅📡🐍 `GET /api/futures/liquidation/max-pain`

**Status:** MS1 Phase 1 OVERLAY — decyzja Sokoła 2026-05-08
**Plan tier:** Hobbyist+
**Cache TTL w MS1:** 5 min
**CG docs:** [liquidation-max-pain](https://docs.coinglass.com/reference/liquidation-max-pain)

Single max-pain price — punkt maksymalnej presji liquidations between current price and liquidation prices.

**Konsumenci:** d3pth UI overlay (linia na heatmap), Hydra (alerty gdy `current_price` blisko max-pain).

---

##### 🔵 `GET /api/futures/liquidation/map`

**Status:** Phase 2 (per-exchange map — mniej użyteczne dla MVP) — decyzja Sokoła 2026-05-08
**CG docs:** [liquidation-map](https://docs.coinglass.com/reference/liquidation-map)

Per pojedyncza giełda map. Cross-exchange `aggregated-map` jest primary; `map` deferred do post-bootstrap.

---

#### 4.2.6 Taker Buy/Sell

##### ✅🐍 `GET /api/futures/taker-buy-sell-volume/history`

**Status:** MS1 Phase 1 (priorytet 2)
**Cache TTL w MS1:** 5 min
**CG docs:** [taker-buysell-volume](https://docs.coinglass.com/reference/taker-buysell-volume)

Pair Taker Buy/Sell history — agresory (market buyers vs sellers).

---

#### 4.2.7 Premium / Index / Mark

`TODO: requires manual verification` — exact endpoint paths dla Premium Index, Mark Price, Index Price nie potwierdzone w bieżącym recon. Zostają jako Phase 2 placeholder do bootstrapu MS1.

---

### 4.3 Spot

#### 🔵 `GET /api/spot/coins-markets`

**Status:** Phase 2 (d3pth ma własne spot data via Binance/Coinbase fetchers — nie dublujemy)
**CG docs:** [spot-coins-markets](https://docs.coinglass.com/reference/spot-coins-markets)

Per-coin spot performance metrics.

---

#### 🔵 `GET /api/spot/pairs-markets`

**Status:** Phase 2
**CG docs:** [spot-pairs-markets](https://docs.coinglass.com/reference/spot-pairs-markets)

Per-pair spot metrics.

---

#### 🔵 `GET /api/spot/aggregated-taker-buy-sell-volume/history`

**Status:** Phase 2
**CG docs:** [spot-aggregated-taker-buysell-history](https://docs.coinglass.com/reference/spot-aggregated-taker-buysell-history)

Aggregated spot taker buy/sell history.

---

#### 🔵 `GET /api/spot/footprint` (Footprint History 90d)

**Status:** Phase 2 / OUT (advanced order flow analytics — niewykorzystywane w MVP)
**CG docs:** [spot-footprint](https://docs.coinglass.com/reference/spot-footprint)

---

### 4.4 Options

#### 🔵 `GET /api/option/max-pain`

**Status:** Phase 2 / consideration dla Hydry (composite score)
**CG docs:** [option-max-pain](https://docs.coinglass.com/reference/option-max-pain)

Options max pain price. Update co 30s wszystkie tiery.

---

#### 🔵 `GET /api/option/exchange-open-interest-history`

**Status:** Phase 2
**CG docs:** [exchange-open-interest-history](https://docs.coinglass.com/reference/exchange-open-interest-history)

Historical options OI per exchange.

---

#### ⚪ `GET /api/option/info`

**Status:** OUT (overview info — discoverable via inne endpointy)
**CG docs:** [info](https://docs.coinglass.com/reference/info)

---

### 4.5 ETF

Wszystkie ETF endpointy w **Phase 2** — Hydra może w przyszłości użyć dla institutional sentiment, ale nie w MVP.

#### 🔵 `GET /api/etf/bitcoin-etfs`

**CG docs:** [bitcoin-etfs](https://docs.coinglass.com/reference/bitcoin-etfs) — ETF list.

#### 🔵 `GET /api/etf/aum`

**CG docs:** [etf-aum](https://docs.coinglass.com/reference/etf-aum) — AUM history.

#### 🔵 `GET /api/etf/flows-history`

**CG docs:** [etf-flows-history](https://docs.coinglass.com/reference/etf-flows-history) — daily inflows/outflows.

#### 🔵 `GET /api/etf/bitcoin-etf-netassets-history`

**CG docs:** [bitcoin-etf-netassets-history](https://docs.coinglass.com/reference/bitcoin-etf-netassets-history)

#### 🔵 `GET /api/etf/history`

**CG docs:** [etf-history](https://docs.coinglass.com/reference/etf-history) — full historical (price, NAV, premium, shares).

---

### 4.6 On-chain

#### 🔵 `GET /api/exchange/balance/list`

**Status:** Phase 2
**CG docs:** [exchange-balance-list](https://docs.coinglass.com/reference/exchange-balance-list)

Exchange-level asset balance + 1d/7d/30d % change.

---

#### 🔵 `GET /api/exchange/balance/chart`

**Status:** Phase 2
**CG docs:** [exchange-balance-chart](https://docs.coinglass.com/reference/exchange-balance-chart)

Historical exchange balance chart.

---

#### 🔵 `GET /api/exchange/assets`

**Status:** Phase 2
**CG docs:** [exchange-assets](https://docs.coinglass.com/reference/exchange-assets)

Wallet address-level asset holdings.

---

### 4.7 Hyperliquid

Wszystkie Hyperliquid endpointy są dostępne w CG API — alternatywa dla direct Hyperliquid WebSocket (`plan_futures_pulse_coinglass.md` Issue #6).

#### ✅📡 `GET /api/hyperliquid/whale-position`

**Status:** MS1 Phase 1 (priorytet 3 — whale alerty HL)
**Cache TTL w MS1:** 1 min
**CG docs:** [hyperliquid-whale-position](https://docs.coinglass.com/reference/hyperliquid-whale-position)

Whale positions na Hyperliquid > $1M notional.

---

#### ✅📡 `GET /api/hyperliquid/whale-alert`

**Status:** MS1 Phase 1 (priorytet 3)
**Cache TTL w MS1:** 1 min
**CG docs:** [hyperliquid-whale-alert](https://docs.coinglass.com/reference/hyperliquid-whale-alert)

Real-time whale alerts > $1M notional.

---

#### 🔵 `GET /api/hyperliquid/position`

**Status:** Phase 2 (wallet positions by coin — granular)
**CG docs:** [hyperliquid-position](https://docs.coinglass.com/reference/hyperliquid-position)

---

#### 🔵 `GET /api/hyperliquid/user-position`

**Status:** Phase 2 (wallet positions by address)
**CG docs:** [hyperliquid-user-position](https://docs.coinglass.com/reference/hyperliquid-user-position)

---

### 4.8 Indicators

#### 🔵 `GET /api/indicator/pi-cycle-top`

**Status:** Phase 2 / Hydra macro signals
**CG docs:** [pi](https://docs.coinglass.com/reference/pi)

Pi Cycle Top: ma110, ma350Mu2, price.

---

#### 🔵 `GET /api/indicator/bull-market-peak-indicator`

**Status:** Phase 2 / Hydra macro signals
**CG docs:** [bull-market-peak-indicator](https://docs.coinglass.com/reference/bull-market-peak-indicator)

Bull Market Peak signal checklist.

---

#### 🔵 `GET /api/indicator/fear-greed-index`

**Status:** Phase 2
**CG docs:** [cryptofear-greedindex](https://docs.coinglass.com/reference/cryptofear-greedindex)

Crypto Fear & Greed Index (daily).

---

### 4.9 Economic Data

#### 🔵 `GET /api/economic-data`

**Status:** Phase 2 (FOMC, CPI calendar — niski priorytet w MVP)
**Cache TTL w MS1:** 24h
**CG docs:** [economic-data](https://docs.coinglass.com/reference/economic-data)

`TODO: requires manual verification` — query params (date range? country filter?) i response schema (event-by-event vs aggregate?) nie potwierdzone w snippetach.

---

### 4.10 Order Book L2/L3

#### 🔵 `GET /api/futures/orderbook-heatmap`

**Status:** Phase 2 (Standard+ tier — ostrożnie, może wymagać upgrade z Startup)
**Plan tier:** Standard+ (L2/L3 OB tier)
**CG docs:** [orderbook-heatmap](https://docs.coinglass.com/reference/orderbook-heatmap)

Historical order book depth for futures, heatmap visualization.

**Uwaga budżetowa:** L2/L3 zaczyna od Standard ($299/msc). MS1 startuje na Startup ($79). Plan upgradu **odłożony do post-bootstrap** — d3pth ma własny order book aggregator (z Binance/Coinbase/Kraken/HyperLiquid).

`TODO: requires manual verification` — czy Startup widzi `orderbook-heatmap` w jakiejś formie ograniczonej (snapshot only?) czy 100% Standard+.

---

### 4.11 Anomaly / Alert

`coinglass_service_todo.md` Faza 1 priorytet 4 mówi o endpoint `GET /api/v1/anomalies` w MS1 (NIE w CG API — to nasz własny endpoint w mikroserwisie agregujący wyniki anomaly detectors).

**Detektory anomalii w MS1:**

| Detektor | Źródło CG | Trigger | Cooldown |
|----------|-----------|---------|----------|
| Kaskada likwidacji | `liquidation/order` (1 min poll) | >$50M long lub short liq w 5 min | 15 min |
| Flip funding | `funding-rate/oi-weight-ohlc-history` | Sign flip (positive ↔ negative) | 1h |
| Skok OI | `openInterest/ohlc-history` (1 min) | OI Δ > +5% w 5 min | 15 min |
| Ekstremalny L/S | `longShort/global-account-ratio` | ratio > 2.0 lub < 0.5 | 1h |

**Hydra konsumuje listę** — decyduje o TG alercie (deduplikacja + cooldown po stronie Hydry).

---

## 5. WebSocket Streams

**Połączenie:** `wss://open-api-v4.coinglass.com/ws`
**Auth:** header `CG-API-KEY` lub query param (potwierdzić przy bootstrapie)
**Format:** JSON messages, subscribe/unsubscribe pattern
**CG docs:** [ws-getting-started](https://docs.coinglass.com/reference/ws-getting-started)

`TODO: requires manual verification` — pełna lista dostępnych streamów (liquidations live? OI? funding?) nie potwierdzona w snippetach. Sprawdzić przy bootstrapie MS1.

**Decyzja MVP:** MS1 startuje **REST-only** (poll co 1-5 min). WebSocket integration → Phase 2 jeśli okaże się potrzebne (np. dla real-time whale alerts < 1 min latency).

**Reconnect strategy (planowana):** auto-reconnect z exponential backoff, snapshot stale tolerance 30s.

---

## 6. Mapowanie d3pth ecosystem

Sekcja-skrót dla wszystkich endpointów ze statusem. Pełen indeks ~160 endpointów — niektóre niedoreprezentowane w sekcji 4 powyżej (FULL format) trafiają tu jako INDEX entries.

### 6.1 MS1 Phase 0 (Foundation — discovery, refresh 24h)

| Endpoint | Path | CG docs | Konsumenci |
|----------|------|---------|------------|
| Supported coins | `GET /api/futures/supported-coins` | [link](https://docs.coinglass.com/reference/coins) | MS1 internal validation |
| Supported exchanges | `GET /api/futures/supported-exchanges` | [link](https://docs.coinglass.com/reference/supported-exchanges) | MS1 internal |
| Supported exchange pairs | `GET /api/futures/supported-exchange-pairs` | [link](https://docs.coinglass.com/reference/instruments) | MS1 internal |

### 6.2 MS1 Phase 1 (priorytet wysoki, refresh < 15 min)

| Endpoint | Path | TTL | CG docs | Konsumenci |
|----------|------|-----|---------|------------|
| Coins markets bulk | `/api/futures/coins-markets` | 5 min | [link](https://docs.coinglass.com/reference/coins-markets) | Hydra Block C, d3pth scanner |
| Pairs markets deep-dive | `/api/futures/pairs-markets` | 5 min lazy | [link](https://docs.coinglass.com/reference/pairs-markets) | d3pth tab Futures Pulse |
| OI OHLC history | `/api/futures/openInterest/ohlc-history` | 1-5 min | [link](https://docs.coinglass.com/reference/oi-ohlc-histroy) | Hydra Block C, d3pth |
| OI exchange list | `/api/futures/openInterest/exchange-list` | 5 min | [link](https://docs.coinglass.com/reference/oi-exchange-list) | Hydra dominance ranking |
| Funding OI-weight OHLC | `/api/futures/funding-rate/oi-weight-ohlc-history` | 15 min | [link](https://docs.coinglass.com/reference/oi-weight-ohlc-history) | Hydra `funding_flip_count_7d` |
| Funding exchange list | `/api/futures/fundingRate/exchange-list` | 1h | [link](https://docs.coinglass.com/reference/fr-exchange-list) | d3pth Futures Pulse Lite |
| L/S global account | `/api/futures/longShort/global-account-ratio` | 5 min | [link](https://docs.coinglass.com/reference/global-longshort-account-ratio) | Hydra |
| L/S top account | `/api/futures/longShort/top-account-ratio` | 5 min | [link](https://docs.coinglass.com/reference/top-longshort-account-ratio) | Hydra institutional |
| L/S top position | `/api/futures/longShort/top-position-ratio` | 5 min | [link](https://docs.coinglass.com/reference/top-longshort-position-ratio) | Hydra institutional |
| Liquidation aggregated map | `/api/futures/liquidation/aggregated-map` | 5 min | [link](https://docs.coinglass.com/reference/liquidation-aggregated-map) | **d3pth `fetchers/coinglass.py` PRIMARY**, Hydra |
| Liquidation max-pain | `/api/futures/liquidation/max-pain` | 5 min | [link](https://docs.coinglass.com/reference/liquidation-max-pain) | d3pth UI overlay, Hydra alerty |
| Taker buy/sell history | `/api/futures/taker-buy-sell-volume/history` | 5 min | [link](https://docs.coinglass.com/reference/taker-buysell-volume) | Hydra agresor flow |
| Hyperliquid whale-position | `/api/hyperliquid/whale-position` | 1 min | [link](https://docs.coinglass.com/reference/hyperliquid-whale-position) | d3pth Futures Pulse |
| Hyperliquid whale-alert | `/api/hyperliquid/whale-alert` | 1 min | [link](https://docs.coinglass.com/reference/hyperliquid-whale-alert) | d3pth Futures Pulse, Hydra alerty |
| Liquidation order (anomaly) | `/api/futures/liquidation/order` | NO CACHE | [link](https://docs.coinglass.com/reference/liquidation-order) | MS1 anomaly pipeline |
| Liquidation aggregated history (anomaly) | `/api/futures/liquidation/aggregated-history` | 5 min | [link](https://docs.coinglass.com/reference/aggregated-liquidation-history) | MS1 anomaly pipeline |

### 6.3 MS1 Phase 1 candidates (verify response shape FIRST)

| Endpoint | Path | CG docs | Decyzja warunkowa |
|----------|------|---------|---------------------|
| Liquidation aggregated heatmap | `/api/futures/liquidation/aggregated-heatmap/model1` | [link](https://docs.coinglass.com/reference/liquidation-aggregate-heatmap) | Phase 1 jeśli schema parsable, Phase 2 jeśli czysto wizualny |

### 6.4 MS1 Phase 2 — Futures (deferred post-bootstrap)

| Endpoint | Path | CG docs | Powód / nota |
|----------|------|---------|--------------|
| Coins price change | `GET /api/futures/coins-price-change` | [link](https://docs.coinglass.com/reference/coins-price-change) | Duplikuje pole z `coins-markets` |
| Coins markets v2 | `GET /api/futures/futures-coins-markets-v2` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Następca `coins-markets` (verify migration path przy bootstrapie) — `TODO: requires manual verification` (różnice względem v1) |
| Futures ticker | `GET /api/futures/futures-ticker` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Live ticker per pair |
| Futures market statistics | `GET /api/futures/futures-market-statistics` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Aggregated market stats |
| Perpetual market | `GET /api/futures/perpetual-market` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Perpetual-specific markets |
| Price OHLC history | `GET /api/futures/price-ohlc-history` | [link](https://docs.coinglass.com/reference/price-ohlc-history) | Historical price OHLC (alternatywa do giełdowych) |
| OI exchange-history-chart | `GET /api/futures/openInterest/exchange-history-chart` | [exchange-open-interest-history](https://docs.coinglass.com/reference/exchange-open-interest-history) | Chart visualization, agregat z `oi-ohlc-history` + `exchange-list`. Errata #6 (futures vs options niejednoznaczne) |
| OI OHLC aggregated history | `GET /api/futures/oi-ohlc-aggregated-history` | [link](https://docs.coinglass.com/reference/oi-ohlc-aggregated-history) | Cross-exchange OI aggregated OHLC |
| RSI list (heatmap) | `GET /api/futures/rsi-list` | [futures-rsi-list](https://docs.coinglass.com/reference/futures-rsi-list) | RSI heatmap multi-symbol multi-timeframe |
| Basis | `GET /api/futures/basis` | [link](https://docs.coinglass.com/reference/basis) | Futures basis history (open/close + APY) |
| Basis chart | `GET /api/futures/futures-basis-chart` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Basis visualization |
| Net position | `GET /api/futures/net-position` | [link](https://docs.coinglass.com/reference/net-position) | Net long/short position aggregate |
| Funding rates arbitrage | `GET /api/futures/funding-rates-arbitrage` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Arbitrage opportunities cross-exchange — Hydra signal candidate |
| Funding rates history (USDⓈ-M) | `GET /api/futures/funding-rates-history-usds-m` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | USDⓈ-margined funding history; alternatywa do `oi-weight-ohlc-history` |
| Aggregated taker buy/sell history | `GET /api/futures/aggregated-taker-buysell-volume-history` | [link](https://docs.coinglass.com/reference/aggregated-taker-buysell-volume-history) | Cross-exchange taker volume |
| Exchange long/short ratio | `GET /api/futures/exchange-longshort-ratio` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Per-exchange L/S ratio |
| Liquidation map (per-exchange) | `GET /api/futures/liquidation/map` | [link](https://docs.coinglass.com/reference/liquidation-map) | Cross-exchange `aggregated-map` jest primary |
| Liquidation heatmap | `GET /api/futures/liquidation-heatmap` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Per-coin heatmap |
| Liquidation info | `GET /api/futures/liquidation-info` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Liquidation summary metadata |
| Liquidations history | `GET /api/futures/liquidations-history` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Alternatywa do `aggregated-liquidation-history`. `TODO: requires manual verification` (różnice schematu) |
| Exchange liquidations | `GET /api/futures/exchange-liquidations` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Per-exchange liquidations summary |

### 6.5 MS1 Phase 2 — Spot

| Endpoint | Path | CG docs | Powód / nota |
|----------|------|---------|--------------|
| Spot supported coins | `GET /api/spot/supported-coins` | [coins](https://docs.coinglass.com/reference/coins) | Lista coin-level dla spot (może być wspólna z futures — `TODO: requires manual verification`) |
| Spot supported exchanges | `GET /api/spot/supported-exchanges` | [supported-exchanges](https://docs.coinglass.com/reference/supported-exchanges) | Lista giełd spot |
| Spot supported exchange pairs | `GET /api/spot/supported-exchange-pairs` | [spot-suported-exchange-pairs](https://docs.coinglass.com/reference/spot-suported-exchange-pairs) | Errata #9: literówka slug `suported` zamiast `supported` po stronie CG |
| Spot coins markets | `GET /api/spot/coins-markets` | [link](https://docs.coinglass.com/reference/spot-coins-markets) | Per-coin spot — d3pth ma własne spot fetchery |
| Spot pairs markets | `GET /api/spot/pairs-markets` | [link](https://docs.coinglass.com/reference/spot-pairs-markets) | Per-pair spot |
| Spot price OHLC history | `GET /api/spot/price-ohlc-history` | [price-history](https://docs.coinglass.com/reference/price-history) | Spot OHLC alternatywa |
| Spot taker buy/sell ratio history | `GET /api/spot/taker-buysell-ratio-history` | [link](https://docs.coinglass.com/reference/spot-taker-buysell-ratio-history) | Spot taker ratio |
| Spot taker buy/sell history | `GET /api/spot/taker-buy-sell-volume/history` | [taker-buysell-volume](https://docs.coinglass.com/reference/taker-buysell-volume) | Spot taker volume |
| Spot aggregated taker | `GET /api/spot/aggregated-taker-buy-sell-volume/history` | [link](https://docs.coinglass.com/reference/spot-aggregated-taker-buysell-history) | Cross-exchange spot taker |
| Spot footprint (90d) | `GET /api/spot/footprint` | [link](https://docs.coinglass.com/reference/spot-footprint) | Advanced order flow, post-bootstrap |

### 6.6 MS1 Phase 2 — Options

| Endpoint | Path | CG docs | Powód / nota |
|----------|------|---------|--------------|
| Option max-pain | `GET /api/option/max-pain` | [link](https://docs.coinglass.com/reference/option-max-pain) | Hydra macro Phase 2; cache 30s |
| Option exchange OI history | `GET /api/option/exchange-open-interest-history` | [link](https://docs.coinglass.com/reference/exchange-open-interest-history) | Errata #6 — niejednoznaczne czy futures czy options |
| Option exchange volume history | `GET /api/option/exchange-vol-history` | [link](https://docs.coinglass.com/reference/exchange-volume-history) | Per-exchange options volume |
| Option info | `GET /api/option/info` | [link](https://docs.coinglass.com/reference/info) | Overview info — OUT (discoverable via inne) |
| Option info history | `GET /api/option/info-history` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Historical OI + volume per exchange |
| Option history (OHLC?) | `GET /api/option/history` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | `TODO: requires manual verification` — czy to OHLC, czy trades, czy aggregat |
| Option vol history | `GET /api/option/vol-history` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Volume history aggregat |
| Option top liquidations | `GET /api/option/top-liquidations` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Mirror nazywa `option-history-1` — Top liquidations options |

### 6.7 MS1 Phase 2 — ETF

| Endpoint | Path | CG docs | Powód / nota |
|----------|------|---------|--------------|
| Bitcoin ETF list | `GET /api/etf/bitcoin-etfs` | [link](https://docs.coinglass.com/reference/bitcoin-etfs) | Hydra institutional Phase 2 |
| Bitcoin ETF AUM | `GET /api/etf/aum` | [link](https://docs.coinglass.com/reference/etf-aum) | AUM history |
| Bitcoin ETF flows history | `GET /api/etf/flows-history` | [link](https://docs.coinglass.com/reference/etf-flows-history) | Daily inflows/outflows |
| Bitcoin ETF netassets history | `GET /api/etf/bitcoin-etf-netassets-history` | [link](https://docs.coinglass.com/reference/bitcoin-etf-netassets-history) | NAV history |
| Bitcoin ETF history | `GET /api/etf/history` | [link](https://docs.coinglass.com/reference/etf-history) | Full historical (price, NAV, premium, shares) |
| Ethereum ETF list | `GET /api/etf/ethereum-etf-list` | [link](https://docs.coinglass.com/reference/ethereum-etf-list) | ETH ETF list (post-launch dataset) |
| Ethereum ETF flows history | `GET /api/etf/ethereum-etf-flows-history` | [link](https://docs.coinglass.com/reference/ethereum-etf-flows-history) | Daily inflows/outflows ETH ETF |
| Ethereum ETF netassets history | `GET /api/etf/ethereum-etf-netassets-history` | [link](https://docs.coinglass.com/reference/ethereum-etf-netassets-history) | ETH ETF NAV |
| Hong Kong Bitcoin ETF flow history | `GET /api/etf/hong-kong-bitcoin-etf-flow-history` | [link](https://docs.coinglass.com/reference/hong-kong-bitcoin-etf-flow-history) | HK Bitcoin Spot ETF |

### 6.8 MS1 Phase 2 — On-chain / Exchange flows

| Endpoint | Path | CG docs | Powód / nota |
|----------|------|---------|--------------|
| Exchange balance list | `GET /api/exchange/balance/list` | [link](https://docs.coinglass.com/reference/exchange-balance-list) | Exchange-level asset balance + 1d/7d/30d % |
| Exchange balance chart | `GET /api/exchange/balance/chart` | [link](https://docs.coinglass.com/reference/exchange-balance-chart) | Historical balance chart |
| Exchange assets | `GET /api/exchange/assets` | [link](https://docs.coinglass.com/reference/exchange-assets) | Wallet address-level holdings |
| Stablecoin marketcap history | `GET /api/exchange/stablecoin-marketcap-history` | [link](https://docs.coinglass.com/reference/stablecoin-marketcap-history) | Stablecoin total marketcap (USDT, USDC, etc.) |
| Exchange on-chain transfers (ERC-20) | `GET /api/exchange/onchain-transfers` | [link](https://docs.coinglass.com/reference/exchange-onchain-transfers) | ERC-20 transfers per exchange (whale flow proxy) |

### 6.9 MS1 Phase 2 — Hyperliquid (granular)

| Endpoint | Path | CG docs | Powód / nota |
|----------|------|---------|--------------|
| Hyperliquid position by coin | `GET /api/hyperliquid/position` | [link](https://docs.coinglass.com/reference/hyperliquid-position) | Wallet positions per coin (Phase 2 — granular) |
| Hyperliquid user-position | `GET /api/hyperliquid/user-position` | [link](https://docs.coinglass.com/reference/hyperliquid-user-position) | Per-address positions + margin summary |

### 6.10 MS1 Phase 2 — Indicators (Hydra macro signals)

Wszystkie indicators są kandydatami do Hydry "Block macro" — ale w MVP MS1/d3pth nie używamy.

| Endpoint | Path | CG docs | Powód / nota |
|----------|------|---------|--------------|
| Pi Cycle Top | `GET /api/indicator/pi-cycle-top` | [pi](https://docs.coinglass.com/reference/pi) | ma110, ma350Mu2, price |
| Bull Market Peak | `GET /api/indicator/bull-market-peak-indicator` | [link](https://docs.coinglass.com/reference/bull-market-peak-indicator) | Peak signal checklist |
| Fear & Greed Index | `GET /api/indicator/fear-greed-index` | [cryptofear-greedindex](https://docs.coinglass.com/reference/cryptofear-greedindex) | Daily F&G |
| AHR999 | `GET /api/indicator/ahr999` | [link](https://docs.coinglass.com/reference/ahr999) | DCA timing indicator |
| Stock-to-Flow | `GET /api/indicator/stock-flow` | [link](https://docs.coinglass.com/reference/stock-flow) | S2F model + halving countdown |
| Puell Multiple | `GET /api/indicator/puell-multiple` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Mining-revenue model |
| Two Year MA Multiplier | `GET /api/indicator/two-year-ma-multiplier` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | 2-year MA × multipliers |
| Two Hundred Week MA Heatmap | `GET /api/indicator/two-hundred-week-moving-avg-heatmap` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | 200-week MA heatmap |
| Golden Ratio Multiplier | `GET /api/indicator/golden-ratio-multiplier` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Fibonacci multipliers |
| Grayscale market history | `GET /api/indicator/grayscale-market-history` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Grayscale Trust premium/discount |
| Bitcoin profitable days | `GET /api/indicator/bitcoin-profitable-days` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | % dni gdy BTC closed > entry price |
| Bitcoin bubble index | `GET /api/indicator/bitcoin-bubble-index` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Composite bubble metric |
| Log-log regression | `GET /api/indicator/log-log-regression` | [endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) | Bitcoin log-log regression model |

### 6.11 MS1 Phase 2 — Economic + Order Book + WebSocket

| Endpoint | Path | CG docs | Powód / nota |
|----------|------|---------|--------------|
| Economic data | `GET /api/economic-data` | [link](https://docs.coinglass.com/reference/economic-data) | FOMC + CPI calendar (niski priorytet MVP) |
| Orderbook heatmap | `GET /api/futures/orderbook-heatmap` | [link](https://docs.coinglass.com/reference/orderbook-heatmap) | **Standard+ tier** — wymaga upgrade z Startup ($79 → $299). d3pth ma własny OB aggregator |
| WebSocket streams | `wss://open-api-v4.coinglass.com/ws` | [ws-getting-started](https://docs.coinglass.com/reference/ws-getting-started) | REST-only w MVP; WS → Phase 2 jeśli potrzebna < 1 min latency |

### 6.12 OUT (nie planujemy konsumować)

| Endpoint | Powód |
|----------|-------|
| `GET /api/exchange-list` | Duplicate `supported-exchanges` ([link](https://docs.coinglass.com/reference/exchange-list)) |

### 6.13 Pełen indeks — audyt liczbowy

Sumarycznie powyższe 6.1-6.12 obejmuje:

| Sekcja | Liczba endpointów INDEX | Status |
|--------|-------------------------|--------|
| 6.1 Phase 0 / Foundation | 3 | FULL |
| 6.2 Phase 1 (priorytet) | 16 | FULL |
| 6.3 Phase 1 candidates | 1 | FULL (warunkowo) |
| 6.4 Phase 2 — Futures | 21 | INDEX |
| 6.5 Phase 2 — Spot | 10 | INDEX |
| 6.6 Phase 2 — Options | 8 | INDEX |
| 6.7 Phase 2 — ETF | 9 | INDEX |
| 6.8 Phase 2 — On-chain | 5 | INDEX |
| 6.9 Phase 2 — Hyperliquid (granular) | 2 | INDEX |
| 6.10 Phase 2 — Indicators | 13 | INDEX |
| 6.11 Phase 2 — Economic + OB + WS | 3 | INDEX |
| 6.12 OUT | 1 | INDEX |
| **SUMA** | **~92 unique paths + ~30 FULL = 122 udokumentowane** | spójne z planem (~120-130 INDEX) |

**Pełen target ~160 (Pro tier):** różnica ~38-40 endpointów to prawdopodobnie:
- WebSocket streams indywidualne (subscribe/unsubscribe types — liquidations live, OI live, funding flips, mark price)
- Wewnętrzne sub-endpointy parametryzowane (np. wariacje `liquidation/...` per timeframe lub per leverage tier)
- Endpointy które CG dodaje od czasu do czasu między release'ami

**Procedura uzupełniania (przy bootstrapie MS1):**
1. Otwórz [docs.coinglass.com/reference/endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) w przeglądarce
2. Diff vs sekcje 6.1-6.12 powyżej
3. Brakujące endpointy → wpisz tutaj jako Phase 2 z odpowiednim podsekcji (lub OUT jeśli jasna rola)
4. Zaktualizuj sekcję 8.1 (data sync) i sekcję 9 (errata)
5. NIE edytuj `MD/research/coinglass_service_todo.md` (FROZEN parking-lot — S21)

---

## 7. Operational guidance

### 7.1 Token bucket strategy w coinglass-service (Startup 80 RPM)

Dystrybucja 80 RPM między ~16 priorytetowych endpointów MS1 Phase 1:

| Grupa endpointów | Cadence | Ilość | RPM consumption |
|-------------------|---------|-------|-----------------|
| Foundation (Phase 0) | 1× / 24h | 3 | ~0 RPM |
| `coins-markets` bulk | co 5 min | 1 | 0.2 RPM |
| `pairs-markets` lazy | on-demand cache 5 min | n/a | ~5 RPM avg |
| OI history (1m, 5m intervals) | co 1 min | 1× per top symbol (10 sym) | ~10 RPM |
| Funding rate OHLC | co 15 min | 1 | <0.1 RPM |
| Funding exchange list | co 1h | 1 | <0.1 RPM |
| L/S ratio (3 endpointy) | co 5 min | 3 | 0.6 RPM |
| Liquidation aggregated-map | co 5 min × top sym | 10 sym | 2 RPM |
| Liquidation max-pain | co 5 min × top sym | 10 sym | 2 RPM |
| Liquidation order (anomaly) | co 1 min | 1 | 1 RPM |
| Liquidation aggregated history | co 5 min | 1 | 0.2 RPM |
| Taker buy/sell | co 5 min × top sym | 10 sym | 2 RPM |
| Hyperliquid whale-position | co 1 min | 1 | 1 RPM |
| Hyperliquid whale-alert | co 1 min | 1 | 1 RPM |
| **Subtotal** | | | **~25 RPM** |

**Margines:** ~55 RPM zostaje na Hydra ad-hoc deep-dives + d3pth user-triggered queries + retry/backoff buffer.

`TODO: requires manual verification` — actual RPM consumption po bootstrapie. Token bucket może wymagać tuningu.

### 7.2 Circuit breaker pattern

Wzorzec z `plan_futures_pulse_coinglass.md` Issue #1:

```
CLOSED → 5× 5xx z CG → OPEN (30s)
OPEN → po 30s → HALF-OPEN (1 probe call)
HALF-OPEN → success → CLOSED
HALF-OPEN → fail → OPEN (kolejne 30s)
```

W MS1: per-endpoint CB (nie global), żeby awaria 1 endpointu nie blokowała reszty.

W d3pth thin client: per-MS1-endpoint CB (NIE per-CG-endpoint, bo d3pth widzi tylko MS1).

### 7.3 Fallback gdy CG niedostępny

| Scenariusz | Reakcja MS1 | Reakcja d3pth |
|------------|-------------|---------------|
| CG 5xx pojedyncze | Retry 3× exponential backoff | Pasywny — czeka na MS1 cache |
| CG 5xx persistent (CB open) | Serve stale cache + flag `data_freshness="stale"` | UI banner "data >5 min stale" |
| CG 429 (rate limit) | Token bucket queue + backoff | Ignore — MS1 cache działa |
| MS1 down | n/a | UI banner "Liquidation data unavailable, retrying in 30s" + fallback `[]` |

**Hyperliquid public data fallback:** jeśli CG offline ALE Hyperliquid public API działa, d3pth może użyć direct HL fetcher (`backend/fetchers/hyperliquid.py`) jako uzupełnienie. Tylko HL data, nie cross-exchange.

### 7.4 Anomaly detection thresholds

`coinglass_service_todo.md` Faza 1 priorytet 4 — patrz sekcja 4.11 powyżej. Hydra konsumuje listę, decyduje o TG alercie.

---

## 8. Changelog źródła

### 8.1 Data ostatniej synchronizacji

**2026-05-08** — initial sync z `docs.coinglass.com` (kanon) + `coinglass.com/pricing` (plan tiers).

### 8.2 Lista WebSearch / WebFetch URL użytych

**Kanon (`docs.coinglass.com`):**
- [/reference/coins](https://docs.coinglass.com/reference/coins)
- [/reference/supported-exchanges](https://docs.coinglass.com/reference/supported-exchanges)
- [/reference/instruments](https://docs.coinglass.com/reference/instruments)
- [/reference/coins-markets](https://docs.coinglass.com/reference/coins-markets)
- [/reference/pairs-markets](https://docs.coinglass.com/reference/pairs-markets)
- [/reference/coins-price-change](https://docs.coinglass.com/reference/coins-price-change)
- [/reference/oi-exchange-list](https://docs.coinglass.com/reference/oi-exchange-list)
- [/reference/oi-ohlc-histroy](https://docs.coinglass.com/reference/oi-ohlc-histroy) (literówka po stronie CG)
- [/reference/oi-weight-ohlc-history](https://docs.coinglass.com/reference/oi-weight-ohlc-history)
- [/reference/exchange-open-interest-history](https://docs.coinglass.com/reference/exchange-open-interest-history)
- [/reference/fr-exchange-list](https://docs.coinglass.com/reference/fr-exchange-list)
- [/reference/global-longshort-account-ratio](https://docs.coinglass.com/reference/global-longshort-account-ratio)
- [/reference/top-longshort-account-ratio](https://docs.coinglass.com/reference/top-longshort-account-ratio)
- [/reference/top-longshort-position-ratio](https://docs.coinglass.com/reference/top-longshort-position-ratio)
- [/reference/net-position](https://docs.coinglass.com/reference/net-position)
- [/reference/liquidation-order](https://docs.coinglass.com/reference/liquidation-order)
- [/reference/aggregated-liquidation-history](https://docs.coinglass.com/reference/aggregated-liquidation-history)
- [/reference/liquidation-map](https://docs.coinglass.com/reference/liquidation-map)
- [/reference/liquidation-aggregated-map](https://docs.coinglass.com/reference/liquidation-aggregated-map)
- [/reference/liquidation-aggregate-heatmap](https://docs.coinglass.com/reference/liquidation-aggregate-heatmap)
- [/reference/liquidation-max-pain](https://docs.coinglass.com/reference/liquidation-max-pain)
- [/reference/taker-buysell-volume](https://docs.coinglass.com/reference/taker-buysell-volume)
- [/reference/spot-coins-markets](https://docs.coinglass.com/reference/spot-coins-markets)
- [/reference/spot-pairs-markets](https://docs.coinglass.com/reference/spot-pairs-markets)
- [/reference/spot-aggregated-taker-buysell-history](https://docs.coinglass.com/reference/spot-aggregated-taker-buysell-history)
- [/reference/spot-footprint](https://docs.coinglass.com/reference/spot-footprint)
- [/reference/option-max-pain](https://docs.coinglass.com/reference/option-max-pain)
- [/reference/info](https://docs.coinglass.com/reference/info)
- [/reference/bitcoin-etfs](https://docs.coinglass.com/reference/bitcoin-etfs)
- [/reference/etf-aum](https://docs.coinglass.com/reference/etf-aum)
- [/reference/etf-flows-history](https://docs.coinglass.com/reference/etf-flows-history)
- [/reference/bitcoin-etf-netassets-history](https://docs.coinglass.com/reference/bitcoin-etf-netassets-history)
- [/reference/etf-history](https://docs.coinglass.com/reference/etf-history)
- [/reference/exchange-balance-list](https://docs.coinglass.com/reference/exchange-balance-list)
- [/reference/exchange-balance-chart](https://docs.coinglass.com/reference/exchange-balance-chart)
- [/reference/exchange-assets](https://docs.coinglass.com/reference/exchange-assets)
- [/reference/hyperliquid-whale-position](https://docs.coinglass.com/reference/hyperliquid-whale-position)
- [/reference/hyperliquid-whale-alert](https://docs.coinglass.com/reference/hyperliquid-whale-alert)
- [/reference/hyperliquid-position](https://docs.coinglass.com/reference/hyperliquid-position)
- [/reference/hyperliquid-user-position](https://docs.coinglass.com/reference/hyperliquid-user-position)
- [/reference/pi](https://docs.coinglass.com/reference/pi)
- [/reference/bull-market-peak-indicator](https://docs.coinglass.com/reference/bull-market-peak-indicator)
- [/reference/cryptofear-greedindex](https://docs.coinglass.com/reference/cryptofear-greedindex)
- [/reference/economic-data](https://docs.coinglass.com/reference/economic-data)
- [/reference/orderbook-heatmap](https://docs.coinglass.com/reference/orderbook-heatmap)
- [/reference/ws-getting-started](https://docs.coinglass.com/reference/ws-getting-started)
- [/reference/responses-error-codes](https://docs.coinglass.com/reference/responses-error-codes)
- [/reference/endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview)
- [/reference/getting-started-with-your-api](https://docs.coinglass.com/reference/getting-started-with-your-api)
- [/reference/authentication](https://docs.coinglass.com/reference/authentication)
- [/reference/exchange-list](https://docs.coinglass.com/reference/exchange-list)
- [/reference/change-log](https://docs.coinglass.com/reference/change-log)
- [/reference/futures-rsi-list](https://docs.coinglass.com/reference/futures-rsi-list)
- [/reference/basis](https://docs.coinglass.com/reference/basis)
- [/reference/oi-ohlc-aggregated-history](https://docs.coinglass.com/reference/oi-ohlc-aggregated-history)
- [/reference/price-ohlc-history](https://docs.coinglass.com/reference/price-ohlc-history)
- [/reference/price-history](https://docs.coinglass.com/reference/price-history)
- [/reference/spot-suported-exchange-pairs](https://docs.coinglass.com/reference/spot-suported-exchange-pairs) (literówka slug — Errata #9)
- [/reference/spot-taker-buysell-ratio-history](https://docs.coinglass.com/reference/spot-taker-buysell-ratio-history)
- [/reference/aggregated-taker-buysell-volume-history](https://docs.coinglass.com/reference/aggregated-taker-buysell-volume-history)
- [/reference/exchange-volume-history](https://docs.coinglass.com/reference/exchange-volume-history)
- [/reference/ethereum-etf-list](https://docs.coinglass.com/reference/ethereum-etf-list)
- [/reference/ethereum-etf-flows-history](https://docs.coinglass.com/reference/ethereum-etf-flows-history)
- [/reference/ethereum-etf-netassets-history](https://docs.coinglass.com/reference/ethereum-etf-netassets-history)
- [/reference/hong-kong-bitcoin-etf-flow-history](https://docs.coinglass.com/reference/hong-kong-bitcoin-etf-flow-history)
- [/reference/stablecoin-marketcap-history](https://docs.coinglass.com/reference/stablecoin-marketcap-history)
- [/reference/exchange-onchain-transfers](https://docs.coinglass.com/reference/exchange-onchain-transfers)
- [/reference/ahr999](https://docs.coinglass.com/reference/ahr999)
- [/reference/stock-flow](https://docs.coinglass.com/reference/stock-flow)

**Mirror fallback (`coinglass.readme.io`):** [api-new-version](https://coinglass.readme.io/reference/api-new-version) — sprawdzone jako fallback dla SPA blank pages, ale w 2026-05-08 sweepie nieużywane (WebSearch snippets pokrył discovery).

**Pricing reference (`coinglass.com`):** [pricing](https://www.coinglass.com/pricing), [CryptoApi](https://www.coinglass.com/CryptoApi), [Full Guide](https://www.coinglass.com/learn/CoinGlass-API-Full-Guide-en).

### 8.3 Procedura odświeżania

**Kiedy:** przy bootstrapie `coinglass-service` (osobny sprint, nie wcześniej).

**Jak:**
1. Sprawdź [docs.coinglass.com/reference/change-log](https://docs.coinglass.com/reference/change-log) — czy CG dodał/usunął endpointy od 2026-05-08
2. Sprawdź [docs.coinglass.com/reference/endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) — czy nasz indeks (sekcja 6) pokrywa wszystko
3. Re-verify wszystkie `TODO: requires manual verification` markery przez Postman / Insomnia z prawdziwym kluczem
4. Update sekcji 8.1 (data sync) i sekcji 9 (errata)
5. NIE edytuj `MD/research/coinglass_service_todo.md` (FROZEN parking-lot) — wszystkie zmiany trafiają TUTAJ

---

## 9. Errata / MS1 bootstrap notes

> **Cel sekcji:** notatki dla osoby/agenta startującego coinglass-service. Sprawdź ten plik PRZED edycją parking-lotu `MD/research/coinglass_service_todo.md` (zgodnie z S21).

### 9.1 Errata #1: `coins-markets` ∧ `pairs-markets` — to dwa różne endpointy, NIE rozbieżność

**Data:** 2026-05-08 (audyt MS1, krok 0)
**Źródła:**
- [docs.coinglass.com/reference/coins-markets](https://docs.coinglass.com/reference/coins-markets) — performance metrics per coin (BTC, ETH symbol-level: funding, market cap, OI, price change)
- [docs.coinglass.com/reference/pairs-markets](https://docs.coinglass.com/reference/pairs-markets) — performance metrics per trading pair (BTCUSDT_BinanceFutures: price, index, 24h change, volume, OI, liquidations)

**Opis:** `coinglass_service_todo.md` Faza 1 Priorytet 1 cytuje `coins-markets` "bulk co 5 min" — to **nadal poprawne**. Ale `pairs-markets` to **osobny endpoint** (per-pair drill-down), NIE rozbieżność.

**Decyzja Sokoła 2026-05-08 (runda 2 sign-off):**
- `coins-markets` → MS1 Phase 1 **scheduled bulk** (polling co 5 min, ranking symboli dla Hydry)
- `pairs-markets` → MS1 Phase 1 **on-demand cached deep-dive** (lazy load per-symbol, NIE bulk polling — chronimy 80 RPM Startup limit)

**Rekomendowana akcja przy bootstrapie MS1:**
- Wystaw OBA endpointy na coinglass-service:5555
- `coins-markets` ma scheduler polling
- `pairs-markets` ma per-symbol cache z lazy load (TTL 5 min)
- W `coinglass_service_todo.md` Faza 1 Priorytet 1: dodać linię o `pairs-markets` jako uzupełnienie `coins-markets` (UWAGA: edycja parking-lotu DOPIERO przy faktycznym bootstrapie, nie teraz — S21)

### 9.2 Errata #2: 29 supported exchanges potwierdzonych w audycie

**Data:** 2026-05-08
**Źródło:** [docs.coinglass.com/reference/supported-exchanges](https://docs.coinglass.com/reference/supported-exchanges)

OKX, Binance, HTX, Bitmex, Bitfinex, Bybit, Deribit, Gate, Kraken, KuCoin, CME, Bitget, dYdX, CoinEx, BingX, Coinbase, Gemini, Crypto.com, Hyperliquid, Bitunix, MEXC, WhiteBIT, Aster, Lighter, EdgeX, Drift, Paradex, Extended, ApeX Omni.

**Implikacja:** Hyperliquid jest dostępny w CG API. Może uprościć `plan_futures_pulse_coinglass.md` Issue #6 (Hyperliquid liquidations heatmap przez WebSocket) — alternatywnie można konsumować przez CG zamiast direct WS connection. Decyzja przy implementacji Issue #6.

### 9.3 Errata #3: Liquidation rodzina — 6 endpointów, decyzja MS1 Phase 1

**Data:** 2026-05-08 (audyt + sign-off Sokoła runda 2)

**Decyzja MS1 Phase 1 dla "Liquidation Levels"** (`plan_futures_pulse_coinglass.md` Issue #1 fundament):

| Endpoint | Status MS1 | Rola |
|----------|-----------|------|
| `liquidation/aggregated-map` | **PRIMARY Phase 1** | Cross-exchange clusters per leverage; główne źródło `find_wall_liq_coincidences()` |
| `liquidation/max-pain` | Phase 1 OVERLAY | Single max-pain price; UI overlay + Hydra alerty |
| `liquidation/aggregated-heatmap/model1` | **Phase 1 candidate** | Verify response shape FIRST. Jeśli `price_buckets[]` parsable → secondary source. Jeśli wizualny → Phase 2 |
| `liquidation/order` | Anomaly pipeline | Raw events → "kaskada likwidacji" detector |
| `liquidation/aggregated-history` | Anomaly pipeline | Time-series → squeeze trigger detector |
| `liquidation/map` (per-exchange) | Phase 2 | Cross-exchange `aggregated-map` jest primary |

**Rekomendowana akcja przy bootstrapie MS1:**
1. Najpierw weryfikacja schema `aggregated-heatmap/model1` (Postman / Insomnia z kluczem) — decyduje Phase 1 vs Phase 2
2. Implementacja `aggregated-map` + `max-pain` jako fundament `GET /api/v1/liquidations/{symbol}` w MS1
3. Anomaly pipeline (`order` + `aggregated-history`) → osobny moduł `anomaly_detector.py` w MS1

### 9.4 Errata #4: Slug typo `oi-ohlc-histroy` (literówka po stronie CG)

**Data:** 2026-05-08
**Źródło:** [docs.coinglass.com/reference/oi-ohlc-histroy](https://docs.coinglass.com/reference/oi-ohlc-histroy) — uwaga literówka `histroy` zamiast `history`

**Implikacja:** używaj literałem URL-a w CG, nie próbuj poprawiać. Sam endpoint `/api/futures/openInterest/ohlc-history` ma poprawny path (literówka tylko w slug dokumentacji).

### 9.5 Errata #5: `oi-weight-ohlc-history` confusing naming

**Data:** 2026-05-08
**Źródło:** [docs.coinglass.com/reference/oi-weight-ohlc-history](https://docs.coinglass.com/reference/oi-weight-ohlc-history)

**Opis:** Nazwa slug sugeruje Open Interest, ale dokumentacja CG opisuje go jako **Funding Rate OI-weighted OHLC**. Z polskich źródeł (`coinglass_service_todo.md`) wiemy że to funding endpoint. Sprawdzić przy bootstrapie czy nazwa po stronie CG zostanie poprawiona.

### 9.6 Errata #6: `exchange-open-interest-history` — futures czy options?

**Data:** 2026-05-08
**Źródło:** [docs.coinglass.com/reference/exchange-open-interest-history](https://docs.coinglass.com/reference/exchange-open-interest-history)

**Opis:** WebSearch pokazał ten endpoint w wynikach dla Options OI history, ale slug sugeruje futures. **Niejednoznaczne.** Sprawdzić przy bootstrapie:
- Czy istnieją dwa różne endpointy (futures + options)?
- Czy parametr `marketType=futures|options` przełącza?
- Czy slug jest błędny i to faktycznie tylko options?

**Rekomendowana akcja:** Postman test z prawdziwym kluczem przed wpisaniem do MS1 schedulera.

### 9.7 Errata #7: Plan tier dla `liquidation-aggregated-heatmap`

**Data:** 2026-05-08

**Opis:** Sokół wskazał że Liquidation aggregated heatmap "może być Phase 1" — ale niektóre heatmapy w CG są L2/L3 OB tier (Standard $299+). Sprawdzić plan tier dla tego konkretnego endpointu — Startup ($79) ma 130+ endpointów, Standard 150+; może `aggregated-heatmap/model1` mieści się w Startup, ale potwierdzić Postman test'em.

**Implikacja:** Jeśli endpoint wymaga Standard, decyzja Phase 1 vs Phase 2 zmienia się — wpływa na decyzję Orkiestratora o upgrade tieru ($79 → $299).

### 9.8 Errata #8: Pełen indeks 160+ endpointów — model dwupoziomowy

**Data:** 2026-05-08
**Decyzja:** Plan zatwierdzony S16 (Sokół runda 1 sign-off) — pełny FULL opis tylko dla MS1 Phase 0/1/2 + d3pth direct consumer + kluczowe Block C. Po S22 (Sokół 2026-05-08): sekcja 6 zawiera **~92 unique paths INDEX** (sumarycznie z FULL — 122 udokumentowane endpointy). Różnica do CG Pro tier ~160 to WebSocket subscribe/unsubscribe types + parametryzowane sub-endpointy + nowe od 2026-05-08.

**Rekomendowana akcja przy bootstrapie:** cross-check listy z `endpoint-overview` — jeśli CG dodało endpointy od 2026-05-08, zaktualizuj sekcję 6 (procedura w 6.13).

### 9.9 Errata #9: Slug typo `spot-suported-exchange-pairs` (literówka po stronie CG)

**Data:** 2026-05-08
**Źródło:** [docs.coinglass.com/reference/spot-suported-exchange-pairs](https://docs.coinglass.com/reference/spot-suported-exchange-pairs) — `suported` zamiast `supported`

**Implikacja:** to **drugi typo** w slug-ach CG (po `oi-ohlc-histroy` z Errata #4). Używaj literałem URL-a, nie poprawiaj. Sam endpoint path `/api/spot/supported-exchange-pairs` ma poprawną pisownię (literówka tylko w slug docs).

### 9.10 Errata #10: `coins-markets-v2` istnieje (sukcesor `coins-markets`?)

**Data:** 2026-05-08
**Źródło:** [docs.coinglass.com/reference/endpoint-overview](https://docs.coinglass.com/reference/endpoint-overview) (mirror readme.io listing)

**Opis:** Endpoint `/api/futures/futures-coins-markets-v2` istnieje obok `/api/futures/coins-markets`. `TODO: requires manual verification`:
- Czy v2 zastępuje v1 (deprecation)?
- Czy v2 ma inne pola w response (extra metrics)?
- Czy oba są dostępne równolegle (różne use-cases)?

**Implikacja dla MS1 bootstrap:** przy implementacji Phase 1 `coins-markets` (sekcja 6.2) sprawdź czy v2 nie jest preferred — może być korzystniejszy (więcej pól, nowsze dane).

### 9.11 Errata #11: Mirror readme.io ujawnił dodatkowe endpointy nie widoczne w docs.coinglass.com SPA

**Data:** 2026-05-08
**Źródło:** [coinglass.readme.io/reference/api-new-version](https://coinglass.readme.io/reference/api-new-version) — boczny menu

**Endpointy odkryte przez mirror (NIE potwierdzone bezpośrednio w `docs.coinglass.com/reference`):** `futures-ticker`, `futures-market-statistics`, `perpetual-market`, `funding-rates-arbitrage`, `funding-rates-history-usds-m`, `liquidations-history`, `exchange-liquidations`, `exchange-longshort-ratio`, `liquidation-info`, `liquidation-heatmap`, `option-history`, `option-vol-history`, `option-history-1` (Top liquidations options), `info-history`, `puell-multiple`, `two-year-ma-multiplier`, `two-hundred-week-moving-avg-heatmap`, `golden-ratio-multiplier`, `grayscale-market-history`, `bitcoin-profitable-days`, `bitcoin-bubble-index`, `log-log-regression`.

**Implikacja:** mirror może być częściowo nieaktualny (legacy v3 vs v4 prefixing). Wszystkie powyższe wpisałem do sekcji 6.4-6.10 z markerem `[endpoint-overview]` zamiast bezpośredniego linku — przy bootstrapie zweryfikuj że path `/api/...` nadal działa lub został zmieniony.

---

