
![[Pasted image 20250728114157.png]]


![[Pasted image 20250728114248.png]]



![[Pasted image 20250728114305.png]]




![[Pasted image 20250728114333.png]]



don't need link in version 2 because docker compose will create a new host network. depends_on means these container must run before .



![[Pasted image 20250728114956.png]]



some compose command: 

- `docker compose up` - Start services
- `docker compose up -d` - Start in detached mode
- `docker compose down` - Stop and remove containers
- `docker compose ps` - List running services
- `docker compose logs` - View service logs
- `docker compose build` - Build or rebuild services


.env : 
``` env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=my-super-secret-password
POSTGRES_DB=myapp
PORT=8080
SECRET_MESSAGE=Hello Docker!
```


docker-compose.yaml
```yaml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    env_file:
      - .env
    environment:
      - PORT=8080
    volumes:
      - mydata:/data
    networks:
      - myapp-network
    depends_on:
      - db
  db:
    image: postgres
    ports:
      - "5432:5432"
    env_file:
      - .env
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - myapp-network
  nginx:
    image: nginx
    ports:
      - "80:80"
    networks:
      - myapp-network
    depends_on:
      - api
volumes:
  mydata:
  postgres_data:
networks:
  myapp-network:
```


## Healthchecks

Docker Compose provides a healthcheck feature to check ==if a service is healthy, and to wait for a service to be healthy before starting another service if it depends on it.==

```yaml
services:
  api:
    build: .
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

Here we've added:

- `healthcheck`: Defines how Docker should check if a service is healthy
- `depends_on` with `condition`: Waits for the database to be healthy before starting the API

This ensures our API only starts when the database is ready and your application is healthy.

## Compose Watch

Docker Compose Watch is a fantastic feature for development. It automatically syncs your code changes into containers and can trigger rebuilds:  

``` yaml
services:
  api:
    build: .
    ports:
      - "8081:8081"
    environment:
      - PORT=8081
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    develop:
      watch:
        - path: ./
          target: /go
          action: rebuild
```

With this configuration:

1. Any changes to the `./` directory are synced instantly into the /go in container and trigger a rebuild

Start your development environment with:  

```
docker compose watch
```

## Profiles

Profiles help you manage different service combinations. For example, you might want development tools only in development:  

```
services:
  api:
    build: .
  db:
    image: postgres
  adminer:
    image: adminer
    profiles:
      - dev
    ports:
      - "9000:8080"
```

Now you can:

- Start normal services: `docker compose up`
- Include dev tools: `docker compose --profile dev up`

## Best Practices

1. **Use health checks** for critical services
2. **Never store secrets** in your compose file
    - Use secret files
    - Or use environment variables
3. **Use profiles** to separate development and production services
4. **Leverage watch** for faster development
5. **Document dependencies** using `depends_on`




volumes:
  superset_home:
    external: false
  db_home:
    external: false
  redis:
    external: false

## ✅ `external: false` là gì?

### Khi bạn khai báo:

yaml

CopyEdit

`volumes:   superset_home:     external: false`

Điều đó có nghĩa là:

> "Tôi muốn Docker **tự tạo volume này khi chạy Compose**, nếu nó chưa tồn tại."

- Docker sẽ tạo một volume mới với tên `projectname_superset_home` (nếu bạn không dùng tên riêng).
    
- Volume sẽ được gắn vào đúng container khi bạn chạy `docker-compose up`.
    

🔁 Đây là cách **tự động và phổ biến** nhất khi bạn chỉ dùng Docker Compose trong dự án của mình.

---

## 🔗 `external: true` là gì?

Ngược lại, nếu bạn viết:

yaml

CopyEdit

`volumes:   superset_home:     external: true`

Thì bạn đang nói với Docker:

> "Tôi sẽ **sử dụng một volume đã tồn tại trước đó**. Đừng tự tạo mới!"

👉 Nếu volume tên `superset_home` chưa tồn tại, Docker sẽ báo lỗi:

pgsql

CopyEdit

`ERROR: Volume superset_home declared as external, but could not be found.`

📦 Điều này hữu ích khi bạn:

- Muốn dùng lại một volume được tạo bằng lệnh `docker volume create superset_home`
    
- Hoặc bạn quản lý volumes bên ngoài (CI/CD, backup hệ thống,...)


Some docker compose trick
## 1. Variable Substitution

Tired of tweaking your docker-compose.yml for every environment? Enter variable substitution:  

```
services:
  web:
    image: nginx
    ports:
      - "${PORT}:80"
```

Set `PORT=8080` in your environment, and voilà! Your configuration adjusts without a single edit.

### Cách 1: Đặt trực tiếp khi chạy

`PORT=8080 docker-compose up`

### Cách 2: Đặt trong `.env` file (phổ biến)

Tạo một file tên `.env` trong cùng thư mục:

`PORT=8080`

> Docker Compose sẽ tự động đọc file `.env` này.


## 2. Profiles

Not all services need to run all the time. Profiles let you pick and choose:  

```
services:
  web:
    image: nginx
    profiles:
      - web
  db:
    image: postgres
    profiles:
      - db
```

Start just the services you need with:  

```
docker-compose --profile web up
```

A lot more fun if you have a lot of services than defining them separately in your `docker compose up` command!

## 3. Extends

Why repeat yourself when you can extend? Here's how:  
base.yml
```
services:
  common:
    image: nginx
```

Then in your docker-compose.yml:  

```
services:
  web:
    extends:
      file: base.yml
      service: common
    ports:
      - "80:80"
```

Remember DRY (Don't Repeat Yourself)? Turns out that also works for docker compose :D


## 4. Merge Compose Files

Mix different Docker Compose files like you're crafting a cocktail (or a potion if you're becoming a Docker Wizard):  
docker-compose.yml
```
services:
  web:
    image: nginx
```

Add some zest with:  
docker-compose.prod.yml
```
services:
  web:
    ports:
      - "8080:80"
```

Run `docker-compose up`, and they merge seamlessly. Or, blend for a specific taste:  

```
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```

📌 Các file này sẽ được **gộp (merge)** lại theo thứ tự từ trái → phải:

> Các giá trị ở **file sau sẽ ghi đè** hoặc bổ sung lên file trước nếu trùng tên service.



## 5. Dry Run Mode

Before you dive headfirst into changes (and potentially a brick wall, aka a production outage), check them out in dry run mode:  

```
docker-compose --dry-run up
```

This lets you preview the actions Docker Compose would take without actually performing them. 