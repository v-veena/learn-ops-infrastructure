# Tech Stack Manual

## 1. Run Questions

### 1a. Config Files

| Config File | Location | Config Value | What it's for | How it's used |
|.env |root |POSTGRES_DB|Names the actual database/schema that gets created inside the Postgres container|When Postgres starts up for the first time, it creates a database with this name — all your app's tables live inside it|
|.env |root  | POSTGRES_USER| The username Postgres creates and that the app uses to authenticate| Combined into the connection string below; whichever service needs to talk to the DB uses this credential to log in|
|.env | root | POSTGRES_PASSWORD | The password paired with that user | Same as above — part of the authentication handshake between app and DB|
| .env | root| DATA_SOURCE_NAME | A full connection string that combines everything above into one URL your app's code can use directly | This is likely read by your API service (probably via an env var lookup in its code) to actually connect to Postgres. It says @database:5432 — database isn't a real hostname on the internet, it's the service name from docker-compose.yml, which is how containers find each other on the same Docker network |

### 1b. How to Start It

pull → just pulls the Docker images (docker compose pull), doesn't start anything
up (depends on pull) → this is the main one: builds and starts all services in detached mode (-d), then tails the logs.
up-api → builds and starts only the api service (and whatever it depends on, per docker-compose.yml —  api depends on database) 
up-client-api → builds and starts only api and client (skips prometheus/grafana/postgres_exporter)

### 1c. Where to Access It

| Service | Port | URL |
|database|5433| it's a db port|
| api  | 8000 | http://localhost:8000|
| client | 3000 | http://localhost:3000 |
| prometheus | 9090 | http://localhost:9090 |
| grafana | 3001 | http://localhost:3001 |
| postgres_exporter | 9187 | http://localhost:9187 |


### 1d. Service Dependencies

| Service | Depends On | Why |
|api |database| it can't serve requests with out data to read/write|
| prometheus | api | prometheus scrapes metrics from the api service, so api has to exist first |
| grafana | prometheus | grafana visualizes data that prometheus collects, so it needs prometheus running to have anything to show |
| postgres_exporter | database | it exports database metrics, so it needs the database to connect to and monitor |


### 1e. Main Entry Points

| Service | Startup File | Routes / URL Config File |
|api|manage.py|LearningPlatform/urls.py|
|client | src/index.js | src/components/ApplicationViews.js |
| | | |
| | | |

---

## 2. Services

| Service Name | Tech Stack (including version) | Purpose |
|database | postgressql16 | store data|
| api | python,Django,Django REST Framework  | backend rest api handling businesslogic, authentication(github oauth) and database access |
Client| node22.13.0 ,React 16.13.1 | frontend web application
Prometheus | prometheus(latest) | scrapes and stores metrics from api
grafana| Grafana(latest) | visuvalizes metrics
postgres_exporter | postgres-exporter(latest) | exports postgresSQL database metrics from prometheus to escape


---

## 3. System Overview
