# Kubernetes Monitoring Setup — Prometheus & Grafana

This repository documents the setup of a full monitoring stack (Prometheus + Grafana + Alertmanager + node-exporter + kube-state-metrics) on a Kubernetes cluster, giving complete visibility into **every node** and **every pod** in the cluster.

## Architecture Overview

| Component | Purpose |
|---|---|
| **Prometheus** | Scrapes and stores metrics as time-series data |
| **Grafana** | Visualizes metrics via dashboards |
| **Alertmanager** | Handles and routes alerts (email, Slack, etc.) |
| **node-exporter** | Runs on every node (DaemonSet), exposes hardware/OS metrics (CPU, RAM, disk, network) |
| **kube-state-metrics** | Exposes Kubernetes object state (pod status, restarts, deployments, replicasets) |
| **cAdvisor** | Built into kubelet, auto-scraped by Prometheus — gives per-container resource usage |
| **Prometheus Operator** | Manages Prometheus configuration declaratively via CRDs (ServiceMonitors, PodMonitors) |

---

## Prerequisites

- A working Kubernetes cluster with `kubectl` configured
- `helm` v3+ installed

Verify both before starting:

```bash
kubectl get nodes
helm version
```
- `kubectl get nodes` confirms your cluster is reachable and lists all nodes.
- `helm version` confirms Helm (the Kubernetes package manager used to install the stack) is installed correctly.

---

## Step 1: Add Helm repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

**Explanation:**
- `helm repo add` registers a remote chart repository so Helm knows where to download chart packages from.
- `prometheus-community` hosts the `kube-prometheus-stack` chart — an all-in-one bundle containing Prometheus, Grafana, Alertmanager, node-exporter, and kube-state-metrics.
- `helm repo update` refreshes the local cache of available charts/versions from these repos.

---

## Step 2: Create a dedicated namespace

```bash
kubectl create namespace monitoring
```

**Explanation:** Isolates all monitoring components in their own namespace (`monitoring`) instead of mixing them with application workloads — cleaner organization and easier RBAC/resource management.

---

## Step 3: Install the kube-prometheus-stack

```bash
helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.enabled=true \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false
```

**Explanation of each flag:**
- `helm install kube-prom prometheus-community/kube-prometheus-stack` — installs the chart under the release name `kube-prom` (this name prefixes all created resources, e.g. `kube-prom-grafana`).
- `--namespace monitoring` — deploys all resources into the `monitoring` namespace created earlier.
- `--set grafana.enabled=true` — explicitly ensures Grafana is deployed alongside Prometheus (enabled by default, set explicitly for clarity).
- `--set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false` — tells Prometheus Operator to discover **all** ServiceMonitors in the cluster, not just ones matching the Helm release's own labels. Ensures nothing is missed.
- `--set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false` — same as above, but for PodMonitors.

This single command deploys the Prometheus Operator, Prometheus itself, Alertmanager, Grafana, node-exporter (as a DaemonSet — one pod per node), and kube-state-metrics.

---

## Step 4: Verify all pods are running

```bash
kubectl get pods -n monitoring
```

Expected healthy output:
```
alertmanager-kube-prom-kube-prometheus-alertmanager-0   2/2   Running
kube-prom-grafana-xxxxxxxxx-xxxxx                       3/3   Running
kube-prom-kube-prometheus-operator-xxxxxxxxx-xxxxx       1/1   Running
kube-prom-kube-state-metrics-xxxxxxxxx-xxxxx             1/1   Running
kube-prom-prometheus-node-exporter-xxxxx                 1/1   Running   (one per node)
prometheus-kube-prom-kube-prometheus-prometheus-0        2/2   Running
```

**Explanation:** The `READY` column shows `containers-ready/total-containers` per pod. All pods should reach full readiness (e.g. `3/3`, `2/2`, `1/1`) within a couple of minutes of install. It's normal to see a restart or two on sidecar containers immediately after install while they wait for dependent services to become available.

### Troubleshooting a pod stuck at partial readiness (e.g. `2/3`)

```bash
kubectl describe pod <pod-name> -n monitoring
```
Shows Events at the bottom (readiness probe failures, image pull issues, etc.)

```bash
kubectl logs <pod-name> -n monitoring -c <container-name>
```
Shows logs for a **specific container** inside a multi-container pod (Grafana pods have 3 containers: `grafana`, `grafana-sc-dashboards`, `grafana-sc-datasources`).

---

## Step 5: Access Grafana

### Retrieve the admin password

```bash
kubectl get secret -n monitoring kube-prom-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

**Explanation:** Grafana's admin password is auto-generated by the chart and stored as a Kubernetes Secret (base64-encoded, not plaintext). This command extracts the `admin-password` field and decodes it to plaintext.

### Port-forward the Grafana service

```bash
kubectl port-forward -n monitoring svc/kube-prom-grafana 3000:80
```

**Explanation:** Exposes the Grafana Service (listening on port 80 inside the cluster) on `localhost:3000` on the machine running this command. Must stay running in the foreground for the connection to work.

### If accessing from a separate local machine (e.g. Windows laptop → remote Ubuntu server)

```bash
ssh -L 3000:localhost:3000 <user>@<server-ip>
```

**Explanation:** Creates an SSH tunnel. This forwards `localhost:3000` on your **local** machine through the SSH connection to `localhost:3000` on the **remote server**, where `kubectl port-forward` is running. Without this, `localhost:3000` on your laptop has nothing listening on it since the port-forward is bound to the remote machine's loopback interface, not yours.

Then open in a browser: `http://localhost:3000`
Login: `admin` / *(password retrieved above)*

---

## Step 6: Access Prometheus (for verifying scrape targets)

```bash
kubectl port-forward -n monitoring svc/kube-prom-kube-prometheus-prometheus 9090:9090
```

Open `http://localhost:9090` → **Status → Targets** to confirm all exporters (`node-exporter`, `kube-state-metrics`, `kubelet`, `apiserver`) show state **UP**.

---

## Step 7: Explore dashboards in Grafana

The `kube-prometheus-stack` chart auto-provisions dashboards — no manual import needed. Under **Dashboards** in Grafana, look for:

| Dashboard | Shows |
|---|---|
| Kubernetes / Compute Resources / Cluster | Cluster-wide overview |
| Kubernetes / Compute Resources / Namespace (Pods) | Every pod's resource usage in a chosen namespace |
| Kubernetes / Compute Resources / Node (Pods) | Every pod running on a chosen node |
| Node Exporter / Nodes | Per-node CPU, memory, disk, network |

(Optional manual imports via **Dashboards → New → Import** using IDs from grafana.com: `315`, `1860`, `6417`, `15757` — only needed if the built-in ones are missing.)

---

## Step 8: Useful PromQL queries

Run these in Prometheus (`:9090`) or Grafana **Explore**:

```promql
# Node CPU usage (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Pod memory usage, grouped by pod/namespace
sum(container_memory_working_set_bytes{container!="",pod!=""}) by (pod, namespace)

# Total restart count per container/pod (all time)
kube_pod_container_status_restarts_total

# Pods that restarted in the last 1 hour
increase(kube_pod_container_status_restarts_total[1h]) > 0

# Top 10 most-restarted pods right now
topk(10, kube_pod_container_status_restarts_total)

# Node ready status
kube_node_status_condition{condition="Ready",status="true"}
```

**Explanation:**
- `rate()` calculates per-second average rate of change over a time window — used for counters like CPU seconds.
- `increase()` calculates the total increase in a counter over a time window — ideal for "how many times did this restart in the last hour" type questions.
- `topk(N, metric)` returns the top N series by value — useful for quickly spotting the worst offenders (e.g. most-restarted pods).
- `kube_pod_container_status_restarts_total` is a **counter** metric — it only records a new point when the value changes, so sparse graphs are expected and normal (it does not mean data is missing).

---

## Step 9: Viewing metrics as a table (instead of graph)

1. In Grafana **Explore**, run your query.
2. Above the results panel, switch from **Graph** view to **Table** view (or set visualization type to **Table** when building a panel in a dashboard).
3. Click any column header (e.g. `Value`) to sort — highest restart counts appear at the top.

To make this permanent:
1. **Dashboards → New → New Dashboard → Add visualization**
2. Select the Prometheus datasource
3. Enter the query (e.g. `kube_pod_container_status_restarts_total`)
4. Change visualization type from **Time series** to **Table**
5. **Apply** → **Save dashboard**

---

## Common `kubectl` commands used for pod/restart inspection

```bash
# All pods across all namespaces, with restart counts
kubectl get pods --all-namespaces

# Sorted by restart count (highest at bottom)
kubectl get pods --all-namespaces --sort-by='.status.containerStatuses[0].restartCount'

# Clean custom table: namespace, pod, restarts, status, node
kubectl get pods --all-namespaces -o custom-columns='NAMESPACE:.metadata.namespace,POD:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount,STATUS:.status.phase,NODE:.spec.nodeName'

# Restart count + last termination reason for a specific pod
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{range .status.containerStatuses[*]}{.name}{"\t"}Restarts: {.restartCount}{"\t"}LastState: {.lastState.terminated.reason}{"\n"}{end}'

# Full event history and state for a pod
kubectl describe pod <pod-name> -n <namespace>
```

**Explanation:**
- `--sort-by` uses a JSONPath expression to sort output by a specific field.
- `-o custom-columns` builds a custom table view instead of the default columns, letting you pick exactly which fields to display.
- `-o jsonpath` extracts specific fields directly from the pod's JSON representation — useful for scripting/automation.
- `describe` shows the full object spec, status, and recent Events — the most detailed single-pod troubleshooting command.

---

## Summary of what this setup provides

- ✅ Per-node hardware metrics (CPU, memory, disk, network) — via **node-exporter**
- ✅ Per-pod/container resource usage — via **cAdvisor** (built into kubelet)
- ✅ Kubernetes object state (pod status, restarts, deployment health) — via **kube-state-metrics**
- ✅ Dashboards for cluster, namespace, and node-level views — via **Grafana**
- ✅ Alerting foundation ready via **Alertmanager** (rules need to be configured)

## Possible next steps (not yet implemented)

- [ ] Configure Alertmanager receivers (Slack/email/Telegram) for notifications
- [ ] Verify Prometheus/Grafana persistence via PVCs (`kubectl get pvc -n monitoring`) so metric history survives pod restarts
- [ ] Replace `kubectl port-forward` + SSH tunnel with a permanent Ingress or NodePort for Grafana/Prometheus access
- [ ] Define custom alert rules (node CPU > 85%, disk > 80%, pod restart spikes, NodeNotReady)
