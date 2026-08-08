Nhìn terminal thì máy bạn đang dùng Linux kiểu Ubuntu/Debian và đã đăng nhập `root`, nên làm lần lượt như dưới đây.

## 1. Cập nhật hệ thống và cài công cụ cơ bản

```
apt update && apt upgrade -y
apt install -y curl ca-certificates jq
```

Kiểm tra hệ điều hành:

```
cat /etc/os-release
```

## 2. Tạo 4 GB swap

Máy chỉ có 11 GiB RAM và hiện chưa có swap. Swap không giúp model chạy nhanh hơn, nhưng hạn chế Ollama hoặc OpenClaw bị OOM kill khi RAM tăng đột ngột.

```
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

Cho swap tự bật sau khi reboot:

```
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

Kiểm tra:

```
free -h
```

Bạn phải thấy đại loại:

```
Swap: 4.0Gi
```

Giảm xu hướng dùng swap:

```
echo 'vm.swappiness=10' > /etc/sysctl.d/99-ollama.conf
sysctl --system
```

## 3. Cài Ollama

Lệnh cài Linux chính thức của Ollama là:

```
curl -fsSL https://ollama.com/install.sh | sh
```

Kiểm tra:

```
ollama --version
```

Kiểm tra service:

```
systemctl status ollama --no-pager
```

Nếu chưa chạy:

```
systemctl enable --now ollama
```

Kiểm tra Ollama API:

```
curl http://127.0.0.1:11434/api/tags
```

Nếu nhận được JSON thì Ollama đã hoạt động.

## 4. Tải `qwen2.5:3b`

Ollama có sẵn tag chính thức `qwen2.5:3b`; đây là model đa ngôn ngữ, hỗ trợ tool và có bản 3B phù hợp với máy bạn.

```
ollama pull qwen2.5:3b
```

Dung lượng tải khoảng vài GB. Kiểm tra model:

```
ollama list
```

Bạn sẽ thấy tương tự:

```
NAME           ID              SIZE
qwen2.5:3b     xxxxxxxxxxxx    1.9 GB
```

## 5. Chạy thử

```
ollama run qwen2.5:3b
```

Sau khi vào giao diện chat, bật thống kê:

```
/set verbose
```

Hỏi thử:

```
Giải thích Kubernetes namespace trong khoảng 300 từ.
```

Cuối câu trả lời sẽ có các thông số như:

```
prompt eval rate: ...
eval rate: ...
```

`eval rate` là tốc độ sinh câu trả lời mà bạn quan tâm.

Thoát bằng:

```
/bye
```

## 6. Test bằng API

```
curl http://127.0.0.1:11434/api/chat \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen2.5:3b",
    "messages": [
      {
        "role": "user",
        "content": "Xin chào, hãy giới thiệu ngắn gọn về Kubernetes."
      }
    ],
    "stream": false
  }' | jq
```

Nếu có trường `message.content`, mọi thứ đã chạy đúng.

## 7. Giới hạn context để tránh tốn RAM

Tạo một model riêng tối ưu cho VM:

```
cat > /root/Modelfile-qwen3b <<'EOF'
FROM qwen2.5:3b

PARAMETER num_ctx 8192
PARAMETER num_thread 4
PARAMETER temperature 0.6
PARAMETER top_p 0.9

SYSTEM Bạn là một trợ lý AI hữu ích. Hãy trả lời chính xác, rõ ràng và ưu tiên tiếng Việt khi người dùng sử dụng tiếng Việt.
EOF
```

Tạo model:

```
ollama create qwen2.5-3b-vm -f /root/Modelfile-qwen3b
```

Chạy:

```
ollama run qwen2.5-3b-vm
```

Cấu hình này:

- Dùng đủ 4 CPU thread.
- Giới hạn context ở 8K để không ăn quá nhiều RAM.
- Phù hợp chat thông thường.
- Để lại RAM cho OpenClaw.

Lưu ý: không nên đặt `num_ctx 128000` chỉ vì model hỗ trợ context lớn. Context càng lớn thì RAM và thời gian xử lý prompt càng tăng.

## 8. Theo dõi tài nguyên khi chạy

Mở một terminal SSH khác:

```
apt install -y htop
htop
```

Hoặc dùng:

```
watch -n 1 'free -h; echo; ps -C ollama -o pid,%cpu,%mem,rss,cmd'
```

Trong lúc model sinh câu trả lời, CPU gần 400% là bình thường vì máy có 4 vCPU.

Kiểm tra log Ollama:

```
journalctl -u ollama -f
```

Ollama hướng dẫn dùng `journalctl -e -u ollama` để xem log service trên Linux.

## 9. Kết nối với OpenClaw

OpenClaw có Ollama provider tích hợp, dùng native Ollama API và local server không cần API key. Model được tham chiếu theo dạng `ollama/tên-model`.

Với model gốc:

```
ollama/qwen2.5:3b
```

Với model mình vừa tạo:

```
ollama/qwen2.5-3b-vm
```

Kiểm tra model OpenClaw nhìn thấy:

```
openclaw models list --provider ollama
```

Đặt model chính:

```
openclaw models set ollama/qwen2.5-3b-vm
```

Nếu phiên bản OpenClaw của bạn chưa có lệnh đó, chạy onboarding:

```
openclaw onboard
```

Sau đó chọn:

```
Provider: Ollama
Model: qwen2.5-3b-vm
```

OpenClaw khuyến nghị Ollama là một trong những cách đơn giản nhất để bắt đầu với local model.

## Toàn bộ lệnh cài nhanh

Có thể copy nguyên khối này:

```
apt update && apt upgrade -y
apt install -y curl ca-certificates jq htop

fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
echo 'vm.swappiness=10' > /etc/sysctl.d/99-ollama.conf
sysctl --system

curl -fsSL https://ollama.com/install.sh | sh
systemctl enable --now ollama

ollama pull qwen2.5:3b
ollama run qwen2.5:3b
```

Mình khuyên trước mắt **chỉ chạy đúng `qwen2.5:3b`**, đo `eval rate` thực tế rồi mới quyết định giữ nó hay chuyển xuống `qwen3:1.7b`.



<div contenteditable="plaintext-only">root@k8s-vm3:~# df -h Filesystem Size Used Avail Use% Mounted on tmpfs 1.2G 1.1M 1.2G 1% /run /dev/vda2 99G 3.2G 92G 4% / tmpfs 5.9G 0 5.9G 0% /dev/shm tmpfs 5.0M 0 5.0M 0% /run/lock tmpfs 1.2G 12K 1.2G 1% /run/user/0 root@k8s-vm3:~# apt install -y curl ca-certificates jq Reading package lists... Done Building dependency tree... Done Reading state information... Done curl is already the newest version (8.5.0-2ubuntu10.11). curl set to manually installed. ca-certificates is already the newest version (20260601~24.04.1). ca-certificates set to manually installed. jq is already the newest version (1.7.1-3ubuntu0.24.04.2). jq set to manually installed. 0 upgraded, 0 newly installed, 0 to remove and 78 not upgraded. root@k8s-vm3:~# fallocate -l 4G /swapfile chmod 600 /swapfile mkswap /swapfile swapon /swapfile Setting up swapspace version 1, size = 4 GiB (4294963200 bytes) no label, UUID=d5d6cbef-8e56-4dbc-a677-d1109cdfff7f root@k8s-vm3:~# free -h total used free shared buff/cache available Mem: 11Gi 589Mi 8.9Gi 4.1Mi 2.5Gi 11Gi Swap: 4.0Gi 0B 4.0Gi root@k8s-vm3:~# echo 'vm.swappiness=10' > /etc/sysctl.d/99-ollama.conf sysctl --system * Applying /usr/lib/sysctl.d/10-apparmor.conf ... * Applying /etc/sysctl.d/10-bufferbloat.conf ... * Applying /etc/sysctl.d/10-console-messages.conf ... * Applying /etc/sysctl.d/10-ipv6-privacy.conf ... * Applying /etc/sysctl.d/10-kernel-hardening.conf ... * Applying /etc/sysctl.d/10-magic-sysrq.conf ... * Applying /etc/sysctl.d/10-map-count.conf ... * Applying /etc/sysctl.d/10-network-security.conf ... * Applying /etc/sysctl.d/10-ptrace.conf ... * Applying /etc/sysctl.d/10-zeropage.conf ... * Applying /usr/lib/sysctl.d/50-pid-max.conf ... * Applying /etc/sysctl.d/99-ollama.conf ... * Applying /usr/lib/sysctl.d/99-protect-links.conf ... * Applying /etc/sysctl.d/99-sysctl.conf ... * Applying /etc/sysctl.conf ... kernel.apparmor_restrict_unprivileged_userns = 1 net.core.default_qdisc = fq_codel kernel.printk = 4 4 1 7 net.ipv6.conf.all.use_tempaddr = 2 net.ipv6.conf.default.use_tempaddr = 2 kernel.kptr_restrict = 1 kernel.sysrq = 176 vm.max_map_count = 1048576 net.ipv4.conf.default.rp_filter = 2 net.ipv4.conf.all.rp_filter = 2 kernel.yama.ptrace_scope = 1 vm.mmap_min_addr = 65536 kernel.pid_max = 4194304 vm.swappiness = 10 fs.protected_fifos = 1 fs.protected_hardlinks = 1 fs.protected_regular = 2 fs.protected_symlinks = 1 root@k8s-vm3:~# curl -fsSL https://ollama.com/install.sh | sh >>> Installing ollama to /usr/local >>> Downloading ollama-linux-amd64.tar.zst ######################################################################## 100.0% >>> Creating ollama user... >>> Adding ollama user to render group... >>> Adding ollama user to video group... >>> Adding current user to ollama group... >>> Creating ollama systemd service... >>> Enabling and starting ollama service... Created symlink /etc/systemd/system/default.target.wants/ollama.service → /etc/systemd/system/ollama.service. >>> The Ollama API is now available at 127.0.0.1:11434. >>> Install complete. Run "ollama" from the command line. WARNING: No NVIDIA/AMD GPU detected. Ollama will run in CPU-only mode. root@k8s-vm3:~# ollama --version ollama version is 0.32.5 root@k8s-vm3:~# systemctl status ollama --no-pager ● ollama.service - Ollama Service Loaded: loaded (/etc/systemd/system/ollama.service; enabled; preset: enabled) Active: active (running) since Wed 2026-08-05 17:35:48 +07; 11s ago Main PID: 90178 (ollama) Tasks: 10 (limit: 14263) Memory: 13.8M (peak: 23.6M) CPU: 299ms CGroup: /system.slice/ollama.service └─90178 /usr/local/bin/ollama serve Aug 05 17:35:48 k8s-vm3 ollama[90178]: time=2026-08-05T17:35:48.474+07:00 level=INFO source=routes.go:1949 msg="Ollama cloud disabled: false" Aug 05 17:35:48 k8s-vm3 ollama[90178]: time=2026-08-05T17:35:48.475+07:00 level=INFO source=images.go:883 msg="total blobs: 0" Aug 05 17:35:48 k8s-vm3 ollama[90178]: time=2026-08-05T17:35:48.475+07:00 level=INFO source=images.go:890 msg="total unused blobs removed: 0" Aug 05 17:35:48 k8s-vm3 ollama[90178]: time=2026-08-05T17:35:48.476+07:00 level=INFO source=routes.go:2004 msg="Listening on 127.0.0.1:11434 (version 0.32.5)" Aug 05 17:35:48 k8s-vm3 ollama[90178]: time=2026-08-05T17:35:48.476+07:00 level=INFO source=model_list_cache.go:112 msg="model list cache hydration complete" mode…psed=227.089µs Aug 05 17:35:48 k8s-vm3 ollama[90178]: time=2026-08-05T17:35:48.477+07:00 level=INFO source=runner.go:60 msg="discovering available GPUs..." Aug 05 17:35:48 k8s-vm3 ollama[90178]: time=2026-08-05T17:35:48.663+07:00 level=INFO source=types.go:50 msg="inference compute" id=cpu library=cpu compute="" name…ble="11.0 GiB" Aug 05 17:35:48 k8s-vm3 ollama[90178]: time=2026-08-05T17:35:48.664+07:00 level=INFO source=routes.go:2054 msg="vram-based default context" total_vram="0 B" default_num_ctx=4096 Aug 05 17:35:48 k8s-vm3 ollama[90178]: time=2026-08-05T17:35:48.946+07:00 level=INFO source=model_recommendations.go:177 msg="model recommendations cache sleep sc…ive_failures=0 Aug 05 17:35:55 k8s-vm3 ollama[90178]: [GIN] 2026/08/05 - 17:35:55 | 200 | 90.378µs | 127.0.0.1 | GET "/api/version" Hint: Some lines were ellipsized, use -l to show in full. root@k8s-vm3:~# curl http://127.0.0.1:11434/api/tags {"models":[]}rooollama pull qwen2.5:3bll qwen2.5:3b pulling manifest pulling 5ee4f07cdb9b: 100% ▕██████████████████████████████████████████████████████████████████████████████████████████████████████████████████ ▏ 1.9 GB/1.9 GB 4.9 MB/s 0s verifying sha256 digest writing manifest success root@k8s-vm3:~# ollama list NAME ID SIZE MODIFIED qwen2.5:3b 357c53fb659c 1.9 GB 2 seconds ago root@k8s-vm3:~# ollama run qwen2.5:3b >>> /set verbose Set 'verbose' mode. >>> Giải thích Kubernetes namespace trong khoảng 300 từ. Kubernetes namespace là một tính năng quan trọng giúp quản lý các ứng dụng và cấu trúc tài nguyên trong môi trường Kubernetes. Nó hoạt động như một lớp wrapper quanh các deployments, pod, service,... giúp chúng được phân biệt với nhau. Các namespace giúp giải quyết vấn đề khi bạn có nhiều tổ chức hoặc nhiều nhóm phát triển sử dụng cùng hệ thống Kubernetes, tránh xung đột giữa các tài nguyên. Kubernetes namespace hoạt động dựa trên một số nguyên tắc cơ bản: 1. Mỗi ứng dụng và cấu trúc trong Kubernetes đều được đặt trong một namespace. 2. Nếu không chỉ định rõ namespace, Kubernetes sẽ sử dụng default namespace mặc định. 3. Người dùng có thể tạo nhiều namespaces tùy ý, mỗi namespace cho phép quản lý các tài nguyên của riêng mình. Các lợi ích chính từ việc sử dụng namespace bao gồm: - Quản lý và phân biệt tài nguyên: Dùng namespace để quản lý và tránh xung đột giữa các ứng dụng. - Tính khả tầm mở rộng: Cung cấp một môi trường thử nghiệm không ảnh hưởng đến environment chính, hoặc cho phép triển khai các application mới mà không cần cập nhật cấu hình gốc của hệ thống. - Tính riêng tư: Bảo vệ tài nguyên quan trọng khỏi truy cập trái phép, thông qua việc đặt nó trong namespace riêng. Tóm lại, mặc dù Kubernetes đã có sẵn một concept là pod cho quản lý các ứng dụng nhỏ và nhẹ, thì namespace giúp nâng cấp khả năng này lên một tầm cao mới, hỗ trợ người dùng tổ chức tài nguyên phức tạp hơn, từ đó tăng cường hiệu suất và tính năng của hệ thống Kubernetes. total duration: 33.809135252s load duration: 271.790457ms prompt eval count: 42 token(s) prompt eval duration: 889.587ms prompt eval rate: 47.21 tokens/s eval count: 341 token(s) eval duration: 32.64367s eval rate: 10.45 tokens/s >>> /bye /bye Rất vui được giúp bạn. Nếu bạn có thêm câu hỏi hoặc cần thông tin về gì khác, hãy đừng ngần ngại liên hệ với tôi. total duration: 4.140529092s load duration: 293.604132ms prompt eval count: 395 token(s) prompt eval duration: 408.832ms prompt eval rate: 966.17 tokens/s eval count: 36 token(s) eval duration: 3.429231s eval rate: 10.50 tokens/s >>> /bye root@k8s-vm3:~# curl http://127.0.0.1:11434/api/chat \ -H 'Content-Type: application/json' \ -d '{ "model": "qwen2.5:3b", "messages": [ { "role": "user", "content": "Xin chào, hãy giới thiệu ngắn gọn về Kubernetes." } ], "stream": false }' | jq % Total % Received % Xferd Average Speed Time Time Time Current Dload Upload Total Spent Left Speed 100 1776 100 1577 100 199 59 7 0:00:28 0:00:26 0:00:02 349 { "model": "qwen2.5:3b", "created_at": "2026-08-05T10:39:47.493133884Z", "message": { "role": "assistant", "content": "Chào bạn! Kubernetes, thường được gọi là \"K8s\" (do số ký tự trong tên bắt đầu bằng \"k\" và kết thúc bằng \"s\"), là một nền tảng quản lý quy mô lớn cho ứng dụng chạy trên đám mây. Nó cung cấp giải pháp để triển khai, duy trì và quản lý các dịch vụ container hóa.\n\nKubernetes được thiết kế nhằm tạo ra một môi trường đáng tin cậy và tự động hóa cho việc phân chia và điều phối các dịch vụ có thể thực thi trong một loạt các môi trường khác nhau. Nó cung cấp các công cụ mạnh mẽ để quản lý ứng dụng bao gồm:\n\n1. Quản lý các nhóm dịch vụ (pods): Kubernetes xắp xếp và chạy các dịch vụ container hóa cùng nhau.\n2. Quản lý tài nguyên: Điều chỉnh lượng CPU, bộ nhớ cho các pod.\n3. Điều phối và giám sát các dịch vụ trên một mạng lớn, không chỉ bao gồm các máy tính, mà còn cả đám mây và vật lý.\n\nKubernetes hỗ trợ nhiều ngôn ngữ lập trình khác nhau như Node.js, Java, Go, Python, Ruby và cung cấp khả năng tạo môi trường đa nền tảng cho ứng dụng, bao gồm cả trong doanh nghiệp và đám mây." }, "done": true, "done_reason": "stop", "total_duration": 26531992207, "load_duration": 231702629, "prompt_eval_count": 41, "prompt_eval_duration": 380868000, "eval_count": 271, "eval_duration": 25902527000 } root@k8s-vm3:~#</div>

Gather data sampling: Not affected

Itlb multihit: Not affected

L1tf: Not affected

Mds: Not affected

Meltdown: Not affected

Mmio stale data: Vulnerable: Clear CPU buffers attempted, no microcode; SMT Host state unknown

Reg file data sampling: Vulnerable: No microcode

Retbleed: Not affected

Spec rstack overflow: Not affected

Spec store bypass: Mitigation; Speculative Store Bypass disabled via prctl

Spectre v1: Mitigation; usercopy/swapgs barriers and __user pointer sanitization

Spectre v2: Mitigation; Enhanced / Automatic IBRS; IBPB conditional; RSB filling; PBRSB-eIBRS Not affected; BHI SW loop, KVM SW loop

Srbds: Not affected

Tsx async abort: Mitigation; TSX disabled

root@k8s-vm3:~# df -h

Filesystem Size Used Avail Use% Mounted on

tmpfs 1.2G 1.1M 1.2G 1% /run

/dev/vda2 99G 3.2G 92G 4% /

tmpfs 5.9G 0 5.9G 0% /dev/shm

tmpfs 5.0M 0 5.0M 0% /run/lock

tmpfs 1.2G 12K 1.2G 1% /run/user/0

root@k8s-vm3:~#

apt install -y curl ca-certificates jq

Reading package lists... Done

Building dependency tree... Done

Reading state information... Done

curl is already the newest version (8.5.0-2ubuntu10.11).

curl set to manually installed.

ca-certificates is already the newest version (20260601~24.04.1).

ca-certificates set to manually installed.

jq is already the newest version (1.7.1-3ubuntu0.24.04.2).

jq set to manually installed.

0 upgraded, 0 newly installed, 0 to remove and 78 not upgraded.

root@k8s-vm3:~# fallocate -l 4G /swapfile

chmod 600 /swapfile

mkswap /swapfile

swapon /swapfile

Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)