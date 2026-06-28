# Kubernetes Cluster Bootstrap Tutorial

This tutorial creates a Kubernetes cluster with one control-plane node and three worker nodes on Ubuntu 24.04 LTS.

## Cluster Overview

| Role          | Hostname      |     Internal IP | API alias |
| ------------- | ------------- | --------------: | --------- |
| Control plane | `fdp-k8s-cp1` | `172.21.92.163` | `k8s-api` |
| Worker        | `fdp-k8s-wk1` | `172.21.92.175` | —         |
| Worker        | `fdp-k8s-wk2` | `172.21.92.123` | —         |
| Worker        | `fdp-k8s-wk3` | `172.21.92.157` | —         |

## Software and Network Configuration

| Component | Configuration |
|---|---|
| Operating system | Ubuntu 24.04 LTS |
| Kubernetes repository | Kubernetes `v1.36` stable channel |
| Container runtime | `containerd` |
| CNI plugin | Calico `v3.32.0` |
| Metrics API | Metrics Server `v0.8.1` |
| Kubernetes API endpoint | `k8s-api:6443` |
| Pod network CIDR | `192.168.0.0/16` |
| Container runtime socket | `/run/containerd/containerd.sock` |

## Deployment Topology

```text
                         Kubernetes API
                         k8s-api:6443
                               |
                    172.21.92.163
                      fdp-k8s-cp1
                     Control plane
                               |
          ------------------------------------------------
          |                      |                       |
  172.21.92.175          172.21.92.123           172.21.92.157
   fdp-k8s-wk1            fdp-k8s-wk2             fdp-k8s-wk3
      Worker                 Worker                  Worker
```


---

# Phase 1 — Prepare Every Node

## 1. Configure Local Hostname Resolution

> Run on: **All nodes**

Add every cluster node and the Kubernetes API alias to `/etc/hosts`:

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'

# Kubernetes FDP cluster
172.21.92.163  fdp-k8s-cp1 k8s-api
172.21.92.175  fdp-k8s-wk1
172.21.92.123  fdp-k8s-wk2
172.21.92.157  fdp-k8s-wk3
EOF
```

Verify name resolution:

```bash
getent hosts k8s-api
getent hosts fdp-k8s-cp1
getent hosts fdp-k8s-wk1
getent hosts fdp-k8s-wk2
getent hosts fdp-k8s-wk3
```

## 2. Create the Administration User

> Run on: **All nodes**

Create the `devops` user and add it to the `sudo` group:

```bash
sudo useradd -m -s /bin/bash -G sudo devops
sudo passwd devops
```

Switch to the new user with a login shell:

```bash
su - devops
```

Confirm the active account and home directory:

```bash
whoami
pwd
groups
```

All remaining commands should be run as `devops` unless a section says otherwise.

## 3. Disable Swap

> Run on: **All nodes**

Disable swap immediately and prevent it from being enabled again after reboot:

```bash
sudo swapoff -a
sudo sed -ri '/[[:space:]]swap[[:space:]]/s/^#?/#/' /etc/fstab
```

Verify that no swap device is active:

```bash
swapon --show
free -h
```

## 4. Load Required Kernel Modules

> Run on: **All nodes**

Configure the required modules to load automatically at boot:

```bash
sudo tee /etc/modules-load.d/k8s.conf >/dev/null <<'EOF'
overlay
br_netfilter
EOF
```

Load them immediately:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Verify the modules:

```bash
lsmod | grep -E 'overlay|br_netfilter'
```

## 5. Configure Kubernetes Networking Parameters

> Run on: **All nodes**

Create the Kubernetes sysctl configuration:

```bash
sudo tee /etc/sysctl.d/k8s.conf >/dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Apply all system sysctl settings:

```bash
sudo sysctl --system
```

Verify the required values:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

Each value should be `1`.

---

# Phase 2 — Install and Configure containerd

## 6. Add the Docker APT Repository

> Run on: **All nodes**

Install the repository prerequisites:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's repository signing key:

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the Docker repository:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

## 7. Install containerd

> Run on: **All nodes**

Update the package index and install containerd:

```bash
sudo apt-get update
sudo apt-get install -y containerd.io
```

## 8. Generate the containerd Configuration

> Run on: **All nodes**

Generate the default configuration file:

```bash
containerd config default |
  sudo tee /etc/containerd/config.toml >/dev/null
```

Enable the systemd cgroup driver:

```bash
sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml
```

Verify the setting:

```bash
grep -n 'SystemdCgroup' /etc/containerd/config.toml
```

The result must contain:

```text
SystemdCgroup = true
```

## 9. Start and Enable containerd

> Run on: **All nodes**

Restart containerd to load the new configuration and enable it at boot:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Verify the service:

```bash
systemctl is-active containerd
systemctl is-enabled containerd
```

Expected results:

```text
active
enabled
```

---

# Phase 3 — Install Kubernetes Packages

## 10. Add the Kubernetes APT Repository

> Run on: **All nodes**

Install the required packages:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings
```

Download and install the Kubernetes repository signing key:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key |
  sudo gpg --dearmor --yes \
    -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the Kubernetes `v1.36` stable repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' |
  sudo tee /etc/apt/sources.list.d/kubernetes.list >/dev/null
```

## 11. Install kubelet, kubeadm, and kubectl

> Run on: **All nodes**

Install the Kubernetes components:

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

Prevent automatic package upgrades:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Enable and start kubelet:

```bash
sudo systemctl enable --now kubelet
```

Kubelet may restart until the node is initialized or joined to the cluster. This is expected at this stage.

## 12. Verify the Installed Versions

> Run on: **All nodes**

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

# Phase 4 — Configure the Kubernetes Node IPs

Each node must advertise its own private IP address through kubelet.

## 13. Configure the Control-Plane Node IP

> Run on: **`fdp-k8s-cp1` only**

```bash
echo 'KUBELET_EXTRA_ARGS=--node-ip=172.21.92.163' |
  sudo tee /etc/default/kubelet
```

## 14. Configure Worker Node 1

> Run on: **`fdp-k8s-wk1` only**

```bash
echo 'KUBELET_EXTRA_ARGS=--node-ip=172.21.92.175' |
  sudo tee /etc/default/kubelet
```

## 15. Configure Worker Node 2

> Run on: **`fdp-k8s-wk2` only**

```bash
echo 'KUBELET_EXTRA_ARGS=--node-ip=172.21.92.123' |
  sudo tee /etc/default/kubelet
```

## 16. Configure Worker Node 3

> Run on: **`fdp-k8s-wk3` only**

```bash
echo 'KUBELET_EXTRA_ARGS=--node-ip=172.21.92.157' |
  sudo tee /etc/default/kubelet
```

## 17. Restart kubelet

> Run on: **All nodes**

```bash
sudo systemctl restart kubelet
```

---

# Phase 5 — Initialize the Control Plane

## 18. Pre-pull the Kubernetes Images

> Run on: **`fdp-k8s-cp1` only**

```bash
sudo kubeadm config images pull \
  --cri-socket=unix:///run/containerd/containerd.sock
```

## 19. Initialize the Kubernetes Cluster

> Run on: **`fdp-k8s-cp1` only**

```bash
sudo kubeadm init \
  --apiserver-advertise-address=172.21.92.163 \
  --control-plane-endpoint=k8s-api:6443 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --node-name=fdp-k8s-cp1
```

Keep the `kubeadm join` command printed at the end of the initialization output. It will be used when adding the worker nodes.

## 20. Configure kubectl for the `devops` User

> Run on: **`fdp-k8s-cp1` only**

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

Verify access to the API server:

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

The original administrator kubeconfig is stored at:

```text
/etc/kubernetes/admin.conf
```

The `devops` user's working kubeconfig is stored at:

```text
/home/devops/.kube/config
```

---

# Phase 6 — Install Calico Networking

## 21. Install the Calico Custom Resource Definitions

> Run on: **`fdp-k8s-cp1` only**

```bash
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/v1_crd_projectcalico_org.yaml
```

## 22. Install the Tigera Operator

> Run on: **`fdp-k8s-cp1` only**

```bash
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
```

## 23. Download the Calico Custom Resources

> Run on: **`fdp-k8s-cp1` only**

```bash
curl -LO \
  https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml
```

Confirm that the Calico IP pool matches the Kubernetes Pod CIDR:

```bash
grep -A8 'ipPools:' custom-resources.yaml
```

The output must contain:

```yaml
cidr: 192.168.0.0/16
```

## 24. Create the Calico Resources

> Run on: **`fdp-k8s-cp1` only**

```bash
kubectl create -f custom-resources.yaml
```

## 25. Wait for Calico to Become Available

> Run on: **`fdp-k8s-cp1` only**

```bash
watch kubectl get tigerastatus
```

Press `Ctrl+C` after every component reports:

```text
AVAILABLE=True
PROGRESSING=False
DEGRADED=False
```

Example healthy status:

```text
NAME        AVAILABLE   PROGRESSING   DEGRADED   MESSAGE
apiserver   True        False         False      All objects available
calico      True        False         False      All objects available
goldmane    True        False         False      All objects available
ippools     True        False         False      All objects available
tiers       True        False         False      All objects available
whisker     True        False         False      All objects available
```

## 26. Verify the Control Plane

> Run on: **`fdp-k8s-cp1` only**

```bash
kubectl get pods -A -o wide
kubectl get nodes -o wide
```

Before continuing, `fdp-k8s-cp1` must report `Ready`.

---

# Phase 7 — Join the Worker Nodes

## 27. Generate a Fresh Worker Join Command (Optional, can use the token generated by kubeadmin init process in controlplane above)

> Run on: **`fdp-k8s-cp1` only**

Generate a fresh join command whenever needed:

```bash
sudo kubeadm token create --print-join-command
```

The command will look similar to this:

```bash
kubeadm join k8s-api:6443 \
  --token <BOOTSTRAP_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_CERTIFICATE_HASH>
```

## 28. Join Each Worker to the Cluster

> Run on: **`fdp-k8s-wk1`, `fdp-k8s-wk2`, and `fdp-k8s-wk3`**

Run the generated join command with `sudo` and append the containerd CRI socket:

```bash
sudo kubeadm join k8s-api:6443 \
  --token <BOOTSTRAP_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_CERTIFICATE_HASH> \
  --cri-socket=unix:///run/containerd/containerd.sock
```

Use the real token and CA hash printed by `kubeadm`; do not copy the placeholder values.

---

# Phase 8 — Label and Verify the Worker Nodes

## 29. Add Worker Role Labels

> Run on: **`fdp-k8s-cp1` only**

```bash
kubectl label node fdp-k8s-wk1 node-role.kubernetes.io/worker=worker
kubectl label node fdp-k8s-wk2 node-role.kubernetes.io/worker=worker
kubectl label node fdp-k8s-wk3 node-role.kubernetes.io/worker=worker
```

## 30. Verify the Complete Cluster

> Run on: **`fdp-k8s-cp1` only**

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

Expected node layout:

```text
NAME          STATUS   ROLES           INTERNAL-IP     OS-IMAGE             CONTAINER-RUNTIME
fdp-k8s-cp1   Ready    control-plane   172.21.92.163   Ubuntu 24.04.4 LTS   containerd://2.2.5
fdp-k8s-wk1   Ready    worker          172.21.92.175   Ubuntu 24.04.4 LTS   containerd://2.2.5
fdp-k8s-wk2   Ready    worker          172.21.92.123   Ubuntu 24.04.4 LTS   containerd://2.2.5
fdp-k8s-wk3   Ready    worker          172.21.92.157   Ubuntu 24.04.4 LTS   containerd://2.2.5
```

All four nodes must report `Ready` before deploying workloads.

---

# Phase 9 — Install Metrics Server

## 31. Deploy Metrics Server

> Run on: **`fdp-k8s-cp1` only**

```bash
kubectl apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.8.1/components.yaml
```

## 32. Allow Metrics Server to Read the Kubelet Metrics Endpoint

> Run on: **`fdp-k8s-cp1`  or your host which can access the cluster via kubeconfig**

For this private cluster, add the `--kubelet-insecure-tls` argument:

```bash
kubectl patch deployment metrics-server \
  -n kube-system \
  --type=json \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/args/-",
      "value": "--kubelet-insecure-tls"
    }
  ]'
```

> **Security note:** `--kubelet-insecure-tls` is appropriate for this private lab environment. Use trusted kubelet certificates instead for a production environment.

## 33. Wait for Metrics Server

```bash
kubectl rollout status deployment/metrics-server -n kube-system
kubectl get pods -n kube-system -l k8s-app=metrics-server
```

## 34. Verify Resource Metrics

```bash
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory
```

---

# Phase 10 — Safely Maintain a Worker Node

Use the following procedure before rebooting a worker, upgrading its kernel, changing CPU or RAM, or performing storage maintenance.

The example below uses `fdp-k8s-wk1`.

## 35. Drain the Worker Node

> Run on: **`fdp-k8s-cp1` only**

```bash
kubectl drain fdp-k8s-wk1 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Confirm that the node is marked unschedulable:

```bash
kubectl get nodes
```

## 36. Perform the Maintenance

Perform the required operation on `fdp-k8s-wk1`, for example:

```bash
sudo reboot
```

After the node returns, verify its services locally:

```bash
systemctl is-active containerd
systemctl is-active kubelet
```

## 37. Return the Worker Node to Service

> Run on: **`fdp-k8s-cp1` only**

```bash
kubectl uncordon fdp-k8s-wk1
```

Verify that the node is schedulable and ready again:

```bash
kubectl get nodes -o wide
```

---

# Final Validation Checklist

Run the following commands from `fdp-k8s-cp1`:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get tigerastatus
kubectl top nodes
```

The cluster is ready when:

- All four nodes report `Ready`.
- The control-plane components are running.
- Calico reports `AVAILABLE=True` and `DEGRADED=False`.
- CoreDNS is running.
- Metrics Server returns node and Pod resource usage.

## Important Paths

| Purpose | Path |
|---|---|
| Administrator kubeconfig | `/etc/kubernetes/admin.conf` |
| `devops` user kubeconfig | `/home/devops/.kube/config` |
| containerd configuration | `/etc/containerd/config.toml` |
| kubelet environment options | `/etc/default/kubelet` |
| Kubernetes static Pod manifests | `/etc/kubernetes/manifests/` |
| kubelet configuration | `/var/lib/kubelet/config.yaml` |
