 
# Ticketing — Microservices Example

This repository contains a microservices-based ticketing application used for learning and demos. The services are split into separate folders (for example, `auth`, `tickets`, `orders`, `payments`, `expiration`, and `client`) and are designed to be developed, tested, and deployed independently. This README focuses on the overall project, the `auth` service (which is present in this workspace), and how to run the system locally or on Kubernetes using the provided manifests and `skaffold.yaml`.

## Table of contents

- Project overview
- Services (what's included)
- Auth service — routes and behaviour
- Local development
- Running tests
- Docker / Kubernetes (skaffold)
- Environment variables
- Example API requests
- Contributing
- License

## Project overview

The ticketing project models an event ticketing system implemented as a set of microservices. Each service owns its own database and REST API and communicates with others using HTTP or events (depending on the service). The repo uses TypeScript and Node.js for services and includes docker/Kubernetes manifests to run the system in a cluster.

This workspace contains (at least) the following directories:

- `auth/` — Authentication service (signup, signin, signout, current-user). This service is implemented with Express + MongoDB and issues JWTs stored in a cookie-session.
- `tickets/` — (expected) Tickets service (create/list tickets).
- `orders/` — (expected) Orders service.
- `payments/` — (expected) Payments service.
- `expiration/` — (expected) Expiration service (background job to expire unpaid orders).
- `client/` — (expected) Frontend client.
- `infra/k8s/` — Kubernetes manifests for services and MongoDB used by the examples.
- `skaffold.yaml` — Skaffold configuration to build and run images locally within a Kubernetes cluster.

> Note: In this workspace I inspected the `auth` service code and the `infra/k8s` manifests. The README below gives concrete instructions for running `auth` locally and for deploying the overall system.

## Services (high-level)

- Auth: Handles user accounts and authentication. Endpoints in `auth/src/routes` include:
	- `POST /api/users/signup` — create a user and set JWT cookie
	- `POST /api/users/signin` — validate credentials and set JWT cookie
	- `POST /api/users/signout` — clear session cookie
	- `GET /api/users/currentuser` — return current user (from JWT)

- Other microservices (tickets, orders, payments, expiration, client) follow similar patterns (separate folders, Dockerfiles, and manifests). The `skaffold.yaml` lists all images and contexts.

## Auth service — implementation notes

- Language / runtime: TypeScript, Node.js, Express
- Data store: MongoDB (uses `mongoose`)
- Auth: JWT generated with a `JWT_KEY` environment variable and stored in a cookie via `cookie-session`.
- Passwords: `crypto.scrypt` hashing with a salt (see `auth/src/services/password.ts`).
- Validation: `express-validator` + a shared `validateRequest` middleware (the project imports a `@rallycoding/common` package with common middleware and error classes used across services).

Model: `User` (in `auth/src/models/user.ts`)
- fields: `email`, `password`.
- Password stored hashed; the schema strips `password` and internal fields when converting to JSON.

## Local development

Prerequisites

- Node.js (v16+ recommended)
- npm or yarn
- MongoDB running locally or via Docker

Run the auth service locally

1. Install dependencies for the `auth` service:

```bash
cd auth
npm install
```

2. Start a local MongoDB instance (one easy way is via Docker):

```bash
# from the project root, or anywhere
docker run --name ticketing-auth-mongo -p 27017:27017 -d mongo
```

3. Start the auth service with required env vars (in a separate terminal):

```bash
# from project root
cd auth
# set env vars in your shell or use a .env manager. Example (zsh):
export MONGO_URI='mongodb://localhost:27017/auth'
export JWT_KEY='your_jwt_secret_here'
npm start
```

The `auth` service listens on port `3000` by default.

Run the whole system with skaffold + a local Kubernetes cluster (recommended for multi-service integration)

1. Ensure you have a local k8s cluster with an ingress controller (e.g., minikube, kind + nginx-ingress) and `skaffold` installed.

2. Run skaffold from the repository root (this will build images and apply manifests in `infra/k8s`):

```bash
skaffold dev
```

Note: `skaffold.yaml` in this project references images and contexts for each service and the `infra/k8s` manifests (for example `auth-depl.yaml`, `auth-mongo-depl.yaml`, and `ingress-srv.yaml`). The `auth` deployment references a secret `jwt-secret` for `JWT_KEY` — you must create that secret in the cluster before deploying:

```bash
kubectl create secret generic jwt-secret --from-literal=JWT_KEY='your_jwt_secret_here'
kubectl apply -f infra/k8s/auth-mongo-depl.yaml
kubectl apply -f infra/k8s/auth-depl.yaml
# apply other service manifests or use skaffold to apply all
```

## Environment variables

Common env vars used by the `auth` service:

- `MONGO_URI` — MongoDB connection string (e.g. `mongodb://localhost:27017/auth`)
- `JWT_KEY` — secret used to sign JWTs (required)

When deploying to Kubernetes the `auth` manifest expects a secret named `jwt-secret` with key `JWT_KEY`.

## Tests

Run unit/integration tests for the `auth` service:

```bash
cd auth
npm test          # runs jest in watch mode by default
npm run test:ci   # run tests once (CI-friendly)
```

The project uses `mongodb-memory-server` for tests so you don't need a running MongoDB to execute the test suite.

## Example API requests

Signup

```bash
curl -X POST http://localhost:3000/api/users/signup \
	-H "Content-Type: application/json" \
	-d '{"email":"user@example.com", "password":"pass1234"}' \
	-i
```

Signin

```bash
curl -X POST http://localhost:3000/api/users/signin \
	-H "Content-Type: application/json" \
	-d '{"email":"user@example.com", "password":"pass1234"}' \
	-i
```

Signout

```bash
curl -X POST http://localhost:3000/api/users/signout -i
```

Current user

```bash
curl http://localhost:3000/api/users/currentuser -i
```

Note: The service uses a cookie-based session to store a JWT. When calling the endpoints from curl you may need to enable cookie handling or use a client (Postman / browser) that persists cookies.

## Contributing

- Follow standard GitHub workflow: branch off `main`, open PRs for changes, and run tests locally before pushing.
- If you add or modify services, update `skaffold.yaml` and `infra/k8s` manifests as needed.

## Where to go from here

- Implement additional services (`tickets`, `orders`, `payments`) following the same folder structure and patterns used in `auth`.
- Add CI workflows to run tests and build images automatically.

---

If you'd like, I can also:

- Add a `.env.example` with required environment variables.
- Create a simple `Makefile` or NPM scripts at the repo root to bring up the local stack with Docker Compose.
- Open a pull request with the README update.

Tell me which of those you'd like me to do next.

