# Техническое задание: Сервис временных email-адресов

## 1. Описание проекта

Сервис для создания временных (disposable) email-адресов с возможностью получения входящих писем через веб-интерфейс и REST API.

**Пример использования:** Пользователь получает адрес `abc123@tempmail.dev`, все входящие письма доступны через API или веб-интерфейс.

## 2. Технологический стек

| Компонент | Технология |
|-----------|------------|
| Язык | Go 1.21+ |
| SMTP-сервер | emersion/go-smtp |
| HTTP API | Fiber v2 |
| База данных | PostgreSQL 15+ |
| Кэширование | Redis |
| Отправка писем | go-mail |
| Миграции | golang-migrate |

## 3. Функциональные требования

### 3.1 Управление email-адресами

- Генерация случайного временного адреса
- Создание кастомного адреса (если свободен)
- Настройка TTL (время жизни) адреса: 10 мин, 1 час, 24 часа
- Продление времени жизни адреса
- Удаление адреса вручную

### 3.2 Приём и хранение писем

- Приём входящих писем по SMTP (порт 25/2525)
- Парсинг заголовков, тела письма, вложений
- Хранение вложений в файловой системе или S3
- Автоматическое удаление писем по истечении TTL

### 3.3 Фильтрация

- Базовый спам-фильтр (по ключевым словам, размеру)
- Ограничение размера письма (макс. 10 MB)
- Ограничение размера вложений (макс. 5 MB на файл)
- Блокировка подозрительных отправителей

### 3.4 REST API

```
POST   /api/v1/mailbox           - создать новый ящик
GET    /api/v1/mailbox/{id}      - информация о ящике
DELETE /api/v1/mailbox/{id}      - удалить ящик
GET    /api/v1/mailbox/{id}/messages      - список писем
GET    /api/v1/mailbox/{id}/messages/{mid} - получить письмо
DELETE /api/v1/mailbox/{id}/messages/{mid} - удалить письмо
GET    /api/v1/mailbox/{id}/messages/{mid}/attachments/{aid} - скачать вложение
```

### 3.5 WebSocket (опционально)

- Подписка на новые письма в реальном времени
- Уведомления о входящих сообщениях

## 4. Нефункциональные требования

- Обработка до 100 одновременных SMTP-соединений
- Время ответа API < 100ms (p95)
- Хранение до 1 млн писем
- Автоматическая очистка устаревших данных (cron job)

## 5. Структура проекта

```
tempmail/
├── cmd/
│   ├── api/          # HTTP API сервер
│   └── smtp/         # SMTP сервер
├── internal/
│   ├── config/       # Конфигурация
│   ├── domain/       # Доменные модели
│   ├── handler/      # HTTP handlers
│   ├── repository/   # Работа с БД
│   ├── service/      # Бизнес-логика
│   ├── smtp/         # SMTP backend
│   └── spam/         # Спам-фильтр
├── migrations/       # SQL миграции
├── pkg/
│   └── email/        # Утилиты для работы с email
├── docker-compose.yml
├── Dockerfile
└── go.mod
```

## 6. Схема базы данных

```sql
-- Почтовые ящики
CREATE TABLE mailboxes (
    id UUID PRIMARY KEY,
    address VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,
    is_active BOOLEAN DEFAULT TRUE
);

-- Письма
CREATE TABLE messages (
    id UUID PRIMARY KEY,
    mailbox_id UUID REFERENCES mailboxes(id) ON DELETE CASCADE,
    from_address VARCHAR(255) NOT NULL,
    subject VARCHAR(500),
    body_text TEXT,
    body_html TEXT,
    received_at TIMESTAMP DEFAULT NOW(),
    is_read BOOLEAN DEFAULT FALSE,
    is_spam BOOLEAN DEFAULT FALSE
);

-- Вложения
CREATE TABLE attachments (
    id UUID PRIMARY KEY,
    message_id UUID REFERENCES messages(id) ON DELETE CASCADE,
    filename VARCHAR(255) NOT NULL,
    content_type VARCHAR(100),
    size_bytes BIGINT,
    storage_path VARCHAR(500)
);

-- Индексы
CREATE INDEX idx_mailboxes_address ON mailboxes(address);
CREATE INDEX idx_mailboxes_expires ON mailboxes(expires_at);
CREATE INDEX idx_messages_mailbox ON messages(mailbox_id);
```

## 7. Конфигурация

```yaml
server:
  http_port: 8080
  smtp_port: 2525

database:
  host: localhost
  port: 5432
  name: tempmail
  user: postgres
  password: ${DB_PASSWORD}

redis:
  host: localhost
  port: 6379

mailbox:
  default_ttl: 1h
  max_ttl: 24h
  domain: tempmail.dev

limits:
  max_message_size: 10485760  # 10 MB
  max_attachment_size: 5242880 # 5 MB
  max_messages_per_mailbox: 100
```

## 8. Этапы разработки

### Этап 1: Базовая инфраструктура (2-3 дня)
- Настройка проекта, Docker, миграции
- Подключение PostgreSQL и Redis
- Базовые модели и репозитории

### Этап 2: SMTP-сервер (3-4 дня)
- Реализация SMTP backend на go-smtp
- Парсинг входящих писем
- Сохранение в БД

### Этап 3: REST API (2-3 дня)
- CRUD для ящиков и писем
- Скачивание вложений
- Документация Swagger

### Этап 4: Фильтрация и очистка (2 дня)
- Спам-фильтр
- Cron для удаления устаревших данных

### Этап 5: Оптимизация (2 дня)
- Кэширование в Redis
- Нагрузочное тестирование

**Общая оценка: 11-14 дней**

## 9. Монетизация (опционально)

- Премиум-домены (кастомные адреса)
- Увеличенное время хранения (до 7 дней)
- Коммерческий API для QA-инструментов
- Приоритетная поддержка

## 10. Риски

| Риск | Митигация |
|------|-----------|
| Спам-атаки | Rate limiting, капча, блокировка IP |
| Использование для мошенничества | Логирование, сотрудничество с abuse-службами |
| Высокая нагрузка на SMTP | Горизонтальное масштабирование, очереди |

