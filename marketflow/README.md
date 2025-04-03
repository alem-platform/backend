# marketflow

## Learning Objectives

- Concurrency and Parallelism
- Concurrency Patterns
- Real-time Data Processing
- Data Caching (Redis)

## Abstract

In this project, you will build a **Real-Time Market Data Processing System**. This system will be able to process a large volume of incoming data concurrently and efficiently.

Financial markets, especially cryptocurrency exchanges, generate vast amounts of data. Traders rely on real-time data to make decisions. This project will simulate the backend of a system that processes real-time cryptocurrency price updates from multiple sources. However, to ensure the system remains functional without external dependencies, the project will also support a test mode where prices are generated locally.

This project will give you hands-on experience in **real-time data ingestion, processing, caching, and storage** while ensuring data consistency and performance using **Go's concurrency primitives**.

## Context

Imagine you are building a system for a financial firm that needs real-time price updates from multiple cryptocurrency exchanges. This system must handle multiple sources of data concurrently, process updates efficiently, and provide an API for querying recent prices and market statistics.  
~~This project is real (a similar project is used at the workplace of one of the graduates). It integrates into a more complex system and addresses some business challenges.~~
To achieve this, the project will implement:

- WebSocket-based real-time data fetching (Live Mode)
- WebSocket-based real-time test data fetching (Test Mode)
- Concurrent data processing using channels & worker pools
- Data storage in PostgreSQL
- Redis caching for quick access to frequently requested prices
- REST API for querying aggregated price data

This project mirrors real-world applications used in trading platforms, giving you practical experience with Go’s concurrency model and backend architecture.

## Resources

- Read about Redis [here](https://redis.io/docs/latest/).
- Read about concurrency [here](https://go.dev/doc/effective_go#concurrency), [here](https://go.dev/blog/pipelines), [here](https://dev.to/dwarvesf/approaches-to-manage-concurrent-workloads-like-worker-pools-and-pipelines-52ed) and [here](https://go.dev/talks/2012/concurrency.slide#1).
- Read about WebSocket [here](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API).

## General Criteria

- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will be graded `0` automatically.
- Your program MUST be able to compile successfully.
- Your program MUST not exit unexpectedly (any panics: `nil-pointer dereference`, `index out of range` etc.). If so, you will get `0` during the defence.
- External packages are allowed only for working with the database and cache. If you use any other external packages, you will receive a grade of `0`.
- The project MUST be compiled by the following command in the project's root directory:

```sh
$ go build -o marketflow .
```
- If an error occurs during startup (e.g., invalid command-line arguments, failure to bind to a port), the program must exit with a non-zero status code and display a clear, understandable error message.
  During normal operation, the server must handle errors gracefully, returning appropriate HTTP status codes to the client without crashing.
---

## Mandatory Part

### Baseline

You will create a `marketflow` application, a system designed to process market data in real-time and offer a RESTful API for accessing price updates and more. The application should follow a hexagonal software architecture. Emphasis should be placed on clean code, maintainability, and scalability to ensure a robust and efficient solution.

#### Outcomes:

- Use a hexagonal architecture:

  - Domain Layer: Defines core business logic and models.

  - Application Layer: Implements use cases and orchestrates interactions between components.

  - Adapters (Ports & Adapters Layer):

    - Web Adapter (HTTP handlers) for REST API.

    - Storage Adapter (PostgreSQL) for data persistence.

    - Cache Adapter (Redis) for caching.

    - Exchange Adapter (WebSockets) for fetching live market data.

- Support two data modes:

  - Live Mode: Fetch real-time prices from at least two cryptocurrency exchanges via WebSockets.

  - Test Mode: Generate synthetic market data locally.

- Store market data in PostgreSQL.

- Implement Redis caching for latest prices.

- Implement concurrency patterns to efficiently process multiple price updates.

  - Provide REST API endpoints for retrieving price information and statistics.

#### Where to get data from?

Choose at least two crypto exchanges [here](https://coinmarketcap.com/rankings/exchanges/) or [here](https://www.coingecko.com/en/exchanges) or [elsewhere](https://duckduckgo.com/).  
Try to avoid exchanges that require registration to receive streaming data.  
Then read the documentation of the exchanges on how to receive market data via WebSocket.

You will also receive data from the local generator for `Test Mode`.  
Run the `provided program` and receive information on ports `40101`, `40102`, `40103`.

#### Additional conditions for live data handling

- **Avoiding IP Bans:** The system should implement [proxy](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/) support to prevent IP bans when fetching data from cryptocurrency exchanges.
- **Redundancy and Failover:** If a WebSocket connection fails, the system should automatically attempt to reconnect (Note: turn off your internet connection to check 😉).
- **Proxy Support:** The system should support configurable proxies to distribute requests across multiple IPs, ensuring continuous access to exchange APIs.

##### Avoiding IP bans and proxy support

Your program should work in several ways proxy modes:

- Without a proxy.
- Switching to a proxy when data cannot be retrieved from the exchange without it
- Only proxy. All traffic from exchanges should go through the proxy.

Q: Where to get proxies?  
A: Use your own proxy or search the internet. [Here](https://github.com/Yariya/Zmap-ProxyScanner), for example, is a proxy search tool.

### Concurrency Implementation and Patterns:

This project heavily relies on concurrency to handle large volumes of real-time data. Key concurrency patterns that should be used include:

- **Fan-in:** Aggregating multiple market data streams into a single channel for centralized processing.

- **Fan-out:** Distributing incoming data updates to multiple workers to process them in parallel.

- **[Worker Pool](https://gobyexample.com/worker-pools):** Managing a set of workers that process WebSocket updates efficiently, ensuring balanced workload distribution.

Example:
```

                      +---------------+       +---------------+       +---------------+
                      |  Source 1     |       |  Source 2     |       |  Source 3     |
                      |  (Generator)  |       |  (Generator)  |       |  (Generator)  |
                      +-------+-------+       +-------+-------+       +-------+-------+
                              |                       |                       |
                              v                       v                       v
                      +-------+-------+       +-------+-------+       +-------+-------+
                      |   Fan-Out 1   |       |   Fan-Out 2   |       |   Fan-Out 3   |
                      |  (Distributor)|       |  (Distributor)|       |  (Distributor)|
                      +---+---+---+---+       +---+---+---+---+       +---+---+---+---+
                          |   |   |               |   |   |               |   |   |
          +---------------+-+-+-+-+-+-------------+-+-+-+-+-+---------------------------+
          |               | | | | | |                 | | | | | |                       |
          v               v v v v v v                 v v v v v v                       v
      +---+---+       +---+---+---+---+-----+       +---+---+---+---+---+---+       +---+---+
      |Worker1|       |Worker2| ... |WorkerN|       |WorkerN+1| ... |WorkerM|      |WorkerM+1|
      +---+---+       +---+---+---+---+-----+       +---+---+---+---+---+---+       +---+---+
          |               |                   ...       |                   ...         |
          +---------------+-----------------------------+-------------------------------+
                              | (all output channels)
                              v
                      +-------+-------+
                      |    Fan-In     |
                      |  (Aggregator) |
                      +-------+-------+
                              | (resultCh)
                              v
                      +-------+-------+
                      |   Receiver    |
                      | (Collector)   |
                      +---------------+

```

- WebSocket listeners must send updates to a shared channel.
- Worker Pool must handle multiple concurrent updates per exchange.
- Data must be batched (instead of per-update writes) before inserting into PostgreSQL.
- If Redis is down, PostgreSQL should still receive data (fallback mechanism).

By implementing these patterns, the system will ensure efficient data ingestion, processing, and storage, leveraging Go’s built-in concurrency primitives.

### Data Storage and Caching

#### Real-Time Data Processing:

- Use **Redis** to cache recent price data.
- Keep the latest price for all price updates for at least the last minute for each trading pair from all exchanges.
- Every minute, calculate the average price for each pair based on the last 60 seconds of data, store the result in PostgreSQL, and also save the minimum and maximum price values.

You will need to decide which key and which value to save in Redis and how best to implement it.

#### Data Storage

- Use **PostgreSQL** to store aggregated data.
- Store data in a single table with the following fields:
  - `pair_name` (string) – the trading pair name.
  - `exchange` (string) – the exchange from which the data was received.
  - `timestamp` (timestamp) – the time when the data is stored.
  - `average_price` (float) – the average price of the trading pair over the last minute.
  - `min_price` (float) – the minimum price of the trading pair over the last minute.
  - `max_price` (float) – the maximum price of the trading pair over the last minute.

### API Endpoints

**Market Data API**

`GET /prices/latest/{symbol}` – Get the latest price for a given symbol.

`GET /prices/latest/{exchange}/{symbol}` – Get the latest price for a given symbol from a specific exchange.

`GET /prices/highest/{symbol}` – Get the highest price over a period.

`GET /prices/highest/{exchange}/{symbol}` – Get the highest price over a period from a specific exchange.

`GET /prices/highest/{symbol}?period={duration}` – Get the highest price within the last `{duration}` (e.g., `1s`,  `3s`, `5s`, `10s`, `30s`, `1m`, `3m`, `5m`).

`GET /prices/highest/{exchange}/{symbol}?period={duration}` – Get the highest price within the last `{duration}` from a specific exchange.

`GET /prices/lowest/{symbol}` – Get the lowest price over a period.

`GET /prices/lowest/{exchange}/{symbol}` – Get the lowest price over a period from a specific exchange.

`GET /prices/lowest/{symbol}?period={duration}` – Get the lowest price within the last {duration}.

`GET /prices/lowest/{exchange}/{symbol}?period={duration}` – Get the lowest price within the last `{duration}` from a specific exchange.

`GET /prices/average/{symbol}` – Get the average price over a period.

`GET /prices/average/{exchange}/{symbol}` – Get the average price over a period from a specific exchange.

`GET /prices/average/{exchange}/{symbol}?period={duration}` – Get the average price within the last `{duration}` from a specific exchange

**Data Mode API**

`POST /mode/test` – Switch to Test Mode (use generated data).

`POST /mode/live` – Switch to Live Mode (fetch data from exchanges).

**Note:** active Test Mode should disable proxy.

**Proxy Settings API**

`GET /proxy/status` – Get the current proxy configuration status.

`POST /proxy/enable` – Enable proxy usage for API requests.

`POST /proxy/disable` – Disable proxy usage for API requests.

`POST /proxy/set-static` – Set the proxy mode to static.

`POST /proxy/set-rotating` – Set the proxy mode to rotating.

`POST /proxy/set-disabled` – Disable proxy usage and remove settings.

**Note:** it should not be possible to enable the proxy in the Test Mode.

**System Health**

`GET /health` - Returns system status (e.g., WebSocket connections, Redis availability).

### Configuration

The application should read configuration from a file. The configuration should include the following parameters:
- PostgreSQL connection details (host, port, user, password, database and etc.).
- Redis connection details (host, port, password and etc.).
- WebSocket connection details (host, port and etc.).
- Proxy settings (host, port and etc.).
- Other necessary settings.

### Logging

- Use Go's log/slog package for logging throughout the application.
- Log significant events, errors information with appropriate levels (`Info`, `Warning`, `Error`).
- Include contextual information in logs (e.g., timestamps, IDs).

### Shutdown

Implement graceful shutdown handling to ensure the application cleans up resources and exits cleanly when receiving a termination signal (e.g., `SIGINT`, `SIGTERM`).

### Usage

Your program must be able to print usage information.

Outcomes:
- Program prints usage text.

```sh
$ ./marketflow --help

Usage:
  marketflow [--port <N>]
  marketflow --help

Options:
  --port N     Port number
```

---

## Support

Believe in yourself and you will succeed.

---
## Guidelines from Author

First of all, implement data acquisition from at least one source.  
Study the patterns and think about how to apply them.  
Then implement the storage of data in PostgreSQL.  
Then implement the caching of data in Redis.  
Then implement the API.  
Then implement the rest of the sources.  
Then implement the rest of the patterns.  
Then implement the rest of the API.  
Then implement the rest of the features.

Good luck, Buddy 😊

---
## Author

This project has been created by:

Savva Savostyanov

Contacts:

- Email: [savvax@savvax.com](mailto:savvax@savvax.com)
- [GitHub](https://github.com/savvax/)
- [LinkedIn](https://www.linkedin.com/in/savvax/)
