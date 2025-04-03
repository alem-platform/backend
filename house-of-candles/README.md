# house-of-candles

## Learning Objectives

- ClickHouse

## Abstract

In "house-of-candles," you’ll upgrade the `marketflow` real-time market data system by swapping out PostgreSQL for ClickHouse—a high-performance database built for analytics. Your mission is to store market price updates, cache them efficiently, and serve Japanese candlestick data through new API endpoints. This hands-on project will boost your ability to integrate cutting-edge tools into existing systems, optimize performance for large datasets, and deliver real-world financial data—all while keeping the code clean and scalable.

## Context

Imagine you’re a backend developer at a fintech startup. Your team’s `marketflow` app tracks real-time cryptocurrency prices, but the PostgreSQL database is struggling with the growing volume of data. Traders need faster access to candlestick charts for quick decisions. Your task? Replace PostgreSQL with ClickHouse to handle millions of price updates efficiently and add [OHLC](https://en.wikipedia.org/wiki/Open-high-low-close_chart) endpoints for charting—all without breaking the system. This is your chance to solve a real-world problem and level up your tech skills!

## Resources

- Read about ClickHouse [here](https://clickhouse.com/docs).

## General Criteria

- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will be graded `0` automatically.
- Your program MUST be able to compile successfully.
- Your program MUST not exit unexpectedly (any panics: `nil-pointer dereference`, `index out of range` etc.). If so, you will get `0` during the defence.
- External packages are allowed only for working with the database and cache. If you use any other external packages, you will receive a grade of `0`.
- The project MUST be compiled by the following command in the project's root directory:

```sh
$ go build -o house-of-candles .
```

- If an error occurs during startup (e.g., invalid command-line arguments, failure to bind to a port), the program must exit with a non-zero status code and display a clear, understandable error message.
  During normal operation, the server must handle errors gracefully, returning appropriate HTTP status codes to the client without crashing.
---

## Mandatory Part

### Baseline

- **Goal**: Transform `marketflow` by replacing PostgreSQL with ClickHouse for market data storage, focusing on Japanese candlestick (OHLC) data.
- **What You’ll Do**:
  - Store price updates in ClickHouse.
  - Add API endpoints for OHLC data.
  - Keep `marketflow`’s clean code and scalability principles.
- **Why ClickHouse?**: It’s fast, columnar, and perfect for aggregating large datasets—like computing OHLC from millions of price ticks.

#### Japanese Candlesticks

[Japanese candlesticks](https://www.forex.com/en/trading-academy/courses/technical-analysis/japanese-candlesticks/) are a popular type of financial chart used by traders to represent the price movement of an asset (like a stock, currency pair, or cryptocurrency) over a specific time period (e.g., 10 seconds, one minute, one hour, one day). Originating in 18th-century Japan for tracking rice prices, they provide a more visually informative picture of price action than a simple line chart.

#### ClickHouse

ClickHouse is a columnar database management system for online analytical processing. It is designed to enable fast real-time analytics on large datasets. ClickHouse is known for its high performance, scalability, and ability to handle large volumes of data efficiently.

ClickHouse is particularly well-suited for scenarios where you need to perform complex queries on large datasets quickly, making it a popular choice for analytics applications, data warehousing, and real-time data processing.

#### Outcomes:
The same application as `marketflow` but

- Store market data in ClickHouse.
- The ability to receive [OHLC](https://en.wikipedia.org/wiki/Open-high-low-close_chart) through endpoints.
- Modified table structure for data storage.

#### Where to get data from?

Get the data the same way as in the `marketflow` project.

### Data Storage, Caching, Processing

#### Real-Time Processing:

- R`edis Caching`: Store the latest price for each trading pair and exchange in Redis. Keep at least 1 minute of history.
- `ClickHouse Storage`: Batch-insert all price updates into ClickHouse every 10 seconds.
- `Cleanup`: Purge Redis data older than 2 minutes to save memory.

#### Data Storage

- Use **ClickHouse** to store aggregated data.
- Store data in a single table with the following fields:
    - `pair_name` (string) – the trading pair name.
    - `exchange` (string) – the exchange from which the data was received.
    - `timestamp` (timestamp) – the time when the data is stored.
    - `price` (float) – the price of the trading pair.

### API Endpoints

**Market Data API**

_Add new endpoints to get OHLC data for a specific exchange and symbol_

1. `GET /prices/ohlc/{exchange}/{symbol}`
- **Purpose**: Fetch OHLC candles for a symbol on one exchange.
- **Parameters**:
  - `interval` (required): e.g., "10s", "1m", "5m".
  - `startTime` (optional): [ISO 8601](https://www.cl.cam.ac.uk/~mgk25/iso-time.html) (e.g., "2023-10-27T00:00:00Z").
  - `endTime` (optional): Defaults to now if omitted.
  - `limit` (optional): Max number of candles (e.g., 100).
- Example: `GET /prices/ohlc/exchange1/BTCUSD?interval=1m&limit=10`
- Response:

```json
[
  {"timestamp": "2023-10-27T10:00:00Z", "open": 42000.0, "high": 42100.0, "low": 41950.0, "close": 42050.0},
  {"timestamp": "2023-10-27T10:01:00Z", "open": 42050.0, "high": 42200.0, "low": 42000.0, "close": 42180.0}
]
```

2. `GET /prices/ohlc/{symbol}`
- **Purpose**: Fetch OHLC candles for a symbol across all exchanges.
- **Parameters**: Same as above.
- **Aggregation**: Treat all price updates for the symbol as one stream, then compute OHLC.
- **Example**: `GET /prices/ohlc/BTCUSD?interval=5m&startTime=2023-10-27T00:00:00Z`
- **Response**: Same format as above.

These endpoints provide a standard way to query the time-series OHLC data necessary for drawing Japanese candlestick charts.

### Configuration

The application should read configuration from a file. The configuration should include the following parameters:
- ClickHouse connection details (host, port, user, password, database and etc.).
- Redis connection details (host, port, password and etc.).
- Connection details (host, port, etc.) for the provided exchange emulations and your test implementation.

### Logging

- Use Go's `log/slog` package for logging throughout the application.
- Log significant events, errors information with appropriate levels (`INFO`, `WARNING`, `ERROR`).
- Include contextual information in logs (e.g., timestamps, IDs).

### Shutdown

Implement graceful shutdown handling to ensure the application cleans up resources and exits cleanly when receiving a termination signal (e.g., `SIGINT`, `SIGTERM`).

### Usage

Your program must be able to print usage information.

Outcomes:
- Program prints usage text.

```sh
$ ./house-of-candles --help

Usage:
  house-of-candles [--port <N>]
  house-of-candles --help

Options:
  --port N     Port number
```

---

## Support

It's an easy project; you'll manage just fine.

---
## Guidelines from Author

Just switch the database from PostgreSQL to ClickHouse.  
Use Docker to run ClickHouse.

Good luck, Buddy 😊

---
## Author

This project has been created by:

Savva Savostyanov

Contacts:

- Email: [savvax@savvax.com](mailto:savvax@savvax.com)
- [GitHub](https://github.com/savvax/)
- [LinkedIn](https://www.linkedin.com/in/savvax/)
