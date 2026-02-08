# 🚀 Deployment с Kamal 2

> **Инструмент:** Kamal 2.3+  
> **Сервер:** VPS (Hetzner / DigitalOcean)  
> **OS:** Ubuntu 22.04 LTS

---

## 📋 Подготовка Сервера

### 1. Требования к VPS

**Минимальные характеристики для MVP:**
- **CPU:** 2 cores
- **RAM:** 4 GB
- **Disk:** 40 GB SSD
- **Bandwidth:** 2 TB/month
- **Cost:** ~$10-15/month (Hetzner CX21)

**Рекомендуется:**
- Hetzner Cloud (дешево, быстро, европейский DC)
- DigitalOcean (простота, хорошая документация)
- Vultr (альтернатива)

### 2. Initial Server Setup

```bash
# На локальной машине: SSH в сервер
ssh root@YOUR_SERVER_IP

# Обновляем систему
apt update && apt upgrade -y

# Устанавливаем Docker
curl -fsSL https://get.docker.com | sh

# Добавляем пользователя deploy
adduser deploy
usermod -aG sudo deploy
usermod -aG docker deploy

# Настраиваем SSH для deploy пользователя
mkdir -p /home/deploy/.ssh
cp ~/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys

# Отключаем root login (безопасность)
nano /etc/ssh/sshd_config
# Изменяем: PermitRootLogin no
systemctl restart sshd

# Выходим и логинимся как deploy
exit
ssh deploy@YOUR_SERVER_IP
```

---

## ⚙️ Kamal Configuration

### 1. Установка Kamal локально

```bash
# На локальной машине
gem install kamal

# Проверяем версию
kamal version
# Ожидаем: Kamal 2.3.0+
```

### 2. Инициализация Kamal в проекте

```bash
# В корне проекта
kamal init

# Это создаст:
# - config/deploy.yml
# - .kamal/secrets
```

### 3. Конфигурация `config/deploy.yml`

```yaml
# config/deploy.yml
service: industrialprofi
image: YOUR_DOCKERHUB_USERNAME/industrialprofi

servers:
  web:
    hosts:
      - YOUR_SERVER_IP
    labels:
      traefik.http.routers.industrialprofi.rule: Host(`industrialprofi.ru`)
      traefik.http.routers.industrialprofi.entrypoints: websecure
      traefik.http.routers.industrialprofi.tls.certresolver: letsencrypt

registry:
  username: YOUR_DOCKERHUB_USERNAME
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  clear:
    RAILS_ENV: production
    RAILS_LOG_LEVEL: info
  secret:
    - RAILS_MASTER_KEY

volumes:
  - "storage:/rails/storage"

asset_path: /rails/public/assets

ssh:
  user: deploy

builder:
  arch: amd64

accessories:
  litestream:
    image: litestream/litestream:0.3.13
    host: YOUR_SERVER_IP
    volumes:
      - "storage:/data"
    env:
      clear:
        LITESTREAM_ACCESS_KEY_ID: YOUR_S3_KEY
        LITESTREAM_SECRET_ACCESS_KEY: YOUR_S3_SECRET
      files:
        - config/litestream.yml:/etc/litestream.yml
    cmd: replicate

# Healthcheck
healthcheck:
  path: /up
  interval: 30s
  timeout: 10s
```

### 4. Секреты `.kamal/secrets`

```bash
# .kamal/secrets
#!/bin/sh

# Docker Hub credentials
export KAMAL_REGISTRY_PASSWORD="YOUR_DOCKERHUB_TOKEN"

# Rails Master Key (из config/master.key)
export RAILS_MASTER_KEY=$(cat config/master.key)

# S3 для Litestream backups (получить на AWS/Backblaze/Cloudflare R2)
export LITESTREAM_ACCESS_KEY_ID="YOUR_S3_KEY"
export LITESTREAM_SECRET_ACCESS_KEY="YOUR_S3_SECRET"
```

**Важно:** Добавить в `.gitignore`:
```
.kamal/secrets
config/master.key
```

---

## 🐳 Dockerfile

```dockerfile
# Dockerfile
FROM ruby:3.4.1-slim AS base

# Установка зависимостей
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      curl \
      libjemalloc2 \
      libvips \
      sqlite3 && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

WORKDIR /rails

# Set production environment
ENV RAILS_ENV=production \
    BUNDLE_DEPLOYMENT=1 \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="development:test"

# ───────────────────────────────────────
# Build stage
# ───────────────────────────────────────
FROM base AS build

# Установка build dependencies
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      build-essential \
      git \
      node-gyp \
      pkg-config \
      python-is-python3 && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Install Node.js и npm
ARG NODE_VERSION=20.11.0
ENV PATH=/usr/local/node/bin:$PATH
RUN curl -sL https://github.com/nodenv/node-build/archive/master.tar.gz | tar xz -C /tmp/ && \
    /tmp/node-build-master/bin/node-build "${NODE_VERSION}" /usr/local/node && \
    rm -rf /tmp/node-build-master

# Install gems
COPY Gemfile Gemfile.lock ./
RUN bundle install && \
    rm -rf ~/.bundle/ "${BUNDLE_PATH}"/ruby/*/cache "${BUNDLE_PATH}"/ruby/*/bundler/gems/*/.git

# Install node modules
COPY package.json package-lock.json ./
RUN npm install

# Copy application code
COPY . .

# Precompile assets (Vite)
RUN npm run build && \
    bundle exec rails assets:precompile

# ───────────────────────────────────────
# Final stage
# ───────────────────────────────────────
FROM base

# Copy built artifacts
COPY --from=build /usr/local/bundle /usr/local/bundle
COPY --from=build /rails /rails

# Create non-root user
RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash && \
    chown -R rails:rails /rails /rails/tmp /rails/storage

USER 1000:1000

# Entrypoint для миграций
ENTRYPOINT ["/rails/bin/docker-entrypoint"]

EXPOSE 3000
CMD ["./bin/thrust", "./bin/rails", "server"]
```

### `bin/docker-entrypoint`

```bash
#!/bin/bash
set -e

# Запускаем миграции при старте
bin/rails db:migrate 2>/dev/null || echo "Database not ready yet"

# Запускаем команду из CMD
exec "$@"
```

Сделать исполняемым:
```bash
chmod +x bin/docker-entrypoint
```

---

## 📦 Litestream (SQLite Backups)

### `config/litestream.yml`

```yaml
# config/litestream.yml
dbs:
  - path: /data/production.sqlite3
    replicas:
      - type: s3
        bucket: industrialprofi-backups
        path: database
        region: eu-central-1
        endpoint: https://YOUR_S3_ENDPOINT  # Для Backblaze B2 или Cloudflare R2
        access-key-id: ${LITESTREAM_ACCESS_KEY_ID}
        secret-access-key: ${LITESTREAM_SECRET_ACCESS_KEY}
        retention: 168h  # 7 дней
        sync-interval: 10s
```

**Альтернативы S3:**
- **Backblaze B2:** $0.005/GB/month (дешево!)
- **Cloudflare R2:** $0.015/GB/month (без egress fees)
- **AWS S3:** $0.023/GB/month

---

## 🚀 Первый Деплой

### 1. Подготовка

```bash
# Проверяем что все секреты на месте
source .kamal/secrets
echo $RAILS_MASTER_KEY  # Должен вывести ключ

# Проверяем SSH доступ
ssh deploy@YOUR_SERVER_IP
```

### 2. Setup сервера (первый раз)

```bash
# Устанавливает Traefik (reverse proxy), Docker registry access
kamal setup

# Это займет 2-3 минуты
```

### 3. Deploy приложения

```bash
# Билдит Docker image, пушит в DockerHub, деплоит на сервер
kamal deploy

# При первом деплое это займет 5-10 минут
```

### 4. Запуск seeds (только первый раз!)

```bash
# SSH в контейнер и запускаем seeds
kamal app exec 'bin/rails db:seed'
```

### 5. Проверка

```bash
# Открываем в браузере
https://YOUR_SERVER_IP

# Или если домен уже настроен:
https://industrialprofi.ru
```

---

## 🔄 Последующие Деплои

```bash
# После изменений в коде
git push origin main  # Пушим в Git
kamal deploy          # Деплоим на сервер

# Kamal автоматически:
# 1. Билдит новый Docker image
# 2. Пушит в registry
# 3. Останавливает старый контейнер
# 4. Запускает новый
# 5. Zero-downtime deployment!
```

---

## 🌐 Настройка Домена

### 1. DNS Records (у регистратора домена)

```
Type    Name    Value           TTL
────────────────────────────────────
A       @       YOUR_SERVER_IP  3600
A       www     YOUR_SERVER_IP  3600
```

### 2. SSL Сертификат (автоматически через Let's Encrypt)

Kamal настраивает Traefik, который автоматически получает SSL от Let's Encrypt.

Проверка:
```bash
# Должен открыться с зеленым замком
https://industrialprofi.ru
```

---

## 📊 Мониторинг и Логи

### Просмотр логов

```bash
# Логи приложения (real-time)
kamal app logs -f

# Логи Traefik
kamal traefik logs -f

# Логи конкретного контейнера
kamal app logs --since 1h
```

### Проверка здоровья

```bash
# Статус всех сервисов
kamal app details

# SSH в контейнер
kamal app exec -i bash

# Запуск Rails console на production
kamal app exec 'bin/rails console'
```

### Healthcheck endpoint

```ruby
# config/routes.rb
get "up" => "rails/health#show", as: :rails_health_check

# Rails 8 уже имеет встроенный healthcheck
# Доступен по /up
# Возвращает 200 если БД доступна и приложение запущено
```

---

## 🔧 Troubleshooting

### Проблема: "No route to host"

```bash
# Проверяем что SSH работает
ssh deploy@YOUR_SERVER_IP

# Проверяем firewall на сервере
sudo ufw status
# Если активен, разрешаем порты:
sudo ufw allow 22    # SSH
sudo ufw allow 80    # HTTP
sudo ufw allow 443   # HTTPS
```

### Проблема: "Docker image build failed"

```bash
# Локально билдим image для проверки
docker build -t industrialprofi .

# Если падает на npm install:
# Проверяем package-lock.json (должен быть закоммичен)
git add package-lock.json
git commit -m "Add package-lock.json"
```

### Проблема: "Database locked"

```bash
# SQLite занята другим процессом
# Перезапускаем контейнер
kamal app reboot

# Проверяем что Solid Queue не висит
kamal app exec 'bin/rails runner "puts SolidQueue::Job.count"'
```

### Проблема: "Assets не загружаются (404)"

```bash
# Проверяем что assets precompile прошел
kamal app exec 'ls public/assets'

# Если пусто — пересобираем
kamal deploy --skip-push  # Заново билдит image
```

---

## 💾 Backup & Restore

### Автоматический Backup (Litestream)

Litestream реплицирует каждые 10 секунд автоматически в S3.

### Ручной Backup

```bash
# SSH в сервер
ssh deploy@YOUR_SERVER_IP

# Копируем БД
docker cp $(docker ps -q -f name=industrialprofi):/rails/storage/production.sqlite3 ./backup-$(date +%Y%m%d).sqlite3

# Скачиваем на локальную машину
scp deploy@YOUR_SERVER_IP:~/backup-*.sqlite3 ./
```

### Restore из Backup

```bash
# 1. Останавливаем приложение
kamal app stop

# 2. Копируем backup в контейнер
docker cp backup-20260208.sqlite3 $(docker ps -q -f name=industrialprofi):/rails/storage/production.sqlite3

# 3. Запускаем приложение
kamal app start
```

---

## 📈 Scaling (Когда понадобится)

### Vertical Scaling (Upgrade VPS)

```bash
# 1. На Hetzner: Upgrade сервера (через панель)
# CX21 (2 CPU, 4GB RAM) → CX31 (2 CPU, 8GB RAM)

# 2. Перезапускаем контейнеры
kamal app reboot
```

### Horizontal Scaling (Несколько серверов)

```yaml
# config/deploy.yml
servers:
  web:
    hosts:
      - SERVER_IP_1
      - SERVER_IP_2
    labels:
      traefik.http.services.industrialprofi.loadbalancer.server.port: 3000
```

**Важно:** Для horizontal scaling нужен shared storage (S3) и PostgreSQL (вместо SQLite).

---

## 🔐 Security Checklist

- [ ] SSH key-based authentication (не пароли)
- [ ] Firewall настроен (ufw enable)
- [ ] SSL сертификат активен (HTTPS)
- [ ] RAILS_MASTER_KEY в секретах (не в коде)
- [ ] Database backups работают (Litestream)
- [ ] Регулярные обновления сервера (`apt upgrade`)
- [ ] Non-root user для deploy
- [ ] Rate limiting (Rack::Attack в будущем)

---

## 📝 Deployment Checklist (Перед каждым деплоем)

```bash
# 1. Тесты зеленые
bin/rails test
# ✓ 0 failures

# 2. Frontend билдится
npm run build
# ✓ Build completed

# 3. Миграции созданы (если есть изменения БД)
bin/rails db:migrate:status
# ✓ up

# 4. Коммитим изменения
git add .
git commit -m "Feature: Add skill categories"
git push origin main

# 5. Деплоим
kamal deploy

# 6. Проверяем healthcheck
curl https://industrialprofi.ru/up
# ✓ 200 OK

# 7. Проверяем в браузере
open https://industrialprofi.ru
# ✓ Работает

# 8. Проверяем логи (нет ошибок)
kamal app logs --since 5m
# ✓ No errors
```

---

**Следующий документ:** `09_DEVELOPMENT_PLAN.md` (Финальный 5-недельный план)
