___

# Three-tier App on Kubernetes (Kind) — with Prometheus & Grafana

---

## Overview

This repo contains Kubernetes manifests for a .NET / Python three-tier Voting app and instructions to:

- Launch a cluster (Kind on EC2)
- Install `kubectl`, Docker and Kind
- Deploy the Voting app to Kubernetes
- Install Prometheus & Grafana (Helm)
- Access the web apps and monitoring dashboards
- Clean up resources

---

## Prerequisites

- An EC2 instance (or local machine) with Docker installed and working.
- `kubectl` configured to talk to your cluster (`kubectl cluster-info` should work).
- `kind` installed (the cluster is created with `kind`).
- `helm` installed (see [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)).

---

## Create / Use the Cluster

Create a Kind cluster (example):

```bash
  kind create cluster --config=config.yml
```

> If using a remote EC2 host: make sure Docker is running and Kind can create nodes there. Note: Kind nodes are Docker containers; NodePort behavior differs from a normal VM cluster (see **Accessing services** below).

---

## Deploy the Voting App

All YAML manifests are in `k8s-specifications/`.

Use `apply` (idempotent) to create resources:

```bash
kubectl apply -f k8s-specifications/
```

To delete them:

```bash
kubectl delete -f k8s-specifications/
```

Check resources:

```bash
kubectl get pods,deploy,svc -A
kubectl get all -n voting-app
```

---

## Accessing the Web Apps

Recommended: expose services in your manifests as `NodePort` or use `kubectl port-forward`. Pods have dynamic names, so prefer forwarding services.

Example (port-forward service to local machine):

```bash
# If Vote service exists as 'vote-svc' and listens on 80:
kubectl port-forward svc/vote-svc 31000:80 -n default
# If Result service exists as 'result-svc' and listens on 80:
kubectl port-forward svc/result-svc 31001:80 -n default
```

Then open:

```
http://localhost:31000    # Vote App
http://localhost:31001    # Result App
```

If you used `NodePort` in a real VM cluster (not Kind), you can use `http://<node-ip>:<nodePort>`.

To list services and ports:

```bash
kubectl get svc -A
kubectl describe svc vote-svc -n voting-app
```

---

## Install Helm Chart (Prometheus + Grafana)

Add repo and update:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Create a namespace for monitoring:

```bash
kubectl create namespace monitoring
```

Install `kube-prometheus-stack` and expose via NodePort (example):

```bash
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.service.type=NodePort \
  --set prometheus.service.nodePort=30000 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=31000
```

### What the install command does (short)

- Installs Prometheus + Grafana using the `kube-prometheus-stack` Helm chart.
- Release name: `prometheus-stack`.
- Namespace: `monitoring`.
- Forces both Prometheus and Grafana services to `NodePort` so they are reachable on the host at the given ports.

---

## Access Prometheus & Grafana

### Option A — NodePort (VM / real node)

If you’re running on a VM/EC2 (not Kind) and set the NodePort values above:

```
Prometheus: http://<node-ip>:30000
Grafana:    http://<node-ip>:31000
```

### Option B — kubectl port-forward (works reliably for Kind)

Find service names (they may vary slightly by chart version):

```bash
kubectl get svc -n monitoring
```

Port-forward to local ports:

```bash
# Prometheus (example service name)
kubectl port-forward svc/prometheus-stack-kube-prom-prometheus 9090:9090 -n monitoring
# Grafana (example service name)
kubectl port-forward svc/prometheus-stack-grafana 3000:80 -n monitoring
```

Open in browser:

```
http://localhost:9090   # Prometheus UI
http://localhost:3000   # Grafana UI
```

> Note: the Grafana service in the chart often exposes HTTP on port **80** (targetPort 3000). `kubectl port-forward svc/... 3000:80` forwards local 3000 to service port 80.

### Get Grafana admin credentials

Username is usually `admin`. Retrieve the password from the Helm-created secret:

```bash
kubectl get secret -n monitoring prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

If secret name differs, list secrets:

```bash
kubectl get secrets -n monitoring
```

---

## Useful Commands

Check pods & status:

```bash
kubectl get pods -n monitoring
kubectl logs -n monitoring <pod-name>
kubectl describe pod -n monitoring <pod-name>
```

Helm status / uninstall:

```bash
helm status prometheus-stack -n monitoring
helm uninstall prometheus-stack -n monitoring
```

Remove monitoring namespace:

```bash
kubectl delete namespace monitoring
```

---

## Cleanup

Delete app resources:

```bash
kubectl delete -f k8s-specifications/
```

Uninstall monitoring:

```bash
helm uninstall prometheus-stack -n monitoring
kubectl delete namespace monitoring
```
