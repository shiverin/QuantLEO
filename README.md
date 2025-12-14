# QuantLEO
quanting into janestreet
***

# QuantLEO: Autonomous Portfolio Management System (PMS)

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](https://github.com/yourusername/quantscale)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-MIT-green)](./LICENSE)
[![Status](https://img.shields.io/badge/status-alpha-orange)]()

**QuantScale** is a full-stack algorithmic trading platform designed to simulate the lifecycle of a quantitative hedge fund. It enables users to parameterize strategies, perform rigorous backtesting using vectorized computation, and execute paper trades via the Alpaca API.

Unlike retail trading bots that rely on technical indicators (RSI, MACD), QuantScale utilizes **stochastic calculus**, **convex optimization**, and **unsupervised machine learning** to solve for optimal portfolio allocation while strictly adhering to risk constraints.

---

## 🏗 System Architecture

The system is architected as a set of microservices orchestrated via Docker Compose, ensuring separation of concerns between the user interface, API gateway, and heavy computational workers.

```mermaid
graph TD
    Client(Next.js Dashboard) -->|REST/WebSockets| API(FastAPI Gateway)
    API -->|Async Task| Redis(Message Broker)
    Redis -->|Queue| Worker(Celery Quant Engine)
    
    subgraph "Quant Engine"
        Worker -->|Vectorized Backtest| VBT(VectorBT)
        Worker -->|Optimization| CVX(CVXPY)
        Worker -->|Inference| SKL(Scikit-Learn/HMM)
    end
    
    Worker -->|Fetch/Stream| Alpaca(Alpaca Markets API)
    Worker -->|Persist| DB(PostgreSQL + TimescaleDB)
    DB --> API
```

### Core Components
*   **Frontend:** Next.js (React/TypeScript) with Plotly.js for 3D volatility surface rendering and interactive financial charting.
*   **Backend:** FastAPI for high-throughput asynchronous request handling.
*   **Task Queue:** Redis & Celery to handle long-running optimization tasks without blocking the HTTP main loop.
*   **Database:** PostgreSQL with TimescaleDB extension for efficient time-series storage and windowing functions.

---

## 🧠 Quantitative Methodologies (Deep Dive)

This project moves beyond heuristic trading by implementing rigorous mathematical models for signal generation and portfolio construction.

### 1. State-Space Modeling (Dynamic Pairs Trading)
Standard pairs trading relies on OLS regression, which assumes a static hedge ratio ($\beta$). However, market relationships are non-stationary.
*   **Implementation:** We utilize a **Kalman Filter** to dynamically estimate the "hidden state" (the cointegration coefficient) of a pair.
*   **The Math:**
    We model the hedge ratio $\beta_t$ as a random walk:
    $$ \beta_t = \beta_{t-1} + \omega_t, \quad \omega_t \sim \mathcal{N}(0, Q) $$
    The observed spread $y_t$ is:
    $$ y_t = x_t \beta_t + \epsilon_t, \quad \epsilon_t \sim \mathcal{N}(0, R) $$
    This allows the algorithm to adapt to structural breaks in correlation instantly, reducing drawdown during divergence events.

### 2. Market Regime Detection (Unsupervised Learning)
Strategies that work in bull markets often fail in high-volatility sideways markets.
*   **Implementation:** A **Gaussian Hidden Markov Model (HMM)** is trained on VIX and SPY returns to classify the market into latent states (e.g., *Low Vol/Bull*, *High Vol/Panic*).
*   **Application:** The risk manager module adjusts the leverage ratio dynamically based on the detected state probability $P(S_t | \text{Data})$.

### 3. Robust Portfolio Optimization
We reject Mean-Variance Optimization (MVO) due to its sensitivity to input noise (estimation error maximization).
*   **Implementation:** We use **Convex Optimization (`cvxpy`)** to solve for weights $w$.
*   **Objective:** Maximize Utility subject to **L1 Regularization** (sparsity) and **Turnover Constraints** (transaction costs).
    $$ \text{maximize} \quad w^T \mu - \lambda w^T \Sigma w - \gamma ||w||_1 $$
    $$ \text{subject to} \quad \sum w_i = 1, \quad w_i \geq 0, \quad ||w_t - w_{t-1}||_1 \leq \delta $$
*   **Covariance Cleaning:** We apply **Ledoit-Wolf Shrinkage** to the covariance matrix $\Sigma$ to mitigate off-diagonal noise.

---

## ⚡ Performance Engineering

*   **Vectorized Backtesting:** Utilizing `vectorbt` and `numpy` broadcasting to run 5-year backtests on S&P 500 constituents in sub-second timeframes.
*   **Look-Ahead Bias Prevention:** The backtesting engine enforces strict time-indexing. Signals generated at $t$ are executed at $t+1_{open}$ to simulate realistic slippage and execution delay.
*   **Asynchronous Execution:** Heavy math jobs are offloaded to Celery workers, ensuring the API remains responsive to heartbeat checks and UI updates.

---

## 🛠 Tech Stack

| Component | Technology | Rationale |
| :--- | :--- | :--- |
| **Language** | Python 3.10+ | Standard for Quant Research. |
| **Math** | `numpy`, `pandas`, `scipy` | Core linear algebra and statistical operations. |
| **Optimization** | `cvxpy` | Solving convex optimization problems with constraints. |
| **Filtering** | `pykalman` | Dynamic Bayesian state estimation. |
| **Backtesting** | `vectorbt` | High-performance vectorized backtesting. |
| **API** | FastAPI | Async Python web framework. |
| **Brokerage** | Alpaca API | Market data and Paper Trading execution. |
| **Containerization** | Docker | Consistent environment across Dev/Prod. |

---

## 🚀 Getting Started

### Prerequisites
*   Docker & Docker Compose
*   Alpaca API Keys (Paper Trading)

### Installation

1.  **Clone the repository**
    ```bash
    git clone https://github.com/yourusername/quantscale.git
    cd quantscale
    ```

2.  **Environment Setup**
    Create a `.env` file in the root directory:
    ```env
    ALPACA_API_KEY=your_key_here
    ALPACA_SECRET_KEY=your_secret_here
    POSTGRES_USER=quant
    POSTGRES_PASSWORD=password
    ```

3.  **Build and Run**
    ```bash
    docker-compose up --build
    ```

4.  **Access the Dashboard**
    Navigate to `http://localhost:3000` to access the UI.
    API Docs (Swagger) available at `http://localhost:8000/docs`.

---

## 📉 Example Usage Flow

1.  **Select Universe:** User selects "Energy Sector" (XOM, CVX, COP, etc.).
2.  **Configure Strategy:** Select "Kalman Pair Switching" with a Risk Target of 10% Volatility.
3.  **Simulate:** The system pulls 2 years of OHLCV data, computes the geometric Brownian motion of the spread, and runs the backtest.
4.  **Analyze:** View the Tear Sheet (Sharpe, Sortino, Max Drawdown).
5.  **Deploy:** Click "Start Paper Trading." The Celery worker initializes a cron job to rebalance the portfolio every market open.

---

## ⚠️ Disclaimer

This software is for educational and research purposes only. It is not financial advice. The "Paper Trading" environment simulates trades with fake money. Do not use this code for live trading with real capital without extensive testing and risk management auditing.

---

*Project maintained by [Your Name]. Open for collaboration.*
