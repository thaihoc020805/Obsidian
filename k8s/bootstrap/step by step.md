
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