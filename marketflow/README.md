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

Imagine you're developing a system for a financial firm that requires real-time price updates from multiple cryptocurrency exchanges. The system must efficiently handle concurrent data streams, process updates in real time, and expose an API for querying recent prices and market statistics.  
This project is real (a similar project is used at the workplace of one of the graduates). It integrates into a more complex system and addresses some business challenges.  
To achieve this, the project will implement:

Imagine you're developing a system for a financial firm that requires real-time price updates from multiple cryptocurrency exchanges. The system must efficiently handle concurrent data streams, process updates in real time, and expose an API for querying recent prices and market statistics.

- Real-time data fetching (`Live Mode`)
- Real-time test data fetching (`Test Mode`)
- Concurrent data processing using channels & worker pools
- Data storage in PostgreSQL
- Redis caching for quick access to frequently requested prices
- REST API for querying aggregated price data

This project mirrors real-world applications used in trading platforms, giving you practical experience with Go’s concurrency model and backend architecture.

## Resources

- Read about Redis [here](https://redis.io/docs/latest/).
- Read about concurrency [here](https://go.dev/doc/effective_go#concurrency), [here](https://go.dev/blog/pipelines), [here](https://dev.to/dwarvesf/approaches-to-manage-concurrent-workloads-like-worker-pools-and-pipelines-52ed) and [here](https://go.dev/talks/2012/concurrency.slide#1).

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

You will create a `marketflow` application, a system designed to process market data in real-time and offer a RESTful API for accessing price updates and more. The application should follow a hexagonal architecture. The project should emphasize clean code, maintainability, and scalability to ensure a robust and efficient solution.

#### Outcomes:

- Use a hexagonal architecture:

  - Domain Layer: Defines core business logic and models.

  - Application Layer: Implements use cases and orchestrates interactions between components.

  - Adapters (Ports & Adapters Layer):

    - Web Adapter (HTTP handlers) for REST API.

    - Storage Adapter (PostgreSQL) for data persistence.

    - Cache Adapter (Redis) for caching.

    - Exchange Adapter for fetching live market data.

- Support two data modes:

  - Live Mode: Fetch real-time prices from three cryptocurrency exchanges.

  - Test Mode: Generate synthetic market data locally.

- Store market data in PostgreSQL.

- Implement Redis caching for latest prices.

- Implement concurrency patterns to efficiently process multiple price updates.

  - Provide REST API endpoints for retrieving price information and statistics.
  
- An additional application provided by you for Test Mode should be able to generate data. 

#### Where to get data from?

You will be provided with a programs that simulates the behavior of cryptocurrency exchanges.  
Run the `provided programs` and receive information on ports `40101`, `40102`, `40103`.  
[exchange1_amd64](exchange1_amd64.tar)  
[exchange1_arm64](exchange1_arm64.tar)  
[exchange2_amd64](exchange2_amd64.tar)  
[exchange2_arm64](exchange2_arm64.tar)   
[exchange3_amd64](exchange3_amd64.tar)  
[exchange3_arm64](exchange3_arm64.tar)

##### How to run the provided programs?

1. Define the CPU architecture.
2. Then load the images:

- `docker load -i exchange1_amd64.tar` or `docker load -i exchange1_arm64.tar`

- `docker load -i exchange2_amd64.tar` or `docker load -i exchange2_arm64.tar`

- `docker load -i exchange3_amd64.tar` or `docker load -i exchange3_arm64.tar`

3. Run the images (example):
- `docker run -p 40101:40101 --name exchange1-arm64 -d exchange3-arm64`
- `docker run -p 40102:40102 --name exchange2-arm64 -d exchange2-arm64`
- `docker run -p 40103:40103 --name exchange3-arm64 -d exchange3-arm64`

Try to run ```nc 127.0.0.1 <port>``` after starting the provided programs.

**It is necessary to receive data about pairs of the following types:**
- `BTCUSDT`
- `DOGEUSDT`
- `TONUSDT`
- `SOLUSDT`
- `ETHUSDT`

You need to implement your own test data generator for Test Mode.

#### Additional conditions for live data handling

- **Failover:** If connection fails, the system should automatically attempt to reconnect (Note: Stop and restart the provided program and/or the generator you implemented. 😉).

### Concurrency Implementation and Patterns:

This project heavily relies on concurrency to handle large volumes of real-time data. Key concurrency patterns that should be used include:

- **Fan-in:** Aggregating multiple market data streams into a single channel for centralized processing.

- **Fan-out:** Distributing incoming data updates to multiple workers to process them in parallel.

- **[Worker Pool](https://gobyexample.com/worker-pools):** Managing a set of workers that process live updates efficiently, ensuring balanced workload distribution.

- **Generator:** Implementing a generator to produce synthetic market data for `Test Mode`.

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

- Listeners must send updates to a shared channel.
- The Worker Pool should efficiently process multiple concurrent updates per exchange, allocating five workers to each data source.
- Data must be batched (instead of per-update writes) before inserting into PostgreSQL.
- If Redis is down, PostgreSQL should still receive data (fallback mechanism).

By implementing these patterns, the system will ensure efficient data ingestion, processing, and storage, leveraging Go’s built-in concurrency primitives.

#### Test Mode

You need to create a program that simulates the behavior of the provided programs for `Test Mode`.
Implement the Generator pattern to provide synthetic data.

### Data Storage and Caching

#### Real-Time Data Processing:

- Use **Redis** to cache recent price data.
- Keep the latest price for all price updates for at least the last minute for each trading pair from all exchanges.
- Every minute, calculate the average price for each pair based on the last 60 seconds of data, store the result in PostgreSQL, and also save the minimum and maximum price values.

You will need to decide which key and which value to save in Redis and how best to implement it.

Do not forget to delete irrelevant data.

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

`GET /prices/highest/{symbol}?period={duration}` – Get the highest price within the last `{duration}` (e.g., the last `1s`,  `3s`, `5s`, `10s`, `30s`, `1m`, `3m`, `5m`).

`GET /prices/highest/{exchange}/{symbol}?period={duration}` – Get the highest price within the last `{duration}` from a specific exchange.

`GET /prices/lowest/{symbol}` – Get the lowest price over a period.

`GET /prices/lowest/{exchange}/{symbol}` – Get the lowest price over a period from a specific exchange.

`GET /prices/lowest/{symbol}?period={duration}` – Get the lowest price within the last {duration}.

`GET /prices/lowest/{exchange}/{symbol}?period={duration}` – Get the lowest price within the last `{duration}` from a specific exchange.

`GET /prices/average/{symbol}` – Get the average price over a period.

`GET /prices/average/{exchange}/{symbol}` – Get the average price over a period from a specific exchange.

`GET /prices/average/{exchange}/{symbol}?period={duration}` – Get the average price within the last `{duration}` from a specific exchange

**Data Mode API**

`POST /mode/test` – Switch to `Test Mode` (use generated data).

`POST /mode/live` – Switch to `Live Mode` (fetch data from `provided programs`).

**System Health**

`GET /health` - Returns system status (e.g., connections, Redis availability).  

**You need to come up with the responses yourself.**

### Configuration

The application should read configuration from a file. The configuration should include the following parameters:
- PostgreSQL connection details (host, port, user, password, database and etc.).
- Redis connection details (host, port, password and etc.).
- Connection details (host, port, etc.) for the provided exchange emulations and your test implementation.

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
Then implement the caching of data in Redis.  
Then implement the storage of data in PostgreSQL.  
Then partially implement the API.
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
