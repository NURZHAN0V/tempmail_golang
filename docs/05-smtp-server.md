# Глава 5: Реализация SMTP-сервера

В этой главе мы создадим SMTP-сервер для приёма входящих писем. Это ключевая часть сервиса временных email-адресов.

---

## 1. Что такое SMTP

**Простое объяснение:**  
SMTP (Simple Mail Transfer Protocol) — это протокол для отправки и приёма электронной почты. Когда кто-то отправляет письмо на адрес `abc123@tempmail.dev`, его почтовый сервер находит наш SMTP-сервер и передаёт письмо.

**Как это работает:**
1. Отправитель пишет письмо на `abc123@tempmail.dev`
2. Его почтовый сервер ищет MX-запись домена `tempmail.dev`
3. MX-запись указывает на наш SMTP-сервер
4. Почтовый сервер отправителя подключается к нашему серверу
5. Наш сервер принимает письмо и сохраняет в базу данных

**Важно:**  
Для работы в продакшене нужно настроить DNS-записи (MX, SPF, DKIM). В этом руководстве мы создадим сервер, который можно тестировать локально.

---

## 2. Устанавливаем библиотеку go-smtp

**Что делаем:**  
Библиотека `go-smtp` позволяет легко создать SMTP-сервер на Go.

**Команда:**
```bash
go get github.com/emersion/go-smtp
go get github.com/emersion/go-message
```

---

## 3. Создаём SMTP-бэкенд

**Что такое бэкенд:**  
SMTP-бэкенд — это набор функций, которые вызываются при получении письма. Мы определяем, что делать с каждым письмом.

**Создаём файл `internal/smtp/backend.go`:**
```go
package smtp

import (
    "io"
    "log"

    "github.com/emersion/go-smtp"

    "tempmail/internal/service"
)

// Backend реализует интерфейс smtp.Backend
// Он создаёт сессии для каждого входящего соединения
type Backend struct {
    mailboxService *service.MailboxService // Сервис для проверки ящиков
    messageService *service.MessageService // Сервис для сохранения писем
    domain         string                  // Наш домен (tempmail.dev)
}

// NewBackend создаёт новый SMTP-бэкенд
func NewBackend(
    mailboxService *service.MailboxService,
    messageService *service.MessageService,
    domain string,
) *Backend {
    return &Backend{
        mailboxService: mailboxService,
        messageService: messageService,
        domain:         domain,
    }
}

// NewSession создаёт новую сессию для входящего соединения
// Вызывается при каждом новом подключении к SMTP-серверу
func (b *Backend) NewSession(c *smtp.Conn) (smtp.Session, error) {
    log.Printf("Новое SMTP-соединение от %s", c.Hostname())
    
    return &Session{
        backend: b,
    }, nil
}
```

**Разбор:**

- `smtp.Backend` — интерфейс из библиотеки go-smtp. Мы реализуем его метод `NewSession`.

- **Интерфейс в Go** — это набор методов, которые тип должен реализовать. Если структура имеет все методы интерфейса, она автоматически его реализует.

- `*smtp.Conn` — соединение с клиентом (почтовым сервером отправителя).

- `c.Hostname()` — имя хоста клиента.

---

## 4. Создаём SMTP-сессию

**Что такое сессия:**  
Сессия обрабатывает одно входящее письмо. Она получает информацию об отправителе, получателе и само письмо.

**Создаём файл `internal/smtp/session.go`:**
```go
package smtp

import (
    "bytes"
    "fmt"
    "io"
    "log"
    "net/mail"
    "strings"

    "github.com/emersion/go-smtp"

    "tempmail/internal/domain"
)

// Session обрабатывает одну SMTP-сессию (одно письмо)
type Session struct {
    backend *Backend   // Ссылка на бэкенд
    from    string     // Адрес отправителя
    to      []string   // Адреса получателей
}

// AuthPlain обрабатывает PLAIN-аутентификацию
// Мы не требуем аутентификацию, поэтому всегда возвращаем nil
func (s *Session) AuthPlain(username, password string) error {
    // Аутентификация не требуется для приёма писем
    return nil
}

// Mail вызывается, когда клиент сообщает адрес отправителя (MAIL FROM)
func (s *Session) Mail(from string, opts *smtp.MailOptions) error {
    log.Printf("MAIL FROM: %s", from)
    s.from = from
    return nil
}

// Rcpt вызывается для каждого получателя (RCPT TO)
// Здесь мы проверяем, существует ли почтовый ящик
func (s *Session) Rcpt(to string, opts *smtp.RcptOptions) error {
    log.Printf("RCPT TO: %s", to)

    // Извлекаем email из формата "Name <email@domain.com>"
    address := extractEmail(to)

    // Проверяем, что письмо для нашего домена
    if !strings.HasSuffix(address, "@"+s.backend.domain) {
        return fmt.Errorf("мы не принимаем письма для домена %s", address)
    }

    // Проверяем, существует ли ящик
    mailbox, err := s.backend.mailboxService.GetByAddress(address)
    if err != nil {
        log.Printf("Ошибка проверки ящика: %v", err)
        return &smtp.SMTPError{
            Code:    550,
            Message: "Почтовый ящик не найден",
        }
    }
    if mailbox == nil {
        return &smtp.SMTPError{
            Code:    550,
            Message: "Почтовый ящик не существует",
        }
    }

    // Добавляем получателя
    s.to = append(s.to, address)
    return nil
}

// Data вызывается, когда клиент отправляет содержимое письма
// r — это Reader, из которого можно прочитать письмо
func (s *Session) Data(r io.Reader) error {
    log.Println("Получение данных письма...")

    // Читаем всё письмо в буфер
    var buf bytes.Buffer
    _, err := buf.ReadFrom(r)
    if err != nil {
        return err
    }

    // Парсим письмо
    msg, err := mail.ReadMessage(&buf)
    if err != nil {
        log.Printf("Ошибка парсинга письма: %v", err)
        return err
    }

    // Извлекаем заголовки
    subject := msg.Header.Get("Subject")
    from := msg.Header.Get("From")
    
    // Если From пустой, используем адрес из MAIL FROM
    if from == "" {
        from = s.from
    }

    // Читаем тело письма
    body, err := io.ReadAll(msg.Body)
    if err != nil {
        return err
    }

    log.Printf("Письмо от %s, тема: %s", from, subject)

    // Сохраняем письмо для каждого получателя
    for _, to := range s.to {
        err := s.saveMessage(to, from, subject, string(body))
        if err != nil {
            log.Printf("Ошибка сохранения письма для %s: %v", to, err)
            // Продолжаем для других получателей
        }
    }

    return nil
}

// saveMessage сохраняет письмо в базу данных
func (s *Session) saveMessage(to, from, subject, body string) error {
    // Находим ящик по адресу
    mailbox, err := s.backend.mailboxService.GetByAddress(to)
    if err != nil {
        return err
    }
    if mailbox == nil {
        return fmt.Errorf("ящик %s не найден", to)
    }

    // Создаём сообщение
    message := &domain.Message{
        MailboxID:   mailbox.ID,
        FromAddress: extractEmail(from),
        Subject:     subject,
        BodyText:    body,
        BodyHTML:    "", // TODO: парсинг HTML
        IsRead:      false,
        IsSpam:      false,
    }

    return s.backend.messageService.Create(message)
}

// Reset вызывается для сброса сессии
func (s *Session) Reset() {
    s.from = ""
    s.to = nil
}

// Logout вызывается при завершении сессии
func (s *Session) Logout() error {
    log.Println("SMTP-сессия завершена")
    return nil
}

// extractEmail извлекает email из строки вида "Name <email@domain.com>"
func extractEmail(s string) string {
    // Если есть угловые скобки, извлекаем email из них
    if start := strings.Index(s, "<"); start != -1 {
        if end := strings.Index(s, ">"); end != -1 {
            return strings.TrimSpace(s[start+1 : end])
        }
    }
    // Иначе возвращаем как есть
    return strings.TrimSpace(s)
}
```

**Разбор:**

- `smtp.Session` — интерфейс сессии. Мы реализуем его методы: `Mail`, `Rcpt`, `Data`, `Reset`, `Logout`.

- `func (s *Session) Mail(...)` — вызывается, когда клиент отправляет команду `MAIL FROM:`. Сохраняем адрес отправителя.

- `func (s *Session) Rcpt(...)` — вызывается для команды `RCPT TO:`. Проверяем, существует ли ящик.

- `func (s *Session) Data(r io.Reader)` — вызывается, когда клиент отправляет само письмо. `io.Reader` — интерфейс для чтения данных.

- `bytes.Buffer` — буфер для накопления данных. Метод `ReadFrom` читает всё из Reader.

- `mail.ReadMessage(...)` — стандартная библиотека Go для парсинга email-сообщений.

- `msg.Header.Get("Subject")` — получаем заголовок письма.

- `io.ReadAll(msg.Body)` — читаем всё тело письма.

- `&smtp.SMTPError{Code: 550, ...}` — SMTP-ошибка с кодом. 550 означает "ящик не найден".

- `strings.HasSuffix(address, "@"+s.backend.domain)` — проверяем, что адрес заканчивается на наш домен.

---

## 5. Добавляем метод GetByAddress в сервис

**Обновляем `internal/service/mailbox_service.go`:**
```go
// GetByAddress возвращает ящик по email-адресу
func (s *MailboxService) GetByAddress(address string) (*domain.Mailbox, error) {
    mailbox, err := s.repo.GetByAddress(address)
    if err != nil {
        return nil, err
    }
    if mailbox == nil {
        return nil, nil // Ящик не найден — это не ошибка
    }

    // Проверяем, не истёк ли срок
    if mailbox.IsExpired() {
        return nil, nil // Истёкший ящик считаем несуществующим
    }

    return mailbox, nil
}
```

---

## 6. Создаём SMTP-сервер

**Создаём файл `internal/smtp/server.go`:**
```go
package smtp

import (
    "fmt"
    "log"
    "time"

    "github.com/emersion/go-smtp"

    "tempmail/internal/config"
    "tempmail/internal/service"
)

// Server — SMTP-сервер для приёма писем
type Server struct {
    server  *smtp.Server
    backend *Backend
    config  config.ServerConfig
}

// NewServer создаёт новый SMTP-сервер
func NewServer(
    cfg config.ServerConfig,
    mailCfg config.MailConfig,
    mailboxService *service.MailboxService,
    messageService *service.MessageService,
) *Server {
    // Создаём бэкенд
    backend := NewBackend(mailboxService, messageService, mailCfg.Domain)

    // Создаём SMTP-сервер
    server := smtp.NewServer(backend)

    // Настраиваем параметры сервера
    server.Addr = fmt.Sprintf(":%d", cfg.SMTPPort)  // Адрес для прослушивания
    server.Domain = mailCfg.Domain                   // Наш домен
    server.ReadTimeout = 30 * time.Second            // Таймаут чтения
    server.WriteTimeout = 30 * time.Second           // Таймаут записи
    server.MaxMessageBytes = 10 * 1024 * 1024        // Макс. размер письма (10 MB)
    server.MaxRecipients = 10                        // Макс. получателей
    server.AllowInsecureAuth = true                  // Разрешаем без TLS (для разработки)

    return &Server{
        server:  server,
        backend: backend,
        config:  cfg,
    }
}

// Start запускает SMTP-сервер
func (s *Server) Start() error {
    log.Printf("SMTP-сервер запущен на порту %d", s.config.SMTPPort)
    log.Printf("Домен: %s", s.server.Domain)
    
    // ListenAndServe блокирует выполнение
    return s.server.ListenAndServe()
}

// Close останавливает SMTP-сервер
func (s *Server) Close() error {
    return s.server.Close()
}
```

**Разбор:**

- `smtp.NewServer(backend)` — создаёт SMTP-сервер с нашим бэкендом.

- `server.Addr = ":2525"` — сервер будет слушать порт 2525. Стандартный SMTP-порт 25 требует root-прав.

- `server.Domain` — домен, который сервер будет объявлять при подключении.

- `server.ReadTimeout`, `server.WriteTimeout` — таймауты для защиты от зависших соединений.

- `server.MaxMessageBytes` — максимальный размер письма в байтах.

- `server.AllowInsecureAuth = true` — разрешаем работу без TLS. В продакшене нужно настроить TLS.

- `server.ListenAndServe()` — запускает сервер и блокирует выполнение.

---

## 7. Создаём точку входа для SMTP-сервера

**Создаём файл `cmd/smtp/main.go`:**
```go
package main

import (
    "fmt"
    "log"

    "tempmail/internal/config"
    "tempmail/internal/repository"
    "tempmail/internal/service"
    smtpserver "tempmail/internal/smtp"
)

func main() {
    // Загружаем конфигурацию
    cfg, err := config.Load()
    if err != nil {
        log.Fatal("Ошибка загрузки конфигурации:", err)
    }

    fmt.Println("=== TempMail SMTP Server ===")

    // Подключаемся к базе данных
    fmt.Println("Подключение к PostgreSQL...")
    db, err := repository.NewPostgresDB(cfg.Database)
    if err != nil {
        log.Fatal("Ошибка подключения к БД:", err)
    }
    defer db.Close()
    fmt.Println("Подключение успешно!")

    // Создаём репозитории
    mailboxRepo := repository.NewMailboxRepository(db.DB)
    messageRepo := repository.NewMessageRepository(db.DB)

    // Создаём сервисы
    mailboxService := service.NewMailboxService(mailboxRepo, cfg.Mail)
    messageService := service.NewMessageService(messageRepo, mailboxRepo, cfg.Limits)

    // Создаём и запускаем SMTP-сервер
    server := smtpserver.NewServer(cfg.Server, cfg.Mail, mailboxService, messageService)

    fmt.Printf("\nSMTP-сервер запущен на порту %d\n", cfg.Server.SMTPPort)
    fmt.Printf("Домен: %s\n", cfg.Mail.Domain)
    fmt.Println("Нажмите Ctrl+C для остановки")

    if err := server.Start(); err != nil {
        log.Fatal("Ошибка SMTP-сервера:", err)
    }
}
```

**Разбор:**

- `smtpserver "tempmail/internal/smtp"` — импортируем наш пакет smtp с псевдонимом `smtpserver`, чтобы не путать с библиотекой `go-smtp`.

---

## 8. Тестируем SMTP-сервер

**Шаг 1: Запускаем API-сервер (в первом терминале):**
```bash
go run cmd/api/main.go
```

**Шаг 2: Создаём почтовый ящик:**
```bash
curl -X POST http://localhost:8080/api/v1/mailbox \
  -H "Content-Type: application/json" \
  -d '{"address": "test"}'
```

Запомните адрес из ответа (например, `test@tempmail.dev`).

**Шаг 3: Запускаем SMTP-сервер (во втором терминале):**
```bash
go run cmd/smtp/main.go
```

**Шаг 4: Отправляем тестовое письмо:**

Используем утилиту `swaks` (Swiss Army Knife for SMTP) или telnet:

**С помощью swaks (если установлен):**
```bash
swaks --to test@tempmail.dev \
      --from sender@example.com \
      --server localhost:2525 \
      --header "Subject: Тестовое письмо" \
      --body "Привет! Это тестовое письмо."
```

**С помощью telnet:**
```bash
telnet localhost 2525
```

Затем вводим команды:
```
HELO localhost
MAIL FROM:<sender@example.com>
RCPT TO:<test@tempmail.dev>
DATA
Subject: Тестовое письмо
From: sender@example.com
To: test@tempmail.dev

Привет! Это тестовое письмо.
.
QUIT
```

**Шаг 5: Проверяем, что письмо сохранилось:**
```bash
curl http://localhost:8080/api/v1/mailbox/{id}/messages
```

Замените `{id}` на ID ящика из шага 2.

---

## 9. Парсинг MIME-сообщений

**Что такое MIME:**  
MIME (Multipurpose Internet Mail Extensions) — стандарт для отправки вложений, HTML и текста в одном письме.

**Обновляем `internal/smtp/session.go` для поддержки MIME:**
```go
package smtp

import (
    "bytes"
    "fmt"
    "io"
    "log"
    "mime"
    "mime/multipart"
    "net/mail"
    "strings"

    gosmtp "github.com/emersion/go-smtp"

    "tempmail/internal/domain"
)

// ... (предыдущий код Session)

// Data вызывается, когда клиент отправляет содержимое письма
func (s *Session) Data(r io.Reader) error {
    log.Println("Получение данных письма...")

    // Читаем всё письмо в буфер
    var buf bytes.Buffer
    _, err := buf.ReadFrom(r)
    if err != nil {
        return err
    }

    // Парсим письмо
    msg, err := mail.ReadMessage(&buf)
    if err != nil {
        log.Printf("Ошибка парсинга письма: %v", err)
        return err
    }

    // Извлекаем заголовки
    subject := decodeHeader(msg.Header.Get("Subject"))
    from := msg.Header.Get("From")
    contentType := msg.Header.Get("Content-Type")

    if from == "" {
        from = s.from
    }

    // Парсим тело письма
    bodyText, bodyHTML := parseBody(msg.Body, contentType)

    log.Printf("Письмо от %s, тема: %s", from, subject)

    // Сохраняем письмо для каждого получателя
    for _, to := range s.to {
        err := s.saveMessage(to, from, subject, bodyText, bodyHTML)
        if err != nil {
            log.Printf("Ошибка сохранения письма для %s: %v", to, err)
        }
    }

    return nil
}

// saveMessage сохраняет письмо в базу данных
func (s *Session) saveMessage(to, from, subject, bodyText, bodyHTML string) error {
    mailbox, err := s.backend.mailboxService.GetByAddress(to)
    if err != nil {
        return err
    }
    if mailbox == nil {
        return fmt.Errorf("ящик %s не найден", to)
    }

    message := &domain.Message{
        MailboxID:   mailbox.ID,
        FromAddress: extractEmail(from),
        Subject:     subject,
        BodyText:    bodyText,
        BodyHTML:    bodyHTML,
        IsRead:      false,
        IsSpam:      false,
    }

    return s.backend.messageService.Create(message)
}

// parseBody парсит тело письма и извлекает текст и HTML
func parseBody(body io.Reader, contentType string) (text, html string) {
    // Если Content-Type не указан, считаем plain text
    if contentType == "" {
        data, _ := io.ReadAll(body)
        return string(data), ""
    }

    // Парсим Content-Type
    mediaType, params, err := mime.ParseMediaType(contentType)
    if err != nil {
        data, _ := io.ReadAll(body)
        return string(data), ""
    }

    // Если это multipart (письмо с несколькими частями)
    if strings.HasPrefix(mediaType, "multipart/") {
        boundary := params["boundary"]
        if boundary == "" {
            data, _ := io.ReadAll(body)
            return string(data), ""
        }

        // Читаем все части
        mr := multipart.NewReader(body, boundary)
        for {
            part, err := mr.NextPart()
            if err == io.EOF {
                break
            }
            if err != nil {
                break
            }

            partType := part.Header.Get("Content-Type")
            partData, _ := io.ReadAll(part)

            if strings.HasPrefix(partType, "text/plain") {
                text = string(partData)
            } else if strings.HasPrefix(partType, "text/html") {
                html = string(partData)
            }
        }
        return text, html
    }

    // Простое письмо (не multipart)
    data, _ := io.ReadAll(body)
    if strings.HasPrefix(mediaType, "text/html") {
        return "", string(data)
    }
    return string(data), ""
}

// decodeHeader декодирует заголовок письма (поддержка UTF-8)
func decodeHeader(s string) string {
    // Декодируем MIME-encoded слова (=?UTF-8?B?...?=)
    dec := new(mime.WordDecoder)
    decoded, err := dec.DecodeHeader(s)
    if err != nil {
        return s
    }
    return decoded
}
```

**Разбор:**

- `mime.ParseMediaType(contentType)` — парсит Content-Type и извлекает тип и параметры (например, boundary для multipart).

- `multipart.NewReader(body, boundary)` — создаёт Reader для чтения multipart-сообщений.

- `mr.NextPart()` — читает следующую часть multipart-сообщения.

- `mime.WordDecoder` — декодирует MIME-encoded заголовки (например, `=?UTF-8?B?0J/RgNC40LLQtdGC?=` → "Привет").

---

## 10. Запуск обоих серверов вместе

Для удобства можно запускать оба сервера из одного процесса.

**Обновляем `cmd/api/main.go`:**
```go
package main

import (
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"

    "github.com/gofiber/fiber/v2"

    "tempmail/internal/config"
    "tempmail/internal/handler"
    "tempmail/internal/repository"
    "tempmail/internal/service"
    smtpserver "tempmail/internal/smtp"
)

func main() {
    // Загружаем конфигурацию
    cfg, err := config.Load()
    if err != nil {
        log.Fatal("Ошибка загрузки конфигурации:", err)
    }

    fmt.Println("=== TempMail Server ===")

    // Подключаемся к базе данных
    fmt.Println("Подключение к PostgreSQL...")
    db, err := repository.NewPostgresDB(cfg.Database)
    if err != nil {
        log.Fatal("Ошибка подключения к БД:", err)
    }
    defer db.Close()
    fmt.Println("Подключение успешно!")

    // Создаём репозитории
    mailboxRepo := repository.NewMailboxRepository(db.DB)
    messageRepo := repository.NewMessageRepository(db.DB)

    // Создаём сервисы
    mailboxService := service.NewMailboxService(mailboxRepo, cfg.Mail)
    messageService := service.NewMessageService(messageRepo, mailboxRepo, cfg.Limits)

    // Создаём обработчики
    mailboxHandler := handler.NewMailboxHandler(mailboxService)
    messageHandler := handler.NewMessageHandler(messageService)

    // Создаём Fiber-приложение
    app := fiber.New(fiber.Config{
        AppName: "TempMail API",
    })

    // Настраиваем маршруты
    handler.SetupRoutes(app, mailboxHandler, messageHandler)

    // Создаём SMTP-сервер
    smtpServer := smtpserver.NewServer(cfg.Server, cfg.Mail, mailboxService, messageService)

    // Запускаем SMTP-сервер в отдельной горутине
    go func() {
        if err := smtpServer.Start(); err != nil {
            log.Printf("SMTP-сервер остановлен: %v", err)
        }
    }()

    // Запускаем HTTP-сервер в отдельной горутине
    go func() {
        addr := fmt.Sprintf(":%d", cfg.Server.HTTPPort)
        if err := app.Listen(addr); err != nil {
            log.Printf("HTTP-сервер остановлен: %v", err)
        }
    }()

    fmt.Printf("\nHTTP API: http://localhost:%d\n", cfg.Server.HTTPPort)
    fmt.Printf("SMTP: localhost:%d\n", cfg.Server.SMTPPort)
    fmt.Println("\nНажмите Ctrl+C для остановки")

    // Ожидаем сигнал завершения
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    fmt.Println("\nОстановка серверов...")
    smtpServer.Close()
    app.Shutdown()
}
```

**Разбор:**

- `go func() { ... }()` — запускает функцию в отдельной **горутине**. Горутина — это легковесный поток выполнения в Go.

- `make(chan os.Signal, 1)` — создаёт **канал** для сигналов. Каналы используются для общения между горутинами.

- `signal.Notify(quit, ...)` — подписываемся на системные сигналы (Ctrl+C, kill).

- `<-quit` — ожидаем значение из канала. Программа "зависает" здесь, пока не придёт сигнал.

- `syscall.SIGINT` — сигнал прерывания (Ctrl+C).
- `syscall.SIGTERM` — сигнал завершения (kill).

---

## Словарь терминов

| Термин | Объяснение |
|--------|------------|
| **SMTP** | Протокол для отправки и приёма email |
| **MX-запись** | DNS-запись, указывающая на почтовый сервер домена |
| **MIME** | Стандарт для вложений и форматирования email |
| **Multipart** | Письмо, состоящее из нескольких частей (текст + HTML + вложения) |
| **Горутина** | Легковесный поток выполнения в Go |
| **Канал** | Механизм общения между горутинами |
| **Бэкенд** | Код, обрабатывающий входящие соединения |
| **Сессия** | Обработка одного входящего письма |

---

## Что мы узнали

- Как работает протокол SMTP
- Как создать SMTP-сервер с помощью go-smtp
- Как реализовать бэкенд и сессию
- Как парсить email-сообщения
- Как обрабатывать MIME и multipart
- Как запускать несколько серверов в одном процессе
- Что такое горутины и каналы

---

[Следующая глава: Спам-фильтр и очистка данных](./06-spam-filter.md)

[Вернуться к оглавлению](./README.md)

