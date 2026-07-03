## 1. Key của server — để client xác minh “đây đúng là server”

SSH server có **host private key** và **host public key**:

```text
SSH server:
- /etc/ssh/ssh_host_ed25519_key       ← private key của server
- /etc/ssh/ssh_host_ed25519_key.pub   ← public key của server
```

Khi client kết nối lần đầu, client nhận public key/fingerprint của server và lưu vào:

```text
~/.ssh/known_hosts
```

Phần này đúng với ý bạn:

```text
Server giữ private key
Client biết public key của server
```

Mục đích là chống việc bạn kết nối nhầm server giả mạo.

---

## 2. Key của user/client — để server xác minh “client này được phép đăng nhập”

Đăng nhập không cần password dùng một cặp key khác:

```text
SSH client:
- id_ed25519       ← private key của client
- id_ed25519.pub   ← public key của client
```

Public key của client được copy lên server vào:

```text
~/.ssh/authorized_keys
```

Luồng này là:

```text
Client giữ private key
Server giữ public key của client
```

Mục đích là server kiểm tra client có thật sự sở hữu private key tương ứng hay không.

---

## Toàn bộ mô hình

```text
SSH CLIENT                              SSH SERVER
────────────────────────────────────────────────────────

Client private key                     Client public key
~/.ssh/id_ed25519        ───────────▶   ~/.ssh/authorized_keys
Dùng để đăng nhập                       Dùng để kiểm tra client


Server public key         ◀───────────  Server private key
~/.ssh/known_hosts                     /etc/ssh/ssh_host_*_key
Dùng để nhận diện server                Dùng chứng minh server thật
```

Vì thế cả client và server đều có thể có private/public key, nhưng chúng thuộc **hai cặp key khác nhau**:

- **Host key**: server giữ private, client lưu public.
    
- **User authentication key**: client giữ private, server lưu public.
    
