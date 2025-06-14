# aggreGATOR 🐊 [gator]

## Learning Objectives

- Working with XML and RSS formats
- Concurrency and channels
- Redis
- PostgreSQL
- Elasticsearch
- Kibana
- Docker Compose

## Abstract

As part of this project, you will create a **CLI application—an [RSS](https://en.wikipedia.org/wiki/RSS) feed aggregator** that:

- Fetches and processes [RSS](https://en.wikipedia.org/wiki/RSS) feeds.
- Stores newly fetched articles in PostgreSQL.
- Caches recent feed results in Redis.
- Sends structured logs to Elasticsearch.
- Enables log analysis and dashboards via Kibana.

This is a service that collects posts from various sources that provide RSS feeds (news websites, blogs, forums). It helps users stay updated in one place without needing to visit each resource manually.

Such a tool is useful for journalists, researchers, analysts, and anyone who wants to stay informed on topics of interest without unnecessary clutter. This kind of application makes information more accessible and centralized.

## Context

You are developing a CLI application that periodically downloads articles from RSS feeds added by the user and saves them to PostgreSQL. Repeated requests are served from Redis. All actions are logged in Elasticsearch and displayed in Kibana. All services are deployed using Docker Compose.

## General Criteria

- Your code MUST be written in accordance with [gofumpt](https://github.com/mvdan/gofumpt). If not, you will be graded `0` automatically.
- Your program MUST be able to compile successfully.
- Your program MUST not exit unexpectedly (any panics: `nil-pointer dereference`, `index out of range` etc.). If so, you will get `0` during the defence.
- External packages are allowed only for working with Redis/PostgreSQL/Elasticsearch. If you use any other external packages, you will receive a grade of `0`.
- The project MUST be compiled by the following command in the project's root directory:

```sh
$ go build -o gator .
```

- If an error occurs during startup (e.g., invalid command-line arguments), the program must exit with a non-zero status code and display a clear, understandable error message.

## Mandatory Part

#### Infrastructure

Include a docker-compose.yml that runs:

- PostgreSQL – for storing articles
- Redis – for caching feeds
- Elasticsearch – for storing logs
- Kibana – for viewing logs and dashboards

#### Important Notes

```sh
gator add --name "alem-platform" --url "https://platform.alem.school/news" --interval 2m
```

Adds a new RSS feed to PostgreSQL. Then reads/writes/updates feeds immediately in database.

```sh
gator list
```

Shows a list of all the added feeds.

```sh
gator delete --name "alem-platform"
```

Deletes the RSS feed from PostgreSQL.

```sh
gator articles --num 5
```

Shows the latest N articles from Redis or the database. By default -- 3 articles.

```sh
gator --help
```

Shows the help for commands.

#### Architecture

Follow the principles of Clean Architecture (also known as Hexagonal or Layered Architecture):

- `domain/` — Domain models and interfaces (pure business logic):

  - Feed, Article structs
  - Repository interfaces (FeedRepository, ArticleRepository, Cache, etc.)

- `app/` — Application services (use cases, orchestrating logic):

  - Logic for adding feeds, fetching and deduplicating articles
  - Coordinating persistence and caching
  - Stateless, depends only on domain interfaces

- `adapters/` — Concrete implementations of external systems:

  - `postgres/` — PostgreSQL repository implementations
  - `redis/` — Redis-based cache implementation
  - `rss/` — RSS and Atom feed fetcher (via HTTP + XML parsing)
  - `logger/` — Elasticsearch logger via structured JSON logs

- `cli/` — Command Line Interface:

  - User interaction layer (e.g., add-feed, fetch, list)
  - Depends on app, not on adapters directly

#### RSS

The whole point of the gator program is to fetch the RSS feed of a website and store its content in a structured format in our database. That way we can display it nicely in our CLI.

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
    <description>Here's the content of the second article.</description>
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

#### Aggregate

Feeds are essentially just lists of posts. A post represents a single web page. The entire point of the gator program is to fetch the actual posts from the feed URLs and store them in database. That way we can display them nicely in CLI.

Need to create a mechanism that will regularly receive articles from the database, compare them with incoming articles from the RSS feed and record changes in the database. The priority is the one that has not been updated the longest or has never been updated.

The mechanism should run against the background at the interval specified by the command line argument (for example: 1m, 1h).

A message should appear at startup.

```sh
./gator add --name "alem-platform" --url "https://platform.alem.school/news" --interval 2m
$ Collecting feeds every 2m...
```

A small hint for run mechanism against the background is use infinity `for` loop with time.Ticker

```go
ticker := time.NewTicker(interval)
defer ticker.Stop()

for range ticker.C {
	scrapeFeeds(s)
}
```

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
migrate -path ./db/migrations -database "postgres://user:pass@localhost:5432/gator?sslmode=disable" up
```

Rollback migration

```sh
migrate -path ./db/migrations -database "postgres://user:pass@localhost:5432/gator?sslmode=disable" down
```

#### Redis

- Key: `articles:<feed_name>`
- Value: JSON list of the latest articles
- TTL: 10 minutes
- If Redis is unavailable — fallback to PostgreSQL with a warning.

#### Elasticsearch

Log all actions (fetch, list, errors, deduplication).

Example:

```json
{
  "timestamp": "2025-06-13T10:05:22Z",
  "level": "INFO",
  "event": "article_saved",
  "feed": "Golang Blog",
  "title": "Go 1.23 Released",
  "link": "https://blog.golang.org/go1.23"
}
```

- Use `log/slog` with JSON handler
- Send logs to Elasticsearch over HTTP
- Levels: `INFO`, `ERROR`, `DEBUG`

#### Kibana

- Deployed via Docker Compose
- Port: `5601`
- Indexes: `gator-logs`

Dashboard includes:

- Logs with histogram (x=timestamp, y=count_of-logs)

### Initial Setup

#### Configuration

```yaml
postgres:
  host: localhost
  port: 5432
  user: postgres
  password: changem
  dbname: gator

redis:
  host: localhost
  port: 6379

elasticsearch:
  host: http://localhost:9200
  index: gator-logs
```

#### Docker Compose

Launches:

- PostgreSQL (port: `5432`)
- Redis (port: `6379`)
- Elasticsearch (port: `9200`)
- Kibana (port: `5601`)

## Example Usage

```sh
$ ./gator --help

  Usage:
    gator COMMAND [OPTIONS]

  Common Commands:
       add            Adds a new RSS feed to PostgreSQL. Then reads/writes/updates feeds immediately in database.
       list           Shows a list of all the added feeds
       delete         Deletes the RSS feed from PostgreSQL
       articles       Shows the latest N articles from Redis or the database. By default -- 3 articles.
```

## Suggestions

- Start with a single feed and CLI command
- Add Redis caching
- Implement PostgreSQL saving and create migrations
- Set up Elasticsearch logging
- Build Docker Compose
- Finalize with Kibana dashboard

Here are a few RSS feeds to get you started:

- TechCrunch: `https://techcrunch.com/feed/`
- Hacker News: `https://news.ycombinator.com/rss`

## Additionally _(\*not necessarily)_

For your own development:

- You can add multiple users and everyone can subscribe to certain news.

- Receive multiple RSS feeds with the `gator add ...` command and process them using the worker pool.

- You can take the RSS feed reading and parsing mechanism to a separate service and add it to `docker compose` services and communicate with it using different protocols (HTTP, GRPC, and)
