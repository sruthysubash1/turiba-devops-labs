# Lab 4 · Multi-service app with Compose

GitHub user: sruthysubash1 · Reference files used (Lab 2 fallback): yes

<!-- Keep every heading as it is: the grader finds the sections by name. Fill in every blank (four underscores) and your GitHub user name: check.sh writeup flags anything left. -->

## 1 · Architecture

```mermaid
flowchart LR
  B["Browser on your laptop"]
  subgraph front["network: front"]
    W["web: nginx :80"]
  end
  A["api :5000 · networks: front, back"]
  subgraph back["network: back"]
    D[("db :5432")]
  end
  V[("volume pgdata")]
  B -->|"HTTP localhost:8080"| W
  W -->|"HTTP api:5000"| A
  A -->|"PostgreSQL db:5432"| D
  D --- V
```

Which arrow crosses from your laptop into Docker: The HTTP localhost:8080 arrow from the browser to web.

## 2 · Predictions and what happened

Part 1: fill in "I predicted" (the letter and a one-line reason) before you write any YAML. Part 3: fill in the rest. A wrong prediction that you explain earns full marks.

| # | I predicted | What I saw (command → the deciding output line) | Match? | If not: why |
|---|-------------|--------------------------------------------------|--------|-------------|
| P1 | Inside the api container, 127.0.0.1 refers to that same container, not PostgreSQL. Use DB_HOST=db so Docker Compose can resolve the database service name. | The browser sends a request to localhost:8080. Docker forwards it to Nginx on port 80, Nginx proxies it to api:5000, and the API communicates with PostgreSQL at db:5432. The response returns through the same path. | PostgreSQL stores its data in the named pgdata volume. docker compose down removes containers but keeps the volume; docker compose down -v also removes the volume and its data. | |
| P2 | B — localhost:5432 should be unreachable because db has no published port. | Run the database-stop experiment and record the actual result. | Pending | |
| P3 | A — todos should survive docker compose down because the named volume persists. | Run the persistence experiment and record the actual result. | Pending | |
| P4 | A — the SQL dump should contain the saved todo. | Run the dump experiment and record the actual result. | Pending | |
| P5 | A — database-backed requests should fail when db stops. | Run the database-stop experiment and record the actual result. | Pending | |
| P6 | A — restoring the SQL dump should restore the saved todo. | Run the restore experiment and record the actual result. | Pending | |

## 3 · Backup and restore

```text
(paste the whole output of step 3.3: pg_dump, down -v, the count query, the restore, and the GET with your todo back)
```

Why the restore must run before `api` starts: Restore the database before starting the API so the required tables and data are available when the API connects.

## 4 · Compose clinic

### c1 (worked example)
- Error line (copied): `connect ECONNREFUSED 127.0.0.1:5432`
- Checkpoint: port, inside `api`
- Fix: `DB_HOST: db`
- Why this fix, and not the tempting one: inside `api`, `localhost` is `api` itself, and nothing listens on 5432 there. Docker's DNS answers `db` with the database container's address. Publishing `db`'s port would "work" by opening a door into the database.
- Hints used: 0

### c2
- Committed fix: `lab4/clinic/c2/`
- Problem: The host port was mapped to the wrong container port.
- Fix: Map host port 8102 to container port 3000 (`8102:3000`).
- Why: The Nginx configuration listens on port 3000, so Docker must forward traffic to that port.

### c3
- Committed fix: `lab4/clinic/c3/`
- Problem: The web and API services did not share a network.
- Fix: Connect the API to both `front` and `back` networks.
- Why: Web can reach API over `front`, and API can reach the database over `back`.

### c4
- Committed fix: `lab4/clinic/c4/`
- Problem: Browser JavaScript tried to access `api:5000` directly.
- Fix: Use `/api/todos` as the API URL.
- Why: The hostname `api` is resolvable inside Docker, not directly by the user's browser. Nginx proxies the same-origin request to the API.

### c5
- Committed fix: `lab4/clinic/c5/`
- Problem: PostgreSQL data was not stored in a persistent named volume.
- Fix: Mount `pgdata` at `/var/lib/postgresql/data` and declare `pgdata` as a top-level volume.
- Why: A named volume preserves database data when containers are removed and recreated.

## 5 · Run in one command

```bash
git clone https://github.com/sruthysubash1/turiba-devops-labs.git && cd turiba-devops-labs
docker compose up -d --build --wait
docker compose ps
```

## 6 · Answers

### A1
In c1, 127.0.0.1 refers to the API container itself, not PostgreSQL. Docker Compose DNS resolves db to the database container on the shared network.

### A2
The browser sends requests to localhost:8080. Docker forwards them to Nginx on port 80; Nginx proxies to api:5000; the API connects to PostgreSQL at db:5432. The database needs no published host port because the API reaches it over Docker's internal network.

### A3
PostgreSQL data is stored in the named pgdata volume. docker compose down removes containers and networks but preserves the volume. docker compose down -v also removes the named volume and its data.

### A4
The API process may remain healthy while PostgreSQL is stopped, but database-backed requests can fail. service_healthy delays API startup until the database is healthy; it cannot prevent later outages.
