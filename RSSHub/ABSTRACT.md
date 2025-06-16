# RSSHub

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