# RSSHub

## Learning Objectives

- Working with XML and RSS formats
- Concurrency and channels
- Redis
- PostgreSQL
- Docker Compose
- Microservices

## Abstract

As part of this project, you will create a **CLI application—an [RSS](https://en.wikipedia.org/wiki/RSS) feed aggregator** that:

- Fetches and processes [RSS](https://en.wikipedia.org/wiki/RSS) feeds.
- Stores newly fetched articles in PostgreSQL.
- Caches recent feed results in Redis.

This is a service that collects posts from various sources that provide RSS feeds (news websites, blogs, forums). It helps users stay updated in one place without needing to visit each resource manually.

Such a tool is useful for journalists, researchers, analysts, and anyone who wants to stay informed on topics of interest without unnecessary clutter. This kind of application makes information more accessible and centralized.

### Background Processing Microservice

In addition to the CLI, the project includes a **dedicated background microservice written in Go** that continuously processes RSS feeds. This microservice runs in the background and is responsible for periodically fetching, parsing, and preparing feed data.

Following microservice architecture best practices, this background worker does **not have direct access** to PostgreSQL or Redis. Instead, it communicates with the main CLI application over **HTTP**—sending feed entries and status updates to predefined endpoints exposed by the CLI service (which acts as the API gateway and persistence layer).

## Context

You are developing a CLI application that periodically downloads articles from RSS feeds added by the user and saves them to PostgreSQL. Repeated requests are served from Redis. You will also write a service that will be responsible for processing RSS feeds in the background and connecting to the main service. All services are deployed using Docker Compose.

## General Criteria

- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will be graded `0` automatically.
- Your program MUST be able to compile successfully.
- Your program MUST not exit unexpectedly (any panics: `nil-pointer dereference`, `index out of range` etc.). If so, you will get `0` during the defence.
- External packages are allowed only for working with Redis/PostgreSQL. If you use any other external packages, you will receive a grade of `0`.
- The project MUST be compiled by the following command in the project's root directory:

```sh
$ go build -o rsshub .
```

- If an error occurs during startup (e.g., invalid command-line arguments), the program must exit with a non-zero status code and display a clear, understandable error message.

## Mandatory Part

#### Infrastructure

Include a docker-compose.yml that runs:

- PostgreSQL – for storing articles
- Redis – for caching feeds
- RSSListener - for processing feeds

#### Important Notes

```sh
rsshub add --name "tech-crunch" --url "https://techcrunch.com/feed/"
```

Adds a new RSS feed to PostgreSQL.

```sh
rsshub fetch --interval 2m
```

Sets the interval at which RSS feeds are fetched (e.g. every 2 minutes).

```sh
rsshub list
```

Shows a list of all the added feeds.

```sh
rsshub delete --name "tech-crunch"
```

Deletes the RSS feed from PostgreSQL.

```sh
rsshub articles --num 5
```

Shows the latest N articles from Redis or the database. By default -- 3 articles.

```sh
rsshub --help
```

Shows the help for commands.

#### Architecture

The project is designed with Clean Architecture principles (also known as Hexagonal or Layered Architecture), ensuring separation of concerns, testability, and maintainability. The system is split into two main components:

### 1. CLI Service (Core Application)

This is the primary Go service that:

- Provides a **command-line interface** for user interaction.
- Exposes **HTTP endpoints** for receiving parsed RSS articles.
- Handles **persistence (PostgreSQL)** and **caching (Redis)** internally.
- Orchestrates business logic through the application layer.

Directory structure:

- `cmd/app/` - Initialize all dependencies

  - Starts the application

- `domain/` — Domain models and interfaces (pure business logic):

  - `Feed`, `Article` structs
  - Repository interfaces (`FeedRepository`, `ArticleRepository`, `Cache`, etc.)

- `internal/` — Application services (use cases, orchestration logic):

  - Adding feeds, fetching/deduplication logic
  - Coordination of storage and caching
  - Stateless, depends only on domain interfaces

- `adapters/` — External system implementations:

  - `postgres/` — PostgreSQL storage layer
  - `redis/` — Redis cache

- `handlers/` — HTTP handlers for receiving external input (e.g., from RSSListener)

- `cli/` — Command Line Interface:
  - User commands (`add-feed`, `fetch`, `list`, etc.)
  - Communicates with app layer
  - Can launch an internal HTTP server for receiving feed data from RSSListener

### 2. RSSListener (Background Microservice)

This is a **separate microservice written in Go**, responsible for:

- Continuously polling subscribed RSS feeds.
- Parsing and preparing article data.
- Sending results to the main CLI service **via HTTP POST requests**.

RSSListener does **not directly access** PostgreSQL or Redis, following **microservice best practices**:

- Each service owns its data and exposes interaction **only via well-defined APIs**.
- The CLI service acts as the **gateway** for all data ingestion and persistence.
- This improves **security, scalability, and encapsulation**.

### Microservices in Go

Go is well-suited for microservice architecture due to:

- Its simplicity, fast compile times, and low runtime overhead.
- Powerful standard libraries for HTTP, concurrency, and networking.
- Strong typing and modular code structure that supports clean separation of services.

This architecture ensures that both services can be developed, tested, and deployed independently, with clear communication over HTTP and decoupled responsibilities.

#### RSSListener

The main purpose of the `rsshub` program is to fetch the RSS feed of a website and store its content in a structured format in our database. That way we can display it nicely in our CLI.

RSS stands for "Really Simple Syndication" and is a way to get the latest content from a website in a structured format. It's fairly ubiquitous on the web: most content sites have an RSS feed.

Structure of an RSS Feed
RSS is a specific structure of XML. We will keep it simple and only worry about a few fields. Here's an example of the documents which need to parse:

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

Then directly unmarshal this kind of document into structs like this:

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

If there are any extra fields in the XML, the parser will just discard them, and if any are missing, the parser will leave them as their zero value.

#### Periodic Feed Aggregation

Feeds are essentially just lists of posts. A post represents a single web page. The entire point of the `rsshub` program is to fetch the actual posts from the feed URLs and store them in database. That way we can display them nicely in CLI.

Need to create a mechanism that will regularly receive articles from the database, compare them with incoming articles from the RSS feed and record changes in the database. The priority is the one that has not been updated the longest or has never been updated.

The mechanism should run against the background at the interval specified by the command line argument (for example: 1m, 1h).

A message should appear at startup.

```sh
./rsshub fetch --interval 2m
$ Collecting feeds every 2m...
```

By default interval should be 3 minutes.

```sh
./rsshub fetch
$ Collecting feeds every 3m...
```

A small hint for running the mechanism in the background is to use an infinite `for` loop. with time.Ticker

```go
ticker := time.NewTicker(interval)
defer ticker.Stop()

for range ticker.C {
	scrapeFeeds(s)
}
```

#### Worker Pool for Concurrent Processing

To improve performance, implement a worker pool for concurrently processing multiple RSS feeds:

- On each tick, retrieve N most outdated or never-updated feeds from the database.
- Distribute them across a pool of worker goroutines.
- The number of workers is configurable; the default is 3.

#### Worker Pool Requirements

- Use a buffered channel to queue feed processing tasks.
- Use sync.WaitGroup to wait until all feeds are processed before proceeding to the next tick.

#### Required Endpoints in the Service

To support the mechanism, ensure the service exposes the following endpoints:

1. `GET /feeds/outdated?n=10`
   Returns the n most outdated or never-updated feeds.

2. `POST /feeds/update`
   Accepts updated feed data and saves it to the database.

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

#### PostgreSQL

Tables:

1.  `feeds`

        | Field      | Type          |
        | ---------- | ------------- |
        | id         | UUID (PK)     |
        | name       | TEXT (unique) |
        | url        | TEXT          |
        | description| TEXT          |
        | created_at | TIMESTAMP     |
        | updated_at | TIMESTAMP     |

2.  `articles`

        | Field         | Type          |
        | ------------- | ------------- |
        | id            | UUID (PK)     |
        | title         | TEXT          |
        | url           | TEXT          |
        | description   | TEXT          |
        | feed_id       | UUID          |
        | published_at  | TIMESTAMP     |
        | created_at    | TIMESTAMP     |
        | updated_at    | TIMESTAMP     |

> This is a standard instruction for tables. It's in your best interest to add fields.

Migrations:

Migrations are a set of versioned files that describe changes to the database structure (DDL): creating tables, changing columns, adding indexes, etc.

The recommended tool is `golang-migrate/migrate`

Install:

```sh
go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

Check:

```sh
migrate -version
```

Example:

```
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

postgres:
  host: localhost
  port: 5432
  user: postgres
  password: changem
  dbname: rsshub

redis:
  host: localhost
  port: 6379

rss_listener:
  host: localhost
  port: 8888
  worker_counts: 3
```

```env
# CLI App
CLI_APP_HOST=localhost
CLI_APP_PORT=8080

# PostgreSQL
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=changem
POSTGRES_DBNAME=rsshub

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# RSS Listener
RSS_LISTENER_HOST=localhost
RSS_LISTENER_PORT=8888
RSS_LISTENER_WORKER_COUNTS=3
```

#### Docker Compose

Launches:

- PostgreSQL (port: `5432`)
- Redis (port: `6379`)
- RSSListener (port: `8888`)

## Example Usage

```sh
$ ./rsshub --help

  Usage:
    rsshub COMMAND [OPTIONS]

  Common Commands:
       add            Adds a new RSS feed to PostgreSQL.
       fetch          Sets the interval at which RSS feeds are fetched (e.g. every 2 minutes).
       list           Shows a list of all the added feeds
       delete         Deletes the RSS feed from PostgreSQL
       articles       Shows the latest N articles from Redis or the database. By default -- 3 articles.
```

## Guidelines from Author

- Start with a single RSS feed and a CLI command to fetch it
- Implement an aggregate mechanism with a background ticker
- Add a worker pool to process feeds concurrently
- Use PostgreSQL for storage and apply database migrations
- Add Redis for optional caching of recent articles
- Build a `docker-compose.yml` for local development
- Finish with basic tests for fetch and parse logic

Here are a few RSS feeds to try:

- TechCrunch: `https://techcrunch.com/feed/`
- Hacker News: `https://news.ycombinator.com/rss`

## Additionally _(\*not necessarily)_

For your own development _(Future features)_:

- You can add multiple users and everyone can subscribe to certain news.
- You can add a Web API for external access.
- You can add a Telegram Bot for notifications of new articles,

## Support

It is always unclear where to start, try to break the task into smaller ones that will solve one problem, and then connect and expand them and eventually you will get the result.

Good luck & have fun :3

## Author

This project has been created by:

_@trech_

Contacts:

- [Email](mailto:amir.inkarov.01@gmail.com)
- [GitHub](https://github.com/Tr8ch/)
- [LinkedIn](https://www.linkedin.com/in/trech/)
