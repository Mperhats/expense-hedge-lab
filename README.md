# Expense hedge lab

A reproducible stress test of Vitalik Buterin's idea that households could hold growth assets and buy claims that pay when their expenses rise. **The CPI and S&P 500 histories are observed. The prediction contracts, premiums, fills, counterparties and payouts are hypothetical. This is not proof that such a market exists or that everyone wins.**

![Historical reserve purchasing power and modeled hedge cash flows](figures/purchasing_power.png)

## Read this first

This is a retrospective, ex-post calibration: **2024 survey weights are applied to a Dec. 2019 starting basket**, so this is not an implementable 2019 trading strategy or out-of-sample result. A national-average U.S. spending scenario starts with a $6,544.58 monthly basket and a reserve equal to 12 months of that basket in December 2019. A growth portfolio allocates 60% to a price-only S&P 500 index and 40% to rolling 3-month Treasury yield. A cash comparator earns that Treasury yield; it is **not** a stablecoin return. The hedged portfolio starts with the same growth allocation, pays a premium monthly and receives a modeled CPI claim payoff at each 12-month maturity. Portfolio assets are *not spent* on the actual expenses. The outcome is the number of current monthly baskets each reserve could purchase.

| Dec. 2019 to Dec. 2024, historical inputs + modeled contracts | Cash/T-bill | Growth | Growth + hedge |
|---|---:|---:|---:|
| Final months of basket affordable | 11.05 | 14.96 | 15.53 |
| Worst peak-to-trough fall in affordable months | -13.6% | -19.9% | -17.0% |

Under the base **assumed** 1.2%-of-notional annual premium and 3% annual CPI strike, the hedger pays **$4,712** in premiums and receives **$8,307** in modeled payouts. The writer loses **$3,595** in net contract cash flow before expenses and investment returns. The hedger's average monthly log of affordable months rises by **0.01862** against growth alone. This particular historical path does **not** make both parties profit. With an assumed 3.0% premium, the writer earns **$3,473**, while the hedger's log measure is **0.02139 lower** than growth alone. These are realized path outcomes under invented contract prices, **not** expected returns, observed trades or actuarially fair rates. No claim that the hedge is a good deal can be made until quoted prices, risks and opposing demand are measured.

## Vitalik's exact toy example

The original [post](https://farcaster.xyz/vitalik.eth/0x652b9fac) stipulates two equally likely election outcomes. A biotech stock's price is uniformly distributed from $80 to $120 in Purple and $60 to $100 in Yellow. A Yellow contract that nets +$10 in Yellow and -$10 in Purple moves both post-hedge distributions to $70-$110. Numerical integration of expected log wealth gives a certainty-equivalent gain of **$0.582** in the toy model. `vitalik_example()` reproduces this using a 100,001-point quadrature. It is a hypothetical two-state calculation, **not** a historical election backtest; the post provides no observable election/stock pair from which to estimate counterfactual election causality, probability or contract pricing.

## Data and design

- [BLS CPI-U](https://www.bls.gov/cpi/overview.htm): seasonally adjusted national rent of primary residence (`CUSR0000SEHA`), food at home (`CUSR0000SAF11`), gasoline (`CUSR0000SETB01`) and all items (`CUSR0000SA0`). Snapshot: Dec. 2019-Dec. 2024. Raw data from [BLS API](https://www.bls.gov/developers/api_signature_v2.htm), committed in [`data/cpi.csv`](data/cpi.csv).
- [BLS 2024 Consumer Expenditure Survey](https://www.bls.gov/opub/reports/consumer-expenditures/2024/home.htm), annual $78,535 across all consumer units. Scenario weights use reported annual rents $5,660 (7.2%), food at home $6,224 (7.9%), gasoline $2,645 (3.4%), residual $64,006 (81.5%) priced using all-items CPI. This is **not a renter's expense distribution**. The all-items proxy includes some rent/food/gas, creating imperfect category mapping in the residual; it is not a true custom household index.
- [FRED S&P 500 (`SP500`)](https://fred.stlouisfed.org/series/SP500) price index and [3-month Treasury yield (`DGS3MO`)](https://fred.stlouisfed.org/series/DGS3MO) daily data converted to monthly last observations. `SP500` excludes dividends and costs. Treasury yield is treated as a simple annualized yield divided by 12; no bond price duration, taxes, expenses or borrowing costs.
- [`data/provenance.json`](data/provenance.json) records public series identifiers, retrieval timestamp and SHA-256 snapshot hashes. Existing snapshots keep the run stable. `scripts/fetch_data.py` refreshes snapshots, but refreshed outputs should not be described with the historical results above without rerunning and updating this README.

A $6,544.58 annualized one-month spending notional enters a new 12-month category CPI call ladder each month. At maturity the cash payoff is `monthly_base_spend * category_weight * max(0, CPI_t/CPI_(t-12) - 1 - strike - basis_gap)` summed over four categories. A new one-month tranche is purchased every month. The first twelve months pay premiums and have no maturing payout. Premium per month is `monthly_base_spend * annual_premium_rate`. There is no use of future CPI index values when opening a contract in the ledger, but the scenario weights were selected from 2024 survey data after the 2019 start date. **This is a deliberately stylized annual inflation call, not a currently listed or priced contract.** A real market would specify observation dates, publication lags, revision policy, settlement venue, currency/collateral, spreads, margin, caps and counterparty default treatment.

For every monthly contract, hedger net cashflow is `payout - premium`; writer net cashflow is its opposite. This zero-sum identity is tested and does not establish that both are happy. A writer could rationally accept negative cashflow on one historical path for diversification or because expected P&L is positive across possible future paths. Neither of those motivations is measured here. The writer's portfolio opportunity cost and risk-adjusted utility are not modeled. Prices are assumptions, not fitted from future CPI.

## Reproduce

Python 3.10+; dependencies listed in [`pyproject.toml`](pyproject.toml).

```bash
python -m venv .venv
source .venv/bin/activate
pip install -e '.[dev]'
expense-hedge --data-dir . --out outputs
pytest -q
```

The entry point writes `outputs/summary.json`, `outputs/ledger.csv` and `outputs/purchasing_power.png`. Vary assumptions:

```bash
expense-hedge --data-dir . --out outputs/expensive --premium 0.03
expense-hedge --data-dir . --out outputs/basis --basis-gap 0.02
expense-hedge --data-dir . --out outputs/higher-strike --strike 0.06
python scripts/fetch_data.py # explicitly refresh sources; rerun analysis after refresh
```

`--premium 0.03` means 3% of protected monthly spending each month, corresponding to a 3% annual premium on a ladder of 12 one-month spending tranches. Outputs compare four sensitivity cases plus a 2-percentage-point basis gap. A 6% strike makes hedging worse on this path. We did not fit any hedging parameters against an out-of-sample outcome. The `basis_gap` parameter is an abstract deductible in CPI growth, **not** a measured household-versus-CPI basis error.

## What this cannot establish

1. No historical price quotes for these hypothetical contract baskets means this cannot estimate true premiums, expected writer return, liquidity or clearing conditions. The payoffs are repriced using observed realized CPI only, and the reported outcomes are in-sample scenarios, not out-of-sample forecasts.
2. National CPI is not any person's rent lease or grocery cart. BLS releases have delays and revisions. Real hedges could miss private bills, local prices and sudden household-specific expenses.
3. This comparison mixes a price-only equity index, simulated yield on cash and simple monthly rebalancing. It ignores equity dividends, fees, tax, spread, custody risk, operational risks, capital constraints, option mark-to-market, and settlement timing. No ETH or stablecoin price is used, so no claim about replacing fiat follows.
4. A premium level at which a seller earned money on this path made the buyer worse off than growth alone on the stated mean-log measure. Positive-sum utility needs the counterparty's preferences and forecasts, and actual contract quotes. A useful next study would pair anonymized household bills with local indexes, obtain executable quotes, fit risk premiums using an expanding training window, and test held-out periods.

## Files

- `src/expense_hedge/simulation.py`: formulas, data validation, ledger, sensitivities and chart.
- `tests/test_simulation.py`: source shape, toy calculation, conservation, payoff monotonicity and future-data isolation.
- `data/`: committed source snapshots and provenance.
- `figures/purchasing_power.png`: generated chart on observed index paths.
- `outputs/summary.json`, `outputs/ledger.csv`: exact machine-readable run outputs.

Data retrieved Sept. 25, 2026. Research demonstration only, not investment advice.
