# RSS Aggregator Project – Peer Review Checklist

## Project Architecture & Clean Code

### The project follows Clean Architecture principles (domain, app, adapters, cli separation)

- [ ] Yes
- [ ] No

### All core logic is separated from infrastructure and frameworks

- [ ] Yes
- [ ] No

### The directory structure matches the documented layout (cmd, domain, internal, adapters, handlers, cli)

- [ ] Yes
- [ ] No

### All structs and functions follow Go naming conventions (PascalCase, camelCase)

- [ ] Yes
- [ ] No

### The code is formatted using `gofumpt`

- [ ] Yes
- [ ] No

---

## CLI Functionality

### The CLI supports adding a feed using the `add` command

- [ ] Yes
- [ ] No

### The CLI supports listing feeds using the `list` command

- [ ] Yes
- [ ] No

### The CLI supports deleting feeds using the `delete` command

- [ ] Yes
- [ ] No

### The CLI supports fetching feeds at intervals using `fetch --interval`

- [ ] Yes
- [ ] No

### The `articles` command returns the latest N articles from Redis or DB

- [ ] Yes
- [ ] No

### A helpful message is shown using `--help`

- [ ] Yes
- [ ] No

---

## Error Handling & Stability

### The program does not panic during normal use (e.g. nil dereference, index out of range)

- [ ] Yes
- [ ] No

### All errors return clear messages to the user

- [ ] Yes
- [ ] No

### The program exits with non-zero status on CLI errors (invalid args, etc.)

- [ ] Yes
- [ ] No

---

## HTTP Endpoints

### The CLI exposes `GET /feeds/outdated?n=10`

- [ ] Yes
- [ ] No

### The CLI accepts `POST /feeds/update` with parsed articles

- [ ] Yes
- [ ] No

### The endpoints are tested or verified manually

- [ ] Yes
- [ ] No

---

## Background RSS Listener

### The RSSListener service polls RSS feeds at intervals

- [ ] Yes
- [ ] No

### The RSSListener does **not** access Redis or PostgreSQL directly

- [ ] Yes
- [ ] No

### The RSSListener sends feed data via HTTP to the CLI service

- [ ] Yes
- [ ] No

### The interval for polling can be configured via CLI argument

- [ ] Yes
- [ ] No

---

## Worker Pool

### The background service implements a worker pool using goroutines

- [ ] Yes
- [ ] No

### The number of workers is configurable

- [ ] Yes
- [ ] No

### Workers use a channel to queue feeds

- [ ] Yes
- [ ] No

### Workers use `sync.WaitGroup` to wait for all tasks to complete

- [ ] Yes
- [ ] No

---

## Redis Caching

### Articles are cached in Redis under `articles:<feed_name>`

- [ ] Yes
- [ ] No

### Redis entries have a TTL of 10 minutes

- [ ] Yes
- [ ] No

### If Redis is unavailable, data is fetched from PostgreSQL with a warning

- [ ] Yes
- [ ] No

---

## PostgreSQL & Migrations

### Feed and article data is persisted in PostgreSQL

- [ ] Yes
- [ ] No

### The `feeds` table contains fields: id, name, url, description, created_at, updated_at

- [ ] Yes
- [ ] No

### The `articles` table contains fields: id, title, url, feed_id, description, published_at, etc.

- [ ] Yes
- [ ] No

### Database migrations are used to manage schema changes

- [ ] Yes
- [ ] No

### Migrations follow the up/down format and are tested

- [ ] Yes
- [ ] No

---

## Docker & Environment

### Docker Compose is used to run PostgreSQL, Redis, and RSSListener

- [ ] Yes
- [ ] No

### The application reads configuration from environment variables or config files

- [ ] Yes
- [ ] No

---

## Final Sanity Checks

### The project builds with `go build -o rsshub .` without error

- [ ] Yes
- [ ] No

### The app prints the message "Collecting feeds every Xm..." on start

- [ ] Yes
- [ ] No

## Detailed Feedback

### What was great? What you liked the most about the program and the team performance?

### What could be better? How those improvements could positively impact the outcome?
