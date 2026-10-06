# Trading & Order Execution Reference

## Overview

This reference covers the complete workflow for executing trades from backtest results to live orders. The process involves:

1. **Position Calculation**: Convert backtest results to share quantities
2. **Broker Connection**: Configure and connect to your broker account
3. **Order Execution**: Create, update, and manage orders via OrderExecutor
4. **Realtime Sync** *(v2.0.0)*: Subscribe to live position/fill streams via `PositionStreamMixin`
5. **Review** *(v2.2.4)*: Rebuild actual NAV and trades from fills via `TradeReview`

---

## Typed Data Contracts — `finlab.schemas`

*(v1.5.9)* `finlab.schemas` exposes formal dataclass / TypedDict contracts used across the trading stack. Prefer these over untyped dict access when writing production integrations — the types are enforced at boundaries (`PortfolioSyncManager`, `OrderExecutor`) and catch schema drift at code-review time.

```python
from finlab.schemas import (
    PositionEntry,   # one holding line: stock_id, quantity, order_condition, weight, ...
    OrderEntry,      # one planned order: stock_id, quantity, action, price, ...
    PortfolioData,   # full portfolio snapshot consumed by PortfolioSyncManager
)
```

**Deprecation note:** `stock_id` continues to work, but new typed APIs prefer `symbol` — see `docs/details/typed_data_interfaces.md` in the finlab repo. The migration policy is: `stock_id` remains accepted but aliased to `symbol` in dataclasses.

---

## Position Class

The `Position` class represents target holdings and provides methods for converting backtest results to executable positions.

**Import:**
```python
from finlab.online.order_executor import Position
```

### Position.from_report()

Convert a backtest report to tradeable positions.

**Signature:**
```python
Position.from_report(
    report,
    fund: float,
    odd_lot: bool = False
) -> Position
```

**Parameters:**
- `report` (Report, required): Backtest report object from `sim()`
- `fund` (float, required): Total capital for position sizing (in the broker's account currency)
- `odd_lot` (bool, default=False): Enable odd lot (零股) trading for smaller positions

**Returns:**
- `Position`: List of position dictionaries with stock_id, quantity, and order_condition

**Example:**
```python
from finlab import backtest
from finlab.online.order_executor import Position

report = backtest.sim(position, resample="M")

# Standard lot trading
position = Position.from_report(report, fund=1000000)
print(position)
# [{'stock_id': '2330', 'quantity': 1, 'order_condition': <OrderCondition.CASH: 1>}]

# Odd lot trading (smaller positions)
position = Position.from_report(report, fund=1000000, odd_lot=True)
```

---

### Custom Position

Create a position manually without backtest.

**Signature:**
```python
Position(holdings: dict) -> Position
```

**Example:**
```python
# Simple position with share counts
position = Position({'2330': 1, '1101': 2})

# Fractional shares (for odd lot)
position = Position({'2330': 1, '1101': 1.001})
```

---

### Position Arithmetic

Combine or modify positions using arithmetic operations.

**Subtraction:**
```python
# Remove stocks from position
new_position = position - Position({'2330': 1})
```

**Addition:**
```python
# Add stocks to position
new_position = position + Position({'1101': 1})
```

**Multi-strategy combination:**
```python
# Combine positions from multiple strategies
position1 = Position.from_report(report1, fund=500000)
position2 = Position.from_report(report2, fund=500000)
total_position = position1 + position2
```

---

## Pre-Open Data Checks *(v2.1.0)*

Before generating orders, confirm the trading calendar and data freshness:

```python
import datetime as dt
from finlab import data

cal = data.get_calendar('tw')               # announced TW sessions incl. future holidays; no login or quota
today = dt.date.today()
print(cal.is_session(today), cal.next_session(today), cal.previous_session(today))

close = data.get('price:收盤價')             # readiness() checks datasets loaded in this process
status = data.readiness()                   # are they current for the next open?
if not status['final_ready']:
    print('data not ready:', status.get('reason'), status['datasets'])
```

`get_calendar()` raises `CalendarUnavailable` for unannounced years instead of guessing weekdays. `readiness()` returns `final_ready=False` for unknown evidence, unsupported markets, stale calendars, no loaded data (`reason='no_loaded_data'`), or reads restricted by `start`/`end` (`reason='historical_data_selection'`). The checks follow scheduled publication times, which are not guaranteed delivery deadlines.

---

## Broker Account Setup

### Environment Variables Summary

| Broker | Required Environment Variables |
|--------|-------------------------------|
| Esun (玉山) | `ESUN_CONFIG_PATH`, `ESUN_MARKET_API_KEY`, `ESUN_ACCOUNT_PASSWORD`, `ESUN_CERT_PASSWORD` |
| Sinopac (永豐) | `SHIOAJI_API_KEY`, `SHIOAJI_SECRET_KEY`, `SHIOAJI_CERT_PERSON_ID`, `SHIOAJI_CERT_PATH`, `SHIOAJI_CERT_PASSWORD` |
| Masterlink (元富) | `MASTERLINK_NATIONAL_ID`, `MASTERLINK_ACCOUNT`, `MASTERLINK_ACCOUNT_PASS`, `MASTERLINK_CERT_PATH`, `MASTERLINK_CERT_PASS` |
| Fubon (富邦) | `FUBON_NATIONAL_ID`, `FUBON_ACCOUNT_PASS`, `FUBON_CERT_PATH` |

---

### Esun (玉山證券)

**Import:**
```python
from finlab.online.esun_account import EsunAccount
```

**Environment Variables:**
```bash
export ESUN_CONFIG_PATH='/path/to/config.ini'
export ESUN_MARKET_API_KEY='your_market_api_key'
export ESUN_ACCOUNT_PASSWORD='your_password'
export ESUN_CERT_PASSWORD='your_cert_password'
```

**Usage:**
```python
import os

os.environ['ESUN_CONFIG_PATH'] = '/path/to/config.ini'
os.environ['ESUN_MARKET_API_KEY'] = 'your_market_api_key'
os.environ['ESUN_ACCOUNT_PASSWORD'] = 'your_password'
os.environ['ESUN_CERT_PASSWORD'] = 'your_cert_password'

acc = EsunAccount()
```

**Install SDK:**
```bash
pip install esun-trade
```

---

### Sinopac (永豐證券)

**Import:**
```python
from finlab.online.sinopac_account import SinopacAccount
```

**Environment Variables:**
```bash
export SHIOAJI_API_KEY='api_key'
export SHIOAJI_SECRET_KEY='secret_key'
export SHIOAJI_CERT_PERSON_ID='A123456789'
export SHIOAJI_CERT_PATH='/path/to/cert'
export SHIOAJI_CERT_PASSWORD='cert_password'
```

**Usage:**
```python
import os

os.environ['SHIOAJI_API_KEY'] = 'api_key'
os.environ['SHIOAJI_SECRET_KEY'] = 'secret_key'
os.environ['SHIOAJI_CERT_PERSON_ID'] = 'A123456789'
os.environ['SHIOAJI_CERT_PATH'] = '/path/to/cert'
os.environ['SHIOAJI_CERT_PASSWORD'] = 'cert_password'

acc = SinopacAccount()
```

**Install SDK:**
```bash
pip install shioaji
```

---

### Masterlink (元富證券)

**Import:**
```python
from finlab.online.masterlink_account import MasterlinkAccount
```

**Environment Variables:**
```bash
export MASTERLINK_NATIONAL_ID='A123456789'
export MASTERLINK_ACCOUNT='account'
export MASTERLINK_ACCOUNT_PASS='password'
export MASTERLINK_CERT_PATH='/path/to/cert'
export MASTERLINK_CERT_PASS='cert_password'
```

**Usage:**
```python
import os

os.environ['MASTERLINK_NATIONAL_ID'] = 'A123456789'
os.environ['MASTERLINK_ACCOUNT'] = 'account'
os.environ['MASTERLINK_ACCOUNT_PASS'] = 'password'
os.environ['MASTERLINK_CERT_PATH'] = '/path/to/cert'
os.environ['MASTERLINK_CERT_PASS'] = 'cert_password'

acc = MasterlinkAccount()
```

---

### Fubon (富邦證券)

**Import:**
```python
from finlab.online.fubon_account import FubonAccount
```

**Environment Variables:**
```bash
export FUBON_NATIONAL_ID='A123456789'
export FUBON_ACCOUNT_PASS='password'
export FUBON_CERT_PATH='/path/to/cert.pfx'
```

**Usage:**
```python
import os

os.environ['FUBON_NATIONAL_ID'] = 'A123456789'
os.environ['FUBON_ACCOUNT_PASS'] = 'password'
os.environ['FUBON_CERT_PATH'] = '/path/to/cert.pfx'

acc = FubonAccount()
```

---

## OrderExecutor Class

The `OrderExecutor` manages order creation, modification, and cancellation.

**Import:**
```python
from finlab.online.order_executor import OrderExecutor
```

**Signature:**
```python
OrderExecutor(
    position: Position,
    account: BrokerAccount
) -> OrderExecutor
```

**Parameters:**
- `position` (Position, required): Target position to execute
- `account` (BrokerAccount, required): Connected broker account instance

**Example:**
```python
from finlab.online.order_executor import OrderExecutor, Position
from finlab.online.sinopac_account import SinopacAccount

# Setup
position = Position.from_report(report, fund=1000000)
acc = SinopacAccount()
executor = OrderExecutor(position, account=acc)
```

---

### OrderExecutor Methods

#### show_alerting_stocks()

Display stocks that require pre-deposit (處置股/警示股).

```python
executor.show_alerting_stocks()
```

---

#### create_orders()

Create orders based on the target position.

**Signature:**
```python
create_orders(view_only: bool = False) -> None
```

**Parameters:**
- `view_only` (bool, default=False): If True, preview orders without execution

**Example:**
```python
# Preview orders first (recommended)
executor.create_orders(view_only=True)

# Execute orders
executor.create_orders()
```

---

#### update_order_price()

Update limit price for pending orders.

```python
executor.update_order_price()
```

---

#### cancel_orders()

Cancel all pending orders.

```python
executor.cancel_orders()
```

---

#### generate_orders() / generate_order_entries()

*(v1.5.9)* Compute the order diff (current position → target position) without sending anything to the broker. Useful for dry-runs, order inspection, and custom execution pipelines.

**Signature:**
```python
executor.generate_orders(
    as_entries: bool = False,
    quantity_type: str = 'shares'   # 'shares' | 'lots' | 'weight'
) -> list[dict] | list[OrderEntry]

executor.generate_order_entries() -> list[OrderEntry]  # typed convenience API
```

**Example:**
```python
# Raw dict list (legacy)
orders = executor.generate_orders()

# Typed OrderEntry list — preferred for new code
entries = executor.generate_order_entries()
for e in entries:
    print(f"{e.stock_id} {e.action} qty={e.quantity} @ {e.price}")

# Use weight units (fraction of fund) instead of share counts
w_orders = executor.generate_orders(quantity_type='weight')
```

---

## Check Account Position

Query current holdings from broker.

```python
# Get current holdings
print(acc.get_position())
```

---

## Real-Trade Review — `TradeReview` *(v2.2.4)*

`finlab.portfolio.TradeReview` rebuilds daily holdings, cash, NAV, time-weighted returns, FIFO trades and a FinLab `Report` from **actual fills**, so you can compare live results against the backtest.

**Signature:**
```python
TradeReview(
    fills: pd.DataFrame,                 # time, stock_id, action, quantity (shares), price; optional fee, tax (NTD amounts)
    prices: pd.DataFrame,                # UNADJUSTED close, dates x stock IDs, for daily valuation
    *,
    start=None,                          # default: first fill day
    end=None,                            # default: last price date
    initial_holdings: dict[str, float] | None = None,  # shares held before start
    initial_cash: float | None = None,   # account mode; None = holdings mode (see below)
    cash_events: pd.DataFrame | None = None,         # time, amount (+ = in), type ('deposit'/'withdrawal' = external flow; others = P&L)
    corporate_actions: pd.DataFrame | None = None,   # stock_id, date, cash (per share), ratio (share multiplier)
    market: Market | None = None,
    limitations: list[str] = (),
)
TradeReview.from_account(account, start, *, prices=None, initial_cash=None,
                         cash_events=None, corporate_actions='auto', fee_ratio=None, **broker_kwargs)
```

**Example (fills you already have):**
```python
import pandas as pd
from finlab import data
from finlab.portfolio import TradeReview

fills = pd.DataFrame({
    'time':     ['2026-07-01 09:05', '2026-07-01 09:10', '2026-08-03 13:20'],
    'stock_id': ['2330', '2317', '2330'],
    'action':   ['buy', 'buy', 'sell'],     # also 'B'/'S', Action.BUY
    'quantity': [1000, 2000, 500],          # shares, not lots
    'price':    [1105.0, 230.5, 1180.0],
    'fee':      [1574, 656, 840],           # actual NTD amounts, not rates
    'tax':      [0, 0, 1770],
})
review = TradeReview(
    fills,
    prices=data.get('price:收盤價'),
    initial_cash=2_000_000,
    cash_events=pd.DataFrame({'time': ['2026-08-15'], 'amount': [500_000], 'type': ['deposit']}),
)
print(review.summary())          # base_nav, end_nav, net_inflow, income, fees_and_tax, pnl, time_weighted_return
review.report.to_html('trade_review.html')

target = pd.DataFrame({'2330': [1000], '2317': [2000]}, index=pd.to_datetime(['2026-08-31']))
print(review.compare(target, kind='shares'))   # date, stock_id, target, actual, diff
```

**From a logged-in broker account:** `TradeReview.from_account(SinopacAccount(), start='2026-01-01')` fetches fills and works backward from current broker holdings to the starting holdings. Supported: `SinopacAccount`, `PocketAccount`, `FubonAccount`, `MasterlinkAccount`, `FugleAccount`, `SchwabAccount`, `BinanceAccount`. Other accounts raise `NotImplementedError`; use the constructor instead. For TW accounts, `corporate_actions='auto'` loads ex-dividend and split data from FinLab. The constructor applies no corporate actions unless you pass them.

| Mode | When | NAV and flows |
|---|---|---|
| Account (`initial_cash` given) | Broker reports cash, deposits and withdrawals | NAV = cash + market value. Only `deposit`/`withdrawal` count as external flows. |
| Holdings (`initial_cash=None`) | TW settlement accounts with no cash history | Buy amounts count as invested before the open. Sale proceeds and dividends count as withdrawn after the close. |

| Output | Contents |
|---|---|
| `review.daily` | `cash`, `market_value`, `nav`, `inflow`, `outflow`, `income`, `fees_and_tax`, `return`, `pnl` |
| `review.holdings` / `review.weights` | Daily shares / NAV weights per stock |
| `review.trades` | FIFO-matched trades; open lots have `exit_date = NaT` |
| `review.returns` / `review.creturn` | Time-weighted daily / cumulative returns |
| `review.limitations` | Data gaps found, e.g. rebuilt holdings that disagree with the broker |

`review.report` raises `ValueError` if any held stock lacks a closing price. Check `review.limitations` first.

---

## PortfolioSyncManager — Typed Data APIs

*(v1.5.9)* In addition to `to_file()` / `from_file()` / `to_cloud()` / `from_cloud()`, `PortfolioSyncManager` now exposes typed data access:

```python
from finlab.portfolio import PortfolioSyncManager
from finlab.schemas import PortfolioData

pm = PortfolioSyncManager(...)

# Untyped dict — legacy
raw = pm.get_data()
pm.set_data(raw)

# Typed — preferred for new code
data: PortfolioData = pm.get_data_typed()
pm.set_data_typed(data)
```

The typed pair validates the payload against the `PortfolioData` schema at the boundary, so schema regressions surface immediately instead of propagating into persisted state.

**Budget check** *(v2.2.2)*: `pm.update(...)` raises `finlab.exceptions.PortfolioError` when it rebuilds a positive-weight strategy and `total_balance` minus the market value of holdings in strategies not being updated is ≤ 0. The whole update aborts. No config or history is written, no positions are removed or rebuilt, and no strategy's stop-loss/take-profit is processed. Raise `total_balance` above the lower bound quoted in the error and rerun `update()`.

---

## Realtime Position Streaming — `PositionStreamMixin`

*(v2.0.0)* `RealtimeProvider` subclasses auto-inherit a `subscribe_positions()` / `on_position()` interface for push-based position updates. Internally this uses a hybrid strategy — initialize via `get_position()`, update in real-time via broker Fill events, and periodically reconcile via polling (default 30s).

```python
from finlab.online.sinopac_account import SinopacAccount

acc = SinopacAccount()

# Subscribe to live position updates
def on_update(update):
    # update is a PositionUpdate dataclass; snapshot_key() deduplicates
    print(update.snapshot_key(), update.symbol, update.quantity)

acc.subscribe_positions()
acc.on_position(on_update)
```

**Why hybrid:** Fill-event streams catch fills with sub-second latency but can miss events during disconnects; polling catches missed state but is slow. Combining both gives real-time responsiveness without state divergence risk.

The `PositionUpdate` dataclass has a `snapshot_key()` method that you should use for deduplication in your consumer — duplicate messages are expected when fills and polling reconciliation arrive close together.

---

## Cloud Strategy Deployment — `python -m finlab cloud` *(v2.0.1)*

For users who want a strategy to run automatically every trading day without managing their own server, FinLab ships a CLI that deploys the strategy to the `finlab-auto-update` Cloud Functions runtime (Asia/Taipei schedule). This is operational tooling — it does not place broker orders by itself; pair it with the `OrderExecutor` workflow above if you want the cloud run to fire live trades.

### Command Map

```bash
python -m finlab cloud deploy <sid>     # upload strategy and (optionally) schedule it
python -m finlab cloud get    <sid>     # inspect metadata + inline source
python -m finlab cloud list             # list deployed strategies
python -m finlab cloud run    <sid>     # trigger one ad-hoc execution
python -m finlab cloud logs   <sid>     # recent execution history
python -m finlab cloud schedule set <sid> HH:MM   # e.g. schedule set my-strat 14:30
python -m finlab cloud schedule delete <sid>
python -m finlab cloud delete <sid>     # remove strategy + schedule (history preserved)
python -m finlab cloud status           # monthly budget / tier usage
```

`<sid>` is your strategy identifier (free-form, but must stay stable — used as the Firestore document id).

### Deploy

**Single file:**
```bash
python -m finlab cloud deploy my_value_strategy \
  --code strategy.py \
  --time 14:35 \
  --tier s
```

**Multi-file project (zip):**
```bash
python -m finlab cloud deploy my_value_strategy \
  --zip project.zip --entry-script main.py \
  --time 14:35 --tier m
```

**Key flags:**
- `--code <file.py>` *or* `--zip <archive.zip> --entry-script <relpath.py>` — choose one. Zip is for multi-file strategies.
- `--time HH:MM` — daily trigger time in Asia/Taipei. Omit to deploy without a schedule (run manually with `cloud run`).
- `--tier {s,m,l,xl}` — compute tier. Larger tiers cost more per run; check `cloud status` for available tiers and monthly budget.
- `--contest` / `--no-contest` / `--contest-alias <name>` — participate in FinLab's strategy contest under an alias.

### Inspect, Trigger, Monitor

```bash
# Read back what's deployed
python -m finlab cloud get my_value_strategy --code-only > strategy_remote.py

# Trigger a one-off run (rate-limited: 5/hour per strategy)
python -m finlab cloud run my_value_strategy --tier m

# See the last few executions including stdout/stderr
python -m finlab cloud logs my_value_strategy --show-output
```

`cloud list` and `cloud logs` give you SID, tier, schedule, type, contest status, runtime, cost, and queue time per run.

### Budget Awareness

```bash
python -m finlab cloud status                # current month
python -m finlab cloud status --month 2026-04
```

Returns monthly budget, used quota, remaining quota, available tiers, and per-tier execution counts. Check this before deploying a high-tier strategy; you cannot exceed the monthly budget.

### Typical Flow

```bash
# 1. Local development — finalize strategy in strategy.py and verify with sim()
python -m finlab cloud deploy momentum_top10 --code strategy.py --time 13:35 --tier s

# 2. Confirm it ran today
python -m finlab cloud logs momentum_top10 --show-output

# 3. Iterate: redeploy overwrites the existing SID's code and schedule
python -m finlab cloud deploy momentum_top10 --code strategy.py --time 14:00 --tier m

# 4. Pause for the weekend
python -m finlab cloud schedule delete momentum_top10
```

---

## Related References

- [backtesting-reference.md](backtesting-reference.md): Backtest configuration and report generation
- [best-practices.md](best-practices.md): Coding patterns and anti-patterns
