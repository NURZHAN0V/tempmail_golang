# Глава 3: Работа с базой данных PostgreSQL

В этой главе мы подключим PostgreSQL, создадим таблицы и научимся выполнять CRUD-операции (создание, чтение, обновление, удаление).

---

## 1. Запускаем PostgreSQL через Docker

**Зачем Docker:**  
Docker позволяет запустить PostgreSQL одной командой, без установки на компьютер. База данных будет работать в изолированном контейнере.

**Создаём файл `docker-compose.yml` в корне проекта:**
```yaml
version: '3.8'

services:
  # PostgreSQL — основная база данных
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

  # Redis — для кэширования (понадобится позже)
  redis:
    image: redis:7-alpine
    container_name: tempmail-redis
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

**Разбор:**
- `version: '3.8'` — версия формата docker-compose
- `services` — список сервисов (контейнеров)
- `image: postgres:15-alpine` — образ PostgreSQL версии 15 (alpine — лёгкая версия Linux)
- `environment` — переменные окружения для контейнера
- `ports: "5432:5432"` — пробрасываем порт из контейнера наружу
- `volumes` — сохраняем данные, чтобы они не пропали при перезапуске

**Запускаем:**
```bash
docker-compose up -d
```

**Что означает `-d`:**  
Флаг `-d` (detached) запускает контейнеры в фоновом режиме. Терминал освобождается для дальнейшей работы.

**Проверяем:**
```bash
docker-compose ps
```

**Результат:**  
Вы увидите два запущенных контейнера: `tempmail-postgres` и `tempmail-redis`.

---

## 2. Создаём модели данных

**Что такое модель:**  
Модель — это Go-структура, которая описывает данные. Каждая модель соответствует таблице в базе данных.

**Создаём файл `internal/domain/mailbox.go`:**
```go
package domain

import (
    "time"
)

// Mailbox — почтовый ящик
// Каждый ящик имеет уникальный адрес и время жизни
type Mailbox struct {
    ID        string    `json:"id"`         // Уникальный идентификатор (UUID)
    Address   string    `json:"address"`    // Email адрес (например, abc123@tempmail.dev)
    CreatedAt time.Time `json:"created_at"` // Дата создания
    ExpiresAt time.Time `json:"expires_at"` // Дата истечения срока
    IsActive  bool      `json:"is_active"`  // Активен ли ящик
}

// IsExpired проверяет, истёк ли срок действия ящика
func (m *Mailbox) IsExpired() bool {
    // time.Now() возвращает текущее время
    // After проверяет, наступило ли время ExpiresAt
    return time.Now().After(m.ExpiresAt)
}
```

**Разбор:**

- `` `json:"id"` `` — тег для JSON. Когда структура преобразуется в JSON, поле будет называться "id" (маленькими буквами).

- `func (m *Mailbox) IsExpired() bool` — это **метод** структуры
  - `(m *Mailbox)` — получатель метода. `m` — это ссылка на конкретный экземпляр Mailbox
  - Методы позволяют добавлять функции к структурам
  - Вызывается так: `mailbox.IsExpired()`

- `time.Now().After(m.ExpiresAt)` — проверяем, что текущее время позже времени истечения

---

## 3. Создаём модель сообщения

**Создаём файл `internal/domain/message.go`:**
```go
package domain

import (
    "time"
)

// Message — входящее письмо
type Message struct {
    ID          string    `json:"id"`           // Уникальный идентификатор
    MailboxID   string    `json:"mailbox_id"`   // ID почтового ящика
    FromAddress string    `json:"from_address"` // Адрес отправителя
    Subject     string    `json:"subject"`      // Тема письма
    BodyText    string    `json:"body_text"`    // Текстовое содержимое
    BodyHTML    string    `json:"body_html"`    // HTML содержимое
    ReceivedAt  time.Time `json:"received_at"`  // Дата получения
    IsRead      bool      `json:"is_read"`      // Прочитано ли
    IsSpam      bool      `json:"is_spam"`      // Помечено как спам
}

// Attachment — вложение к письму
type Attachment struct {
    ID          string `json:"id"`           // Уникальный идентификатор
    MessageID   string `json:"message_id"`   // ID письма
    Filename    string `json:"filename"`     // Имя файла
    ContentType string `json:"content_type"` // MIME-тип (например, image/png)
    SizeBytes   int64  `json:"size_bytes"`   // Размер в байтах
    StoragePath string `json:"storage_path"` // Путь к файлу на диске
}
```

**Разбор:**
- `int64` — 64-битное целое число. Используется для больших чисел (размер файла может быть большим).
- Структуры связаны через ID: `Message.MailboxID` ссылается на `Mailbox.ID`, `Attachment.MessageID` ссылается на `Message.ID`.

---

## 4. Создаём SQL-миграции

**Что такое миграция:**  
Миграция — это SQL-скрипт, который изменяет структуру базы данных. Миграции позволяют версионировать изменения БД.

**Создаём файл `migrations/001_init.up.sql`:**
```sql
-- Создаём таблицу почтовых ящиков
CREATE TABLE IF NOT EXISTS mailboxes (
    id UUID PRIMARY KEY,                           -- Уникальный идентификатор
    address VARCHAR(255) UNIQUE NOT NULL,          -- Email адрес (уникальный)
    created_at TIMESTAMP DEFAULT NOW(),            -- Дата создания
    expires_at TIMESTAMP NOT NULL,                 -- Дата истечения
    is_active BOOLEAN DEFAULT TRUE                 -- Активен ли ящик
);

-- Создаём таблицу писем
CREATE TABLE IF NOT EXISTS messages (
    id UUID PRIMARY KEY,                           -- Уникальный идентификатор
    mailbox_id UUID REFERENCES mailboxes(id) ON DELETE CASCADE, -- Связь с ящиком
    from_address VARCHAR(255) NOT NULL,            -- Адрес отправителя
    subject VARCHAR(500),                          -- Тема письма
    body_text TEXT,                                -- Текстовое содержимое
    body_html TEXT,                                -- HTML содержимое
    received_at TIMESTAMP DEFAULT NOW(),           -- Дата получения
    is_read BOOLEAN DEFAULT FALSE,                 -- Прочитано ли
    is_spam BOOLEAN DEFAULT FALSE                  -- Спам ли
);

-- Создаём таблицу вложений
CREATE TABLE IF NOT EXISTS attachments (
    id UUID PRIMARY KEY,                           -- Уникальный идентификатор
    message_id UUID REFERENCES messages(id) ON DELETE CASCADE, -- Связь с письмом
    filename VARCHAR(255) NOT NULL,                -- Имя файла
    content_type VARCHAR(100),                     -- MIME-тип
    size_bytes BIGINT,                             -- Размер в байтах
    storage_path VARCHAR(500)                      -- Путь к файлу
);

-- Создаём индексы для ускорения поиска
CREATE INDEX IF NOT EXISTS idx_mailboxes_address ON mailboxes(address);
CREATE INDEX IF NOT EXISTS idx_mailboxes_expires ON mailboxes(expires_at);
CREATE INDEX IF NOT EXISTS idx_messages_mailbox ON messages(mailbox_id);
CREATE INDEX IF NOT EXISTS idx_messages_received ON messages(received_at);
```

**Разбор SQL:**

- `CREATE TABLE IF NOT EXISTS` — создать таблицу, если её ещё нет
- `UUID` — тип для уникальных идентификаторов
- `PRIMARY KEY` — первичный ключ (уникальный идентификатор записи)
- `VARCHAR(255)` — строка до 255 символов
- `UNIQUE` — значение должно быть уникальным в таблице
- `NOT NULL` — поле обязательное (не может быть пустым)
- `DEFAULT NOW()` — значение по умолчанию — текущее время
- `REFERENCES mailboxes(id)` — внешний ключ (ссылка на другую таблицу)
- `ON DELETE CASCADE` — при удалении ящика удалятся и все его письма
- `TEXT` — текст неограниченной длины
- `BIGINT` — большое целое число
- `CREATE INDEX` — создаёт индекс для ускорения поиска

**Создаём файл отката `migrations/001_init.down.sql`:**
```sql
-- Удаляем таблицы в обратном порядке (из-за зависимостей)
DROP TABLE IF EXISTS attachments;
DROP TABLE IF EXISTS messages;
DROP TABLE IF EXISTS mailboxes;
```

---

## 5. Подключаемся к базе данных

**Создаём файл `internal/repository/postgres.go`:**
```go
package repository

import (
    "database/sql"
    "fmt"

    // Импортируем драйвер PostgreSQL
    // Символ _ означает, что мы импортируем пакет только ради его побочных эффектов
    // (драйвер регистрирует себя при импорте)
    _ "github.com/lib/pq"

    "tempmail/internal/config"
)

// PostgresDB — обёртка над подключением к PostgreSQL
type PostgresDB struct {
    DB *sql.DB // Стандартный интерфейс Go для работы с БД
}

// NewPostgresDB создаёт новое подключение к PostgreSQL
func NewPostgresDB(cfg config.DatabaseConfig) (*PostgresDB, error) {
    // Формируем строку подключения
    // Формат: postgres://user:password@host:port/dbname?sslmode=disable
    connStr := fmt.Sprintf(
        "postgres://%s:%s@%s:%d/%s?sslmode=disable",
        cfg.User,
        cfg.Password,
        cfg.Host,
        cfg.Port,
        cfg.Name,
    )

    // Открываем соединение с базой данных
    // sql.Open не устанавливает соединение сразу, только проверяет параметры
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, fmt.Errorf("ошибка открытия БД: %w", err)
    }

    // Проверяем, что соединение работает
    // Ping отправляет запрос к БД и ждёт ответа
    err = db.Ping()
    if err != nil {
        return nil, fmt.Errorf("ошибка подключения к БД: %w", err)
    }

    // Возвращаем обёртку с подключением
    return &PostgresDB{DB: db}, nil
}

// Close закрывает соединение с базой данных
func (p *PostgresDB) Close() error {
    return p.DB.Close()
}
```

**Разбор:**

- `_ "github.com/lib/pq"` — импорт с `_` означает, что мы не используем пакет напрямую. Драйвер PostgreSQL при импорте регистрирует себя в системе, и потом `sql.Open("postgres", ...)` знает, как работать с PostgreSQL.

- `*sql.DB` — стандартный тип Go для работы с базами данных. Предоставляет методы для выполнения запросов.

- `fmt.Sprintf(...)` — форматирует строку, подставляя значения вместо `%s` (строка) и `%d` (число).

- `fmt.Errorf("...: %w", err)` — создаёт новую ошибку, оборачивая исходную. `%w` позволяет сохранить оригинальную ошибку внутри новой.

- `db.Ping()` — отправляет тестовый запрос к базе данных, чтобы убедиться, что соединение работает.

---

## 6. Создаём репозиторий для почтовых ящиков

**Что такое репозиторий:**  
Репозиторий — это слой, который отвечает за работу с базой данных. Он скрывает детали SQL от остального кода.

**Создаём файл `internal/repository/mailbox_repo.go`:**
```go
package repository

import (
    "database/sql"
    "time"

    "github.com/google/uuid"

    "tempmail/internal/domain"
)

// MailboxRepository — репозиторий для работы с почтовыми ящиками
type MailboxRepository struct {
    db *sql.DB // Подключение к базе данных
}

// NewMailboxRepository создаёт новый репозиторий
func NewMailboxRepository(db *sql.DB) *MailboxRepository {
    return &MailboxRepository{db: db}
}

// Create создаёт новый почтовый ящик
func (r *MailboxRepository) Create(address string, ttl time.Duration) (*domain.Mailbox, error) {
    // Генерируем уникальный ID
    id := uuid.New().String()
    
    // Вычисляем время истечения
    now := time.Now()
    expiresAt := now.Add(ttl)

    // SQL-запрос для вставки записи
    // $1, $2, $3, $4 — это плейсхолдеры для параметров
    // Они защищают от SQL-инъекций
    query := `
        INSERT INTO mailboxes (id, address, created_at, expires_at, is_active)
        VALUES ($1, $2, $3, $4, $5)
    `

    // Выполняем запрос
    // Exec используется для запросов, которые не возвращают данные (INSERT, UPDATE, DELETE)
    _, err := r.db.Exec(query, id, address, now, expiresAt, true)
    if err != nil {
        return nil, err
    }

    // Возвращаем созданный ящик
    return &domain.Mailbox{
        ID:        id,
        Address:   address,
        CreatedAt: now,
        ExpiresAt: expiresAt,
        IsActive:  true,
    }, nil
}

// GetByID находит ящик по ID
func (r *MailboxRepository) GetByID(id string) (*domain.Mailbox, error) {
    // SQL-запрос для выборки одной записи
    query := `
        SELECT id, address, created_at, expires_at, is_active
        FROM mailboxes
        WHERE id = $1
    `

    // Создаём пустую структуру для результата
    mailbox := &domain.Mailbox{}

    // QueryRow выполняет запрос и возвращает одну строку
    // Scan читает значения из строки в поля структуры
    err := r.db.QueryRow(query, id).Scan(
        &mailbox.ID,
        &mailbox.Address,
        &mailbox.CreatedAt,
        &mailbox.ExpiresAt,
        &mailbox.IsActive,
    )

    // Проверяем ошибки
    if err == sql.ErrNoRows {
        // Специальная ошибка — запись не найдена
        return nil, nil
    }
    if err != nil {
        return nil, err
    }

    return mailbox, nil
}

// GetByAddress находит ящик по email-адресу
func (r *MailboxRepository) GetByAddress(address string) (*domain.Mailbox, error) {
    query := `
        SELECT id, address, created_at, expires_at, is_active
        FROM mailboxes
        WHERE address = $1 AND is_active = true
    `

    mailbox := &domain.Mailbox{}
    err := r.db.QueryRow(query, address).Scan(
        &mailbox.ID,
        &mailbox.Address,
        &mailbox.CreatedAt,
        &mailbox.ExpiresAt,
        &mailbox.IsActive,
    )

    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }

    return mailbox, nil
}

// Delete удаляет почтовый ящик
func (r *MailboxRepository) Delete(id string) error {
    query := `DELETE FROM mailboxes WHERE id = $1`
    _, err := r.db.Exec(query, id)
    return err
}

// DeleteExpired удаляет все истёкшие ящики
func (r *MailboxRepository) DeleteExpired() (int64, error) {
    query := `DELETE FROM mailboxes WHERE expires_at < NOW()`
    
    // Exec возвращает Result, из которого можно узнать количество затронутых строк
    result, err := r.db.Exec(query)
    if err != nil {
        return 0, err
    }

    // RowsAffected возвращает количество удалённых записей
    return result.RowsAffected()
}
```

**Разбор:**

- `uuid.New().String()` — генерирует новый UUID (уникальный идентификатор) и преобразует в строку.

- `time.Now().Add(ttl)` — добавляет длительность `ttl` к текущему времени.

- `$1, $2, $3...` — плейсхолдеры в PostgreSQL. Значения передаются отдельно от запроса, что защищает от SQL-инъекций (атак через ввод данных).

- `r.db.Exec(query, ...)` — выполняет SQL-запрос без возврата данных. Используется для INSERT, UPDATE, DELETE.

- `r.db.QueryRow(query, ...).Scan(...)` — выполняет запрос, возвращающий одну строку, и читает значения в переменные.
  - `&mailbox.ID` — передаём адрес поля, чтобы `Scan` мог записать туда значение

- `sql.ErrNoRows` — специальная ошибка, означающая "запись не найдена". Это не ошибка программы, просто данных нет.

---

## 7. Создаём репозиторий для сообщений

**Создаём файл `internal/repository/message_repo.go`:**
```go
package repository

import (
    "database/sql"
    "time"

    "github.com/google/uuid"

    "tempmail/internal/domain"
)

// MessageRepository — репозиторий для работы с письмами
type MessageRepository struct {
    db *sql.DB
}

// NewMessageRepository создаёт новый репозиторий
func NewMessageRepository(db *sql.DB) *MessageRepository {
    return &MessageRepository{db: db}
}

// Create создаёт новое письмо
func (r *MessageRepository) Create(msg *domain.Message) error {
    // Генерируем ID, если не задан
    if msg.ID == "" {
        msg.ID = uuid.New().String()
    }
    
    // Устанавливаем время получения
    if msg.ReceivedAt.IsZero() {
        msg.ReceivedAt = time.Now()
    }

    query := `
        INSERT INTO messages (id, mailbox_id, from_address, subject, body_text, body_html, received_at, is_read, is_spam)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
    `

    _, err := r.db.Exec(query,
        msg.ID,
        msg.MailboxID,
        msg.FromAddress,
        msg.Subject,
        msg.BodyText,
        msg.BodyHTML,
        msg.ReceivedAt,
        msg.IsRead,
        msg.IsSpam,
    )

    return err
}

// GetByMailboxID возвращает все письма для указанного ящика
func (r *MessageRepository) GetByMailboxID(mailboxID string) ([]*domain.Message, error) {
    query := `
        SELECT id, mailbox_id, from_address, subject, body_text, body_html, received_at, is_read, is_spam
        FROM messages
        WHERE mailbox_id = $1
        ORDER BY received_at DESC
    `

    // Query возвращает несколько строк
    rows, err := r.db.Query(query, mailboxID)
    if err != nil {
        return nil, err
    }
    // defer гарантирует, что rows.Close() выполнится при выходе из функции
    defer rows.Close()

    // Создаём срез для результатов
    var messages []*domain.Message

    // Перебираем все строки результата
    for rows.Next() {
        msg := &domain.Message{}
        err := rows.Scan(
            &msg.ID,
            &msg.MailboxID,
            &msg.FromAddress,
            &msg.Subject,
            &msg.BodyText,
            &msg.BodyHTML,
            &msg.ReceivedAt,
            &msg.IsRead,
            &msg.IsSpam,
        )
        if err != nil {
            return nil, err
        }
        // Добавляем письмо в срез
        messages = append(messages, msg)
    }

    // Проверяем ошибки, возникшие при переборе
    if err = rows.Err(); err != nil {
        return nil, err
    }

    return messages, nil
}

// GetByID находит письмо по ID
func (r *MessageRepository) GetByID(id string) (*domain.Message, error) {
    query := `
        SELECT id, mailbox_id, from_address, subject, body_text, body_html, received_at, is_read, is_spam
        FROM messages
        WHERE id = $1
    `

    msg := &domain.Message{}
    err := r.db.QueryRow(query, id).Scan(
        &msg.ID,
        &msg.MailboxID,
        &msg.FromAddress,
        &msg.Subject,
        &msg.BodyText,
        &msg.BodyHTML,
        &msg.ReceivedAt,
        &msg.IsRead,
        &msg.IsSpam,
    )

    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }

    return msg, nil
}

// MarkAsRead помечает письмо как прочитанное
func (r *MessageRepository) MarkAsRead(id string) error {
    query := `UPDATE messages SET is_read = true WHERE id = $1`
    _, err := r.db.Exec(query, id)
    return err
}

// Delete удаляет письмо
func (r *MessageRepository) Delete(id string) error {
    query := `DELETE FROM messages WHERE id = $1`
    _, err := r.db.Exec(query, id)
    return err
}

// CountByMailboxID возвращает количество писем в ящике
func (r *MessageRepository) CountByMailboxID(mailboxID string) (int, error) {
    query := `SELECT COUNT(*) FROM messages WHERE mailbox_id = $1`
    
    var count int
    err := r.db.QueryRow(query, mailboxID).Scan(&count)
    if err != nil {
        return 0, err
    }
    
    return count, nil
}
```

**Разбор:**

- `msg.ReceivedAt.IsZero()` — проверяет, установлено ли время. Нулевое время (time.Time{}) считается "пустым".

- `r.db.Query(query, ...)` — выполняет запрос, возвращающий несколько строк.

- `defer rows.Close()` — `defer` откладывает выполнение до выхода из функции. Гарантирует, что ресурсы будут освобождены, даже если произойдёт ошибка.

- `rows.Next()` — переходит к следующей строке результата. Возвращает `true`, пока есть строки.

- `rows.Err()` — возвращает ошибку, если она произошла во время перебора строк.

- `ORDER BY received_at DESC` — сортировка по дате получения, новые письма первыми.

---

## 8. Применяем миграции

**Что делаем:**  
Выполняем SQL-скрипт, чтобы создать таблицы в базе данных.

**Выполняем миграцию из PowerShell:**
```powershell
Get-Content migrations/001_init.up.sql | docker exec -i tempmail-postgres psql -U postgres -d tempmail
```

**Проверяем таблицы:**
```powershell
docker exec -it tempmail-postgres psql -U postgres -d tempmail -c "\dt"
```

**Результат:**
```
             List of relations
 Schema |    Name     | Type  |  Owner   
--------+-------------+-------+----------
 public | attachments | table | postgres
 public | mailboxes   | table | postgres
 public | messages    | table | postgres
```


---

## 9. Тестируем подключение

**Обновляем `cmd/api/main.go`:**
```go
package main

import (
    "fmt"
    "log"

    "tempmail/internal/config"
    "tempmail/internal/repository"
)

func main() {
    // Загружаем конфигурацию
    cfg, err := config.Load()
    if err != nil {
        log.Fatal("Ошибка загрузки конфигурации:", err)
    }

    fmt.Println("=== TempMail API Server ===")

    // Подключаемся к базе данных
    fmt.Println("Подключение к PostgreSQL...")
    db, err := repository.NewPostgresDB(cfg.Database)
    if err != nil {
        log.Fatal("Ошибка подключения к БД:", err)
    }
    // defer гарантирует закрытие соединения при выходе из main
    defer db.Close()

    fmt.Println("Подключение успешно!")

    // Создаём репозиторий
    mailboxRepo := repository.NewMailboxRepository(db.DB)

    // Тестируем создание ящика
    fmt.Println("\nСоздаём тестовый ящик...")
    mailbox, err := mailboxRepo.Create("test123@tempmail.dev", cfg.Mail.DefaultTTL)
    if err != nil {
        log.Fatal("Ошибка создания ящика:", err)
    }
    fmt.Printf("Создан ящик: %s (ID: %s)\n", mailbox.Address, mailbox.ID)
    fmt.Printf("Истекает: %s\n", mailbox.ExpiresAt.Format("2006-01-02 15:04:05"))

    // Проверяем чтение
    fmt.Println("\nЧитаем ящик из базы...")
    found, err := mailboxRepo.GetByID(mailbox.ID)
    if err != nil {
        log.Fatal("Ошибка чтения:", err)
    }
    if found != nil {
        fmt.Printf("Найден: %s, активен: %v\n", found.Address, found.IsActive)
    }

    // Удаляем тестовый ящик
    fmt.Println("\nУдаляем тестовый ящик...")
    err = mailboxRepo.Delete(mailbox.ID)
    if err != nil {
        log.Fatal("Ошибка удаления:", err)
    }
    fmt.Println("Ящик удалён!")
}
```

**Запускаем:**
```bash
go run cmd/api/main.go
```

**Ожидаемый результат:**
```
=== TempMail API Server ===
Подключение к PostgreSQL...
Подключение успешно!

Создаём тестовый ящик...
Создан ящик: test123@tempmail.dev (ID: abc123-...)
Истекает: 2024-01-15 12:30:00

Читаем ящик из базы...
Найден: test123@tempmail.dev, активен: true

Удаляем тестовый ящик...
Ящик удалён!
```

---

## Словарь терминов

| Термин | Объяснение |
|--------|------------|
| **CRUD** | Create, Read, Update, Delete — базовые операции с данными |
| **Миграция** | SQL-скрипт для изменения структуры базы данных |
| **Репозиторий** | Слой, отвечающий за работу с базой данных |
| **UUID** | Универсальный уникальный идентификатор (128-битное число) |
| **Плейсхолдер** | Заглушка в SQL ($1, $2), куда подставляются значения |
| **defer** | Откладывает выполнение до выхода из функции |
| **SQL-инъекция** | Атака через ввод вредоносного SQL-кода |

---

## Что мы узнали

- Как запустить PostgreSQL через Docker
- Как создавать модели данных (структуры)
- Как писать SQL-миграции
- Как подключаться к PostgreSQL из Go
- Как создавать репозитории для работы с БД
- Что такое плейсхолдеры и зачем они нужны
- Как использовать `defer` для освобождения ресурсов
- Как выполнять CRUD-операции

---

[Следующая глава: Создание REST API с Fiber](./04-rest-api.md)

[Вернуться к оглавлению](./README.md)

