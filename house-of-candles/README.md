# house-of-candles

## Learning Objectives

- ClickHouse

## Abstract

In this project, you will enhance your **Real-Time Market Data Processing System** ([`marketflow`](https://github.com/alem-platform/backend/tree/main/marketflow)) by replacing one of its existing technologies with a more efficient alternative.

This change will improve the system's performance, enabling it to process market data more rapidly. Through this hands-on experience, you will also gain valuable skills in quickly and effectively integrating new technologies into an existing software system.

## Context

At work, sometimes you have to replace one technology with another. C'est la vie.

Business needs have changed and we need to add slightly expanded data to our database faster.  
Now we need to change `marketflow` a bit and replace PostgreSQL with [ClickHouse](https://clickhouse.com/).

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

This project, `house-of-candles`, serves as an enhancement to the `marketflow` application. The original `marketflow` system processes real-time market data and provides a RESTful API.

The primary objective of the `house-of-candles` enhancement is to replace the `PostgreSQL` database with `ClickHouse` specifically for storing market data values (e.g., candle data). This involves modifying `marketflow`'s persistence layer to efficiently interact with ClickHouse for storing these values. The project must maintain the established principles of clean code, maintainability, and scalability.

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

#### Real-Time Data Processing:

- Use **Redis** to cache recent price data.
- Keep the latest price for all price updates for at least the last minute for each trading pair from all exchanges.
- Add data to ClickHouse every 10 seconds, save each received value.
- Ensure outdated data (older than two minutes) is purged from Redis.

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

1. **Get OHLC data for a specific exchange and symbol:**
- `GET /prices/ohlc/{exchange}/{symbol}`  
- **Description:** Retrieves a series of OHLC (candlestick) data points for a given symbol on a specific exchange over a defined time range and interval.  
- **Query Parameters:**
  - `interval={duration}` (Required): The time interval for each candlestick (e.g., `10s`, `1m`, `5m`, `1h`, `1d`). This should match the intervals you are storing in ClickHouse.
  - `startTime={timestamp}` (Optional): The start timestamp (inclusive) for the data range (e.g., [ISO 8601](https://www.cl.cam.ac.uk/~mgk25/iso-time.html) format `YYYY-MM-DDTHH:MM:SSZ` or Unix timestamp). If omitted, might default to a recent period or require limit.
  - `endTime={timestamp}` (Optional): The end timestamp (exclusive or inclusive - be consistent!) for the data range. If omitted, defaults to the current time.
  - `limit={integer}` (Optional): Limits the number of candlestick data points returned. Useful for fetching the most recent 'N' candles. If startTime is provided, it limits results starting from that time. If only endTime is provided, it limits results ending at that time. If neither startTime nor endTime is provided, it typically returns the latest N candles.
- Example:  
`GET /prices/ohlc/exchange2/BTCUSDT?interval=1m&limit=100` (Get the last 100 one-minute candles for BTC/USDT on `exchange2`)
- Example:  
`GET /prices/ohlc/exchange3/ETHUSD?interval=1h&startTime=2023-10-26T00:00:00Z&endTime=2023-10-27T00:00:00Z` (Get hourly candles for a specific day)
- Response: An array of JSON objects, each representing a candlestick:
```
[
  {
    "timestamp": "2023-10-27T10:00:00Z", // Start time of the interval
    "open": 41500.50,
    "high": 41550.20,
    "low": 41480.00,
    "close": 41530.90
  },
  {
    "timestamp": "2023-10-27T10:01:00Z",
    "open": 41530.90,
    "high": 41600.00,
    "low": 41525.50,
    "close": 41595.10
  }
  // ... more candles
]
```

2. **Get OHLC data for an aggregated symbol (across exchanges):**
   `GET /prices/ohlc/{symbol}`
- **Description:** Retrieves a series of OHLC data points for a given symbol, aggregated across all tracked exchanges.
- **Query Parameters:** Same as above (interval, startTime, endTime, limit).
- **Example:**  
`GET /prices/ohlc/BTCUSD?interval=5m&limit=50`  
- **Response:** Same format as the exchange-specific endpoint, but the OHLC values represent the aggregated data according to your defined strategy.

These endpoints provide a standard way to query the time-series OHLC data necessary for drawing Japanese candlestick charts. Remember to clearly document the required interval parameter and how startTime, endTime, and limit interact.


### Configuration

The application should read configuration from a file. The configuration should include the following parameters:
- ClickHouse connection details (host, port, user, password, database and etc.).
- Redis connection details (host, port, password and etc.).
- Connection details (host, port, etc.) for the provided exchange emulations and your test implementation.

### Logging

- Use Go's `log/slog` package for logging throughout the application.
- Log significant events, errors information with appropriate levels (`Info`, `Warning`, `Error`).
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
