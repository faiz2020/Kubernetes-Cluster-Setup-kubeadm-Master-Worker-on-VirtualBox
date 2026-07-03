# DevOps Hands-On Projects — Kubernetes Cluster (kubeadm, VirtualBox)

A progression of hands-on projects built on top of a self-hosted kubeadm cluster (1 master + 1 worker, Calico CNI, containerd) to practice real-world DevOps skills — from basic app deployment to production-grade HA, GitOps, and security.

Base cluster setup: see [`README.md`](./README.md) (cluster bootstrap guide).

---

## Suggested order

```
1 → 2 → 3   (core workloads: CI/CD, ingress/TLS, storage)
   → 9      (IaC-ify everything built so far — high leverage)
   → 4      (HA control plane)
   → 5 → 6  (observability: metrics + logs)
   → 10 → 11 (security: RBAC + network policies)
   → 7      (backup / disaster recovery)
   → 8      (GitOps)
   → 12     (chaos engineering — capstone)
```

---

## Beginner

### 1. CI/CD Pipeline to the Cluster
Build a simple app (Node/Flask/Go), containerize it, and set up a GitHub Actions pipeline:
`build image → push to Docker Hub/GHCR → kubectl apply to cluster`.
Add separate `staging` and `prod` namespaces.

**Skills:** CI/CD, Docker, kubectl automation, GitHub Actions

---

### 2. Ingress + TLS
Install `nginx-ingress-controller`. Deploy 2–3 apps and route them through Ingress rules using different hostnames/paths. Add `cert-manager` with a self-signed or Let's Encrypt (staging) issuer for HTTPS.

**Skills:** Ingress, TLS termination, cert-manager

---

### 3. Persistent Storage Demo
Deploy a stateful app (Postgres/MySQL) using a StatefulSet + PersistentVolumeClaim. Install a CSI driver such as **Longhorn** (works well on bare VMs). Prove data survives pod deletion and rescheduling.

**Skills:** StorageClass, PV/PVC, StatefulSets, CSI

---

## Intermediate

### 4. Convert to 3-Node HA Control Plane
Add 2 more master VMs. Put HAProxy + keepalived in front as a load balancer with a VIP. Rebuild the cluster using `--control-plane-endpoint` pointing at the VIP.

**Skills:** HA architecture, etcd quorum, load balancing
**Why it matters:** single most valuable production-readiness upgrade to this lab

---

### 5. Monitoring Stack
Deploy `kube-prometheus-stack` via Helm (Prometheus + Grafana + Alertmanager). Build a dashboard tracking node CPU/memory and pod restarts. Configure an alert (Slack/Discord webhook) for crash-looping pods.

**Skills:** Helm, Prometheus, Grafana, alerting

---

### 6. Centralized Logging
Deploy Loki + Promtail to ship container logs from all pods into Grafana. Practice searching logs cluster-wide instead of `kubectl logs` per pod.

**Skills:** Log aggregation, Grafana, Loki

---

### 7. etcd Backup Automation
Write a CronJob or systemd timer to run `etcdctl snapshot save` on a schedule. Store snapshots off-cluster (S3, MinIO, or scp to another VM). Practice a full restore drill by deliberately breaking etcd and recovering it.

**Skills:** Disaster recovery, backup strategy, etcd internals

---

## Advanced

### 8. GitOps Deployment with ArgoCD or Flux
Install ArgoCD and point it at a Git repo of Kubernetes manifests. Replace manual `kubectl apply` with GitOps: commit → ArgoCD detects drift → auto-syncs and deploys.

**Skills:** GitOps, ArgoCD, declarative infrastructure

---

### 9. Infrastructure as Code for the Whole Lab
Rebuild the entire VirtualBox + kubeadm setup using **Terraform** (VM provisioning via Vagrant/libvirt provider) + **Ansible** (automating swap-off, containerd install, kubeadm init/join — everything done manually in the base setup).
Goal: `terraform apply && ansible-playbook site.yml` rebuilds the cluster from scratch.

**Skills:** IaC, Ansible, Terraform
**Why it matters:** highest-value project for job interviews

---

### 10. RBAC + Multi-Tenancy
Create namespaces per team. Set up ServiceAccounts, Roles, and RoleBindings restricting each team to their own namespace. Generate a kubeconfig for a "developer" role that can only deploy within their namespace, with no cluster-admin access.

**Skills:** RBAC, security, multi-tenancy

---

### 11. NetworkPolicy Lockdown
Deploy a 3-tier app (frontend / backend / db). Use Calico NetworkPolicies to enforce that frontend can only reach backend, and backend only reaches db — deny-all by default. Verify by attempting (and failing) to curl the db directly from frontend.

**Skills:** Zero-trust networking, Calico NetworkPolicies

---

### 12. Chaos Engineering (Capstone)
Install `chaos-mesh`, or manually kill nodes/pods/network links, and observe how the cluster self-heals — or doesn't. Document failures, root causes, and fixes.

**Skills:** Resilience testing, incident response, operational maturity

---

## Portfolio tip

Push each project to its own folder or repo with a project-specific README documenting:
- What was built
- Commands/manifests used
- What broke and how it was fixed
- Before/after (screenshots, `kubectl get` output, Grafana dashboards, etc.)

This turns the progression into a narrative: *"I can deploy a pod" → "I can run this like a production platform."*
