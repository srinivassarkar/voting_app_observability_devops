# Kubernetes Observability for Voting Application

## 📌 Overview

This project demonstrates an **end-to-end observability setup** for a Kubernetes-based microservices application (Voting App). The goal is to simulate a **production-like environment** locally using **Kind**, deploy a real multi-service application, apply **metrics, logs, tracing, and load testing**, and understand system behavior under load.

The project intentionally includes **real-world issues and fixes** encountered during setup, mirroring how observability is built and debugged in actual DevOps/SRE work.

---

## 📁 Repository Structure

```bash
.
├── README.md
├── dashboards/                 # Grafana dashboards (custom/imported)
├── helm-charts/
│   ├── loki-values.yaml        # Loki + Promtail configuration overrides
│   └── prometheus-values.yaml  # Prometheus configuration overrides
├── manifests/
│   ├── 00-namespaces.yaml      # All required namespaces
│   ├── 01-redis.yaml           # Redis service
│   ├── 02-postgres.yaml        # PostgreSQL service
│   ├── 03-vote.yaml            # Vote frontend service
│   ├── 04-worker.yaml          # Background worker
│   ├── 05-result.yaml          # Result frontend service
│   ├── 06-jaeger.yaml          # Jaeger deployment
│   ├── 07-otel-collector.yaml  # OpenTelemetry Collector
│   └── 08-locust.yaml          # Locust master & workers
└── scripts/
    └── kind-cluster.yaml       # Kind cluster configuration
```

---

## 🏗️ Step-by-Step Implementation

### 1️⃣ Kubernetes Cluster Setup (Kind)

A local Kubernetes cluster was created using **Kind** to simulate a real cluster environment.

```bash
kind create cluster --config scripts/kind-cluster.yaml
```

Validation:

```bash
kubectl get nodes
kubectl cluster-info
```

---

### 2️⃣ Namespace Isolation

Namespaces were created to separate concerns (application, monitoring, observability).

```bash
kubectl apply -f manifests/00-namespaces.yaml
```

Namespaces used:

* `voting-app`
* `monitoring`
* `observability`

---

### 3️⃣ Deploy Voting Application (Microservices)

The classic **Voting App** consists of multiple independent services.

Deployment order:

```bash
kubectl apply -f manifests/01-redis.yaml
kubectl apply -f manifests/02-postgres.yaml
kubectl apply -f manifests/03-vote.yaml
kubectl apply -f manifests/04-worker.yaml
kubectl apply -f manifests/05-result.yaml
```

Verification:

```bash
kubectl get pods -n voting-app
kubectl get svc -n voting-app
```

The Vote service was exposed via NodePort for external access.

---

### 4️⃣ Metrics: Prometheus & Grafana

Installed **kube-prometheus-stack** using Helm with custom values.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f helm-charts/prometheus-values.yaml
```

Validation:

* Prometheus targets showed application services as **UP**
* Grafana dashboards displayed CPU and memory metrics

---

### 5️⃣ Logging: Loki + Promtail

Centralized logging was implemented using Loki.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  -n monitoring \
  -f helm-charts/loki-values.yaml
```

Logs were queried in Grafana using **LogQL**, confirming logs from all namespaces.

---

### 6️⃣ Distributed Tracing: Jaeger + OpenTelemetry

Jaeger was deployed as a tracing backend.

```bash
kubectl apply -f manifests/06-jaeger.yaml
kubectl apply -f manifests/07-otel-collector.yaml
```

Jaeger UI was accessible and healthy.

⚠️ **Important Note**: Application services were **not instrumented** with OpenTelemetry SDKs, so traces were not generated. This reflects a real-world limitation when application code cannot be modified.

---

### 7️⃣ Load Testing: Locust (Distributed Mode)

Locust was deployed using a **master-worker architecture**.

```bash
kubectl apply -f manifests/08-locust.yaml
```

Validation:

```bash
kubectl get pods -n voting-app | grep locust
```

Traffic was generated against the Vote service, simulating concurrent users.

---

## 🧨 Issues Faced & How They Were Solved

### ❌ Issue 1: Locust Workers Not Connecting to Master

**Symptom:**

> "You can't start a distributed test before at least one worker process has connected"

**Root Cause:**

* Incorrect service name / port resolution between Locust master and workers

**Fix:**

* Ensured workers pointed to `locust-master:5557`
* Verified ClusterIP service and DNS resolution
* Restarted Locust deployment cleanly

---

### ❌ Issue 2: Zero RPS Despite Active Users

**Symptom:**

* Users spawned but RPS stayed at 0

**Root Cause:**

* Incorrect target Host configured in Locust UI

**Fix:**

* Corrected host to the **Vote service NodePort URL**
* Confirmed POST requests appearing in Locust stats

---

### ❌ Issue 3: Missing Traces in Jaeger

**Symptom:**

* Jaeger UI showed no traces

**Root Cause:**

* Application services lacked OpenTelemetry instrumentation

**Resolution:**

* Validated tracing backend readiness
* Documented instrumentation as a future improvement

---

## 🔍 Observability Correlation

During load testing:

* CPU and memory usage increased in Grafana
* Log volume spiked in Loki
* Locust showed request latency under load

This confirmed end-to-end observability across **metrics, logs, and traffic**.

---

## 🧠 Key Learnings

* Observability is incomplete without **correlation**
* Infrastructure-level monitoring provides value even without app instrumentation
* Load testing is essential to validate monitoring assumptions
* Clean teardown and recreation improves system reliability

---

## 🚀 Future Improvements

* Add OpenTelemetry SDKs to application services
* Introduce SLOs and alerting
* Export traces to Grafana Tempo
* Add CI pipeline for Helm + manifests validation

---

## 🛠️ Tech Stack

`Kubernetes | Kind | Helm | Prometheus | Grafana | Loki | Jaeger | OpenTelemetry | Locust`

---

## 🧹 Cleanup

```bash
kind delete cluster
```

---

## ✅ Status

**Project completed with full observability validation and documented production-style troubleshooting.**
