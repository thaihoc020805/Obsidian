
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

