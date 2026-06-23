
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
sudo mkdir -p /etc/containerd

containerd config default |
  sudo tee /etc/containerd/config.toml >/dev/null
```

