# Bitcoin Options Iron Condor Strategy

## Strategy Overview

This project implements and backtests a **short-term, delta-neutral Iron Condor strategy** on Bitcoin options traded on Deribit.

The strategy is designed to be **direction-neutral at entry** and to profit primarily from **theta decay** during the final days before expiration. Given the very short holding period and proximity to expiration, risk management is driven by strict delta constraints to mitigate losses caused by rapid gamma exposure.

### Strategy Parameters

* **Underlying**: Bitcoin (BTC)
* **Options type**: European vanilla options (Deribit)
* **Days to expiration (DTE) at entry**: 3 days
* **Entry time**: 6 hours before expiration
* **Holding period**: up to 4 hours

### Iron Condor Construction

All four legs are opened simultaneously:

* Short Call with delta **+0.20**
* Long Call with delta **+0.05**
* Short Put with delta **−0.20**
* Long Put with delta **−0.05**

### Exit Conditions

The position is closed upon the first occurrence of any of the following events:

1. **Profit target reached**

   * Profit equals **25% of the collected premium**, i.e. the total value of the structure has decreased by 25% relative to entry.

2. **Time-based exit**

   * **4 hours** have passed since entry.
   * Exit is executed at the current market price.

3. **Delta risk breach**

   * The absolute delta of **any short leg** (call or put) reaches **0.35**.

This delta-based stop is critical due to the extremely high gamma close to expiration, where even small spot price movements can cause rapid directional exposure.

---

## Data Structures

### `Leg`

The `Leg` class represents a single option leg and stores:

* Strike price
* Option type (`CALL` or `PUT`)
* Side (`BUY` or `SELL`)
* Premium (price)
* Delta
* Implied volatility (IV)

### `Position`

The `Position` class represents a full multi-leg options position and contains:

* Entry timestamp
* Expiration timestamp
* List of option legs
* Total entry premium

### `OptionType`

An enumeration defining the option type:

* `CALL`
* `PUT`

---

## Greeks Calculation (`Greeks` Class)

The `Greeks` class provides a complete toolkit for working with option sensitivities.

### Analytical Greeks (Black–Scholes)

* Delta
* Gamma
* Vega
* Theta
* Rho

These are calculated for **European vanilla options** using the Black–Scholes framework and can be applied to both individual options and entire option dataframes.

### Numerical Greeks (CRR Binomial Model)

To support more realistic sensitivity estimates close to expiration, the class also implements numerical Greeks:

* `crr_euro_call`
* `crr_euro_put`

These methods use the **Cox–Ross–Rubinstein (CRR)** binomial tree model:

* Number of steps: **n = 300**
* Option price is computed via the binomial tree
* Greeks are obtained using **finite difference approximations** via `binomial_greeks`

Because numerical Greeks are computationally expensive, they are **calculated only for the Iron Condor legs**, not for the entire option chain. The resulting values are reasonably close to analytical Greeks; higher accuracy can be achieved by increasing the number of tree steps.

---

## Data Loading and Preprocessing (`DataLoader`)

The `DataLoader` class is responsible for loading, preprocessing, and aggregating historical Bitcoin option data from Deribit, as well as creating **option chain snapshots** for backtesting.

### `get_data()`

This method either:

* Loads a previously saved CSV file (if available), or
* Downloads data directly from the **Deribit API**

Key features:

* Data is fetched in **batches of 10,000 trades**
* The timestamp is updated sequentially to ensure a **continuous historical range**
* Instrument names are parsed to extract:

  * Expiration date
  * Strike price
  * Option type
* Time to expiration (T) is calculated
* Options are filtered by **DTE limit** (only **3 DTE options** are retained)

---

## Option Chain Snapshots (`chain_snapshot()`)

The `chain_snapshot()` method constructs an option chain at a specific moment in time, which is used by both the strategy and the backtest.

Functionality:

* Selects options for a given expiration within a time window before a specified timestamp
* Aggregates data by strike and option type
* Keeps the **latest available values** for:

  * Mark price
  * Implied volatility
  * Greeks
  * Spot price
* Removes options with **zero implied volatility**

This process creates a realistic “market snapshot” representing the information available at the decision point.

---

## Strategy Logic (`Strategy` Class)

The `Strategy` class encapsulates strategy construction and valuation logic.

Key methods:

* `fit_legs`
  Selects the optimal options based on target deltas and strategy constraints.

* `iron_condor`
  Builds the four-leg Iron Condor structure using the selected options.

* `calc_premium`
  Calculates the total premium received at entry.

* `calc_current_value`
  Computes the current market value of the full structure.

* `check_delta_breach`
  Checks whether the delta of any short leg exceeds the predefined threshold.

---

## Backtesting Engine (`Backtest`)

The `Backtest` class performs a full historical simulation of the strategy.

### Inputs

* Historical option data (`df`)
* `DataLoader` instance
* `Strategy` instance
* Entry timing parameters
* Exit thresholds (profit, time, delta)
* Initial capital
* `position_size`

> **Note:** `position_size` directly affects equity curve and several performance metrics. It was introduced to make the framework more flexible and reusable.

### `backtest()`

The main method executes the backtest step-by-step for each expiration:

1. Sorts data by timestamp
2. Builds option chain snapshots at the entry time
3. Constructs the Iron Condor
4. Calculates entry premium
5. Tracks position value over time
6. Recalculates Greeks using **CRR binomial Greeks**
7. Checks exit conditions at each step
8. Closes the trade upon the first triggered condition
9. Computes PnL

If no exit condition is triggered earlier, the position is closed exactly **4 hours after entry**.

---

## Performance Evaluation

The backtest provides two main evaluation methods:

### `calc_sharpe()`

* Computes the **Sharpe ratio** using daily strategy returns
* Based on cumulative PnL and standard deviation

### `calc_stats()`

Returns detailed performance statistics, including:

* Total number of trades
* Win rate
* Total return
* CAGR
* Maximum drawdown
* Average profit and loss
* Breakdown of exit reasons

---

## Data Output and Assumptions

After processing, the system generates a final table containing:

* Option prices (bid/ask or mark)
* Implied volatility
* Spot price
* Greeks (Black–Scholes)
* Timestamps

The results are saved in **CSV format** for reuse.

Because **historical Greeks are not available** via the free Deribit API, all Greeks are calculated internally.

Since all Deribit options are **European**, Black–Scholes Greeks provide a reasonable approximation. The only modeling assumption is that the risk-free rate **r** is set to the **US Federal Reserve rate**.

---

## Limitations

The current implementation does **not** account for:

* Short borrowing costs
* Slippage (market exits are used)
* Trading commissions

These factors can be incorporated in future extensions.

---

