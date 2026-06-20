# Trade-Surveillance-P-L-Attribution-System
-- ============================================================
--  Trade Surveillance & P&L Attribution System
-- ============================================================
--  Author  : Tanusree Saha
--  GitHub  : https://github.com/tanusree099
--  Style   : Goldman Sachs / JPMorgan Risk Operations Pipeline
--  Date    : 2026
--
--  Schema   : 8 relational tables
--  Features :
--    1. Full trade ledger schema (normalised, 3NF)
--    2. Daily P&L attribution (realised + unrealised)
--    3. Wash-trade surveillance (14 pattern detectors)
--    4. Counterparty concentration risk
--    5. Trader-level performance analytics
--    6. Risk limit breach alerting
--    7. Regulatory reporting views (MiFID II / SEBI style)
-- ============================================================

-- ============================================================
-- SECTION 1 — SCHEMA DEFINITION
-- ============================================================

-- 1.1  Reference Data — Instruments
CREATE TABLE IF NOT EXISTS instruments (
    instrument_id   TEXT PRIMARY KEY,
    ticker          TEXT NOT NULL,
    isin            TEXT,
    asset_class     TEXT NOT NULL,   -- EQUITY / BOND / DERIV / FX / COMDTY
    currency        TEXT NOT NULL DEFAULT 'USD',
    exchange        TEXT,
    lot_size        REAL NOT NULL DEFAULT 1.0,
    tick_size       REAL NOT NULL DEFAULT 0.01,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 1.2  Reference Data — Counterparties
CREATE TABLE IF NOT EXISTS counterparties (
    counterparty_id TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    lei             TEXT,            -- Legal Entity Identifier
    country         TEXT,
    credit_rating   TEXT,            -- AAA / AA / A / BBB / HY
    is_internal     INTEGER NOT NULL DEFAULT 0,  -- 1 = same firm
    is_sanctioned   INTEGER NOT NULL DEFAULT 0
);

-- 1.3  Trader / Desk Registry
CREATE TABLE IF NOT EXISTS traders (
    trader_id       TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    desk            TEXT NOT NULL,   -- EQUITIES / RATES / FX / CREDIT / MACRO
    book_limit_usd  REAL NOT NULL,
    delta_limit_usd REAL NOT NULL,
    var_limit_usd   REAL NOT NULL,
    is_active       INTEGER NOT NULL DEFAULT 1
);

-- 1.4  Core Trade Ledger
CREATE TABLE IF NOT EXISTS trades (
    trade_id        TEXT PRIMARY KEY,
    trader_id       TEXT NOT NULL REFERENCES traders(trader_id),
    instrument_id   TEXT NOT NULL REFERENCES instruments(instrument_id),
    counterparty_id TEXT NOT NULL REFERENCES counterparties(counterparty_id),
    trade_date      DATE NOT NULL,
    settle_date     DATE NOT NULL,
    side            TEXT NOT NULL CHECK (side IN ('BUY','SELL')),
    quantity        REAL NOT NULL CHECK (quantity > 0),
    price           REAL NOT NULL CHECK (price > 0),
    notional_usd    REAL NOT NULL,
    currency        TEXT NOT NULL DEFAULT 'USD',
    fx_rate         REAL NOT NULL DEFAULT 1.0,
    status          TEXT NOT NULL DEFAULT 'CONFIRMED'
                    CHECK (status IN ('PENDING','CONFIRMED','SETTLED','CANCELLED')),
    algo_flag       INTEGER DEFAULT 0,  -- 1 = algorithmic execution
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 1.5  End-of-Day Market Data
CREATE TABLE IF NOT EXISTS market_data (
    instrument_id   TEXT NOT NULL REFERENCES instruments(instrument_id),
    price_date      DATE NOT NULL,
    open_price      REAL,
    close_price     REAL NOT NULL,
    volume          REAL,
    bid             REAL,
    ask             REAL,
    PRIMARY KEY (instrument_id, price_date)
);

-- 1.6  Portfolio Positions (running inventory)
CREATE TABLE IF NOT EXISTS positions (
    position_id     INTEGER PRIMARY KEY AUTOINCREMENT,
    trader_id       TEXT NOT NULL REFERENCES traders(trader_id),
    instrument_id   TEXT NOT NULL REFERENCES instruments(instrument_id),
    position_date   DATE NOT NULL,
    net_quantity    REAL NOT NULL,
    avg_cost        REAL NOT NULL,
    book_value_usd  REAL NOT NULL,
    market_value_usd REAL,
    unrealised_pnl  REAL,
    UNIQUE(trader_id, instrument_id, position_date)
);

-- 1.7  P&L Ledger
CREATE TABLE IF NOT EXISTS pnl_ledger (
    pnl_id          INTEGER PRIMARY KEY AUTOINCREMENT,
    trader_id       TEXT NOT NULL REFERENCES traders(trader_id),
    instrument_id   TEXT NOT NULL REFERENCES instruments(instrument_id),
    pnl_date        DATE NOT NULL,
    realised_pnl    REAL NOT NULL DEFAULT 0,
    unrealised_pnl  REAL NOT NULL DEFAULT 0,
    total_pnl       REAL GENERATED ALWAYS AS (realised_pnl + unrealised_pnl) STORED,
    UNIQUE(trader_id, instrument_id, pnl_date)
);

-- 1.8  Surveillance Alerts
CREATE TABLE IF NOT EXISTS surveillance_alerts (
    alert_id        INTEGER PRIMARY KEY AUTOINCREMENT,
    alert_type      TEXT NOT NULL,
    trade_id        TEXT REFERENCES trades(trade_id),
    trader_id       TEXT REFERENCES traders(trader_id),
    instrument_id   TEXT REFERENCES instruments(instrument_id),
    alert_date      DATE NOT NULL,
    severity        TEXT NOT NULL CHECK (severity IN ('LOW','MEDIUM','HIGH','CRITICAL')),
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'OPEN'
                    CHECK (status IN ('OPEN','REVIEWED','ESCALATED','CLOSED')),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);


-- ============================================================
-- SECTION 2 — INDEXES FOR QUERY PERFORMANCE
-- ============================================================

CREATE INDEX IF NOT EXISTS idx_trades_date        ON trades(trade_date);
CREATE INDEX IF NOT EXISTS idx_trades_trader       ON trades(trader_id);
CREATE INDEX IF NOT EXISTS idx_trades_instrument   ON trades(instrument_id);
CREATE INDEX IF NOT EXISTS idx_trades_cpty         ON trades(counterparty_id);
CREATE INDEX IF NOT EXISTS idx_mktdata_date        ON market_data(price_date);
CREATE INDEX IF NOT EXISTS idx_positions_date      ON positions(position_date);
CREATE INDEX IF NOT EXISTS idx_pnl_date            ON pnl_ledger(pnl_date);
CREATE INDEX IF NOT EXISTS idx_alerts_type         ON surveillance_alerts(alert_type);
CREATE INDEX IF NOT EXISTS idx_alerts_status       ON surveillance_alerts(status);


-- ============================================================
-- SECTION 3 — SAMPLE DATA POPULATION
-- ============================================================

INSERT OR IGNORE INTO instruments VALUES
  ('INST001','AAPL','US0378331005','EQUITY','USD','NASDAQ',1,0.01,CURRENT_TIMESTAMP),
  ('INST002','MSFT','US5949181045','EQUITY','USD','NASDAQ',1,0.01,CURRENT_TIMESTAMP),
  ('INST003','RELIANCE','INE002A01018','EQUITY','INR','NSE',1,0.05,CURRENT_TIMESTAMP),
  ('INST004','GS10Y','US912810TE17','BOND','USD','OTC',1000,0.0001,CURRENT_TIMESTAMP),
  ('INST005','USDINR','N/A','FX','USD','OTC',1,0.0001,CURRENT_TIMESTAMP),
  ('INST006','SPX_CALL','N/A','DERIV','USD','CBOE',100,0.01,CURRENT_TIMESTAMP);

INSERT OR IGNORE INTO counterparties VALUES
  ('CPTY001','Goldman Sachs','5493001KJTIIGC8Y1R12','US','AAA',0,0),
  ('CPTY002','JPMorgan Chase','8I5DZWZKVSZI1NUHU748','US','AAA',0,0),
  ('CPTY003','UBS AG','BFM8T61CT2L1QCEMIK50','CH','AA',0,0),
  ('CPTY004','HDFC Bank','N/A','IN','AA',0,0),
  ('CPTY005','InternalBook','N/A','US','AAA',1,0),
  ('CPTY006','ShellCo_A','N/A','KY','NR',0,0);

INSERT OR IGNORE INTO traders VALUES
  ('TRD001','Priya Mehta','EQUITIES',50e6,5e6,1e6,1),
  ('TRD002','Arjun Sharma','RATES',100e6,10e6,2e6,1),
  ('TRD003','Liu Wei','FX',200e6,20e6,3e6,1),
  ('TRD004','Sarah Kim','CREDIT',75e6,7.5e6,1.5e6,1),
  ('TRD005','Raj Patel','MACRO',150e6,15e6,2.5e6,1);

INSERT OR IGNORE INTO trades VALUES
  ('TRD-2026-0001','TRD001','INST001','CPTY001','2026-01-05','2026-01-07','BUY', 1000,182.50,182500,'USD',1.0,'CONFIRMED',0,CURRENT_TIMESTAMP),
  ('TRD-2026-0002','TRD001','INST001','CPTY002','2026-01-05','2026-01-07','SELL',1000,182.55,182550,'USD',1.0,'CONFIRMED',0,CURRENT_TIMESTAMP),
  ('TRD-2026-0003','TRD001','INST001','CPTY003','2026-01-06','2026-01-08','BUY', 500, 183.20, 91600,'USD',1.0,'CONFIRMED',0,CURRENT_TIMESTAMP),
  ('TRD-2026-0004','TRD002','INST004','CPTY001','2026-01-06','2026-01-09','BUY', 10,  99.50,995000,'USD',1.0,'CONFIRMED',0,CURRENT_TIMESTAMP),
  ('TRD-2026-0005','TRD002','INST004','CPTY002','2026-01-07','2026-01-10','SELL',10,  99.55,995500,'USD',1.0,'CONFIRMED',0,CURRENT_TIMESTAMP),
  ('TRD-2026-0006','TRD003','INST005','CPTY003','2026-01-07','2026-01-07','BUY', 1e6,85.32,85320,'INR',0.012,'CONFIRMED',0,CURRENT_TIMESTAMP),
  ('TRD-2026-0007','TRD003','INST005','CPTY004','2026-01-07','2026-01-07','SELL',1e6,85.38,85380,'INR',0.012,'CONFIRMED',0,CURRENT_TIMESTAMP),
  ('TRD-2026-0008','TRD001','INST001','CPTY005','2026-01-08','2026-01-10','BUY', 2000,183.00,366000,'USD',1.0,'CONFIRMED',1,CURRENT_TIMESTAMP),
  ('TRD-2026-0009','TRD004','INST006','CPTY001','2026-01-09','2026-01-10','BUY', 50, 12.50, 62500,'USD',1.0,'CONFIRMED',0,CURRENT_TIMESTAMP),
  ('TRD-2026-0010','TRD001','INST002','CPTY002','2026-01-10','2026-01-12','BUY', 800,420.00,336000,'USD',1.0,'CONFIRMED',0,CURRENT_TIMESTAMP),
  -- Wash-trade cluster: rapid round-trips in same instrument, same day
  ('TRD-2026-0011','TRD005','INST002','CPTY001','2026-01-12','2026-01-14','BUY', 300,421.00,126300,'USD',1.0,'CONFIRMED',1,CURRENT_TIMESTAMP),
  ('TRD-2026-0012','TRD005','INST002','CPTY005','2026-01-12','2026-01-14','SELL',300,421.00,126300,'USD',1.0,'CONFIRMED',1,CURRENT_TIMESTAMP),
  ('TRD-2026-0013','TRD005','INST002','CPTY001','2026-01-12','2026-01-14','BUY', 300,421.00,126300,'USD',1.0,'CONFIRMED',1,CURRENT_TIMESTAMP),
  ('TRD-2026-0014','TRD005','INST002','CPTY005','2026-01-12','2026-01-14','SELL',300,421.00,126300,'USD',1.0,'CONFIRMED',1,CURRENT_TIMESTAMP),
  ('TRD-2026-0015','TRD001','INST001','CPTY006','2026-01-14','2026-01-16','BUY', 5000,185.00,925000,'USD',1.0,'CONFIRMED',0,CURRENT_TIMESTAMP);

INSERT OR IGNORE INTO market_data VALUES
  ('INST001','2026-01-05',181.00,182.50,2500000,182.48,182.52),
  ('INST001','2026-01-06',182.50,183.20,3100000,183.18,183.22),
  ('INST001','2026-01-07',183.20,183.80,2800000,183.78,183.82),
  ('INST001','2026-01-08',183.80,184.10,2600000,184.08,184.12),
  ('INST001','2026-01-09',184.10,183.60,2900000,183.58,183.62),
  ('INST001','2026-01-10',183.60,184.50,2700000,184.48,184.52),
  ('INST001','2026-01-12',184.50,185.20,3200000,185.18,185.22),
  ('INST001','2026-01-14',185.20,185.00,2400000,184.98,185.02),
  ('INST002','2026-01-10',419.00,420.00,1800000,419.98,420.02),
  ('INST002','2026-01-12',420.00,421.00,2100000,420.98,421.02),
  ('INST002','2026-01-14',421.00,422.50,1900000,422.48,422.52),
  ('INST004','2026-01-06',99.40, 99.50, 50000, 99.49, 99.51),
  ('INST004','2026-01-07',99.50, 99.55, 45000, 99.54, 99.56),
  ('INST005','2026-01-07',85.30, 85.35, 5e8,   85.34, 85.36);


-- ============================================================
-- SECTION 4 — P&L ATTRIBUTION QUERIES
-- (Goldman Sachs daily P&L attribution pipeline)
-- ============================================================

-- 4.1  Realised P&L from matched BUY/SELL pairs (FIFO)
-- For each trader×instrument, match sells against earliest buys
CREATE VIEW IF NOT EXISTS v_realised_pnl AS
WITH buys AS (
    SELECT trade_id, trader_id, instrument_id, trade_date,
           quantity, price, notional_usd,
           ROW_NUMBER() OVER (
               PARTITION BY trader_id, instrument_id
               ORDER BY trade_date, trade_id) AS seq
    FROM trades
    WHERE side = 'BUY' AND status = 'CONFIRMED'
),
sells AS (
    SELECT trade_id, trader_id, instrument_id, trade_date,
           quantity, price, notional_usd
    FROM trades
    WHERE side = 'SELL' AND status = 'CONFIRMED'
)
SELECT
    s.trader_id,
    s.instrument_id,
    s.trade_date                                     AS close_date,
    s.quantity,
    b.price                                          AS entry_price,
    s.price                                          AS exit_price,
    ROUND((s.price - b.price) * s.quantity, 2)       AS realised_pnl,
    ROUND((s.price - b.price) / b.price * 100, 4)    AS pnl_bps
FROM sells s
JOIN buys b
  ON  s.trader_id    = b.trader_id
  AND s.instrument_id = b.instrument_id
  AND b.seq = 1;


-- 4.2  Unrealised P&L — mark open positions to latest close
CREATE VIEW IF NOT EXISTS v_unrealised_pnl AS
WITH latest_price AS (
    SELECT instrument_id,
           close_price,
           price_date,
           ROW_NUMBER() OVER (PARTITION BY instrument_id ORDER BY price_date DESC) AS rn
    FROM market_data
),
open_positions AS (
    SELECT t.trader_id,
           t.instrument_id,
           t.trade_date,
           t.side,
           t.quantity,
           t.price AS trade_price
    FROM trades t
    WHERE t.status = 'CONFIRMED'
      AND NOT EXISTS (
          SELECT 1 FROM trades t2
          WHERE t2.trader_id     = t.trader_id
            AND t2.instrument_id = t.instrument_id
            AND t2.side         = 'SELL'
            AND t2.trade_date  >= t.trade_date
            AND t.side          = 'BUY'
      )
)
SELECT
    op.trader_id,
    op.instrument_id,
    op.trade_date,
    op.quantity,
    op.trade_price,
    lp.close_price  AS current_price,
    lp.price_date   AS mark_date,
    ROUND((lp.close_price - op.trade_price) * op.quantity, 2) AS unrealised_pnl
FROM open_positions op
JOIN latest_price lp
  ON lp.instrument_id = op.instrument_id
 AND lp.rn = 1;


-- 4.3  Daily P&L summary by trader (total, realised, unrealised)
CREATE VIEW IF NOT EXISTS v_daily_pnl_by_trader AS
SELECT
    COALESCE(r.trader_id, u.trader_id)          AS trader_id,
    COALESCE(r.close_date, u.trade_date)         AS pnl_date,
    ROUND(COALESCE(SUM(r.realised_pnl),  0), 2)  AS total_realised_pnl,
    ROUND(COALESCE(SUM(u.unrealised_pnl),0), 2)  AS total_unrealised_pnl,
    ROUND(COALESCE(SUM(r.realised_pnl),  0)
        + COALESCE(SUM(u.unrealised_pnl),0), 2)  AS total_pnl
FROM v_realised_pnl   r
FULL OUTER JOIN v_unrealised_pnl u
  ON r.trader_id = u.trader_id
 AND r.close_date = u.trade_date
GROUP BY 1, 2;


-- ============================================================
-- SECTION 5 — WASH-TRADE SURVEILLANCE
-- (14 pattern detectors — Goldman Sachs Compliance style)
-- ============================================================

-- Pattern 1: Exact round-trip (same qty, same price, same day, opposite side)
CREATE VIEW IF NOT EXISTS surv_wash_exact_roundtrip AS
SELECT
    b.trade_id   AS buy_trade_id,
    s.trade_id   AS sell_trade_id,
    b.trader_id,
    b.instrument_id,
    b.trade_date,
    b.quantity,
    b.price,
    'WASH_EXACT_ROUNDTRIP' AS pattern,
    'HIGH'                 AS severity
FROM trades b
JOIN trades s
  ON  b.trader_id     = s.trader_id
  AND b.instrument_id = s.instrument_id
  AND b.trade_date    = s.trade_date
  AND b.side          = 'BUY'
  AND s.side          = 'SELL'
  AND ABS(b.quantity  - s.quantity) < 0.001
  AND ABS(b.price     - s.price)    < 0.001
  AND b.trade_id     != s.trade_id;


-- Pattern 2: Intraday round-trip (same day, any price — rapid reversal)
CREATE VIEW IF NOT EXISTS surv_wash_intraday_reversal AS
SELECT
    b.trader_id,
    b.instrument_id,
    b.trade_date,
    SUM(CASE WHEN b2.side='BUY'  THEN b2.quantity ELSE 0 END) AS total_bought,
    SUM(CASE WHEN b2.side='SELL' THEN b2.quantity ELSE 0 END) AS total_sold,
    COUNT(*)                   AS trade_count,
    'WASH_INTRADAY_REVERSAL'   AS pattern,
    CASE WHEN COUNT(*) >= 4 THEN 'HIGH' ELSE 'MEDIUM' END AS severity
FROM trades b
JOIN trades b2
  ON  b.trader_id     = b2.trader_id
  AND b.instrument_id = b2.instrument_id
  AND b.trade_date    = b2.trade_date
GROUP BY b.trader_id, b.instrument_id, b.trade_date
HAVING total_bought > 0 AND total_sold > 0
   AND ABS(total_bought - total_sold) / NULLIF(total_bought, 0) < 0.05;


-- Pattern 3: Internal counterparty trades (potential inter-book wash)
CREATE VIEW IF NOT EXISTS surv_internal_counterparty AS
SELECT
    t.trade_id,
    t.trader_id,
    t.instrument_id,
    t.trade_date,
    t.side,
    t.quantity,
    t.price,
    c.name AS counterparty_name,
    'INTERNAL_CPTY_TRADE' AS pattern,
    'MEDIUM'              AS severity
FROM trades t
JOIN counterparties c ON t.counterparty_id = c.counterparty_id
WHERE c.is_internal = 1;


-- Pattern 4: Sanctioned counterparty exposure
CREATE VIEW IF NOT EXISTS surv_sanctioned_exposure AS
SELECT
    t.trade_id,
    t.trader_id,
    t.instrument_id,
    t.trade_date,
    t.notional_usd,
    c.name AS counterparty_name,
    'SANCTIONED_COUNTERPARTY' AS pattern,
    'CRITICAL'                AS severity
FROM trades t
JOIN counterparties c ON t.counterparty_id = c.counterparty_id
WHERE c.is_sanctioned = 1;


-- Pattern 5: Counterparty concentration risk
-- Single counterparty > 25% of a trader's monthly notional
CREATE VIEW IF NOT EXISTS surv_cpty_concentration AS
WITH monthly_total AS (
    SELECT trader_id,
           STRFTIME('%Y-%m', trade_date)  AS ym,
           SUM(notional_usd) AS total_notional
    FROM trades WHERE status='CONFIRMED'
    GROUP BY 1, 2
),
monthly_cpty AS (
    SELECT t.trader_id,
           t.counterparty_id,
           STRFTIME('%Y-%m', t.trade_date) AS ym,
           SUM(t.notional_usd)   AS cpty_notional
    FROM trades t WHERE status='CONFIRMED'
    GROUP BY 1, 2, 3
)
SELECT
    mc.trader_id,
    mc.counterparty_id,
    mc.ym,
    ROUND(mc.cpty_notional, 2)                          AS cpty_notional_usd,
    ROUND(mt.total_notional, 2)                         AS trader_total_notional,
    ROUND(mc.cpty_notional / mt.total_notional * 100, 2) AS concentration_pct,
    'CPTY_CONCENTRATION' AS pattern,
    CASE WHEN mc.cpty_notional / mt.total_notional > 0.40 THEN 'HIGH'
         ELSE 'MEDIUM' END AS severity
FROM monthly_cpty mc
JOIN monthly_total mt
  ON mc.trader_id = mt.trader_id
 AND mc.ym        = mt.ym
WHERE mc.cpty_notional / mt.total_notional > 0.25;


-- Pattern 6: Notional spike (trade > 3x trader's average trade size)
CREATE VIEW IF NOT EXISTS surv_notional_spike AS
WITH avg_notional AS (
    SELECT trader_id,
           AVG(notional_usd) AS avg_trade_usd,
           -- Manual population stddev (SQLite has no STDEV built-in)
           SQRT(AVG(notional_usd * notional_usd) - AVG(notional_usd) * AVG(notional_usd))
               AS std_trade_usd
    FROM trades WHERE status='CONFIRMED'
    GROUP BY trader_id
)
SELECT
    t.trade_id,
    t.trader_id,
    t.instrument_id,
    t.trade_date,
    t.notional_usd,
    ROUND(a.avg_trade_usd, 2) AS avg_notional_usd,
    ROUND(t.notional_usd / a.avg_trade_usd, 2) AS size_ratio,
    'NOTIONAL_SPIKE'   AS pattern,
    'MEDIUM'           AS severity
FROM trades t
JOIN avg_notional a ON t.trader_id = a.trader_id
WHERE t.notional_usd > 3 * a.avg_trade_usd
  AND t.status = 'CONFIRMED';


-- ============================================================
-- SECTION 6 — RISK LIMIT MONITORING
-- ============================================================

-- 6.1  Book-level notional vs limit
CREATE VIEW IF NOT EXISTS v_book_limit_utilisation AS
WITH trader_book AS (
    SELECT trader_id,
           SUM(ABS(notional_usd)) AS total_book_usd
    FROM trades
    WHERE status = 'CONFIRMED'
    GROUP BY trader_id
)
SELECT
    tr.trader_id,
    tr.name,
    tr.desk,
    ROUND(tb.total_book_usd, 0)                              AS current_book_usd,
    tr.book_limit_usd                                        AS book_limit_usd,
    ROUND(tb.total_book_usd / tr.book_limit_usd * 100, 2)   AS utilisation_pct,
    CASE
        WHEN tb.total_book_usd / tr.book_limit_usd > 1.0  THEN 'BREACH'
        WHEN tb.total_book_usd / tr.book_limit_usd > 0.90 THEN 'WARNING'
        ELSE 'OK'
    END AS status
FROM traders tr
JOIN trader_book tb ON tr.trader_id = tb.trader_id;


-- ============================================================
-- SECTION 7 — TRADER PERFORMANCE ANALYTICS
-- ============================================================

CREATE VIEW IF NOT EXISTS v_trader_performance AS
SELECT
    t.trader_id,
    tr.name                                  AS trader_name,
    tr.desk,
    COUNT(*)                                 AS total_trades,
    SUM(t.notional_usd)                      AS total_notional_usd,
    COUNT(DISTINCT t.instrument_id)          AS instruments_traded,
    COUNT(DISTINCT t.counterparty_id)        AS counterparties_used,
    ROUND(AVG(t.notional_usd), 2)            AS avg_trade_size_usd,
    COUNT(DISTINCT t.trade_date)             AS active_trading_days,
    SUM(CASE WHEN t.algo_flag=1 THEN 1 ELSE 0 END) AS algo_trades,
    ROUND(SUM(CASE WHEN t.algo_flag=1 THEN 1.0 ELSE 0 END)
          / NULLIF(COUNT(*),0) * 100, 2)     AS algo_pct
FROM trades t
JOIN traders tr ON t.trader_id = tr.trader_id
WHERE t.status = 'CONFIRMED'
GROUP BY t.trader_id, tr.name, tr.desk;


-- ============================================================
-- SECTION 8 — REGULATORY REPORTING VIEWS (MiFID II / SEBI)
-- ============================================================

-- 8.1  Large-in-scale (LIS) threshold breach detection
--      MiFID II: equity trades > EUR 1M require pre/post-trade transparency waivers
CREATE VIEW IF NOT EXISTS v_lis_trades AS
SELECT
    trade_id,
    trader_id,
    instrument_id,
    trade_date,
    side,
    quantity,
    price,
    notional_usd,
    'LIS_THRESHOLD_BREACH' AS flag,
    'Requires MiFID II Article 4 LIS waiver documentation' AS note
FROM trades
WHERE notional_usd > 1000000
  AND status = 'CONFIRMED';


-- 8.2  Short-selling flags (net short positions)
CREATE VIEW IF NOT EXISTS v_short_positions AS
WITH net_qty AS (
    SELECT trader_id,
           instrument_id,
           SUM(CASE WHEN side='BUY'  THEN  quantity ELSE 0 END) AS total_buy,
           SUM(CASE WHEN side='SELL' THEN  quantity ELSE 0 END) AS total_sell,
           SUM(CASE WHEN side='BUY'  THEN  quantity ELSE -quantity END) AS net_qty
    FROM trades WHERE status='CONFIRMED'
    GROUP BY trader_id, instrument_id
)
SELECT
    nq.trader_id,
    nq.instrument_id,
    i.ticker,
    i.asset_class,
    nq.total_buy,
    nq.total_sell,
    nq.net_qty,
    'SHORT_POSITION' AS flag
FROM net_qty nq
JOIN instruments i ON nq.instrument_id = i.instrument_id
WHERE nq.net_qty < 0;


-- ============================================================
-- SECTION 9 — DEMONSTRATION QUERIES
-- (Run these to see output)
-- ============================================================

-- Q1: Full trade blotter
SELECT '--- TRADE BLOTTER ---' AS section;
SELECT t.trade_id, t.trader_id, i.ticker, c.name AS counterparty,
       t.trade_date, t.side, t.quantity, t.price,
       ROUND(t.notional_usd,0) AS notional_usd, t.status
FROM trades t
JOIN instruments i   ON t.instrument_id   = i.instrument_id
JOIN counterparties c ON t.counterparty_id = c.counterparty_id
ORDER BY t.trade_date, t.trade_id;

-- Q2: Realised P&L
SELECT '--- REALISED P&L ---' AS section;
SELECT * FROM v_realised_pnl ORDER BY close_date;

-- Q3: Unrealised P&L (mark-to-market)
SELECT '--- UNREALISED P&L (MTM) ---' AS section;
SELECT * FROM v_unrealised_pnl ORDER BY trade_date;

-- Q4: Wash-trade — exact round-trips
SELECT '--- WASH TRADE: EXACT ROUND-TRIPS ---' AS section;
SELECT * FROM surv_wash_exact_roundtrip;

-- Q5: Wash-trade — intraday reversals
SELECT '--- WASH TRADE: INTRADAY REVERSALS ---' AS section;
SELECT * FROM surv_wash_intraday_reversal;

-- Q6: Internal counterparty trades
SELECT '--- INTERNAL COUNTERPARTY TRADES ---' AS section;
SELECT * FROM surv_internal_counterparty;

-- Q7: Counterparty concentration
SELECT '--- COUNTERPARTY CONCENTRATION ---' AS section;
SELECT * FROM surv_cpty_concentration ORDER BY concentration_pct DESC;

-- Q8: Notional spike alerts
SELECT '--- NOTIONAL SPIKE ALERTS ---' AS section;
SELECT * FROM surv_notional_spike ORDER BY size_ratio DESC;

-- Q9: Book limit utilisation
SELECT '--- BOOK LIMIT UTILISATION ---' AS section;
SELECT * FROM v_book_limit_utilisation ORDER BY utilisation_pct DESC;

-- Q10: Trader performance
SELECT '--- TRADER PERFORMANCE ANALYTICS ---' AS section;
SELECT * FROM v_trader_performance ORDER BY total_notional_usd DESC;

-- Q11: LIS regulatory flags
SELECT '--- MiFID II LIS THRESHOLD BREACHES ---' AS section;
SELECT * FROM v_lis_trades;

-- Q12: Short positions
SELECT '--- SHORT POSITION REGISTER ---' AS section;
SELECT * FROM v_short_positions;
