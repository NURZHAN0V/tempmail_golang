# Глава 2: Структура проекта и конфигурация

В этой главе мы организуем структуру папок проекта и научимся работать с конфигурацией через переменные окружения.

---

## 1. Создаём структуру папок

**Зачем это нужно:**  
Правильная организация кода помогает не запутаться в проекте. Каждая папка отвечает за свою часть функционала.

**Что делаем:**  
Создаём папки согласно стандартной структуре Go-проектов.

**Команды:**
```bash
# Создаём основные папки
mkdir -p cmd/api
mkdir -p cmd/smtp
mkdir -p internal/config
mkdir -p internal/domain
mkdir -p internal/handler
mkdir -p internal/repository
mkdir -p internal/service
mkdir -p internal/smtp
mkdir -p internal/spam
mkdir -p migrations
mkdir -p pkg/email
```

**Что означает каждая папка:**

| Папка | Назначение |
|-------|------------|
| `cmd/api` | Точка входа для HTTP API сервера |
| `cmd/smtp` | Точка входа для SMTP сервера |
| `internal/config` | Загрузка конфигурации |
| `internal/domain` | Модели данных (структуры) |
| `internal/handler` | Обработчики HTTP-запросов |
| `internal/repository` | Работа с базой данных |
| `internal/service` | Бизнес-логика |
| `internal/smtp` | SMTP-сервер для приёма писем |
| `internal/spam` | Спам-фильтр |
| `migrations` | SQL-миграции для базы данных |
| `pkg/email` | Утилиты для работы с email |

**Что такое `internal`:**  
Папка `internal` — специальная в Go. Код внутри неё можно использовать только в этом проекте. Другие проекты не смогут импортировать эти пакеты.

**Что такое `cmd`:**  
Папка `cmd` содержит точки входа в приложение. Каждая подпапка — отдельная программа, которую можно запустить.

---

## 2. Создаём файл конфигурации

**Зачем это нужно:**  
Конфигурация (порты, пароли, адреса баз данных) не должна быть "зашита" в код. Её нужно хранить отдельно, чтобы легко менять без перекомпиляции.

**Создаём файл `.env` в корне проекта:**
```
# Сервер
HTTP_PORT=8080
SMTP_PORT=2525

# База данных
DB_HOST=localhost
DB_PORT=5432
DB_NAME=tempmail
DB_USER=postgres
DB_PASSWORD=secret

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Настройки почтовых ящиков
MAIL_DOMAIN=tempmail.dev
DEFAULT_TTL=1h
MAX_TTL=24h

# Лимиты
MAX_MESSAGE_SIZE=10485760
MAX_ATTACHMENT_SIZE=5242880
MAX_MESSAGES_PER_MAILBOX=100
```

**Разбор:**
- Каждая строка — это пара "ключ=значение"
- `#` — начало комментария
- Пробелов вокруг `=` быть не должно
- Значения без кавычек

**Важно:**  
Добавьте `.env` в файл `.gitignore`, чтобы случайно не выложить пароли в Git:
```bash
echo ".env" >> .gitignore
```

---

## 3. Устанавливаем библиотеки

**Что делаем:**  
Устанавливаем библиотеки для работы с конфигурацией и другие зависимости проекта.

**Команды:**
```bash
# Библиотека для чтения .env файлов
go get github.com/joho/godotenv

# Библиотека для работы с конфигурацией
go get github.com/kelseyhightower/envconfig

# Веб-фреймворк Fiber
go get github.com/gofiber/fiber/v2

# Драйвер PostgreSQL
go get github.com/lib/pq

# Библиотека для работы с UUID
go get github.com/google/uuid

# SMTP-сервер
go get github.com/emersion/go-smtp

# Парсер email
go get github.com/emersion/go-message
```

**Что такое `go get`:**  
Команда `go get` скачивает внешнюю библиотеку и добавляет её в файл `go.mod`. После этого библиотеку можно использовать в коде.

**Результат:**  
В файле `go.mod` появятся строки с зависимостями, а также создастся файл `go.sum` с контрольными суммами (для безопасности).

---

## 4. Создаём структуру конфигурации

**Что делаем:**  
Создаём Go-структуру, которая будет хранить все настройки приложения.

**Создаём файл `internal/config/config.go`:**
```go
package config

import (
    "time"

    "github.com/joho/godotenv"
    "github.com/kelseyhightower/envconfig"
)

// Config — главная структура конфигурации приложения
// Все поля заполняются из переменных окружения
type Config struct {
    Server   ServerConfig   // Настройки серверов
    Database DatabaseConfig // Настройки базы данных
    Redis    RedisConfig    // Настройки Redis
    Mail     MailConfig     // Настройки почты
    Limits   LimitsConfig   // Лимиты
}

// ServerConfig — настройки HTTP и SMTP серверов
type ServerConfig struct {
    HTTPPort int `envconfig:"HTTP_PORT" default:"8080"` // Порт HTTP сервера
    SMTPPort int `envconfig:"SMTP_PORT" default:"2525"` // Порт SMTP сервера
}

// DatabaseConfig — настройки подключения к PostgreSQL
type DatabaseConfig struct {
    Host     string `envconfig:"DB_HOST" default:"localhost"`     // Адрес сервера БД
    Port     int    `envconfig:"DB_PORT" default:"5432"`          // Порт БД
    Name     string `envconfig:"DB_NAME" default:"tempmail"`      // Имя базы данных
    User     string `envconfig:"DB_USER" default:"postgres"`      // Пользователь БД
    Password string `envconfig:"DB_PASSWORD" required:"true"`     // Пароль БД (обязательный)
}

// RedisConfig — настройки подключения к Redis
type RedisConfig struct {
    Host string `envconfig:"REDIS_HOST" default:"localhost"` // Адрес Redis
    Port int    `envconfig:"REDIS_PORT" default:"6379"`      // Порт Redis
}

// MailConfig — настройки почтовых ящиков
type MailConfig struct {
    Domain     string        `envconfig:"MAIL_DOMAIN" default:"tempmail.dev"` // Домен для email
    DefaultTTL time.Duration `envconfig:"DEFAULT_TTL" default:"1h"`           // Время жизни по умолчанию
    MaxTTL     time.Duration `envconfig:"MAX_TTL" default:"24h"`              // Максимальное время жизни
}

// LimitsConfig — лимиты и ограничения
type LimitsConfig struct {
    MaxMessageSize       int `envconfig:"MAX_MESSAGE_SIZE" default:"10485760"`        // Макс. размер письма (10 MB)
    MaxAttachmentSize    int `envconfig:"MAX_ATTACHMENT_SIZE" default:"5242880"`      // Макс. размер вложения (5 MB)
    MaxMessagesPerMailbox int `envconfig:"MAX_MESSAGES_PER_MAILBOX" default:"100"`    // Макс. писем в ящике
}
```

**Разбор:**

- `package config` — объявляем, что этот файл принадлежит пакету `config`. Имя пакета обычно совпадает с именем папки.

- `import (...)` — импортируем несколько пакетов. Когда импортов много, их группируют в скобках.

- `type Config struct {...}` — создаём структуру `Config`, которая содержит другие структуры (вложенные структуры).

- `` `envconfig:"HTTP_PORT" default:"8080"` `` — это **теги структуры**. Они не влияют на работу Go, но библиотеки могут их читать:
  - `envconfig:"HTTP_PORT"` — библиотека envconfig будет искать переменную окружения `HTTP_PORT`
  - `default:"8080"` — если переменная не найдена, использовать значение 8080
  - `required:"true"` — поле обязательное, без него программа не запустится

- `time.Duration` — специальный тип для хранения промежутков времени. Понимает значения вроде "1h" (1 час), "30m" (30 минут), "24h" (24 часа).

---

## 5. Функция загрузки конфигурации

**Что делаем:**  
Добавляем функцию, которая читает переменные окружения и заполняет структуру `Config`.

**Добавляем в `internal/config/config.go`:**
```go
// Load загружает конфигурацию из переменных окружения
// Сначала пытается прочитать файл .env, затем читает переменные окружения
func Load() (*Config, error) {
    // Пытаемся загрузить .env файл
    // Если файла нет — не страшно, будем читать из системных переменных
    _ = godotenv.Load()

    // Создаём пустую структуру конфигурации
    var cfg Config

    // Заполняем структуру из переменных окружения
    // Если обязательное поле отсутствует — вернётся ошибка
    err := envconfig.Process("", &cfg)
    if err != nil {
        return nil, err
    }

    // Возвращаем указатель на конфигурацию
    return &cfg, nil
}
```

**Разбор:**

- `func Load() (*Config, error)` — функция возвращает два значения:
  - `*Config` — указатель на структуру Config
  - `error` — ошибка (если что-то пошло не так)

- `_ = godotenv.Load()` — вызываем функцию загрузки .env файла
  - `_` (нижнее подчёркивание) означает "игнорируем возвращаемое значение". Мы не проверяем ошибку, потому что файл .env необязателен.

- `var cfg Config` — создаём переменную `cfg` типа `Config`. Все поля будут иметь нулевые значения.

- `envconfig.Process("", &cfg)` — заполняем структуру из переменных окружения
  - `""` — пустой префикс (переменные читаются как есть, без префикса)
  - `&cfg` — передаём **адрес** переменной (указатель). Символ `&` означает "взять адрес". Это нужно, чтобы функция могла изменить нашу переменную.

- `if err != nil` — проверяем, была ли ошибка
  - `nil` — специальное значение "ничего" для указателей, интерфейсов и ошибок
  - Если `err` не `nil`, значит произошла ошибка

- `return &cfg, nil` — возвращаем указатель на конфигурацию и `nil` вместо ошибки (ошибки не было)

**Что такое указатель:**  
Указатель — это адрес в памяти, где хранится значение. Вместо копирования всей структуры, мы передаём только её адрес. Это экономит память и позволяет изменять оригинал.
- `&cfg` — получить адрес переменной `cfg`
- `*Config` — тип "указатель на Config"

---

## 6. Полный код файла config.go

**Создаём файл `internal/config/config.go`:**
```go
package config

import (
    "time"

    "github.com/joho/godotenv"
    "github.com/kelseyhightower/envconfig"
)

// Config — главная структура конфигурации приложения
type Config struct {
    Server   ServerConfig
    Database DatabaseConfig
    Redis    RedisConfig
    Mail     MailConfig
    Limits   LimitsConfig
}

// ServerConfig — настройки HTTP и SMTP серверов
type ServerConfig struct {
    HTTPPort int `envconfig:"HTTP_PORT" default:"8080"`
    SMTPPort int `envconfig:"SMTP_PORT" default:"2525"`
}

// DatabaseConfig — настройки подключения к PostgreSQL
type DatabaseConfig struct {
    Host     string `envconfig:"DB_HOST" default:"localhost"`
    Port     int    `envconfig:"DB_PORT" default:"5432"`
    Name     string `envconfig:"DB_NAME" default:"tempmail"`
    User     string `envconfig:"DB_USER" default:"postgres"`
    Password string `envconfig:"DB_PASSWORD" required:"true"`
}

// RedisConfig — настройки подключения к Redis
type RedisConfig struct {
    Host string `envconfig:"REDIS_HOST" default:"localhost"`
    Port int    `envconfig:"REDIS_PORT" default:"6379"`
}

// MailConfig — настройки почтовых ящиков
type MailConfig struct {
    Domain     string        `envconfig:"MAIL_DOMAIN" default:"tempmail.dev"`
    DefaultTTL time.Duration `envconfig:"DEFAULT_TTL" default:"1h"`
    MaxTTL     time.Duration `envconfig:"MAX_TTL" default:"24h"`
}

// LimitsConfig — лимиты и ограничения
type LimitsConfig struct {
    MaxMessageSize        int `envconfig:"MAX_MESSAGE_SIZE" default:"10485760"`
    MaxAttachmentSize     int `envconfig:"MAX_ATTACHMENT_SIZE" default:"5242880"`
    MaxMessagesPerMailbox int `envconfig:"MAX_MESSAGES_PER_MAILBOX" default:"100"`
}

// Load загружает конфигурацию из переменных окружения
func Load() (*Config, error) {
    // Загружаем .env файл (если есть)
    _ = godotenv.Load()

    // Создаём структуру конфигурации
    var cfg Config

    // Заполняем из переменных окружения
    err := envconfig.Process("", &cfg)
    if err != nil {
        return nil, err
    }

    return &cfg, nil
}
```

---

## 7. Создаём точку входа API

**Что делаем:**  
Создаём главный файл API-сервера, который загружает конфигурацию и запускает сервер.

**Создаём файл `cmd/api/main.go`:**
```go
package main

import (
    "fmt"
    "log"

    "tempmail/internal/config"
)

func main() {
    // Загружаем конфигурацию
    cfg, err := config.Load()
    
    // Проверяем ошибку загрузки
    if err != nil {
        // log.Fatal выводит сообщение и завершает программу
        log.Fatal("Ошибка загрузки конфигурации:", err)
    }

    // Выводим информацию о конфигурации
    fmt.Println("=== TempMail API Server ===")
    fmt.Printf("HTTP порт: %d\n", cfg.Server.HTTPPort)
    fmt.Printf("SMTP порт: %d\n", cfg.Server.SMTPPort)
    fmt.Printf("База данных: %s@%s:%d/%s\n", 
        cfg.Database.User, 
        cfg.Database.Host, 
        cfg.Database.Port, 
        cfg.Database.Name,
    )
    fmt.Printf("Домен почты: %s\n", cfg.Mail.Domain)
    fmt.Printf("TTL по умолчанию: %s\n", cfg.Mail.DefaultTTL)
    
    // TODO: Здесь будет запуск HTTP-сервера
    fmt.Println("\nСервер пока не реализован...")
}
```

**Разбор:**

- `import "tempmail/internal/config"` — импортируем наш пакет config. Путь начинается с имени модуля (`tempmail`), указанного в `go.mod`.

- `log.Fatal(...)` — выводит сообщение и завершает программу с кодом ошибки 1. Используется для критических ошибок, после которых продолжение невозможно.

- `fmt.Printf(...)` — форматированный вывод:
  - `%d` — целое число
  - `%s` — строка
  - `\n` — перенос строки

- `cfg.Server.HTTPPort` — обращаемся к вложенной структуре через точку

---

## 8. Проверяем работу

**Запускаем программу:**
```bash
go run cmd/api/main.go
```

**Ожидаемый результат:**
```
=== TempMail API Server ===
HTTP порт: 8080
SMTP порт: 2525
База данных: postgres@localhost:5432/tempmail
Домен почты: tempmail.dev
TTL по умолчанию: 1h0m0s

Сервер пока не реализован...
```

**Если видите ошибку о `DB_PASSWORD`:**  
Это значит, что переменная `DB_PASSWORD` обязательна, но не найдена. Проверьте, что файл `.env` создан и содержит `DB_PASSWORD=secret`.

---

## Словарь терминов

| Термин | Объяснение |
|--------|------------|
| **Пакет (package)** | Способ организации кода в Go. Каждый файл принадлежит какому-то пакету |
| **Модуль (module)** | Проект Go с файлом go.mod. Содержит один или несколько пакетов |
| **Указатель** | Адрес в памяти, где хранится значение. Обозначается `*` |
| **Теги структуры** | Метаданные полей структуры в обратных кавычках. Читаются библиотеками |
| **nil** | Специальное значение "ничего" для указателей, интерфейсов, ошибок |
| **Переменные окружения** | Системные переменные, доступные всем программам |

---

## Что мы узнали

- Как организовать структуру Go-проекта
- Зачем нужны папки `cmd`, `internal`, `pkg`
- Как хранить конфигурацию в файле `.env`
- Как создавать структуры с тегами
- Как загружать конфигурацию из переменных окружения
- Что такое указатели и зачем они нужны
- Как обрабатывать ошибки в Go

---

[Следующая глава: Работа с базой данных PostgreSQL](./03-database.md)

[Вернуться к оглавлению](./README.md)

