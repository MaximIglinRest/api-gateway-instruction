## Предварительная настройка

---
### Подключимся к стенду по ssh
```shell
ssh root@176.114.67.25
```
### Установим Docker и Git
```shell
sudo apt-get update
sudo apt-get install git

sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o
/etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture)
signed-by=/etc/apt/keyrings/docker.asc]
https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
 sudo apt-get install docker-ce docker-ce-cli containerd.io
docker-buildx-plugin docker-compose-plugin
```

## Сервисы на FastApi
### Зависимости:
```shell
pip install fastapi uvicorn
```

### Delivery-service
#### приложение
```python
from fastapi import FastAPI


app = FastAPI()


@app.get("/health_check")
async def heath_check():
    return {"status": "Delivery service started."}

```
#### Dockerfile
```yaml
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY src .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001", "--workers", "4"]
```
#### docker-compose.yaml
```yaml
version: '3.8'
services:
  web:
    container_name: delivery_service
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8001:8001"
    environment:
      - ENV=dev
    restart: always
    networks:
      - app-network
networks:
  app-network:
    external: true
```
#### gitlab-ci.yml
```yaml
stages:
  - deploy
deploy:
  stage: deploy
  script:
    - echo "Deploying app to remote server..."
    - docker compose up -d --build
```

1. Теперь вернемся в gitlab и сделаем еще пару шагов для настройки CI/CD
2. Создадим runner, который будет работать на стенде и запускать джобы нашего
пайплайна и добавим этот runner в репозитории
3. Открываем settings → CI/CD → New project runner, ставим галочку Run
untagged jobs → create runner
4. Возвращаемся на server и запускаем runner на стенде без dind и прочего, но
не забываем выдать ему права к Docker
```shell
curl -LJO
"https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/la
test/deb/gitlab-runner_amd64.deb"
 dpkg -i gitlab-runner_amd64.deb
 gitlab-runner register   --url https://gitlab.com   --token
glrt-t3_UN3rAqKQy3nySTdayMwc
Теперь выбираем shell, вся сборка будет происходить прямо в оболочке.
 sudo usermod -aG docker gitlab-runner
 sudo systemctl restart gitlab-runner
```
При сборке стрельнула ошибка — докер не увидел нашу сеть. Зайдем на стенд и создадим ее.

```shell
 docker network create app-network
```

Зайдем на стенд и посмотрим активные контейнеры, а также попробуем достучаться до нашего приложения по сети.
http://176.114.67.25:8001/docs — все работает 

### Users-service
#### приложение:
В приложении меняем только порт и в Dockerfile
```shell
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY src .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000",
"--workers", "4"]
```
 
В docker-compose поменяем тот самый container_name и прокинутый наружу порт:
```yaml
 version: '3.8'
services:
  web:
    container_name: users_service
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - ENV=production
    restart: always
    networks:
      - app-network
networks:
  app-network:
    external: true
```

http://176.114.67.25:8000/docs. Видим, что все работает отлично. 
## Разворачиваем KrakenD
kraken.json — файл конфигурации нашего gateway.
```json
{
  "version": 3,
  "name": "My API Gateway",
  "port": 8080,
  "cache_ttl": "300s",
  "endpoints": [
    {
      "endpoint": "/delivery/health_check/",
      "method": "GET",
      "backend": [
        {
          "url_pattern": "/health_check/",
          "host": [ "http://delivery_service:8001" ]
} ]
}, {
      "endpoint": "/users/health_check/",
      "method": "GET",
      "backend": [
        {
          "url_pattern": "/health_check/",
          "host": [ "http://users_service:8000" ]

  } ]
} ]
}
```
#### Dockerfile
```shell
FROM devopsfaith/krakend:latest
COPY krakend.json /etc/krakend/config/krakend.json
CMD [ "run", "-c", "/etc/krakend/config/krakend.json" ]
```
#### docker-compose

```yaml
version: '3.8'
services:
  krakend:
    container_name: krakend
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    networks:
      - app-network
networks:
  app-network:
    external: true
```
.gitlab-ci.yml остается без изменений, как в предыдущих сервисах.

## Nginx
src/nginx.conf
```
 server {
    listen 80;
    server_name 5.35.4.197;
    charset utf-8;
location / {
default_type text/plain; return 200 'Тестовый запуск 2';
}
# Проксирование запросов к FastAPI на /users location /api-gateway/ {
        rewrite ^/api-gateway/(.*)$ /$1 break;
        proxy_pass <http://krakend:8080>;
# Настройки заголовков проксирования proxy_set_header Host $host; proxy_set_header X-Real-IP $remote_addr;

          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Dockerfile
```shell
COPY src/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
## API-composition

```python
# users
@app.get("/users/{user_id}/")
async def get_user_data():
    return {"first_name": "Max.", "last_name": "Iglin"}


# delivery
@app.get("/users/{user_id}/")
async def get_user_deliveries():
    return {"deliveries": [{"id": 1, "date": "2024-01-01"}, {"id": 2, "date": "2024-12-01"}]
}
```

#### krakend.json
```json
{
"endpoint": "/user-delivery/{user}/",
"method": "GET",
"backend": [
  {
    "url_pattern": "/users/{user}/",
    "host": ["http://users_service:8000"]
},
  {
    "url_pattern": "/users/{user}/",
    "host": [
      "http://delivery_service:8001"
    ]
  }  
```

По endpoint http://176.114.67.25/api-gateway/user-delivery/1/ получаем данные из двух сервисов.
