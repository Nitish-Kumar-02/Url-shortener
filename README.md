# URL Shortener with Click Analytics

A production-style URL shortener built with Spring Boot. Shortens long URLs into
compact codes, redirects visitors, and tracks click analytics — with Redis caching
on the hot read path and PostgreSQL for durable storage.

## Why this project

Most high-traffic systems are read-heavy and skewed: a small number of links get
most of the clicks. This project is built around that observation:

- **Cache-aside pattern with Redis** — every redirect checks Redis before hitting
  the database. On a cache miss, it reads from Postgres and populates the cache
  for next time. This keeps the hot path (redirects) fast and takes load off the
  database.
- **Atomic click counting** — click counts are incremented with a single `UPDATE`
  query at the database level (not read-modify-write in application code), so
  concurrent clicks on the same link never lose updates.
- **Random Base62 codes** — short codes are generated randomly rather than from a
  sequential counter, so links aren't guessable/enumerable.
- **Separate click-event log** — every click is also logged as its own row
  (timestamp, referrer, user agent), so analytics can answer "when" and "from
  where", not just "how many".

## Tech stack

- Java 17, Spring Boot 3
- PostgreSQL (persistent storage), Redis (caching)
- Spring Data JPA, Spring Validation
- springdoc-openapi (Swagger UI)
- JUnit 5 + Mockito for tests
- Docker & Docker Compose

## Running it

### With Docker (easiest)

```bash
docker-compose up --build
```

This starts the app, PostgreSQL, and Redis together. The API is at
`http://localhost:8080`, and Swagger UI at `http://localhost:8080/swagger-ui.html`.

### Locally (without Docker)

You'll need PostgreSQL and Redis running locally, then:

```bash
mvn spring-boot:run
```

Configuration (DB/Redis host, port, credentials) is read from environment
variables — see `application.yml` for defaults.

## API

**Create a short URL**
```
POST /api/urls
Content-Type: application/json

{
  "originalUrl": "https://example.com/some/very/long/path",
  "customAlias": "my-link",       // optional
  "expiresInDays": 30              // optional
}
```

**Visit a short link** (redirects, records the click)
```
GET /{shortCode}
```

**View analytics**
```
GET /api/urls/{shortCode}/analytics
```

Full interactive docs at `/swagger-ui.html` once the app is running.

## Running tests

```bash
mvn test
```

Service-layer tests mock the repository and Redis layers, so they run without
needing a real database or Redis instance — useful for CI.

## Possible extensions

- Rate limiting per IP (token bucket in Redis)
- Admin endpoint to deactivate/delete a link (with cache eviction)
- QR code generation for each short link
- Async click logging via a message queue instead of a synchronous DB write

## What to say about it in an interview

> "It's a read-heavy system, so I added a Redis cache-aside layer in front of
> Postgres to keep redirects fast under load. Click counts are incremented
> atomically at the database level to avoid race conditions, and I log each
> click as a separate event so the analytics aren't just a counter — they can
> show click timing and referrer trends too."
