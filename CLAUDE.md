# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Stan's Robot Shop — a sample microservice e-commerce application for learning containerized orchestration and monitoring. Intentionally uses a polyglot tech stack. Pre-instrumented with Instana APM for distributed tracing. This is a demo app; authentication and error handling are intentionally minimal.

## Build and Run Commands

```bash
# Build all service images
export INSTANA_AGENT_KEY="<key>"   # needed for Nginx tracing module
docker-compose build

# Run locally (images are on Docker Hub if you skip the build)
docker-compose pull
docker-compose up                  # app available at http://localhost:8080

# Run with load generation
docker-compose -f docker-compose.yaml -f docker-compose-load.yaml up

# Build/run a single service (e.g. cart)
docker-compose build cart
docker-compose up cart

# Push images to a custom registry (edit .env first)
docker-compose push

# Kubernetes deployment via Helm
helm install robot-shop K8s/helm/ --set image.repo=robotshop --set image.version=latest

# Load generator standalone
cd load-gen && ./load-gen.sh -n 5 -t 10m -h http://localhost:8080
```

The `.env` file controls the Docker image registry (`REPO=robotshop`) and version tag (`TAG=2.1.0`).

## Architecture

### Three-Tier Pattern with API Gateway

```
Nginx (web:8080)  ──→  Microservices  ──→  Data Stores
   API Gateway         Business Logic       MongoDB, MySQL, Redis, RabbitMQ
```

Nginx serves the AngularJS frontend and reverse-proxies all `/api/*` routes to backend services. Clients never talk to backends directly. Routing is defined in `web/default.conf.template`.

### Services and Their Tech Stacks

| Service | Language | Data Store | Port |
|---------|----------|------------|------|
| catalogue | Node.js/Express | MongoDB | 8080 |
| user | Node.js/Express | MongoDB + Redis | 8080 |
| cart | Node.js/Express | Redis | 8080 |
| shipping | Java/Spring Boot | MySQL | 8080 |
| payment | Python/Flask | RabbitMQ (publish) | 8080 |
| ratings | PHP/Symfony | MySQL | 80 |
| dispatch | Go | RabbitMQ (consume) | - |
| web | Nginx + AngularJS 1.x | - | 8080 |

### Communication Patterns

- **Synchronous REST**: Client → Nginx → backend service (all `/api/*` routes)
- **Asynchronous messaging**: `payment` publishes to RabbitMQ → `dispatch` consumes
- **Database-per-service**: each service owns its data store, no shared databases

### Frontend

AngularJS 1.x SPA in `web/static/`. Single controller file (`js/controller.js`) with all route controllers. User state is held in an in-memory AngularJS factory (`currentUser`), not persisted across page refreshes.

### Nginx API Gateway Routing

Defined in `web/default.conf.template` — uses environment variables (`${CATALOGUE_HOST}`, `${USER_HOST}`, etc.) resolved at container startup via `web/entrypoint.sh`.

## Deployment Targets

- **Local**: `docker-compose.yaml`
- **Kubernetes**: `K8s/helm/` (Helm chart, `values.yaml` for config)
- **Cloud-specific**: `EKS/`, `AKS/`, `GKE/` (docs and Helm overrides)
- **Other**: `OpenShift/`, `DCOS/`, `Swarm/`

Helm values of note: `psp.enabled`, `nodeport` (for minikube), `openshift`, `redis.storageClassName`, `payment.gateway`, `eum.key`/`eum.url` for Instana End-User Monitoring.

## Observability

- All services include Instana tracing instrumentation (must be initialized before other imports in Node.js services)
- Prometheus metrics endpoints: `/api/cart/metrics` and `/api/payment/metrics`
- Health check endpoints: `/health` on all services (ratings uses `/_health`)
- Logging: `pino` (Node.js), standard logging (Python/Java/Go/PHP), configured via docker-compose JSON file driver

## Known Deprecations

This codebase has significant deprecated dependencies. Key items:
- Node.js services use `mongodb` ^3.x (callback API), `redis` ^2.x, deprecated `body-parser` and `request` packages
- Java shipping service uses Spring Boot 2.3.x, `javax.*` namespaces, `HandlerInterceptorAdapter`, legacy MySQL driver
- Go dispatch uses archived `streadway/amqp` and `opentracing-go`
- PHP ratings uses PHP 7.4, Symfony `RouteCollectionBuilder`, Doctrine annotations
- Dockerfiles use EOL base images (Node 14, OpenJDK 8, MySQL 5.7, PHP 7.4, Debian 10, Go 1.17)
- K8s manifests use removed `PodSecurityPolicy` and Helm Chart `apiVersion: v1`
- `docker-compose.yaml` uses deprecated `version: '3'` key
