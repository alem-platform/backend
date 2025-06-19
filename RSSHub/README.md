# RSSHub

## Learning Objectives

- Working with XML and RSS formats
- Concurrency and channels
- Worker Pool
- Redis
- PostgreSQL
- Docker Compose

## Abstract

In this project, you will build a **CLI application — an [RSS](https://en.wikipedia.org/wiki/RSS) feed aggregator**, which:

- Provides a command-line interface (CLI)
- Fetches and parses [RSS](https://en.wikipedia.org/wiki/RSS) feeds
- Stores articles in PostgreSQL
- Caches recent data in Redis
- Aggregates RSS feeds using a worker pool in the background

This is a service that collects publications from various sources that provide RSS feeds (news sites, blogs, forums). It helps users stay informed in one place without the need to visit each website manually.

Such a tool is useful for journalists, researchers, analysts, and anyone who wants to stay updated on topics of interest without unnecessary noise. This kind of application makes information more accessible and centralized.

## Context

You are developing a CLI application that periodically fetches articles from user-added RSS feeds and stores them in `PostgreSQL`. Repeated requests are served from `Redis`. You will also implement a mechanism responsible for background, parallel processing of RSS feeds. All services are deployed using `Docker Compose`.

## General Criteria

- Your code **MUST** be formatted according to [gofumpt](https://github.com/mvdan/gofumpt). If not, you will automatically receive a grade of `0`.
- Your program **MUST** compile successfully without errors.

- Your program **MUST NOT** exit unexpectedly (e.g., due to nil pointer dereference, index out of range, etc.). If it does, you will receive a grade of `0` during the defense.

- External packages are allowed only for working with `Redis` and `PostgreSQL`. Using any other external dependencies will result in a grade of `0`.

- If an error occurs during startup (e.g., invalid command-line arguments), the program **MUST**:

  - Exit with a non-zero status code
  - Display a clear and understandable error message

- The project **MUST** compile successfully using the following command from the project's root directory:

```sh
$ go build -o rsshub .
```

- The project **MUST NOT** produce any errors when run with the -race flag:

```sh
$ go run -race main.go
```

- The interval **MUST** be set dynamically via a terminal command. If not, you will automatically receive a grade of `0`.

- The number of workers **MUST** also be configurable dynamically via a terminal command. If not, you will automatically receive a grade of `0`.

## Mandatory Part

### Infrastructure

Include a `docker-compose.yml` file that runs the following services:

- RSSHub – a CLI application
- PostgreSQL – for storing articles
- Redis – for caching feeds

### Architecture

The project is designed using the principles of Clean Architecture (also known as Hexagonal or Layered Architecture), which ensures separation of concerns, testability, and maintainability:

- Provides a **command-line interface** for user interaction.
- Handles **data persistence (PostgreSQL)** and **caching (Redis)** internally.
- Executes business logic through the application layer.

### Directory Structure

- `cmd/app/` — Initialization of all dependencies

  - Starts the application
  - Launches background, parallel processing of RSS feeds

- `domain/` — Domain models and interfaces (pure business logic):

  - Structures: `Feed`, `Article`
  - Repository interfaces: `FeedRepository`, `ArticleRepository`, `Cache`, etc.

- `internal/` — Application services (use cases, orchestration logic):

  - Adding feeds, loading logic, and deduplication
  - Coordination between storage and cache
  - Stateless; depends only on domain interfaces
  - Mechanism for processing RSS feeds and saving articles

- `adapters/` — Implementations of external systems:

  - `postgres/` — PostgreSQL storage layer
  - `redis/` — Redis cache
  - `rss/` — Interaction with external services related to RSS feeds

- `cli/` — Command-line interface:

  - User commands (`add`, `delete`, `list`, etc.)
  - Interacts with the application layer

### RSS

The main goal of the `rsshub` program is to fetch a website's `RSS` feed and store its content in a structured format in our database. This allows us to display the data nicely in the CLI.

`RSS` stands for **"Really Simple Syndication"** — it's a way to receive fresh content from a website in a structured format. It is widely used on the internet: most content-driven websites provide an `RSS` feed.

### Structure of an RSS Feed

`RSS` is a specific `XML` structure. We will simplify the task and focus only on a few fields. Below is an example of the documents that need to be parsed:

```xml
<rss xmlns:atom="http://www.w3.org/2005/Atom" version="2.0">
<channel>
  <title>RSS Feed Example</title>
  <link>https://www.example.com</link>
  <description>This is an example RSS feed</description>
  <item>
    <title>First Article</title>
    <link>https://www.example.com/article1</link>
    <description>This is the content of the first article.</description>
    <pubDate>Mon, 06 Sep 2021 12:00:00 GMT</pubDate>
  </item>
  <item>
    <title>Second Article</title>
    <link>https://www.example.com/article2</link>
    <description>This is the content of the second article.</description>
    <pubDate>Tue, 07 Sep 2021 14:30:00 GMT</pubDate>
  </item>
</channel>
</rss>
```

_Then directly transform this kind of document into structures like this:_

```go
type RSSFeed struct {
	Channel struct {
		Title       string    `xml:"title"`
		Link        string    `xml:"link"`
		Description string    `xml:"description"`
		Item        []RSSItem `xml:"item"`
	} `xml:"channel"`
}

type RSSItem struct {
	Title       string `xml:"title"`
	Link        string `xml:"link"`
	Description string `xml:"description"`
	PubDate     string `xml:"pubDate"`
}
```

If there are additional fields in the `XML`, the parser will simply ignore them. If some fields are missing, they will retain their default (zero) values.

Accordingly, you will need to implement an `RSS` parser that performs an HTTP request using the feed URL stored in the database.

### Periodic Feed Aggregation

Feeds are essentially lists of publications. Each publication is a separate web page. The main goal of the `rsshub` program is to fetch actual publications using the URLs from RSS feeds and store them in the database. This allows us to display them nicely in the CLI.

You need to create a mechanism that regularly retrieves feed URLs from the database, compares them with new articles from the `RSS` feeds, and saves any changes to the database. Priority should be given to feeds that have not been updated for a long time or have never been updated.

This mechanism must run in the background at a specified interval. The default interval should be `3 minutes`. This interval should also be configurable using a CLI command. The command must be able to change the interval while the application is running, without stopping the background aggregation process.

**To improve performance, implement a worker pool for parallel processing of multiple articles:**

1. On each timer tick, retrieve the N most outdated or never-updated feeds from the database.
2. Distribute them across the worker pool (goroutines).
3. Each worker:

   - Downloads the feed by its URL
   - Parses new articles
   - Saves them to `Postgres` and caches `Redis`

4. The application must be able to dynamically:

   - Change the ticker interval (`SetInterval`)
   - Resize the number of workers (`Resize`)

**Default behavior:**

- Default settings must be defined in the application configuration
- Default interval: `3 minutes`
- Default number of workers: `3`

**Mechanism requirements:**

- **The ticker** must be controllable: it should be possible to stop and restart it with a new interval
- **The worker** pool must be scalable: workers can be stopped or added dynamically

**What to Avoid**

| Problem                      | How to Avoid                                                                                            |
| ---------------------------- | ------------------------------------------------------------------------------------------------------- |
| ❌ Data Race                 | Use `sync.Mutex` or `atomic` when accessing shared variables (e.g., `n`, `interval`)                    |
| ❌ Goroutine Leaks           | All spawned goroutines (ticker, workers) must be terminated using `context.Context` or a `done` channel |
| ❌ Duplicate Tickers         | When calling `SetInterval()`, always stop the old ticker before starting a new one                      |
| ❌ Closing Channel Twice     | Channels (e.g., `jobs`) must be closed by only one goroutine                                            |
| ❌ Panic on `ticker.Reset()` | Never call `Reset()` on a stopped ticker                                                                |
| ❌ Unconsumed `jobs` Channel | Workers must read from the `jobs` channel; otherwise, writing to the channel will cause a deadlock      |

**What Needs to Be Implemented**

- Structures for the ticker and worker pool
- Method `Start(ctx)` — starts the processing loop
- Method `SetInterval(time.Duration)` — safely updates the interval
- Method `Resize(n int)` — recreates the worker pool with the desired size
- Method `Stop()` — gracefully shuts down everything via `ctx`
- Workers — read from the `jobs` channel and process articles
- The `jobs` channel must be created and closed correctly
- The code should be structured as a service (in `internal/`)

> ---
>
> # WARNING
>
> Do NOT DoS the servers you're fetching feeds from. Anytime you write code that makes a request to a third party server you should be sure that you are not making too many requests too quickly.
> That's why I recommend printing to the console for each request, and being ready with a quick `Ctrl+C` to stop the program if you see something going wrong.
>
> [DoS attack](https://en.wikipedia.org/wiki/Denial-of-service_attack#:~:text=attacks%20are%20distributed.-,Distributed%20DoS,of%20hosts%20infected%20with%20malware.) - read more.
>
> ---
>
> # WARNING

## Important CLI Commands

```sh
rsshub add --name "tech-crunch" --url "https://techcrunch.com/feed/"
```

Adds a new RSS feed to PostgreSQL.

```sh
rsshub set-interval 2m
```

Sets the interval at which RSS feeds are fetched (e.g., every 2 minutes).

```sh
rsshub set-workers 3
```

Sets the number of workers.

```sh
rsshub list --num 10
```

Displays a N feeds. If without option then the cammand show list of all feeds added.

```sh
rsshub delete --name "tech-crunch"
```

Deletes the RSS feed from PostgreSQL.

```sh
rsshub articles --feed-name "tech-crunch" --num 5
```

Shows the latest N articles from Redis or the database by feed name. Default is 3 articles.

```sh
rsshub --help
```

Displays help information for all available commands.

#### PostgreSQL

Tables:

1.  `feeds`

        | Field      | Type          |
        | ---------- | ------------- |
        | id         | UUID (PK)     |
        | created_at | TIMESTAMP     |
        | updated_at | TIMESTAMP     |
        | name       | TEXT (unique) |
        | url        | TEXT          |

2.  `articles`

        | Field         | Type          |
        | ------------- | ------------- |
        | id            | UUID (PK)     |
        | created_at    | TIMESTAMP     |
        | updated_at    | TIMESTAMP     |
        | title         | TEXT          |
        | url           | TEXT          |
        | published_at  | TIMESTAMP     |
        | description   | TEXT          |
        | feed_id       | UUID          |

> This is a standard instruction for tables. It's in your best interest to add fields.

### Migrations

Migrations are a set of versioned files that describe changes to the database schema (DDL): creating tables, modifying columns, adding indexes, etc.

The recommended tool is `golang-migrate/migrate`.

Install:

```sh
go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

Check:

```sh
migrate -version
```

Example:

```sh
migrations/
├── create_feeds_table.up.sql
└── create_feeds_table.down.sql
```

- `*.up.sql` — The SQL that is used
- `*.down.sql` — SQL that is rolled back

`create_feeds_table.up.sql`:

```sql
CREATE TABLE feeds (
    //TODO
);
```

`create_feeds_table.down.sql`:

```sql
DROP TABLE IF EXISTS feeds;
```

Useful commands:

Create a new migration:

```sh
migrate create -ext sql -dir migrations -seq create_feeds_table
```

Application of migrations

```sh
migrate -path ./db/migrations -database "postgres://user:pass@localhost:5432/rsshub?sslmode=disable" up
```

Rollback migration

```sh
migrate -path ./db/migrations -database "postgres://user:pass@localhost:5432/rsshub?sslmode=disable" down
```

#### Redis

- Key: `articles:<feed_name>`
- Value: JSON list of the latest articles
- TTL: 10 minutes
- If Redis is unavailable — fallback to PostgreSQL with a warning.

### Initial Setup

#### Configuration

```yaml
cli_app:
  host: localhost
  port: 8080
  timer_interval: 3m
  workers_count: 3

postgres:
  host: localhost
  port: 5432
  user: postgres
  password: changem
  dbname: rsshub

redis:
  host: localhost
  port: 6379
```

```env
# CLI App
CLI_APP_HOST=localhost
CLI_APP_PORT=8080
CLI_APP_TIMER_INTERVAL=3m
CLI_APP_WORKERS_COUNT=3

# PostgreSQL
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=changem
POSTGRES_DBNAME=rsshub

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

```

#### Docker Compose

Launches:

- CLI (port: `8080`)
- PostgreSQL (port: `5432`)
  - set username, password, db from `env/yaml`
- Redis (port: `6379`)

## Example Usage

```sh
$ ./rsshub --help

  Usage:
    rsshub COMMAND [OPTIONS]

  Common Commands:
       add             Adds a new RSS feed to PostgreSQL.
       set-interval    Sets the interval at which RSS feeds are fetched (e.g., every 2 minutes).
       set-workers     Sets the number of workers.
       list            Shows a list of all the added feeds.
       delete          Deletes the RSS feed from PostgreSQL.
       articles        Shows the latest N articles from Redis or the database. By default -- 3 articles.
```

## Recommendations from the Author

- Start with a single RSS feed and a CLI command to fetch it
- Implement an aggregation mechanism with a background timer
- Add a worker pool for parallel feed processing
- Use PostgreSQL for storage and configure database migrations
- Add Redis for caching the latest articles
- Create a `docker-compose.yml` file for local development
- Finish by testing your application logic

**Here are some RSS feeds to get started:**

- TechCrunch: `https://techcrunch.com/feed/`
- Hacker News: `https://news.ycombinator.com/rss`
- UN News: `https://news.un.org/feed/subscribe/ru/news/all/rss.xml`
- BBC News: `https://feeds.bbci.co.uk/news/world/rss.xml`
- Ars Technica: `http://feeds.arstechnica.com/arstechnica/index`
- The Verge: `https://www.theverge.com/rss/index.xml`

## Optional _(Nice to Have)_

For your own growth _(Future opportunities)_:

- Add support for multiple users and allow each to subscribe to different feeds
- Implement a Web API for external access
- Add a Telegram bot to notify users about new articles

## Support

It's always hard to know where to start. Try breaking the task into smaller pieces, each solving one specific problem. Then connect and extend them — eventually, you'll have a complete solution.

Good luck, and enjoy the process :3

## Author

This project has been created by:

_@trech_

Contacts:

---

- [Email](mailto:amir.inkarov.01@gmail.com)
- [GitHub](https://github.com/Tr8ch/)
- [Discord](https://discordapp.com/users/394227156915322881/)
- [LinkedIn](https://www.linkedin.com/in/trech/)
