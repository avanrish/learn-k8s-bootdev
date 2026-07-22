# k8s-bootdev — SynergyChat on Kubernetes

Kubernetes manifests for deploying **SynergyChat**, a small multi-service chat
application, to a local cluster. This is the hands-on project from the
[boot.dev "Learn Kubernetes"](https://www.boot.dev/) course.

The repo is a collection of plain YAML manifests (no Helm/Kustomize) applied
with `kubectl apply -f`. It covers Deployments, Services, ConfigMaps, a
PersistentVolumeClaim, HorizontalPodAutoscalers, and the Gateway API
(GatewayClass / Gateway / HTTPRoute) for ingress.

## Architecture

```
                        ┌──────────────────────────────┐
      synchat.internal ─┤                              │
   synchatapi.internal ─┤   Gateway (app-gateway)      │  Envoy Gateway
                        │   HTTP :80                    │
                        └───────┬───────────────┬──────┘
                                │ HTTPRoute      │ HTTPRoute
                                ▼                ▼
                        ┌───────────────┐ ┌───────────────┐
                        │  web-service  │ │  api-service  │
                        │    :80→8080   │ │   :80→8080    │
                        └───────┬───────┘ └───────┬───────┘
                                ▼                 ▼
                        ┌───────────────┐ ┌───────────────────────┐
                        │ synergychat-  │ │  synergychat-api      │
                        │ web (HPA 1–4) │ │  + PVC /persist       │
                        └───────────────┘ └───────────┬───────────┘
                                                       │ CRAWLER_BASE_URL
                                                       ▼
                                        ┌──────────────────────────────┐
                                        │ namespace: crawler           │
                                        │ crawler-service :80→8080     │
                                        │ synergychat-crawler pod:      │
                                        │   3 containers :8080/8081/8082│
                                        └──────────────────────────────┘
```

## Components

| Service   | Namespace | Purpose                                              | Exposure |
|-----------|-----------|------------------------------------------------------|----------|
| `web`     | `default` | Frontend UI, talks to the API                        | `synchat.internal` via Gateway |
| `api`     | `default` | Backend API, persists a JSON DB to a PVC             | `synchatapi.internal` via Gateway |
| `crawler` | `crawler` | Scrapes/analyzes text; single pod, three containers  | Internal (`crawler-service`) |
| `testcpu` | `default` | Load generator to demo CPU-based autoscaling         | None |
| `testram` | `default` | Load generator to demo memory limits                 | None |

### web
- `web-configmap.yaml` — `WEB_PORT: 8080`, `API_URL: http://synchatapi.internal`
- `web-deployment.yaml` — live deployment spec (exported from the cluster; **excluded from the string-quoting convention below**)
- `web-service.yaml` — ClusterIP, `80 → 8080`
- `web-hpa.yaml` — autoscales 1–4 replicas at 50% target CPU
- `web-httproute.yaml` — routes host `synchat.internal` to `web-service`

### api
- `api-configmap.yaml` — `API_PORT`, `API_DB_FILEPATH` (`/persist/db.json`), `CRAWLER_BASE_URL`
- `api-deployment.yaml` — 1 replica, mounts the PVC at `/persist`
- `api-pvc.yaml` — 1Gi `ReadWriteOnce` volume
- `api-service.yaml` — ClusterIP, `80 → 8080`
- `api-httproute.yaml` — routes host `synchatapi.internal` to `api-service`

### crawler (namespace `crawler`)
- `crawler-configmap.yaml` — keywords, DB path, and ports `8080/8081/8082`
- `crawler-deployment.yaml` — one pod running three crawler containers, each on a
  different port (sharing an `emptyDir` cache volume)
- `crawler-service.yaml` — ClusterIP, `80 → 8080`

### Gateway (ingress)
- `app-gatewayclass.yaml` — `GatewayClass` using the Envoy Gateway controller
- `app-gateway.yaml` — `Gateway` with an HTTP listener on port 80
- The `*-httproute.yaml` files bind hostnames to backend Services

### Load-test workloads
- `testcpu-deployment.yaml` + `testcpu-hpa.yaml` — CPU limit `10m`, HPA to demo scaling
- `testram-deployment.yaml` + `testram-configmap.yaml` — memory limit `256Mi`, `MEGABYTES` config

## Prerequisites

- A running Kubernetes cluster (this course uses [minikube](https://minikube.sigs.k8s.io/))
- `kubectl` configured against that cluster
- The [metrics-server](https://github.com/kubernetes-sigs/metrics-server) add-on
  (required for the HPAs) — `minikube addons enable metrics-server`
- [Envoy Gateway](https://gateway.envoyproxy.io/) installed for Gateway API support

## Usage

Create the `crawler` namespace, then apply everything:

```bash
kubectl create namespace crawler
kubectl apply -f .
```

Or apply selectively, e.g. just the API stack:

```bash
kubectl apply -f api-configmap.yaml -f api-pvc.yaml -f api-deployment.yaml -f api-service.yaml -f api-httproute.yaml
```

Check status:

```bash
kubectl get pods,svc,hpa
kubectl get pods -n crawler
kubectl get gateway,httproute
```

### Accessing the app

Ingress is served by the Gateway on the hostnames `synchat.internal` and
`synchatapi.internal`. Point them at the Gateway's address (map them in
`/etc/hosts`, or use `kubectl port-forward` to the Gateway/Service), then open
`http://synchat.internal`.

For a quick local check without DNS setup:

```bash
kubectl port-forward service/web-service 8080:80
# then browse http://localhost:8080
```

## Conventions

All manifests **except `web-deployment.yaml`** follow a "quote every string
scalar" style for consistency — string values (including `apiVersion`, `kind`,
names, ports-as-labels, paths) are wrapped in double quotes, while genuine
numbers (`port: 80`, `replicas: 1`) and empty maps (`{}`) are left bare.
`web-deployment.yaml` is a spec exported directly from the cluster and is left
as-is.