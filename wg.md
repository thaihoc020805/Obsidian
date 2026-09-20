Ok, dưới đây là **bản full từ đầu đến cuối**, gom lại đúng những gì mình đã làm, bao gồm cả các lỗi đã gặp và lý do phải sửa. Mục tiêu cuối cùng là:

```
Ubuntu client
    │
    │ WireGuard
    ▼
VPS Oracle Singapore
    │
    ├─ traffic bình thường của VPS
    │    10.0.0.199 → 161.118.252.60
    │
    └─ traffic của WireGuard client
         SNAT → 10.0.0.121
              → 161.118.197.192
```

Kết quả cuối:

```
Client IPv4        → qua VPN
Public IPv4 client → 161.118.197.192

Client DNS         → 1.1.1.1 qua wg0
Client IPv6        → block, tránh IPv6 leak

VPS bình thường    → 161.118.252.60
SSH khi VPN bật    → ssh ubuntu@10.66.66.1
```

---

# 0. Trạng thái network VPS trước khi cài WireGuard

VPS của bạn có một VNIC `enp0s6` với:

```
10.0.0.199  = primary private IP
              ↔ 161.118.252.60 public reserved IP

10.0.0.121  = secondary private IP
              ↔ 161.118.197.192 public reserved IP
```

Linux:

```
ip addr show enp0s6
```

đang đúng kiểu:

```
inet 10.0.0.199/24 ... scope global enp0s6
inet 10.0.0.121/24 ... scope global secondary enp0s6
```

Route:

```
ip route
```

đang là:

```
default via 10.0.0.1 dev enp0s6 proto static
10.0.0.0/24 dev enp0s6 proto kernel scope link src 10.0.0.199
```

Điều này quan trọng vì:

```
traffic bình thường VPS → source 10.0.0.199
traffic WG client       → mình sẽ ép SNAT thành 10.0.0.121
```

Test:

```
curl -4 https://ifconfig.me
```

trên VPS phải là:

```
161.118.252.60
```

và:

```
curl --interface 10.0.0.121 https://ifconfig.me
```

phải là:

```
161.118.197.192
```

---

# 1. Cài WireGuard trên VPS

Trên VPS:

```
sudo apt update
sudo apt install -y wireguard
```

Tạo thư mục:

```
sudo install -d -m 700 /etc/wireguard
```

Tạo keypair server:

```
sudo sh -c 'umask 077; wg genkey | tee /etc/wireguard/server.key | wg pubkey > /etc/wireguard/server.pub'
```

Kiểm tra:

```
sudo ls -l /etc/wireguard/
```

sẽ có:

```
server.key
server.pub
```

Xem **public key**:

```
sudo cat /etc/wireguard/server.pub
```

Public key share được.

**Không gửi:**

```
/etc/wireguard/server.key
```

cho ai.

Trong setup hiện tại của bạn, server public key là:

```
KRS879mCwoGgCIZeo/Aq26WP44yUfbSQ4xar6bIKpzQ=
```

---

# 2. Bật IPv4 forwarding trên VPS

VPS phải có khả năng nhận packet từ `wg0` rồi forward nó ra `enp0s6`.

Tạo:

```
sudo nano /etc/sysctl.d/99-wireguard.conf
```

ghi:

```
net.ipv4.ip_forward=1
```

Apply:

```
sudo sysctl --system
```

Check:

```
sysctl net.ipv4.ip_forward
```

phải ra:

```
net.ipv4.ip_forward = 1
```

---

# 3. Tạo cấu hình WireGuard server

Lấy private key:

```
sudo cat /etc/wireguard/server.key
```

Không gửi key này đi đâu.

Mở:

```
sudo nano /etc/wireguard/wg0.conf
```

Cấu hình cuối cùng của server là:

```
[Interface]
Address = 10.66.66.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

PostUp = iptables -I FORWARD 1 -i wg0 -o enp0s6 -j ACCEPT; iptables -I FORWARD 1 -i enp0s6 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -A POSTROUTING -s 10.66.66.0/24 -o enp0s6 -j SNAT --to-source 10.0.0.121

PostDown = iptables -D FORWARD -i wg0 -o enp0s6 -j ACCEPT; iptables -D FORWARD -i enp0s6 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -D POSTROUTING -s 10.66.66.0/24 -o enp0s6 -j SNAT --to-source 10.0.0.121

[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.66.66.2/32
```

Thay:

```
SERVER_PRIVATE_KEY
```

bằng nội dung thật của:

```
/etc/wireguard/server.key
```

Client public key của máy bạn hiện tại là:

```
PCkniEJGpYTVePnSy6MbUEma3YqCe/xNeSh3KPRCBBQ=
```

nên `[Peer]` hiện tại của bạn là:

```
[Peer]
PublicKey = PCkniEJGpYTVePnSy6MbUEma3YqCe/xNeSh3KPRCBBQ=
AllowedIPs = 10.66.66.2/32
```

Set permission:

```
sudo chmod 600 /etc/wireguard/wg0.conf
```

## Tại sao là `-I FORWARD` chứ không phải `-A FORWARD`?

Đây là một lỗi thực tế mình đã gặp.

Oracle Ubuntu image của bạn có:

```
FORWARD
1 REJECT everything
```

Nếu dùng:

```
iptables -A FORWARD ...
```

thì rule WG bị append **sau REJECT**:

```
1 REJECT
2 ACCEPT wg0 -> enp0s6
```

=> packet chết ở rule 1.

Vì vậy phải dùng:

```
iptables -I FORWARD 1 ...
```

để rule WG nằm **trước REJECT**.

---

# 4. SNAT WireGuard traffic ra đúng public IP thứ hai

Dòng quan trọng nhất trong server config:

```
iptables -t nat -A POSTROUTING \
  -s 10.66.66.0/24 \
  -o enp0s6 \
  -j SNAT --to-source 10.0.0.121
```

Nó biến:

```
10.66.66.2
```

thành:

```
10.0.0.121
```

khi packet ra Internet.

OCI sau đó NAT:

```
10.0.0.121
↓
161.118.197.192
```

Do đó website nhìn thấy:

```
161.118.197.192
```

chứ không phải IP quản trị VPS:

```
161.118.252.60
```

---

# 5. Mở UDP 51820 trong OCI

OCI Console:

```
Networking
→ Virtual Cloud Networks
→ main-vcn
→ Security Lists
→ security list của public-subnet
→ Add Ingress Rules
```

Rule:

```
Source Type:             CIDR
Source CIDR:             0.0.0.0/0
IP Protocol:             UDP
Source Port Range:       All
Destination Port Range:  51820
Stateless:               No
Description:             WireGuard
```

Nếu client luôn có một public IP cố định thì có thể dùng:

```
x.x.x.x/32
```

thay vì:

```
0.0.0.0/0
```

Nhưng laptop đổi Wi-Fi/mạng liên tục thì `0.0.0.0/0` tiện hơn.

WireGuard vẫn cần đúng private key mới handshake được.

---

# 6. Mở UDP 51820 trong firewall của Ubuntu VPS

Đây là lỗi đầu tiên mình gặp.

`ufw` không tồn tại:

```
sudo ufw status
```

ra:

```
sudo: ufw: command not found
```

nhưng **không có UFW không có nghĩa không có firewall**.

Check:

```
sudo iptables -L INPUT -n -v --line-numbers
```

VPS của bạn ban đầu có:

```
1 ACCEPT RELATED,ESTABLISHED
2 ACCEPT ICMP
3 ACCEPT lo
4 ACCEPT TCP 22
5 REJECT everything
```

Nên UDP 51820:

```
packet UDP 51820
↓
không match 1-4
↓
REJECT
```

Đó là lý do `tcpdump` nhìn thấy packet tới server nhưng WireGuard không handshake.

Thêm rule trước REJECT:

```
sudo iptables -I INPUT 5 -p udp --dport 51820 -j ACCEPT
```

Check:

```
sudo iptables -L INPUT -n -v --line-numbers
```

kết quả cuối:

```
1 ACCEPT RELATED,ESTABLISHED
2 ACCEPT ICMP
3 ACCEPT lo
4 ACCEPT TCP 22
5 ACCEPT UDP 51820
6 REJECT
```

---

# 7. Persist rule INPUT 51820

Cài:

```
sudo apt install -y iptables-persistent
```

Trong máy bạn package đã có sẵn.

Điểm cực kỳ quan trọng:

**Không save firewall trong lúc các rule WireGuard động đang bị duplicate.**

Cách sạch là stop WG trước:

```
sudo systemctl stop wg-quick@wg0
```

Nếu trước đó đã test bằng tay và có duplicate, dọn:

```
while sudo iptables -C FORWARD -i wg0 -o enp0s6 -j ACCEPT 2>/dev/null; do
  sudo iptables -D FORWARD -i wg0 -o enp0s6 -j ACCEPT
done
```

```
while sudo iptables -C FORWARD -i enp0s6 -o wg0 \
  -m conntrack --ctstate RELATED,ESTABLISHED \
  -j ACCEPT 2>/dev/null; do

  sudo iptables -D FORWARD -i enp0s6 -o wg0 \
    -m conntrack --ctstate RELATED,ESTABLISHED \
    -j ACCEPT
done
```

Dọn NAT:

```
while sudo iptables -t nat -C POSTROUTING \
  -s 10.66.66.0/24 \
  -o enp0s6 \
  -j SNAT --to-source 10.0.0.121 2>/dev/null; do

  sudo iptables -t nat -D POSTROUTING \
    -s 10.66.66.0/24 \
    -o enp0s6 \
    -j SNAT --to-source 10.0.0.121
done
```

Đảm bảo INPUT vẫn có UDP 51820:

```
sudo iptables -L INPUT -n --line-numbers
```

Sau đó mới:

```
sudo netfilter-persistent save
```

Ý tưởng là:

```
iptables-persistent
→ persist firewall cơ bản
→ đặc biệt INPUT UDP 51820

wg-quick
→ tự add/remove FORWARD + SNAT
```

Đừng persist các rule động của wg0.

---

# 8. Start WireGuard server

```
sudo systemctl enable --now wg-quick@wg0
```

Check:

```
sudo wg
```

ban đầu sẽ thấy:

```
interface: wg0
public key: ...
private key: (hidden)
listening port: 51820
```

Check IP:

```
ip addr show wg0
```

phải có:

```
inet 10.66.66.1/24
```

Check port:

```
sudo ss -lunp | grep 51820
```

phải có:

```
0.0.0.0:51820
[::]:51820
```

---

# 9. Cài WireGuard trên Ubuntu client

Trên máy Ubuntu client:

```
sudo apt update
sudo apt install -y wireguard
```

Tạo thư mục:

```
sudo install -d -m 700 /etc/wireguard
```

Tạo client keypair:

```
sudo sh -c 'umask 077; wg genkey | tee /etc/wireguard/client.key | wg pubkey > /etc/wireguard/client.pub'
```

Xem client public key:

```
sudo cat /etc/wireguard/client.pub
```

Trong case hiện tại:

```
PCkniEJGpYTVePnSy6MbUEma3YqCe/xNeSh3KPRCBBQ=
```

Copy **public key** này vào server `[Peer]`.

Không share:

```
/etc/wireguard/client.key
```

---

# 10. Test tunnel trước bằng split tunnel

Ở lần đầu tiên, **đừng dùng ngay `0.0.0.0/0`**.

Dùng:

```
AllowedIPs = 10.66.66.0/24
```

trước.

Lý do: nếu WG chưa hoạt động mà để:

```
AllowedIPs = 0.0.0.0/0
```

thì toàn bộ IPv4 client bị nhét vào một tunnel chết, kể cả SSH tới VPS.

Đó là lý do lúc đầu bạn bật WG là:

```
ssh 161.118.252.60
→ timeout
```

Cấu hình test client:

```
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.66.66.2/24

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = 161.118.197.192:51820
AllowedIPs = 10.66.66.0/24
PersistentKeepalive = 25
```

Set quyền:

```
sudo chmod 600 /etc/wireguard/wg0.conf
```

Bật:

```
sudo wg-quick up wg0
```

Test:

```
ping -c 3 10.66.66.1
```

phải trả lời:

```
64 bytes from 10.66.66.1
```

Check:

```
sudo wg
```

phải có:

```
latest handshake: ... seconds ago
transfer: ... received, ... sent
```

---

# 11. Debug nếu không handshake

Đây chính xác là quy trình mình đã dùng.

Trên server:

```
sudo tcpdump -ni enp0s6 udp port 51820
```

Khi client bật WG, mình đã thấy:

```
58.187.x.x:xxxxx > 10.0.0.121:51820 UDP
```

Điều đó chứng minh:

```
Internet
→ OCI Security List
→ public IP
→ OCI NAT
→ VNIC
→ enp0s6
```

đều đã OK.

Nhưng `sudo wg` không có handshake.

Cuối cùng nguyên nhân là:

```
iptables INPUT
→ REJECT UDP 51820
```

sau khi thêm:

```
sudo iptables -I INPUT 5 -p udp --dport 51820 -j ACCEPT
```

thì handshake lên ngay.

---

# 12. Chuyển client sang full tunnel IPv4

Sau khi tunnel cơ bản hoạt động, đổi:

```
AllowedIPs = 10.66.66.0/24
```

thành:

```
AllowedIPs = 0.0.0.0/0
```

Restart:

```
sudo wg-quick down wg0
sudo wg-quick up wg0
```

Lúc đầu IPv4 Internet chưa chạy vì server FORWARD có:

```
1 REJECT
2 ACCEPT wg0 -> enp0s6
```

tức rule WireGuard nằm sai vị trí.

Đó là lý do mình sửa server config từ:

```
iptables -A FORWARD
```

thành:

```
iptables -I FORWARD 1
```

Sau khi sửa, server phải có:

```
sudo iptables -L FORWARD -n -v --line-numbers
```

kết quả kiểu:

```
1 ACCEPT enp0s6 -> wg0 RELATED,ESTABLISHED
2 ACCEPT wg0 -> enp0s6
3 REJECT everything else
```

NAT:

```
sudo iptables -t nat -L POSTROUTING -n -v --line-numbers
```

phải có:

```
SNAT
source 10.66.66.0/24
out enp0s6
to:10.0.0.121
```

---

# 13. Test full IPv4 VPN

Trên client:

```
ping -4 -c 3 1.1.1.1
```

phải OK.

Sau đó:

```
curl -4 https://ifconfig.me
```

kết quả:

```
161.118.197.192
```

Test thêm:

```
curl -4 https://api.ipify.org
echo
```

cũng:

```
161.118.197.192
```

Trong khi trên VPS:

```
curl -4 https://ifconfig.me
```

vẫn:

```
161.118.252.60
```

Đây là mục tiêu quan trọng nhất:

```
VPS own traffic
→ 161.118.252.60

WireGuard client
→ 161.118.197.192
```

---

# 14. Vấn đề IPv6 leak

Client lúc đó chỉ có:

```
AllowedIPs = 0.0.0.0/0
```

nên chỉ IPv4 vào WG.

IPv6 vẫn đi thẳng:

```
client
→ ISP Việt Nam
→ Internet
```

Ví dụ trước đó:

```
curl -6 https://ifconfig.me
```

hoặc:

```
ping google.com
```

có thể dùng IPv6 kiểu:

```
2405:....
```

Đó là IPv6 leak.

Vì mình chưa tunnel IPv6 qua server nên giải pháp hiện tại là:

> Khi WireGuard bật → block toàn bộ IPv6 outbound trên client.

Check có ip6tables:

```
command -v ip6tables
```

máy bạn:

```
/usr/sbin/ip6tables
```

---

# 15. Vấn đề DNS leak

Mục tiêu:

```
application
↓
systemd-resolved
↓
1.1.1.1
↓
IPv4
↓
wg0
↓
VPS
↓
Internet
```

Không muốn:

```
DNS
→ wlo1
→ router/ISP Việt Nam
```

Ban đầu mình thử:

```
resolvectl default-route wg0 yes
```

nhưng máy client báo:

```
Failed to set default route configuration: Access denied
```

=> bỏ dòng đó.

Mình cũng từng thử:

```
resolvectl revert wg0
```

trong `PostDown`, nhưng nó gây:

```
Failed to resolve interface "wg0": No such device
```

hoặc:

```
Failed to revert interface configuration: Access denied
```

=> **bỏ luôn `resolvectl revert`**.

Khi wg0 bị xóa thì per-link DNS config cũng biến mất theo.

---

# 16. Cấu hình client cuối cùng

Đây là **client `wg0.conf` cuối cùng đang chạy ngon**.

```
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.66.66.2/24

PostUp = resolvectl dns %i 1.1.1.1
PostUp = resolvectl domain %i '~.'

PostUp = ip6tables -C OUTPUT ! -o lo -j REJECT 2>/dev/null || ip6tables -I OUTPUT 1 ! -o lo -j REJECT

PreDown = ip6tables -D OUTPUT ! -o lo -j REJECT 2>/dev/null || true

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = 161.118.197.192:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Với máy bạn:

```
SERVER_PUBLIC_KEY =
KRS879mCwoGgCIZeo/Aq26WP44yUfbSQ4xar6bIKpzQ=
```

Không copy private key từ guide. Phải giữ private key thật hiện có trong file của bạn.

### Tại sao IPv6 rule viết hơi dài?

Thay vì:

```
ip6tables -I OUTPUT 1 ! -o lo -j REJECT
```

mỗi lần up, mình dùng:

```
ip6tables -C OUTPUT ! -o lo -j REJECT 2>/dev/null || \
ip6tables -I OUTPUT 1 ! -o lo -j REJECT
```

Nghĩa là:

```
rule đã tồn tại
→ không add thêm

rule chưa tồn tại
→ add
```

tránh duplicate.

`PreDown`:

```
ip6tables -D OUTPUT ! -o lo -j REJECT 2>/dev/null || true
```

nghĩa là nếu rule không tồn tại thì cũng **không làm wg-quick down fail giữa chừng**.

---

# 17. Test DNS

Sau khi:

```
sudo wg-quick up wg0
```

chạy:

```
resolvectl status wg0
```

máy bạn hiện đúng:

```
Current Scopes: DNS
Current DNS Server: 1.1.1.1
DNS Servers: 1.1.1.1
DNS Domain: ~.
Default Route: yes
```

Check query:

```
resolvectl query google.com
```

phải thấy:

```
-- link: wg0
```

Máy bạn hiện đã ra đúng:

```
google.com: ...
-- link: wg0
```

Đây mới là cái chứng minh DNS không còn đi qua `wlo1`.

DNS vẫn có thể trả về cả:

```
142.x.x.x
```

và:

```
2404:....
```

Không sao.

`2404:...` chỉ là **DNS trả về bản ghi AAAA của Google**.

Nó không có nghĩa IPv6 đã leak.

---

# 18. Test IPv6 block

```
curl -6 --max-time 5 https://ifconfig.me
```

phải fail:

```
Failed to connect...
```

Check rule:

```
sudo ip6tables -L OUTPUT -n -v --line-numbers
```

phải chỉ có một rule WireGuard:

```
REJECT ... !lo ... ::/0 ::/0
```

Trước đó bạn từng có 2 rule duplicate vì `PostDown` fail.

Nếu cần dọn:

```
while sudo ip6tables -C OUTPUT ! -o lo -j REJECT 2>/dev/null; do
  sudo ip6tables -D OUTPUT ! -o lo -j REJECT
done
```

sau đó để `wg-quick up` tự add lại đúng một rule.

---

# 19. SSH khi full tunnel đang bật

Một vấn đề mình gặp:

```
AllowedIPs = 0.0.0.0/0
```

có nghĩa là mọi IPv4 client đều vào WG.

Bao gồm cả:

```
161.118.252.60:22
```

nên SSH tới public IP VPS có thể timeout.

Không có nghĩa VPS chết.

Khi VPN đang bật, SSH sạch nhất bằng:

```
ssh ubuntu@10.66.66.1
```

Flow:

```
client
↓ WireGuard
10.66.66.1
↓
sshd trên VPS
```

Khi WG tắt:

```
sudo wg-quick down wg0
```

thì có thể SSH public lại:

```
ssh ubuntu@161.118.252.60
```

Nếu muốn thì sau này còn có thể tạo route exception để `161.118.252.60/32` luôn đi ngoài VPN.

---

# 20. Auto-start WireGuard server

Server:

```
sudo systemctl enable --now wg-quick@wg0
```

Check:

```
systemctl is-enabled wg-quick@wg0
```

phải:

```
enabled
```

---

# 21. Auto-start WireGuard client

Client:

```
sudo systemctl enable wg-quick@wg0
```

Bạn đã check:

```
systemctl is-enabled wg-quick@wg0
```

và nhận:

```
enabled
```

=> reboot client là WG tự lên.

---

# 22. Bộ test cuối cùng

## Server

```
sudo wg
```

muốn thấy client peer + handshake.

```
ip addr show wg0
```

muốn thấy:

```
10.66.66.1/24
```

```
sysctl net.ipv4.ip_forward
```

phải:

```
net.ipv4.ip_forward = 1
```

```
sudo iptables -L INPUT -n -v --line-numbers
```

phải có:

```
ACCEPT udp dpt:51820
```

trước REJECT.

```
sudo iptables -L FORWARD -n -v --line-numbers
```

phải kiểu:

```
1 ACCEPT enp0s6 -> wg0 RELATED,ESTABLISHED
2 ACCEPT wg0 -> enp0s6
3 REJECT
```

```
sudo iptables -t nat -L POSTROUTING -n -v --line-numbers
```

phải có:

```
10.66.66.0/24
→ SNAT
→ 10.0.0.121
```

Server Internet:

```
curl -4 https://ifconfig.me
```

phải:

```
161.118.252.60
```

---

## Client

```
sudo wg
```

muốn thấy:

```
endpoint: 161.118.197.192:51820
allowed ips: 0.0.0.0/0
latest handshake: ... seconds ago
transfer: ... received, ... sent
```

Tunnel:

```
ping -c 3 10.66.66.1
```

phải OK.

IPv4 Internet:

```
ping -4 -c 3 1.1.1.1
```

phải OK.

Public IPv4:

```
curl -4 https://ifconfig.me
```

phải:

```
161.118.197.192
```

Test bằng service khác:

```
curl -4 https://api.ipify.org
echo
```

phải:

```
161.118.197.192
```

IPv6:

```
curl -6 --max-time 5 https://ifconfig.me
```

phải fail.

DNS:

```
resolvectl status wg0
```

phải có:

```
DNS Servers: 1.1.1.1
DNS Domain: ~.
```

và:

```
resolvectl query google.com
```

phải:

```
-- link: wg0
```

---

# 23. Hai file final quan trọng nhất

## VPS — `/etc/wireguard/wg0.conf`

```
[Interface]
Address = 10.66.66.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

PostUp = iptables -I FORWARD 1 -i wg0 -o enp0s6 -j ACCEPT; iptables -I FORWARD 1 -i enp0s6 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -A POSTROUTING -s 10.66.66.0/24 -o enp0s6 -j SNAT --to-source 10.0.0.121

PostDown = iptables -D FORWARD -i wg0 -o enp0s6 -j ACCEPT; iptables -D FORWARD -i enp0s6 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -D POSTROUTING -s 10.66.66.0/24 -o enp0s6 -j SNAT --to-source 10.0.0.121

[Peer]
PublicKey = PCkniEJGpYTVePnSy6MbUEma3YqCe/xNeSh3KPRCBBQ=
AllowedIPs = 10.66.66.2/32
```

## Client — `/etc/wireguard/wg0.conf`

```
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.66.66.2/24

PostUp = resolvectl dns %i 1.1.1.1
PostUp = resolvectl domain %i '~.'

PostUp = ip6tables -C OUTPUT ! -o lo -j REJECT 2>/dev/null || ip6tables -I OUTPUT 1 ! -o lo -j REJECT

PreDown = ip6tables -D OUTPUT ! -o lo -j REJECT 2>/dev/null || true

[Peer]
PublicKey = KRS879mCwoGgCIZeo/Aq26WP44yUfbSQ4xar6bIKpzQ=
Endpoint = 161.118.197.192:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Và server forwarding:

```
/etc/sysctl.d/99-wireguard.conf
```

```
net.ipv4.ip_forward=1
```

---

# 24. Kiến trúc cuối cùng của hệ thống

```
                   UBUNTU CLIENT
                   10.66.66.2
                        │
               IPv4 + DNS only
                        │
                 encrypted WG
                        │
                        ▼
              161.118.197.192:51820
                        │
                    OCI NAT
                        │
                        ▼
                   10.0.0.121
                        │
                ┌───────┴────────┐
                │   Oracle VPS   │
                │                │
                │ wg0            │
                │ 10.66.66.1     │
                │                │
                └───────┬────────┘
                        │
              SNAT → 10.0.0.121
                        │
                        ▼
                     Internet
                        │
                        ▼
              sees 161.118.197.192
```

Trong khi traffic riêng của VPS:

```
VPS
↓
10.0.0.199
↓
OCI NAT
↓
161.118.252.60
```

Và phía client:

```
IPv4
→ WireGuard
→ Singapore

DNS
→ 1.1.1.1
→ wg0
→ Singapore

IPv6
→ REJECT
→ không leak
```

Đây là trạng thái cuối cùng mà chúng ta đã test thực tế bằng handshake, ping, `ifconfig.me`, `ipify`, `resolvectl` và `ip6tables`.