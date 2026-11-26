# Turiba DevOps Capstone

## Run it from zero

### 1. Clone the repository

Clone the GitHub repository and enter the project directory.

`git clone git@github.com:sruthysubash1/turiba-devops-labs.git`

`cd turiba-devops-labs`

### 2. Configure the environment

Create the local environment file using `.env.example`.

`cp .env.example .env`

Replace the example database password in `.env` with your own password. Do not commit `.env` to GitHub.

### 3. Start the application with Docker Compose

Validate the configuration:

`/usr/local/lib/docker/cli-plugins/docker-compose config -q`

Start the services:

`/usr/local/lib/docker/cli-plugins/docker-compose up -d`

Check the containers:

`/usr/local/lib/docker/cli-plugins/docker-compose ps`

Open the application at http://localhost:8080.

Test the API:

`curl -s http://localhost:8080/api/todos`

Create a todo:

`curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"capstone test"}' http://localhost:8080/api/todos`

### 4. Create a Kubernetes cluster

Create a local kind cluster:

`kind create cluster --name capstone-check`

`kubectl get nodes`

Create the namespace:

`kubectl create namespace course`

Create the database Secret. Replace the example password with your own:

`kubectl -n course create secret generic db-secret --from-literal=password='replace-with-your-own-password'`

### 5. Deploy the Helm chart

Validate the chart:

`helm lint charts/course-app`

`helm template course-app charts/course-app -n course`

Install the application:

`helm install course-app charts/course-app -n course`

Wait for the deployments:

`kubectl -n course rollout status deployment/db`

`kubectl -n course rollout status deployment/api`

Check the pods:

`kubectl -n course get pods`

The expected deployment has three API replicas and one database pod.

### 6. Test the Kubernetes application

Start port forwarding:

`kubectl -n course port-forward service/api 8080:80`

Keep this command running in one terminal. Open another terminal to test the API.

`curl -s http://localhost:8080/api/todos`

Create a todo and check it again.

### 7. Verify database persistence

Find the database pod:

`kubectl -n course get pods`

Delete the database pod using its actual name:

`kubectl -n course delete pod DATABASE_POD_NAME`

Wait for the database deployment:

`kubectl -n course rollout status deployment/db`

Check the todos again. The previously created todo should still exist because the database uses persistent storage.

### 8. Test Helm upgrade and rollback

Upgrade the API to two replicas:

`helm upgrade course-app charts/course-app -n course --set api.replicas=2`

Check the pods:

`kubectl -n course get pods`

Roll back to revision 1:

`helm rollback course-app 1 -n course`

Check the restored deployment:

`kubectl -n course get pods`

`helm history course-app -n course`

The API should return to three replicas.

## Architecture

The application has three main services:

- Web: serves the frontend and publishes port 8080.
- API: provides the todo API and communicates with PostgreSQL.
- Database: PostgreSQL 17 stores todo records.

Docker Compose uses two internal networks:

- The front network connects the web service and API.
- The back network connects the API and database.

PostgreSQL uses the pgdata named volume for persistent storage.

In Kubernetes, Helm deploys the application. The API has three replicas. PostgreSQL uses persistent storage, and a Kubernetes Secret supplies the database password. Kubernetes Services provide internal communication.

The API image is published to GitHub Container Registry (GHCR).

## What is where

- `api/` - backend API application.
- `web/` - frontend application.
- `compose.yaml` - Docker Compose configuration.
- `.env.example` - example environment configuration.
- `charts/course-app/` - Helm chart.
- `charts/course-app/Chart.yaml` - chart and application versions.
- `charts/course-app/values.yaml` - image settings and API replicas.
- `charts/course-app/templates/` - Kubernetes resource templates.
- `.github/workflows/ci.yml` - CI workflow.
- `.github/workflows/release.yml` - image publishing and GitHub Release workflow.
- `.github/workflows/lab05-graph.yml` - existing Lab 5 workflow.
- `lab09-package-it/` - Lab 9 instructions, checks and reference material.
- `capstone/README.md` - capstone documentation.

The capstone uses the existing application, CI workflow, release workflow and Helm chart. Lab 9 reference material is available under `lab09-package-it/reference/`.

## Level and evidence

### CI and release

- Enabled GitHub Actions on the repository fork.
- Triggered CI using an empty commit.
- Verified all four CI jobs: lint, unit, integration and image.
- Merged the chart version update through a pull request.
- Published release `v1.0.0`.
- Verified that the GHCR package is public.
- Successfully pulled `ghcr.io/sruthysubash1/turiba-devops-labs/api:1.0.0`.

### Docker Compose

- Cloned the repository into a separate `capstone-check` directory.
- Validated the Compose configuration.
- Started the database, API and web containers.
- Verified that all three containers were running.
- Tested the API GET and POST endpoints.
- Created and retrieved todo records.

### Kubernetes and Helm

- Created a kind cluster named `capstone-check`.
- Installed the application using Helm.
- Verified three API pods and one database pod were running.
- Confirmed Kubernetes used the own GHCR image tagged `1.0.0`.
- Created a todo and verified that it remained available after deleting and recreating the database pod.
- Upgraded the API replica count to two.
- Rolled back to revision 1 and verified that three API replicas were restored.
- Verified Helm release revision 3 was deployed.

## When it breaks

### Docker Compose problems

Check the configuration and container logs:

`/usr/local/lib/docker/cli-plugins/docker-compose config`

`/usr/local/lib/docker/cli-plugins/docker-compose ps`

`/usr/local/lib/docker/cli-plugins/docker-compose logs`

### Kubernetes pod problems

`kubectl -n course get pods`

`kubectl -n course describe pods`

`kubectl -n course get events --sort-by=.metadata.creationTimestamp`

### Database connection problems

`kubectl -n course get secret db-secret`

`kubectl -n course logs deployment/db`

### Application access problems

`kubectl -n course get services`

`kubectl -n course port-forward service/api 8080:80`

### Helm problems

`helm lint charts/course-app`

`helm status course-app -n course`

`helm history course-app -n course`

## Known issues

- In the WSL environment used for testing, the Docker Compose plugin was available through its full executable path even though `docker compose` was not recognised by the Docker CLI. The full executable path worked for configuration validation and starting the services.
- The Kubernetes deployment requires the `db-secret` Secret to be created before installing the Helm chart.
- The database password must be supplied locally and must not be committed to the repository.
- The capstone Kubernetes setup uses a manually created kind cluster rather than automated infrastructure provisioning.

## AI use

ChatGPT was used as a learning and troubleshooting assistant during the capstone.

It helped explain Docker, Docker Compose, Kubernetes, Helm, Git branches and GitHub Actions. It also helped troubleshoot environment and deployment issues and organise the documentation.

Commands were run and tested in the local environment. Container status, API responses, deployment results, Helm operations and GitHub Actions results were checked during the work.
