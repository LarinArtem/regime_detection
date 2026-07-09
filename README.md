# Live Volatility Regime Detection Dashboard

A real-time market dashboard (Python + Tkinter + Matplotlib) that streams tick data from **Interactive Brokers**, aggregates it into OHLC bars on the fly, and classifies each bar into one of three volatility regimes — low, medium, high — using a Markov regime-switching model calibrated on historical data.

## Overview

`project_6_regime.py` is a single-file desktop application. Connect it to IB Trader Workstation, enter a symbol, and it renders a live candlestick chart whose background is color-coded by the detected volatility regime (green = low, amber = medium, red = high). The regime model is calibrated from historical bars at startup and can be recalibrated on demand.

## Architecture

The application is organized around four classes:

### `IBApp` (connectivity)
Subclasses `EWrapper` + `EClient` from the official `ibapi` package. It handles the TWS socket on a background thread, filters informational error codes, receives streaming ticks (`tickPrice`: last/bid/ask) and historical bars, and forwards live prices to the UI through a callback. A `threading.Event` signals when a historical download completes.

### `OHLCBar` (data model)
A lightweight bar object built from raw ticks: it tracks open/high/low/close, tick count, and exposes a `volatility` property defined as the normalized bar range `(high − low) / close`. Each bar also carries its assigned regime label.

### `MarkovRegime` (the model)
A 3-state regime-switching model over bar volatility:

- **Calibration** — `calibrate(hist_bars)` estimates per-regime volatility distributions from historical bars (splitting the volatility distribution into low/medium/high states) and a state-transition structure.
- **Inference** — for each new bar, `_gaussian_likelihood(vol, regime)` evaluates how likely the observed volatility is under each state's Gaussian; combined with the transition probabilities from the previous state, the model updates its state-probability vector and picks the most likely regime. The Markov prior gives the classification persistence — regimes don't flip on a single noisy bar.
- Each regime maps to chart colors (`#3fb950` / `#d29922` / `#f85149`) and matching dark background shades.

### `LiveMarketDashboard` (UI)
A dark-themed Tkinter interface embedding a Matplotlib canvas (`FigureCanvasTkAgg`):

- Connection controls, symbol entry, start/stop stream, and a **Recalibrate** button.
- A background **bar-manager loop** rolls ticks into fixed-interval OHLC bars (kept in a `deque`).
- A **chart-update loop** redraws the candlestick chart with regime-colored bands and updates live statistics (last price, bid/ask, current regime, state probabilities).

## Design decisions

- **Threading** — three concerns run on separate threads (IB socket, bar aggregation, chart refresh) with the GUI on the main thread; queues/callbacks keep them decoupled.
- **Range-based volatility** — using bar range rather than close-to-close returns gives a responsive intrabar volatility signal suitable for tick-level streaming.
- **Probabilistic regimes over hard thresholds** — the Markov formulation smooths regime assignments and exposes state probabilities, not just a label.
- **Graceful data degradation** — delayed market data (IB error 10167) is detected and accepted, so the app works without live-data subscriptions.

## Requirements & setup

1. Install IB Trader Workstation or IB Gateway; enable API access (paper-trading port 7497 by default).
2. Install dependencies:
   ```bash
   pip install ibapi numpy matplotlib
   ```
3. Run:
   ```bash
   python project_6_regime.py
   ```
4. Click **Connect**, enter a symbol (e.g., `SPY`), and start the stream. Use **Recalibrate** after enough bars have accumulated to refresh the regime model.

## Possible extensions

- Replace the calibration heuristic with full Baum–Welch (EM) estimation of the HMM.
- Persist bars and regimes to disk for later backtesting.
- Use regimes as inputs to a strategy layer (e.g., widen stops or cut size in the high-vol state).
