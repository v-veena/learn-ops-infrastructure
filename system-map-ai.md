# System Map (AI)

## 1. System Diagram
```mermaid
graph LR
    Client["learn-ops-client<br/>React 16 (SPA)<br/>:3000"]
    API["learn-ops-api<br/>Django + Django REST Framework<br/>:8000"]
    DB["PostgreSQL 16<br/>:5432"]
    Valkey["Valkey<br/>(Redis-compatible)<br/>:6379"]
    Monarch["service-monarch<br/>Python (asyncio worker)<br/>:8080 metrics / :8081 log UI"]
    GitHub["GitHub API<br/>(external)<br/>api.github.com"]
    Slack["Slack API<br/>(external)"]
    Exporter["postgres_exporter<br/>:9187"]
    Prometheus["Prometheus<br/>:9090"]
    Grafana["Grafana<br/>:3001"]

    Client -->|"HTTP REST/JSON<br/>Token auth :8000"| API
    API -->|"DB query<br/>Django ORM/psycopg2 :5432"| DB
    API -->|"cache GET / PUBLISH<br/>valkey-py :6379"| Valkey
    API -->|"HTTPS OAuth2<br/>allauth/dj-rest-auth"| GitHub
    Valkey -->|"PUB/SUB subscribe<br/>channel_migrate_issue_tickets :6379"| Monarch
    Monarch -->|"HTTPS REST<br/>issues API, GH_PAT"| GitHub
    Monarch -->|"HTTPS POST<br/>chat.postMessage"| Slack
    Exporter -->|"DB query<br/>:5432"| DB
    Prometheus -->|"HTTP scrape<br/>GET /metrics/metrics :8000"| API
    Prometheus -->|"HTTP scrape<br/>:9187"| Exporter
    Grafana -->|"HTTP query<br/>:9090"| Prometheus
```