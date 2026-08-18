Được. Hiểu 3 cái này theo kiểu **“package nằm ở đâu + lúc chạy Node tìm nó kiểu gì”** là dễ nhất.

## 1. `npm install <package>` — cài local vào project

Ví dụ:

```
mkdir demo
cd demo
npm init -y

npm install axios
```

Sau đó thư mục sẽ kiểu:

```
demo/
├── package.json
├── package-lock.json
└── node_modules/
    └── axios/
```

`axios` được cài vào:

```
./node_modules/axios
```

và `package.json` được thêm:

```
{
  "dependencies": {
    "axios": "^1.x.x"
  }
}
```

Flow:

```
npm install axios
        │
        ├─ đọc npm registry
        │
        ├─ download package
        │
        ├─ giữ/download thông qua npm cache
        │
        ├─ giải dependency tree
        │
        └─ đặt package vào:
             project/node_modules/
```

### npm cache ở đâu?

Thường trên Linux:

```
npm config get cache
```

ra kiểu:

```
/home/thaihoc/.npm
```

Nhưng cần phân biệt:

```
~/.npm/
   = cache/download metadata

project/node_modules/
   = package thực tế mà project đang sử dụng
```

Cache không phải nơi bạn normalmente `require()` package từ đó.

---

# 2. `npm install -g <package>` — cài global

Ví dụ:

```
npm install -g typescript
```

`-g` = `--global`.

Package không vào:

```
./node_modules
```

mà vào **global npm prefix**.

Xem bằng:

```
npm prefix -g
```

Ví dụ:

```
/usr/local
```

thì package thường nằm đâu đó kiểu:



```
/usr/local/lib/node_modules/typescript/
```

và executable được expose tại:

```
/usr/local/bin/tsc
```

Nên sau đó ở bất kỳ thư mục nào:

```
tsc
```

đều chạy được, vì `/usr/local/bin` nằm trong $PATH.
Có thể hình dung:

```
npm install -g typescript
          │
          ▼

/usr/local/lib/node_modules/
└── typescript/
       └── ...

/usr/local/bin/
└── tsc -> symlink tới typescript/bin/tsc
```

### Điểm quan trọng

Global install chủ yếu hợp với **CLI tools**:

```
npm install -g prettier
npm install -g typescript
npm install -g @deepseek-ai/dsh
```

Không nên dùng global package làm dependency bình thường cho app Node.

Ví dụ app có:

```
import axios from "axios";
```

thì Node thông thường tìm:

```
./node_modules/axios
```

chứ không tự đi kiếm:

```
/usr/local/lib/node_modules/axios
```

---

# 3. `npx <package>` — chạy package mà không cần bạn cài global

Đây là phần hay bị hiểu nhầm nhất.

Ví dụ:

```
npx cowsay hello
```

`npx` về bản chất nói:
> “Tìm executable tên này. Nếu chưa có package phù hợp, lấy package đó về tạm/cache rồi chạy binary của nó.”

Flow đại khái:

```
npx @deepseek-ai/dsh web
           │
           ▼
1. npm/npx resolve package
           │
           ▼
2. package chưa available?
      ├─ có sẵn → dùng luôn
      └─ chưa → tải
           │
           ▼
3. đặt package vào npm exec cache
           │
           ▼
4. thêm thư mục .bin của package đó vào PATH tạm
           │
           ▼
5. chạy:
   dsh web
```

Bạn thấy prompt:

```
Need to install the following packages:
@deepseek-ai/dsh@0.1.0-rc.7
Ok to proceed? (y)
```

chính là bước này.

---

# Một ví dụ cụ thể với `dsh`

Bạn chạy:

```
npx @deepseek-ai/dsh web
```

Nó **không** tạo:

```
~/node_modules/@deepseek-ai/dsh
```

và cũng không nhất thiết tạo global:

```
/usr/local/lib/node_modules/@deepseek-ai/dsh
```

Thay vào đó npm dùng cache của nó, thường dưới:

```
~/.npm/
```

Có thể thấy dạng:

```
~/.npm/_npx/<hash>/
```

Ví dụ:

```
~/.npm/_npx/8f123abcd/
├── node_modules/
│   └── @deepseek-ai/
│       └── dsh/
└── package.json
```

Sau đó trong process đó, nó tạm thời làm giống:

```
PATH=
~/.npm/_npx/8f123abcd/node_modules/.bin
:
$PATH
```

Thế nên câu:

```
dsh web
```

ở bên trong process của `npx` được resolve thành:

```
~/.npm/_npx/.../node_modules/.bin/dsh
```

Rồi chương trình chạy.

Khi process kết thúc, việc **thêm PATH tạm thời** biến mất.

Package cache thì có thể vẫn còn.

---

# So sánh trực tiếp

Giả sử package tên:

```
foo
```

và nó cung cấp CLI:

```
foo
```

### Local

```
npm install foo
```

sẽ có:

```
my-project/
├── node_modules/
│   ├── foo/
│   └── .bin/
│       └── foo
└── package.json
```

Bạn thường chạy:

```
npx foo
```

hoặc trong npm script:

```
{
  "scripts": {
    "start": "foo"
  }
}
```

---

### Global

```
npm install -g foo
```

sẽ kiểu:

```
/usr/local/lib/node_modules/foo
/usr/local/bin/foo
```

rồi:

```
foo
```

ở đâu cũng được.

---

### NPX trực tiếp

```
npx foo
```

nếu project không có `foo`, nó có thể lấy về cache:

```
~/.npm/_npx/<hash>/node_modules/foo
```

sau đó chạy binary.

Không cần:

```
npm install -g foo
```

---

# Vậy `npx` có “cài” không?

**Có**, nhưng khác nghĩa với `npm install`.

`npm install foo`:

```
Tôi muốn project này phụ thuộc vào foo.
```

Nó ghi dependency vào project.

`npm install -g foo`:

```
Tôi muốn máy/user này có CLI foo dùng mọi nơi.
```

`npx foo`:

```
Tôi chỉ muốn chạy foo.
Tự tìm/cài cần thiết cho tôi.
```

Đấy là khác biệt về **intent**.

---

# Một điểm cực quan trọng: `node_modules/.bin`

Giả sử package:

```
@deepseek-ai/dsh
```

trong `package.json` của package đó khai báo kiểu:

```
{
  "bin": {
    "dsh": "./dist/cli.js"
  }
}
```

npm khi cài package sẽ tạo executable:

```
node_modules/.bin/dsh
```

trỏ tới:

```
@deepseek-ai/dsh/dist/cli.js
```

Nên:

```
npx @deepseek-ai/dsh web
```

cuối cùng về bản chất là tìm cái `bin` đó rồi chạy:

```
dsh web
```

---

# Vậy tại sao local package cũng chạy được bằng `npx`?

Giả sử:

```
npm install prettier
```

project:

```
project/
└── node_modules/
    ├── prettier/
    └── .bin/
        └── prettier
```

Bây giờ:

```
npx prettier .
```

`npx` sẽ ưu tiên tìm package/binary có sẵn ở project trước.

Nên nó dùng:

```
./node_modules/.bin/prettier
```

chứ **không cần tải Prettier mới về**.

Đây là một lý do `npx` rất tiện.

---

# npm cache đóng vai trò gì?

Ví dụ lần đầu:

```
npx @deepseek-ai/dsh web
```

có thể là:

```
Internet
   │
   ▼
npm registry
   │
   ▼
~/.npm cache
   │
   ▼
~/.npm/_npx/<hash>/node_modules/@deepseek-ai/dsh
   │
   ▼
dsh
```

Lần sau:

```
npx @deepseek-ai/dsh web
```

npm có thể tận dụng dữ liệu đã cache, nên không nhất thiết download tất cả từ đầu.

Nhưng **npm cache không phải database của DSH**.

Đây là hai thế giới riêng:

```
                  ┌── npm cache
                  │   chương trình/package
npx dsh ──────────┤
                  │
                  └── DSH application data
                      history / config / sessions / API keys...
```

Nếu DSH lưu data vào:

```
~/.config/...
```

hay:

```
~/.local/share/...
```

thì dù bạn:

```
npx dsh
```

hay:

```
npm install -g dsh
dsh
```

nó vẫn có thể đọc cùng data đó.

---

# Vậy `npm` thực sự là gì?

`npm` có thể hiểu là 3 thứ gắn với nhau:

```
npm
├── registry
├── CLI
└── package ecosystem
```

Registry:

```
https://registry.npmjs.org
```

là kho package.

Ví dụ:

```
axios
react
typescript
@deepseek-ai/dsh
```

CLI `npm` là chương trình bạn chạy:

```
npm install
npm uninstall
npm update
npm publish
```

Nó giao tiếp với registry + quản lý dependency.

---

# `npm install` không ghi version cố định à?

Ví dụ:

```
npm install axios
```

`package.json` có thể:

```
"axios": "^1.12.0"
```

Nhưng `package-lock.json` ghi version resolve chính xác hơn:

```
axios 1.12.0
dependency A 2.3.1
dependency B 4.7.2
...
```

Nên:

```
package.json
= project muốn dependency nào

package-lock.json
= dependency tree thực tế đã resolve

node_modules
= files thực sự đang cài trên disk
```

Ba thứ này nên phân biệt.

---

# Ví dụ toàn flow `npm install`

```
npm install express
```

Flow:

```
package.json
     │
     │ npm thấy cần express
     ▼
npm registry
     │
     │ metadata + package tarball
     ▼
npm cache ~/.npm
     │
     │ resolve dependencies
     ▼
package-lock.json
     │
     ▼
node_modules/
├── express/
├── body-parser/
├── router/
├── ...
└── ...
```

chạy app lên tôi mới test được chứ 

Node app:

```
import express from "express";
```

sau đó module resolver tìm lên cây thư mục:

```
current directory/node_modules
parent directory/node_modules
parent parent/node_modules
...
```

---

# Còn global install thì flow

```
npm install -g @deepseek-ai/dsh
```

```
npm registry
     │
     ▼
npm cache
     │
     ▼
global node_modules
     │
     └── @deepseek-ai/dsh
                │
                │ "bin": ...
                ▼
global bin/dsh
```

Shell:

```
dsh web
```

resolve:

```
$PATH
 │
 ├─ /usr/local/bin
 │       └── dsh
 │
 ▼
DeepSeek Harness
```

---

# NPX thì flow

```
npx @deepseek-ai/dsh web
```

```
Does current project already have it?
          │
      ┌───┴────┐
     yes       no
      │         │
      │         ▼
      │     npm cache /
      │     npx environment
      │         │
      └────┬────┘
           ▼
find package's "bin"
           │
           ▼
temporarily modify PATH
           │
           ▼
dsh web
```

---

## Với trường hợp của bạn

Nếu chỉ muốn **thử DSH**:

```
npx @deepseek-ai/dsh web
```

rất hợp lý.

Nếu dùng DSH hàng ngày và muốn đơn giản:

```
npm install -g @deepseek-ai/dsh
```

rồi:

```
dsh web
```

Nhưng có một lợi thế khá lớn khi tiếp tục dùng `npx`:

```
npx @deepseek-ai/dsh@0.1.0-rc.7 web
```

Bạn có thể **pin chính xác version** rất dễ.

Ví dụ test bản mới:

```
npx @deepseek-ai/dsh@0.1.0-rc.8 web
```

không cần thay package global trên máy.

### Tóm lại bằng một sơ đồ

```
                    npm registry
                         │
                         ▼
                     npm cache
                     ~/.npm
                    /       \
                   /         \
                  ▼           ▼
 npm install foo             npx foo
       │                       │
       ▼                       ▼
project/node_modules      temporary/cache
       │                       │
       ▼                       ▼
project dùng foo           chạy foo ngay


npm install -g foo
       │
       ▼
global node_modules
       │
       ▼
global bin/foo
       │
       ▼
foo chạy được từ mọi nơi
```

Câu ngắn nhất để nhớ là:

> **`npm install` = cài dependency cho project.  
> `npm install -g` = cài command/tool cho toàn user/máy.  
> `npx` = chạy command từ npm package, có thể tự lấy package về nếu cần.**