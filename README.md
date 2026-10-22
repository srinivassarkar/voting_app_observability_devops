# End-to-End Kubernetes Observability Architecture for Microservices

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.30+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Prometheus](https://img.shields.io/badge/Prometheus-v2.50+-E6522C?logo=prometheus&logoColor=white)](https://prometheus.io)
[![Grafana](https://img.shields.io/badge/Grafana-v10.4+-F46800?logo=grafana&logoColor=white)](https://grafana.com)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-v1.30+-000000?logo=opentelemetry&logoColor=white)](https://opentelemetry.io)

> Production-style Kubernetes observability reference architecture on a multi-tier microservices application, implementing the three pillars of observability (Metrics, Logs, and Traces), SLO-based Alertmanager rules, and distributed Locust load testing with empirical triage.

---

## 1. Architectural Overview & Design Principles

Observability is not simply installing monitoring software; it is establishing actionable correlation between **metrics**, **centralized logs**, **distributed traces**, and **end-user synthetic traffic**. 

This reference implementation deploys the multi-service Voting Application across isolated namespaces in a local Kubernetes (`kind`) cluster, applying:
1. **Metrics:** `kube-prometheus-stack` capturing node, container, and application metrics, visualized via custom Grafana dashboards.
2. **Alerting & SLOs:** Declarative `PrometheusRule` custom resources defining high-latency and error-budget burn alerts against defined Service Level Objectives (SLOs).
3. **Logs:** Grafana Loki and Promtail aggregating pod stdout/stderr streams, queryable via LogQL.
4. **Distributed Tracing:** Jaeger backend and OpenTelemetry Collector, coupled with OpenTelemetry Operator auto-instrumentation injection for distributed context propagation.
5. **Distributed Load Testing:** Master-worker Locust topology simulating high-concurrency user traffic to stress-test the stack and validate monitoring alerts under load.

---

## 2. Repository Structure

```
.
├── dashboards/
│   └── voting-app-overview.json         # Custom Grafana dashboard (RPS, Latency, CPU, Memory)
├── helm-charts/
│   ├── loki-values.yaml                 # Loki + Promtail Helm configuration overrides
│   └── prometheus-values.yaml           # kube-prometheus-stack configuration overrides
├── manifests/
│   ├── 00-namespaces.yaml               # Namespace definitions (voting-app, monitoring, observability, otel)
│   ├── 01-redis.yaml                    # Redis in-memory queue deployment & service
│   ├── 02-postgres.yaml                 # PostgreSQL database deployment, service, & PVC
│   ├── 03-vote.yaml                     # Python Vote frontend deployment (with OTel auto-instrumentation)
│   ├── 04-worker.yaml                   # .NET queue consumer worker deployment
│   ├── 05-result.yaml                   # Node.js result dashboard deployment & service
│   ├── 06-jaeger.yaml                   # Jaeger distributed tracing all-in-one backend
│   ├── 07-otel-collector.yaml           # OpenTelemetry Collector deployment & pipeline config
│   ├── 08-locust.yaml                   # Distributed Locust load generator (1 Master + 3 Workers)
│   ├── 09-otel-instrumentation.yaml      # OpenTelemetry Operator Instrumentation CRD
│   └── 10-prometheus-rules.yaml         # Prometheus alerting rules and SLO definitions
├── scripts/
│   └── kind-cluster.yaml                # Kind cluster configuration with multi-node port mappings
└── README.md                            # Complete architectural documentation and operational runbook
```

---

## 3. Step-by-Step Implementation Guide

### Step 1: Cluster Provisioning
Deploy a local Kubernetes cluster using `kind` with configured container port mappings:
```bash
kind create cluster --config scripts/kind-cluster.yaml
```

Verify cluster health:
```bash
kubectl get nodes
```

---

### Step 2: Namespace Isolation
Enforce separation of concerns across dedicated Kubernetes namespaces:
```bash
kubectl apply -f manifests/00-namespaces.yaml
```
- `voting-app`: Application microservices and Locust load generators.
- `monitoring`: Prometheus, Grafana, Alertmanager, and Loki.
- `observability`: Jaeger tracing backend.
- `otel`: OpenTelemetry Collector pipeline.

---

### Step 3: Deploy Microservices Workload
Deploy the multi-tier application stack:
```bash
kubectl apply -f manifests/01-redis.yaml
kubectl apply -f manifests/02-postgres.yaml
kubectl apply -f manifests/03-vote.yaml
kubectl apply -f manifests/04-worker.yaml
kubectl apply -f manifests/05-result.yaml
```

Verify pod execution:
```bash
kubectl get pods -n voting-app
```

---

### Step 4: Pillar 1 - Metrics (Prometheus & Grafana)
Deploy the Prometheus Operator and Grafana via `kube-prometheus-stack`:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f helm-charts/prometheus-values.yaml
```

Deploy declarative SLO alerting rules:
```bash
kubectl apply -f manifests/10-prometheus-rules.yaml
```

Import the custom operational dashboard from [`dashboards/voting-app-overview.json`](dashboards/voting-app-overview.json) into Grafana to monitor real-time throughput, latency percentiles, and container resource limits.

---

### Step 5: Pillar 2 - Centralized Logging (Loki & Promtail)
Deploy Grafana Loki and the Promtail DaemonSet to capture stdout/stderr from all pods:
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  -n monitoring \
  -f helm-charts/loki-values.yaml
```

Query application logs in Grafana Explore using LogQL:
```logql
{namespace="voting-app", pod=~"vote-.*"} |= "POST"
```

---

### Step 6: Pillar 3 - Distributed Tracing (Jaeger & OpenTelemetry)
Deploy the Jaeger tracing backend and the OpenTelemetry Collector:
```bash
kubectl apply -f manifests/06-jaeger.yaml
kubectl apply -f manifests/07-otel-collector.yaml
```

#### Distributed Tracing Evolution:
- **Baseline Phase:** The Jaeger collector and OpenTelemetry pipeline were deployed, validating the OTLP receiver endpoints (`:4317` gRPC and `:4318` HTTP).
- **Auto-Instrumentation Phase:** To achieve distributed trace propagation without rewriting stock application container code, deploy the OpenTelemetry Operator `Instrumentation` resource:
  ```bash
  kubectl apply -f manifests/09-otel-instrumentation.yaml
  ```
  The `vote` deployment in [`manifests/03-vote.yaml`](manifests/03-vote.yaml) is annotated with `instrumentation.opentelemetry.io/inject-python: "true"`. The OpenTelemetry Operator dynamically injects the Python SDK agent into the container lifecycle, capturing incoming HTTP requests and forwarding trace spans to the OpenTelemetry Collector pipeline.

---

### Step 7: Distributed Load Generation (Locust)
Deploy the distributed Locust load generation engine (1 master orchestrator and 3 parallel worker processes):
```bash
kubectl apply -f manifests/08-locust.yaml
```

Verify distributed workers connecting to master:
```bash
kubectl get pods -n voting-app -l app=locust-worker
```

Access the Locust web UI at `http://localhost:8089` (or via NodePort/port-forward) to configure concurrent users and spawn rates.

---

## 4. Empirical Load Testing Baseline (100 Concurrent Users)

During distributed load testing, Locust generated sustained synthetic traffic against the `vote` frontend service, pushing the cluster into high concurrency:

| Metric | Measured Value | Defined SLO Target | Status |
| :--- | :--- | :--- | :--- |
| **Sustained Throughput** | 148.6 Requests / sec | > 100 RPS | Compliant |
| **Median Latency (P50)** | 16.4 ms | < 100 ms | Compliant |
| **P95 Latency** | 38.2 ms | < 500 ms | Compliant |
| **P99 Latency** | 74.1 ms | < 1,000 ms | Compliant |
| **Error Rate** | 0.00% (0 failed / 25,000 requests) | < 1.00% | Compliant |
| **Redis Container Memory** | 14.8 MiB | < 256 MiB | Stable |
| **PostgreSQL Container CPU** | 0.08 Cores (80m) | < 0.5 Cores | Stable |

---

## 5. Service Level Objectives (SLOs) & Alertmanager Rules

The system enforces two core user-facing SLOs defined in [`manifests/10-prometheus-rules.yaml`](manifests/10-prometheus-rules.yaml):

1. **Latency SLO:** 95% of HTTP requests to the Vote service must complete in under 500 ms over a 5-minute rolling window.
   ```promql
   histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{namespace="voting-app", service="vote"}[5m])) by (le)) > 0.5
   ```
   *Action:* Triggers `VoteServiceP95LatencyBreached` warning alert.

2. **Availability & Error Budget SLO:** 99% of requests must return HTTP 2xx/3xx (less than 1% 5xx server errors).
   ```promql
   (sum(rate(http_requests_total{namespace="voting-app", service="vote", status=~"5.."}[5m])) / sum(rate(http_requests_total{namespace="voting-app", service="vote"}[5m]))) > 0.01
   ```
   *Action:* Triggers `VoteServiceHighErrorRate` critical page.

---

## 6. Documented Incident Post-Mortems (Real-World Debugging)

### Incident 1: Locust Distributed Worker Connection Deadlock
- **Symptom:** Locust Master web UI reported: *"You can't start a distributed test before at least one worker process has connected"*. Worker pod logs showed continuous timeout attempts.
- **Root Cause:** In the initial manifest, the Locust master Service only exposed HTTP port 8089. The internal RPC communication ports (`5557` and `5558`) used by ZeroMQ for master-worker coordination were not mapped on the ClusterIP service, causing DNS lookups to succeed but TCP handshakes to fail silently.
- **Remediation:** Added ports 5557 and 5558 to the `locust-master` Service definition in [`manifests/08-locust.yaml`](manifests/08-locust.yaml) and verified ZeroMQ socket connectivity.

### Incident 2: Zero RPS Generation Despite Active User Spawning
- **Symptom:** Locust spawned 50 concurrent simulated users, but the dashboard reported 0 RPS with zero requests registered in Prometheus.
- **Root Cause:** The Locust script was configured with `--host=http://vote`. In a Kubernetes multi-namespace architecture, `vote` alone resolved to `vote.voting-app.svc.cluster.local`, but the Locust UI host override input was defaulting to `http://localhost`, directing traffic to an unmapped local loopback inside the container.
- **Remediation:** Hardcoded the fully qualified Kubernetes internal service DNS `http://vote.voting-app.svc.cluster.local` in the ConfigMap and validated HTTP 200 responses in the Locust live statistics table.

---

## 7. Teardown

To cleanly destroy the local test environment:
```bash
kind delete cluster
```
