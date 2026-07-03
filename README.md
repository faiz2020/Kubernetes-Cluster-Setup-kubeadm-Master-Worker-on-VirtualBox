# Kubernetes Cluster Setup (kubeadm) — Master + Worker on VirtualBox

A 2-node Kubernetes cluster (1 control-plane + 1 worker) built with `kubeadm`, `containerd`, and Calico CNI, tested on VirtualBox VMs simulating a production-like environment.

**Environment:**
- Master: `192.168.1.33`
- Slave (worker): `192.168.1.39`
- Kubernetes version: `v1.32.13`
- Network mode: VirtualBox **Bridged Adapter** (nodes must reach each other directly on the LAN — NAT mode will not work)

---

## 0. Prerequisites (run on BOTH master and slave)

### 0.1 Disable swap

```bash
swapoff -a

# make it permanent
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# verify
free -h
cat /proc/swaps   # should be empty
```

### 0.2 Load required kernel modules

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```

### 0.3 Set required sysctl params

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

### 0.4 Install containerd

```bash
apt update
apt install -y containerd

mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd
systemctl status containerd --no-pager
```

### 0.5 Install kubeadm, kubelet, kubectl (v1.32)

```bash
apt update
apt install -y apt-transport-https ca-certificates curl gpg

mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list

apt update
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

systemctl enable kubelet
```

Verify versions match on both nodes:
```bash
kubeadm version
kubectl version --client
kubelet --version
```

### 0.6 Set hostnames

```bash
# on master
hostnamectl set-hostname master

# on slave
hostnamectl set-hostname slave
```

### 0.7 Add hosts entries (on BOTH nodes)

```bash
cat <<EOF | tee -a /etc/hosts
192.168.1.33 master
192.168.1.39 slave
EOF
```

### 0.8 Verify node-to-node connectivity

```bash
# from slave
ping -c 3 192.168.1.33
curl -k https://192.168.1.33:6443/healthz   # response "ok" = reachable
```

---

## 1. Initialize the control plane (run ONLY on master)

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.1.33 \
  --pod-network-cidr=192.168.0.0/16 \
  --control-plane-endpoint=192.168.1.33
```

> `192.168.0.0/16` is Calico's default pod CIDR. If using Flannel instead, use `10.244.0.0/16` here and swap the CNI manifest in step 3.

### 1.1 Configure kubectl access (on master)

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
```

Master will show `NotReady` until a CNI is installed — that's expected.

---

## 2. Install CNI — Calico (run ONLY on master)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Watch pods come up (can take several minutes on first run — images being pulled):

```bash
kubectl get pods -n kube-system -o wide -w
```

Wait until `calico-node`, `calico-kube-controllers`, and both `coredns` pods show `1/1 Running`, then:

```bash
kubectl get nodes
```

Master should now show `Ready`.

---

## 3. Join the worker node

### 3.1 Generate join command (on master)

```bash
kubeadm token create --print-join-command
```

Copy the output — looks like:
```bash
kubeadm join 192.168.1.33:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

> Tokens expire after 24h. Re-run the command above anytime to get a fresh one.

### 3.2 Run the join command on the slave

Make sure section 0 (prereqs) is fully completed on the slave first, then:

```bash
kubeadm join 192.168.1.33:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Expect output ending in:
```
This node has joined the cluster
```

> **Important:** `kubectl` only works where the kubeconfig (`admin.conf`) was copied — i.e. the master. Running `kubectl` on the slave/worker node will fail with a `localhost:8080 connection refused` error unless you manually copy `admin.conf` there. This is expected behavior; manage the cluster from the master.

### 3.3 Verify from master

```bash
kubectl get nodes
```

Slave will show `NotReady` briefly while Calico initializes networking on it (can take a few minutes). Watch it:

```bash
kubectl get pods -n kube-system -o wide -w
```

Once `calico-node` and `kube-proxy` pods for the slave node are `1/1 Running`, recheck:

```bash
kubectl get nodes
```

Expected final state:
```
NAME     STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   ~20m  v1.32.13
slave    Ready    <none>          ~5m   v1.32.13
```

---

## 4. Sanity test — deploy and verify cross-node networking

```bash
kubectl run nginx-test --image=nginx --port=80
kubectl get pods -o wide
```

Confirm pod reaches `Running` (scheduled onto slave in this setup).

```bash
kubectl expose pod nginx-test --port=80 --type=ClusterIP
kubectl get svc nginx-test
```

```bash
kubectl run curl-test --image=curlimages/curl -it --rm -- curl http://nginx-test
```

A successful response (`Welcome to nginx!` HTML) confirms:
- DNS resolution (CoreDNS) works
- Pod-to-pod networking across nodes (Calico) works
- Service routing (kube-proxy) works

Clean up test resources when done:
```bash
kubectl delete pod nginx-test curl-test --ignore-not-found
kubectl delete svc nginx-test --ignore-not-found
```

---

## Troubleshooting notes

| Symptom | Cause / Fix |
|---|---|
| `kubectl` on worker gives `localhost:8080 connection refused` | Worker nodes don't have `admin.conf` by default — run `kubectl` from master only |
| Pods stuck `Pending`/`ContainerCreating` for a long time on first apply | Normal on fresh VMs — images are being pulled; wait several minutes |
| `curl -k https://<master-ip>:6443/healthz` hangs/times out | Networking issue — check VirtualBox adapter is **Bridged**, not NAT; check firewall on ports 6443, 10250-10252 |
| Node stuck `NotReady` after join | CNI (Calico) still initializing on that node — check `kubectl get pods -n kube-system -o wide -w` |
| `kubeadm join` token expired | Regenerate with `kubeadm token create --print-join-command` on master |
| containerd/kubeadm "command not found" | Package not installed yet — see sections 0.4 and 0.5 |

---

## Cluster reset (if you need to start over)

```bash
# on the node you want to reset
kubeadm reset -f
rm -rf /etc/cni/net.d
rm -rf $HOME/.kube
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

Root cause, for your notes/README
Two compounding issues from the netplan cleanup:

The tee (not tee -a) command used earlier accidentally overwrote /etc/hosts instead of appending, wiping out the 127.0.0.1 localhost / ::1 localhost entries
Without those entries, localhost lookups fell through to DNS instead of resolving instantly and locally — and combined with the extra/conflicting nameservers from the DHCP cleanup, those lookups sometimes timed out
Calico's felix health checks (-felix-ready, -felix-live) call localhost internally, so they kept failing/timing out, triggering repeated liveness-probe restarts

Fixed by restoring proper /etc/hosts entries on both nodes and removing localhost from the ::1 line (since felix has IPv6 disabled), then cycling the calico-node pods to pick up healthy state.
