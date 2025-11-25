# Глава 7: Docker и деплой

В этой финальной главе мы упакуем приложение в Docker-контейнер и подготовим его к развёртыванию.

---

## 1. Что такое Docker

**Простое объяснение:**  
Docker — это инструмент для упаковки приложений в контейнеры. Контейнер содержит всё необходимое для работы: код, библиотеки, настройки. Это гарантирует, что приложение будет работать одинаково везде.

**Преимущества:**
- Изоляция от системы
- Одинаковое поведение на разных серверах
- Простое развёртывание
- Легко масштабировать

---

## 2. Создаём Dockerfile

**Что такое Dockerfile:**  
Dockerfile — это инструкция для создания Docker-образа. Он описывает, как собрать контейнер с нашим приложением.

**Создаём файл `Dockerfile` в корне проекта:**
```dockerfile
# Этап 1: Сборка приложения
# Используем официальный образ Go для сборки
FROM golang:1.21-alpine AS builder

# Устанавливаем рабочую директорию внутри контейнера
WORKDIR /app

# Копируем файлы зависимостей
COPY go.mod go.sum ./

# Скачиваем зависимости
# Это отдельный шаг для кэширования (если зависимости не изменились, этот шаг пропустится)
RUN go mod download

# Копируем весь исходный код
COPY . .

# Собираем приложение
# CGO_ENABLED=0 — отключаем CGO для статической сборки
# -ldflags="-s -w" — уменьшаем размер бинарника
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /tempmail ./cmd/api

# Этап 2: Финальный образ
# Используем минимальный образ Alpine
FROM alpine:3.19

# Устанавливаем ca-certificates для HTTPS и tzdata для часовых поясов
RUN apk --no-cache add ca-certificates tzdata

# Создаём пользователя для безопасности (не запускаем от root)
RUN adduser -D -g '' appuser

# Устанавливаем рабочую директорию
WORKDIR /app

# Копируем собранное приложение из этапа сборки
COPY --from=builder /tempmail .

# Копируем миграции
COPY migrations ./migrations

# Переключаемся на непривилегированного пользователя
USER appuser

# Открываем порты
EXPOSE 8080 2525

# Команда запуска
CMD ["./tempmail"]
```

**Разбор:**

- `FROM golang:1.21-alpine AS builder` — используем образ Go для сборки. `AS builder` даёт имя этому этапу.

- `WORKDIR /app` — устанавливаем рабочую директорию. Все последующие команды выполняются в ней.

- `COPY go.mod go.sum ./` — копируем файлы зависимостей.

- `RUN go mod download` — скачиваем зависимости. Отдельный шаг для кэширования.

- `COPY . .` — копируем весь код.

- `CGO_ENABLED=0` — отключаем CGO (C-библиотеки). Это позволяет создать статический бинарник.

- `GOOS=linux` — собираем для Linux.

- `-ldflags="-s -w"` — убираем отладочную информацию, уменьшая размер.

- `FROM alpine:3.19` — второй этап, используем минимальный образ.

- `COPY --from=builder /tempmail .` — копируем бинарник из первого этапа.

- `USER appuser` — запускаем от непривилегированного пользователя (безопасность).

- `EXPOSE 8080 2525` — документируем, какие порты использует приложение.

- `CMD ["./tempmail"]` — команда запуска контейнера.

---

## 3. Создаём .dockerignore

**Зачем нужен:**  
Файл `.dockerignore` указывает, какие файлы не нужно копировать в контейнер.

**Создаём файл `.dockerignore`:**
```
# Git
.git
.gitignore

# IDE
.idea
.vscode
*.swp

# Локальные файлы
.env
*.log
tmp/

# Документация
docs/
README.md

# Тесты
*_test.go

# Собранные файлы
tempmail
*.exe
```

---

## 4. Обновляем docker-compose.yml

**Обновляем `docker-compose.yml` для полного стека:**
```yaml
version: '3.8'

services:
  # Наше приложение
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: tempmail-app
    environment:
      - HTTP_PORT=8080
      - SMTP_PORT=2525
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=tempmail
      - DB_USER=postgres
      - DB_PASSWORD=secret
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - MAIL_DOMAIN=tempmail.dev
      - DEFAULT_TTL=1h
      - MAX_TTL=24h
      - CLEANUP_INTERVAL=5m
    ports:
      - "8080:8080"   # HTTP API
      - "2525:2525"   # SMTP
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped

  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: tempmail-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: tempmail
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations/001_init.up.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Redis
  redis:
    image: redis:7-alpine
    container_name: tempmail-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

**Разбор:**

- `build: context: .` — собираем образ из текущей директории.

- `depends_on` — указываем зависимости. Приложение запустится после PostgreSQL и Redis.

- `condition: service_healthy` — ждём, пока PostgreSQL не станет "здоровым" (готовым принимать соединения).

- `healthcheck` — проверка здоровья PostgreSQL.

- `./migrations/001_init.up.sql:/docker-entrypoint-initdb.d/init.sql` — автоматически выполняем миграцию при первом запуске.

- `restart: unless-stopped` — автоматический перезапуск при падении.

---

## 5. Собираем и запускаем

**Собираем образ:**
```bash
docker-compose build
```

**Запускаем все сервисы:**
```bash
docker-compose up -d
```

**Проверяем статус:**
```bash
docker-compose ps
```

**Смотрим логи:**
```bash
docker-compose logs -f app
```

**Останавливаем:**
```bash
docker-compose down
```

**Останавливаем и удаляем данные:**
```bash
docker-compose down -v
```

---

## 6. Создаём скрипты для удобства

### Вариант 1: Makefile (Linux/macOS)

**Что такое Makefile:**  
Makefile содержит команды, которые можно запускать одним словом. Упрощает работу с проектом.

**Создаём файл `Makefile`:**
```makefile
# Переменные
APP_NAME=tempmail
DOCKER_COMPOSE=docker-compose

# Цель по умолчанию
.DEFAULT_GOAL := help

# Справка
.PHONY: help
help:
	@echo "Доступные команды:"
	@echo "  make build      - Собрать Docker-образ"
	@echo "  make up         - Запустить все сервисы"
	@echo "  make down       - Остановить все сервисы"
	@echo "  make logs       - Показать логи приложения"
	@echo "  make restart    - Перезапустить приложение"
	@echo "  make clean      - Удалить все данные"
	@echo "  make dev        - Запустить в режиме разработки"
	@echo "  make test       - Запустить тесты"
	@echo "  make lint       - Проверить код"

# Сборка
.PHONY: build
build:
	$(DOCKER_COMPOSE) build

# Запуск
.PHONY: up
up:
	$(DOCKER_COMPOSE) up -d

# Остановка
.PHONY: down
down:
	$(DOCKER_COMPOSE) down

# Логи
.PHONY: logs
logs:
	$(DOCKER_COMPOSE) logs -f app

# Перезапуск
.PHONY: restart
restart:
	$(DOCKER_COMPOSE) restart app

# Полная очистка
.PHONY: clean
clean:
	$(DOCKER_COMPOSE) down -v
	docker system prune -f

# Режим разработки (локальный запуск)
.PHONY: dev
dev:
	$(DOCKER_COMPOSE) up -d postgres redis
	go run cmd/api/main.go

# Тесты
.PHONY: test
test:
	go test -v ./...

# Линтер
.PHONY: lint
lint:
	golangci-lint run
```

**Использование:**
```bash
make build    # Собрать
make up       # Запустить
make logs     # Смотреть логи
make down     # Остановить
```

### Вариант 2: PowerShell-скрипты (Windows)

На Windows `make` обычно не установлен. Создадим PowerShell-скрипты.

**Создаём файл `scripts/build.ps1`:**
```powershell
# Сборка Docker-образа
Write-Host "Сборка Docker-образа..." -ForegroundColor Green
docker-compose build
Write-Host "Готово!" -ForegroundColor Green
```

**Создаём файл `scripts/up.ps1`:**
```powershell
# Запуск всех сервисов
Write-Host "Запуск сервисов..." -ForegroundColor Green
docker-compose up -d
Write-Host "Сервисы запущены!" -ForegroundColor Green
Write-Host "API: http://localhost:8080" -ForegroundColor Cyan
Write-Host "Swagger: http://localhost:8080/swagger/index.html" -ForegroundColor Cyan
```

**Создаём файл `scripts/down.ps1`:**
```powershell
# Остановка всех сервисов
Write-Host "Остановка сервисов..." -ForegroundColor Yellow
docker-compose down
Write-Host "Сервисы остановлены!" -ForegroundColor Green
```

**Создаём файл `scripts/logs.ps1`:**
```powershell
# Просмотр логов приложения
docker-compose logs -f app
```

**Создаём файл `scripts/dev.ps1`:**
```powershell
# Режим разработки: запускаем только БД, приложение локально
Write-Host "Запуск PostgreSQL и Redis..." -ForegroundColor Green
docker-compose up -d postgres redis
Start-Sleep -Seconds 3
Write-Host "Запуск приложения..." -ForegroundColor Green
go run cmd/api/main.go
```

**Создаём файл `scripts/swagger.ps1`:**
```powershell
# Генерация Swagger документации
Write-Host "Генерация Swagger..." -ForegroundColor Green
swag init -g cmd/api/main.go -o docs
Write-Host "Готово! Файлы в папке docs/" -ForegroundColor Green
```

**Использование в PowerShell:**
```powershell
.\scripts\build.ps1    # Собрать
.\scripts\up.ps1       # Запустить
.\scripts\logs.ps1     # Смотреть логи
.\scripts\down.ps1     # Остановить
.\scripts\dev.ps1      # Режим разработки
.\scripts\swagger.ps1  # Генерация Swagger
```

**Примечание:**  
Если PowerShell блокирует выполнение скриптов, выполните:
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

---

## 7. Создаём скрипт для миграций

**Создаём файл `scripts/migrate.sh`:**
```bash
#!/bin/bash

# Скрипт для выполнения миграций

set -e  # Выход при ошибке

# Параметры подключения
DB_HOST=${DB_HOST:-localhost}
DB_PORT=${DB_PORT:-5432}
DB_NAME=${DB_NAME:-tempmail}
DB_USER=${DB_USER:-postgres}
DB_PASSWORD=${DB_PASSWORD:-secret}

# Формируем строку подключения
CONNECTION="postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}?sslmode=disable"

echo "Подключение к базе данных..."
echo "Host: ${DB_HOST}:${DB_PORT}"
echo "Database: ${DB_NAME}"

# Выполняем миграцию
case "$1" in
    up)
        echo "Применение миграций..."
        psql "${CONNECTION}" -f migrations/001_init.up.sql
        echo "Миграции применены!"
        ;;
    down)
        echo "Откат миграций..."
        psql "${CONNECTION}" -f migrations/001_init.down.sql
        echo "Миграции откачены!"
        ;;
    *)
        echo "Использование: $0 {up|down}"
        exit 1
        ;;
esac
```

**Делаем исполняемым:**
```bash
chmod +x scripts/migrate.sh
```

**Использование:**
```bash
./scripts/migrate.sh up    # Применить миграции
./scripts/migrate.sh down  # Откатить миграции
```

---

## 8. Настройка для продакшена

**Создаём файл `docker-compose.prod.yml`:**
```yaml
version: '3.8'

services:
  app:
    image: your-registry/tempmail:latest
    environment:
      - HTTP_PORT=8080
      - SMTP_PORT=25
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=tempmail
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - MAIL_DOMAIN=${MAIL_DOMAIN}
    ports:
      - "80:8080"
      - "25:25"
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: tempmail
    volumes:
      - postgres_data:/var/lib/postgresql/data
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 128M

volumes:
  postgres_data:
  redis_data:
```

**Разбор:**

- `deploy: replicas: 2` — запускаем 2 экземпляра приложения.

- `resources: limits` — ограничиваем ресурсы контейнера.

- `logging` — настраиваем логирование (ротация логов).

- `${DB_PASSWORD}` — переменные из файла `.env`.

---

## 9. Итоговая структура проекта

```
tempmail/
├── cmd/
│   ├── api/
│   │   └── main.go           # Точка входа
│   └── smtp/
│       └── main.go           # SMTP-сервер (опционально)
├── internal/
│   ├── config/
│   │   └── config.go         # Конфигурация
│   ├── domain/
│   │   ├── mailbox.go        # Модель ящика
│   │   └── message.go        # Модель письма
│   ├── handler/
│   │   ├── mailbox_handler.go
│   │   ├── message_handler.go
│   │   ├── error.go
│   │   └── routes.go
│   ├── repository/
│   │   ├── postgres.go
│   │   ├── mailbox_repo.go
│   │   └── message_repo.go
│   ├── service/
│   │   ├── mailbox_service.go
│   │   ├── message_service.go
│   │   ├── scheduler.go
│   │   └── stats.go
│   ├── smtp/
│   │   ├── backend.go
│   │   ├── session.go
│   │   └── server.go
│   └── spam/
│       └── filter.go
├── migrations/
│   ├── 001_init.up.sql
│   └── 001_init.down.sql
├── scripts/
│   └── migrate.sh
├── .env
├── .env.example
├── .gitignore
├── .dockerignore
├── docker-compose.yml
├── docker-compose.prod.yml
├── Dockerfile
├── Makefile
├── go.mod
├── go.sum
└── README.md
```

---

## 10. Проверяем всё вместе

**Шаг 1: Запускаем:**
```bash
make build
make up
```

**Шаг 2: Проверяем здоровье:**
```bash
curl http://localhost:8080/health
```

**Шаг 3: Создаём ящик:**
```bash
curl -X POST http://localhost:8080/api/v1/mailbox
```

**Шаг 4: Отправляем письмо:**
```bash
swaks --to <address>@tempmail.dev \
      --from test@example.com \
      --server localhost:2525 \
      --header "Subject: Test" \
      --body "Hello!"
```

**Шаг 5: Проверяем письма:**
```bash
curl http://localhost:8080/api/v1/mailbox/<id>/messages
```

**Шаг 6: Смотрим статистику:**
```bash
curl http://localhost:8080/stats
```

---

## Что мы создали

Полноценный сервис временных email-адресов:

- **REST API** для управления ящиками и письмами
- **SMTP-сервер** для приёма входящих писем
- **Спам-фильтр** для защиты от нежелательной почты
- **Планировщик** для автоматической очистки
- **Docker-контейнеры** для развёртывания

---

## Дальнейшее развитие

Что можно добавить:
1. **Аутентификация API** (API-ключи, JWT)
2. **WebSocket** для уведомлений о новых письмах
3. **Веб-интерфейс** для просмотра писем
4. **TLS/SSL** для SMTP
5. **Мониторинг** (Prometheus, Grafana)
6. **Тесты** (unit, integration)
7. **CI/CD** (GitHub Actions, GitLab CI)

---

## Словарь терминов

| Термин | Объяснение |
|--------|------------|
| **Docker** | Платформа для контейнеризации приложений |
| **Dockerfile** | Инструкция для создания Docker-образа |
| **Docker Compose** | Инструмент для запуска многоконтейнерных приложений |
| **Образ** | Шаблон для создания контейнеров |
| **Контейнер** | Запущенный экземпляр образа |
| **Том (Volume)** | Постоянное хранилище данных для контейнеров |
| **Healthcheck** | Проверка работоспособности сервиса |
| **Multi-stage build** | Многоэтапная сборка для уменьшения размера образа |

---

## Поздравляем!

Вы создали полноценный бэкенд-сервис на Go с нуля. Теперь вы знаете:

- Основы языка Go
- Как работать с базами данных
- Как создавать REST API
- Как принимать email через SMTP
- Как фильтровать спам
- Как упаковывать приложение в Docker

---

[Следующая глава: Документация API с OpenAPI (Swagger)](./08-openapi.md)

[Вернуться к оглавлению](./README.md)

