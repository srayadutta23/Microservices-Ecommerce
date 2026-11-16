# ShopKart: Microservices E-Commerce Platform

![CI-CD](https://github.com/srayadutta23/Microservices-Ecommerce/actions/workflows/ci-cd.yml/badge.svg)
![Docker](https://img.shields.io/badge/Docker-containers-2496ED?logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-k3s-326CE5?logo=kubernetes&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?logo=amazonaws&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)

A distributed e-commerce platform built with Node.js, Express, React, TypeScript and MongoDB.
It is containerised with **Docker**, built and published automatically with **GitHub Actions**,
orchestrated with **Kubernetes**, and deployed to a free-tier **AWS EC2** server.

## Live demo

| What | URL |
|---|---|
| Web app | **http://16.178.113.48** |
| API gateway health check | http://16.178.113.48:4000/health |
| Product API | http://16.178.113.48:4000/api/products |

> The demo server is stopped when it is not being presented, to save AWS credits. If the link
> does not open, the instance is switched off.

## Table of contents

1. [Architecture](#1-architecture)
2. [Tech stack and project layout](#2-tech-stack-and-project-layout)
3. [Run locally with Docker Compose](#3-run-locally-with-docker-compose)
4. [Docker images and Docker Hub](#4-docker-images-and-docker-hub)
5. [CI/CD with GitHub Actions](#5-cicd-with-github-actions)
6. [Kubernetes (local cluster)](#6-kubernetes-local-cluster)
7. [Deploy to AWS (EC2 + k3s)](#7-deploy-to-aws-ec2--k3s)
8. [Configuration reference](#8-configuration-reference)
9. [API reference](#9-api-reference)
10. [End-to-end testing](#10-end-to-end-testing)
11. [Troubleshooting (problems we hit and fixes)](#11-troubleshooting)
12. [Security notes and known limitations](#12-security-notes-and-known-limitations)
13. [Cost control](#13-cost-control)
14. [Contributors](#14-contributors)

---

## 1. Architecture

### Application

```
                    +-----------------------------+
                    |       React Frontend        |
                    |    (Nginx, port 80 / 5173)  |
                    +-----------------------------+
                                   |
                                   | HTTP (browser requests)
                                   v
                    +-----------------------------+
                    |         API Gateway         |
                    |         (port 4000)         |
                    +-----------------------------+
                      |            |            |
          +-----------+            |            +-----------+
          v                        v                        v
+-------------------+    +-------------------+    +-------------------+
|   User Service    |    |  Product Service  |    |   Order Service   |
|    (port 5001)    |    |    (port 5002)    |    |    (port 5003)    |
+-------------------+    +-------------------+    +-------------------+
          |                        ^             |          |
          |                        +-------------+          |
          |                  inter-service HTTP             |
          v                        v                        v
+-------------------+    +-------------------+    +-------------------+
|  MongoDB: user_db |    |MongoDB: product_db|    | MongoDB: order_db |
+-------------------+    +-------------------+    +-------------------+
```

Each service owns its own database (database-per-service). The Order Service calls the
Product Service over HTTP to reserve stock when an order is placed.

### DevOps pipeline

```
 Developer                GitHub                    Docker Hub              AWS EC2 (k3s)
 ---------                ------                    ----------              -------------
 git push main  ----->  GitHub Actions  ----->   sraya23/ecommerce-*  --->  kubectl apply
                        build 5 images           tags: latest + commit      pods pull images
                        push on main             SHA                        and run
```

### Port matrix

| Component | Container name (Compose) | Host port | Container port | Health probe |
|---|---|---|---|---|
| Frontend | ecommerce-frontend | 5173 | 80 | `GET /` |
| API Gateway | ecommerce-api-gateway | 4000 | 4000 | `GET /health` |
| User Service | ecommerce-user-service | 5001 | 5001 | `GET /health` |
| Product Service | ecommerce-product-service | 5002 | 5002 | `GET /health` |
| Order Service | ecommerce-order-service | 5003 | 5003 | `GET /health` |
| MongoDB | ecommerce-mongodb | 27017 | 27017 | `mongosh` ping |

## 2. Tech stack and project layout

**Stack:** Node.js, Express, React, TypeScript, Vite, MongoDB, Nginx, Docker, Docker Compose,
Kubernetes (Docker Desktop for local, k3s on AWS), GitHub Actions, AWS EC2.

```
Microservices-Ecommerce/
├── .github/workflows/ci-cd.yml     # CI/CD pipeline
├── api-gateway/                    # reverse proxy to the three services
├── services/
│   ├── user-service/               # registration, login, JWT
│   ├── product-service/            # catalogue, search, seed script
│   └── order-service/              # orders, stock reservation
├── frontend/                       # React + Vite, served by Nginx
├── infrastructure/kubernetes/
│   └── shopkart.yaml               # all Kubernetes resources
├── scripts/                        # image push helpers
├── docker-compose.yml              # local multi-container setup
└── README.md
```

## 3. Run locally with Docker Compose

**Prerequisites:** Docker Desktop installed and running.

```bash
git clone https://github.com/srayadutta23/Microservices-Ecommerce.git
cd Microservices-Ecommerce

docker compose config            # validate the file
docker compose up -d --build     # build and start all containers
docker compose ps                # all containers should be "healthy"
docker compose exec product-service npm run seed    # load 100 products
```

Open http://localhost:5173.

```bash
docker compose logs -f           # follow logs
docker compose down              # stop, keep the database volume
docker compose down -v           # stop and wipe the database
```

> **Fix we made:** the original Compose file published the gateway on `4001`, the user service
> on `5005` and the product service on `5004`, while the frontend is built to call
> `http://localhost:4000`. The mappings were changed to `4000:4000`, `5001:5001` and
> `5002:5002` so the host ports match the documentation and the frontend.

## 4. Docker images and Docker Hub

Each service has its own `Dockerfile`. Images are published under the `sraya23` Docker Hub
account:

| Component | Image |
|---|---|
| Frontend | `sraya23/ecommerce-frontend` |
| API Gateway | `sraya23/ecommerce-api-gateway` |
| User Service | `sraya23/ecommerce-user-service` |
| Product Service | `sraya23/ecommerce-product-service` |
| Order Service | `sraya23/ecommerce-order-service` |

Tags used: `v1.0.0` (first manual release), `latest`, and the full Git commit SHA (added by CI).

### Manual push (PowerShell)

```powershell
docker login -u sraya23      # use a Docker Hub access token as the password

$user = "sraya23"
$services = "frontend","api-gateway","user-service","product-service","order-service"

foreach ($s in $services) {
  docker tag "samarth23072005/ecommerce-${s}:latest" "${user}/ecommerce-${s}:v1.0.0"
  docker tag "samarth23072005/ecommerce-${s}:latest" "${user}/ecommerce-${s}:latest"
  docker push "${user}/ecommerce-${s}:v1.0.0"
  docker push "${user}/ecommerce-${s}:latest"
}
```

The braces in `${s}` matter: in PowerShell, `$s:latest` is read as a drive-style variable name
and produces an empty value.

## 5. CI/CD with GitHub Actions

The pipeline lives in `.github/workflows/ci-cd.yml`.

| Event | What happens |
|---|---|
| Pull request to `main` | Builds all 5 images as a check. Nothing is pushed. |
| Push to `main` | Builds all 5 images in parallel, logs in to Docker Hub, pushes `latest` and the commit SHA tag. |

```yaml
name: CI-CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - { name: ecommerce-frontend,        path: frontend,                 args: "VITE_API_GATEWAY_URL=${{ vars.API_URL || 'http://localhost:4000' }}" }
          - { name: ecommerce-api-gateway,     path: api-gateway,              args: "" }
          - { name: ecommerce-user-service,    path: services/user-service,    args: "" }
          - { name: ecommerce-product-service, path: services/product-service, args: "" }
          - { name: ecommerce-order-service,   path: services/order-service,   args: "" }
    steps:
      - uses: actions/checkout@v4
      - name: Login to Docker Hub
        if: github.event_name == 'push'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build, and push on main
        uses: docker/build-push-action@v6
        with:
          context: ${{ matrix.path }}
          build-args: ${{ matrix.args }}
          push: ${{ github.event_name == 'push' }}
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/${{ matrix.name }}:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/${{ matrix.name }}:${{ github.sha }}
```

### One-time GitHub setup

In the repository: **Settings > Secrets and variables > Actions**.

| Type | Name | Value |
|---|---|---|
| Secret | `DOCKERHUB_USERNAME` | `sraya23` (lowercase) |
| Secret | `DOCKERHUB_TOKEN` | a Docker Hub access token with Read & Write |
| Variable | `API_URL` | `http://16.178.113.48:4000` |

Notes:

- Secret **names** must use only letters, numbers and underscores. The workflow refers to the
  name (`secrets.DOCKERHUB_USERNAME`), never to the value.
- The frontend is a static bundle, so the API address is baked in at **build time** through the
  `VITE_API_GATEWAY_URL` build argument. That is why `API_URL` is a variable that CI passes in.
- In a forked repository, GitHub disables Actions by default. Enable them from the **Actions** tab.
- Secrets can only be managed by the repository owner. This is why the work was done in a fork.
- Never paste a token into a file. GitHub push protection blocks any push that contains one.

## 6. Kubernetes (local cluster)

All resources are in `infrastructure/kubernetes/shopkart.yaml`:

- MongoDB: Deployment + PersistentVolumeClaim (1 Gi) + Service
- User, Product, Order, API Gateway and Frontend: 2 replicas each, with a Service
- Readiness and liveness probes on `/health` (the **kubelet** runs these probes and restarts
  unhealthy containers)
- CPU and memory requests and limits on every container
- JWT secret read from a Kubernetes Secret, not stored in the repository

### Setup (Docker Desktop)

1. Docker Desktop > Settings > Kubernetes > Enable Kubernetes (Kubeadm) > Apply.
2. Verify:

```powershell
kubectl config use-context docker-desktop
kubectl get nodes
```

### Deploy

```powershell
docker compose down      # free the ports

kubectl create namespace shopkart
kubectl create secret generic shopkart-secrets -n shopkart --from-literal=JWT_SECRET=<long-random-string>
kubectl apply -f infrastructure/kubernetes/shopkart.yaml
kubectl get pods -n shopkart -w
```

Run each port-forward in its own terminal:

```powershell
kubectl port-forward -n shopkart svc/api-gateway 4000:4000
kubectl port-forward -n shopkart svc/frontend 5173:80
```

Seed the data and open http://localhost:5173:

```powershell
kubectl exec -n shopkart deploy/product-service -- npm run seed
```

### Kubernetes features demonstrated

```powershell
# Scaling
kubectl scale deployment user-service -n shopkart --replicas=4
kubectl get pods -n shopkart

# Self-healing: delete a pod and watch the Deployment recreate it
kubectl delete pod -n shopkart <pod-name>
kubectl get pods -n shopkart -w

# Rolling update and rollback
kubectl set image deployment/frontend frontend=sraya23/ecommerce-frontend:latest -n shopkart
kubectl rollout status deployment/frontend -n shopkart
kubectl rollout undo deployment/frontend -n shopkart

# Probe results and container events (kubelet activity)
kubectl describe pod -n shopkart <pod-name>
```

Roll out a CI-built image by commit SHA:

```powershell
$sha = git rev-parse HEAD
kubectl set image deployment/user-service user-service="sraya23/ecommerce-user-service:${sha}" -n shopkart
kubectl rollout status deployment/user-service -n shopkart
```

## 7. Deploy to AWS (EC2 + k3s)

The same manifest runs on a single EC2 server using **k3s**, a lightweight Kubernetes
distribution that includes its own kubelet.

### 7.1 Create the server

| Setting | Value |
|---|---|
| Account plan | AWS Free plan (credits) |
| Region | Asia Pacific (Sydney), `ap-southeast-2` |
| OS | Ubuntu Server 24.04 LTS, 64-bit (x86) |
| Instance type | a free-plan-eligible x86 type with at least 4 GB RAM (for example `c7i-flex.large` or `m7i-flex.large`) |
| Storage | 20 GiB gp3 |
| Key pair | RSA `.pem`, named `shopkart` (kept private, never committed) |
| Public address | Elastic IP `16.178.113.48` associated with the instance |

Security group inbound rules:

| Type | Port | Source |
|---|---|---|
| SSH | 22 | 0.0.0.0/0 (IPv4) |
| HTTP | 80 | 0.0.0.0/0 |
| Custom TCP | 4000 | 0.0.0.0/0 |

Notes: ARM (`t4g`) instances are avoided because the CI images are built for x86. An SSH rule
containing only an IPv6 range blocks IPv4 connections, so the SSH rule uses an IPv4 source.

### 7.2 Build the frontend for the server address

Set the GitHub variable `API_URL` to `http://16.178.113.48:4000` (see section 5), then push to
`main` so CI builds a frontend image that calls the server instead of `localhost`.

### 7.3 Connect

```powershell
cd $HOME\Downloads
icacls shopkart.pem /inheritance:r
icacls shopkart.pem /grant:r "$($env:USERNAME):R"
ssh -i shopkart.pem ubuntu@16.178.113.48
```

If port 22 is blocked by a campus network, use a phone hotspot or the browser-based
**EC2 Instance Connect** terminal.

### 7.4 Install k3s

Traefik is disabled so that port 80 stays free for the frontend:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
sudo k3s kubectl get nodes          # wait for Ready
alias k='sudo k3s kubectl'
```

### 7.4 Deploy ShopKart

```bash
git clone https://github.com/srayadutta23/Microservices-Ecommerce.git
cd Microservices-Ecommerce

SHA=$(git rev-parse HEAD)
IP=16.178.113.48

# Use the CI image tag and the server address instead of v1.0.0 and localhost
sed -e "s|:v1.0.0|:${SHA}|g" \
    -e "s|http://localhost:5173|http://${IP}|g" \
    -e "s|http://127.0.0.1:5173|http://${IP}|g" \
    infrastructure/kubernetes/shopkart.yaml > /tmp/shopkart-aws.yaml

k create namespace shopkart
k create secret generic shopkart-secrets -n shopkart --from-literal=JWT_SECRET=$(openssl rand -hex 24)
k apply -f /tmp/shopkart-aws.yaml

# Expose the frontend on port 80 and the gateway on port 4000
k patch svc frontend    -n shopkart -p '{"spec":{"type":"LoadBalancer"}}'
k patch svc api-gateway -n shopkart -p '{"spec":{"type":"LoadBalancer"}}'

k get pods -n shopkart -w           # wait for 1/1 Running
k exec -n shopkart deploy/product-service -- npm run seed
```

Open **http://16.178.113.48**.

### 7.5 Updating the deployment

After a new commit is pushed and CI is green:

```bash
cd ~/Microservices-Ecommerce
git pull
SHA=$(git rev-parse HEAD)
k set image deployment/user-service user-service=sraya23/ecommerce-user-service:${SHA} -n shopkart
k rollout status deployment/user-service -n shopkart
```

Repeat for the other services you changed.

## 8. Configuration reference

| Variable | Used by | Purpose |
|---|---|---|
| `PORT` | all services | listening port |
| `NODE_ENV` | all services | `production` |
| `MONGO_URI` | user, product, order | e.g. `mongodb://mongodb:27017/user_db` |
| `JWT_SECRET` | user, order | signs and verifies tokens (Kubernetes Secret) |
| `JWT_EXPIRES_IN` | user | token lifetime, `1d` |
| `ALLOWED_ORIGIN` | user, product, order | CORS origin of the frontend |
| `ALLOWED_ORIGINS`, `CLIENT_URL` | gateway | allowed browser origins |
| `USER_SERVICE_URL`, `PRODUCT_SERVICE_URL`, `ORDER_SERVICE_URL` | gateway, order | internal service addresses |
| `VITE_API_GATEWAY_URL` | frontend (build time) | public address of the API gateway |

## 9. API reference

Base URL: `http://16.178.113.48:4000` (or `http://localhost:4000` locally).

| Method | Path | Description |
|---|---|---|
| POST | `/api/users/register` | create a customer account |
| POST | `/api/users/login` | log in and receive a JWT |
| GET | `/api/products` | list products |
| GET | `/api/products/categories` | list categories |
| GET | `/api/products/search?q=phone` | search products |
| POST | `/api/orders` | place an order (Bearer token required) |
| GET | `/api/orders` | order history (Bearer token required) |
| GET | `/health` | gateway health check |

## 10. End-to-end testing

1. Open the web app. The home page shows products and categories.
2. Register a new user and log in. A JWT is issued.
3. Search for `phone`.
4. Add an item to the cart and check out.
5. Confirm the product's stock went down by the ordered quantity.
6. Confirm the order appears in the order history.

## 11. Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Frontend loads but shows no products | Gateway published on `4001`, frontend calls `4000` | Use `4000:4000` in Compose; on AWS, set `API_URL` correctly and rebuild |
| `invalid reference format` in the push loop | PowerShell reads `$s:latest` as a drive variable | Write `${s}:latest` |
| `Secret names can only contain alphanumeric characters or underscores` | Wrong text in the **Name** box | Name goes in the first box, value in the second; use `DOCKERHUB_USERNAME` |
| Push rejected: "Push cannot contain secrets" | A token was typed into a workflow file | Remove it, rewrite history (`git reset --soft`), revoke the token, store it only in GitHub Secrets |
| Docker login fails to paste a token | Terminal hides input | `Get-Clipboard \| docker login -u sraya23 --password-stdin` |
| `ImagePullBackOff` | The tag does not exist on Docker Hub | Wait for CI to finish and use the matching commit SHA |
| `Connection timed out` on SSH | Security group SSH rule only allows an IPv6 range, or the network blocks port 22 | Add an IPv4 source (`0.0.0.0/0`), or use a hotspot / EC2 Instance Connect |
| `t3.medium` cannot be launched | Not eligible on the AWS free plan | Use a free-plan-eligible type such as `c7i-flex.large` |
| Pods in `CrashLoopBackOff` at first start | Services start before MongoDB is ready | They recover automatically after a minute |

## 12. Security notes and known limitations

- No credentials are stored in the repository. Docker Hub login uses GitHub Secrets, and the JWT
  secret is created with `kubectl create secret`.
- Docker Hub tokens are issued per use (one for the laptop, one for GitHub) and are revocable.
- SSH is open to `0.0.0.0/0` for convenience. Access still requires the private key file.

This is a production-**style** setup for learning and demonstration. Known gaps:

- Single-node cluster, so there is no high availability.
- No HTTPS or domain name (plain HTTP on an IP address).
- MongoDB runs without authentication and on a local volume.
- No automated tests in the pipeline (CI only builds the images).
- Deployment to AWS is a manual `kubectl` step, not automated.
- No monitoring or centralised logging.

Ideas for next steps: an SSH deploy job in GitHub Actions, HTTPS with Let's Encrypt, MongoDB
credentials, a Horizontal Pod Autoscaler, and Prometheus with Grafana.

## 13. Cost control

The AWS Free plan provides credits that are consumed while the instance runs. To keep them:

- Stop the instance (EC2 > Instances > Instance state > Stop) when not presenting. The Elastic
  IP stays attached, so the address does not change.
- Keep a budget alert in AWS Budgets.
- When the project is finished, terminate the instance and release the Elastic IP.

## 14. Contributors

| Area | Contributor |
|---|---|
| Application services, frontend and Docker Compose | [samarth23072005](https://github.com/samarth23072005) |
| CI/CD, Kubernetes and AWS deployment | [srayadutta23](https://github.com/srayadutta23) |
