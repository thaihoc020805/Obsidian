
in all node
```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'

# Kubernetes FDP cluster
172.21.92.163  fdp-k8s-cp1 k8s-api
172.21.92.175  fdp-k8s-wk1
172.21.92.123  fdp-k8s-wk2
172.21.92.157  fdp-k8s-wk3
EOF
```


```bash
useradd -m -s /bin/bash -G sudo devops && echo 'devops:thaihoc285' | chpasswd
su devops
cd
```


```bash
sudo swapoff -a
sudo sed -ri '/[[:space:]]swap[[:space:]]/s/^#?/#/' /etc/fstab
```

```bash
sudo tee /etc/modules-load.d/k8s.conf >/dev/null <<'EOF'
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

```

```bash
sudo tee /etc/sysctl.d/k8s.conf >/dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt-get install -y containerd.io
```

```bash
containerd config default |
  sudo tee /etc/containerd/config.toml >/dev/null
  
sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml
```

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings

```

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key |
  sudo gpg --dearmor --yes \
    -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' |
  sudo tee /etc/apt/sources.list.d/kubernetes.list >/dev/null
```


```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

```bash
kubeadm version
kubectl version --client
kubelet --version
```

## Trên `fdp-k8s-cp1` 
```bash
echo 'KUBELET_EXTRA_ARGS=--node-ip=172.21.92.163' |
  sudo tee /etc/default/kubelet
```

## Trên `fdp-k8s-wk1`

```bash
echo 'KUBELET_EXTRA_ARGS=--node-ip=172.21.92.175' |
  sudo tee /etc/default/kubelet
```

## Trên `fdp-k8s-wk2`

```bash
echo 'KUBELET_EXTRA_ARGS=--node-ip=172.21.92.123' |
  sudo tee /etc/default/kubelet
```

## Trên `fdp-k8s-wk3`

```bash
echo 'KUBELET_EXTRA_ARGS=--node-ip=172.21.92.157' |
  sudo tee /etc/default/kubelet
```

```bash
sudo systemctl restart kubelet
```

## Trên `fdp-k8s-cp1` 
```bash
sudo kubeadm config images pull \
  --cri-socket=unix:///run/containerd/containerd.sock
```

```bash
sudo kubeadm init \
  --apiserver-advertise-address=172.21.92.163 \
  --control-plane-endpoint=k8s-api:6443 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --node-name=fdp-k8s-cp1
```

```bash
mkdir -p "$HOME/.kube"

sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"

sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

```bash
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/v1_crd_projectcalico_org.yaml
```

```bash
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
```

```bash
curl -LO \
  https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml
  
grep -A8 'ipPools:' custom-resources.yaml
```

phải có cidr: 192.168.0.0/16

```bash
kubectl create -f custom-resources.yaml
```


```bash
watch kubectl get tigerastatus

until:

Every 2.0s: kubectl get tigerastatus       fdp-k8s-cp1: Tue Jun 23 22:47:49 2026

NAME        AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
apiserver   True        False         False      2m8s    All objects available
calico      True        False         False      103s    All objects available
goldmane    True        False         False      2m3s    All objects available
ippools     True        False         False      3m19s   All objects available
tiers       True        False         False      2m3s    All objects available
whisker     True        False         False      83s     All objects available
```

```bash
kubectl get pods -A -o wide
kubectl get nodes -o wide
```

control plane phải thành ready

Trên `fdp-k8s-wk[1-3]` 

```bash
sudo kubeadm join k8s-api:6443 \
  --token dnhey2.16b8iqaxvxru1jwk \
  --discovery-token-ca-cert-hash sha256:e2a24ca01f7e7dba40dbfc15963561646dc5a11ae105d4d628779658ba66dcf9 \
  --cri-socket=unix:///run/containerd/containerd.sock
```


Trên `fdp-k8s-cp1`:
```bash
kubectl label node fdp-k8s-wk1 node-role.kubernetes.io/worker=worker
kubectl label node fdp-k8s-wk2 node-role.kubernetes.io/worker=worker
kubectl label node fdp-k8s-wk3 node-role.kubernetes.io/worker=worker
```

```bash
kubectl get nodes -o wide

NAME          STATUS   ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION             CONTAINER-RUNTIME
fdp-k8s-cp1   Ready    control-plane   16m     v1.36.2   172.21.92.163   <none>        Ubuntu 24.04.4 LTS   6.8.0-58-generic (amd64)   containerd://2.2.5
fdp-k8s-wk1   Ready    worker          3m11s   v1.36.2   172.21.92.175   <none>        Ubuntu 24.04.4 LTS   6.8.0-58-generic (amd64)   containerd://2.2.5
fdp-k8s-wk2   Ready    worker          2m58s   v1.36.2   172.21.92.123   <none>        Ubuntu 24.04.4 LTS   6.8.0-58-generic (amd64)   containerd://2.2.5
fdp-k8s-wk3   Ready    worker          2m42s   v1.36.2   172.21.92.157   <none>        Ubuntu 24.04.4 LTS   6.8.0-58-generic (amd64)   containerd://2.2.5
```


kubeconfig lưu ở: /etc/kubernetes/admin.conf