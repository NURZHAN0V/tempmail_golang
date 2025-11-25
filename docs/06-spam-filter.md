# Глава 6: Спам-фильтр и очистка данных

В этой главе мы создадим базовый спам-фильтр и систему автоматической очистки устаревших данных.

---

## 1. Зачем нужен спам-фильтр

**Проблема:**  
Временные email-адреса часто используются для регистрации на сайтах. Спамеры могут отправлять на них нежелательные письма, засоряя базу данных.

**Решение:**  
Создадим простой спам-фильтр, который проверяет письма по нескольким критериям:
- Ключевые слова в теме и теле письма
- Размер письма
- Подозрительные отправители

---

## 2. Создаём спам-фильтр

**Создаём файл `internal/spam/filter.go`:**
```go
package spam

import (
    "strings"

    "tempmail/internal/domain"
)

// SpamFilter — фильтр для обнаружения спама
type SpamFilter struct {
    keywords        []string // Спам-слова
    blockedDomains  []string // Заблокированные домены отправителей
    maxSubjectLen   int      // Максимальная длина темы
    maxBodyLen      int      // Максимальная длина тела
}

// NewSpamFilter создаёт новый спам-фильтр с настройками по умолчанию
func NewSpamFilter() *SpamFilter {
    return &SpamFilter{
        // Типичные спам-слова (можно расширить)
        keywords: []string{
            "viagra",
            "casino",
            "lottery",
            "winner",
            "congratulations",
            "free money",
            "click here",
            "unsubscribe",
            "buy now",
            "limited time",
            "act now",
            "urgent",
            "make money",
            "earn cash",
            "work from home",
            "no obligation",
            "risk free",
            "credit card",
            "bitcoin",
            "crypto",
        },
        // Домены, известные как источники спама
        blockedDomains: []string{
            "spam.com",
            "junk.mail",
            "fake.sender",
        },
        maxSubjectLen: 500,         // Макс. длина темы
        maxBodyLen:    1024 * 1024, // 1 MB
    }
}

// Check проверяет письмо на спам
// Возвращает true, если письмо является спамом, и причину
func (f *SpamFilter) Check(msg *domain.Message) (bool, string) {
    // Проверка 1: Заблокированные домены
    senderDomain := extractDomain(msg.FromAddress)
    for _, blocked := range f.blockedDomains {
        if strings.EqualFold(senderDomain, blocked) {
            return true, "заблокированный домен отправителя"
        }
    }

    // Проверка 2: Слишком длинная тема
    if len(msg.Subject) > f.maxSubjectLen {
        return true, "слишком длинная тема письма"
    }

    // Проверка 3: Слишком большое тело
    bodyLen := len(msg.BodyText) + len(msg.BodyHTML)
    if bodyLen > f.maxBodyLen {
        return true, "слишком большой размер письма"
    }

    // Проверка 4: Спам-слова в теме
    subjectLower := strings.ToLower(msg.Subject)
    for _, keyword := range f.keywords {
        if strings.Contains(subjectLower, keyword) {
            return true, "спам-слово в теме: " + keyword
        }
    }

    // Проверка 5: Спам-слова в теле (только текстовая часть)
    bodyLower := strings.ToLower(msg.BodyText)
    spamWordCount := 0
    for _, keyword := range f.keywords {
        if strings.Contains(bodyLower, keyword) {
            spamWordCount++
        }
    }
    // Если найдено 3+ спам-слова — это спам
    if spamWordCount >= 3 {
        return true, "множество спам-слов в тексте"
    }

    // Проверка 6: Пустая тема и отправитель
    if msg.Subject == "" && msg.FromAddress == "" {
        return true, "пустая тема и отправитель"
    }

    // Проверка 7: Слишком много ссылок
    linkCount := strings.Count(bodyLower, "http://") + strings.Count(bodyLower, "https://")
    if linkCount > 10 {
        return true, "слишком много ссылок"
    }

    // Письмо не является спамом
    return false, ""
}

// AddKeyword добавляет новое спам-слово
func (f *SpamFilter) AddKeyword(keyword string) {
    f.keywords = append(f.keywords, strings.ToLower(keyword))
}

// AddBlockedDomain добавляет домен в чёрный список
func (f *SpamFilter) AddBlockedDomain(domain string) {
    f.blockedDomains = append(f.blockedDomains, strings.ToLower(domain))
}

// extractDomain извлекает домен из email-адреса
func extractDomain(email string) string {
    // email: user@domain.com -> domain.com
    parts := strings.Split(email, "@")
    if len(parts) != 2 {
        return ""
    }
    return strings.ToLower(parts[1])
}
```

**Разбор:**

- `strings.EqualFold(a, b)` — сравнивает строки без учёта регистра (case-insensitive).

- `strings.ToLower(s)` — преобразует строку в нижний регистр.

- `strings.Contains(s, substr)` — проверяет, содержит ли строка подстроку.

- `strings.Count(s, substr)` — считает количество вхождений подстроки.

- `strings.Split(s, sep)` — разбивает строку по разделителю и возвращает срез.

---

## 3. Интегрируем фильтр в сервис сообщений

**Обновляем `internal/service/message_service.go`:**
```go
package service

import (
    "errors"
    "log"

    "tempmail/internal/config"
    "tempmail/internal/domain"
    "tempmail/internal/repository"
    "tempmail/internal/spam"
)

// Ошибки сервиса
var (
    ErrMessageNotFound = errors.New("письмо не найдено")
    ErrMailboxFull     = errors.New("ящик переполнен")
    ErrMessageTooLarge = errors.New("письмо слишком большое")
)

// MessageService — сервис для работы с письмами
type MessageService struct {
    msgRepo     *repository.MessageRepository
    mailboxRepo *repository.MailboxRepository
    limits      config.LimitsConfig
    spamFilter  *spam.SpamFilter // Добавляем спам-фильтр
}

// NewMessageService создаёт новый сервис
func NewMessageService(
    msgRepo *repository.MessageRepository,
    mailboxRepo *repository.MailboxRepository,
    limits config.LimitsConfig,
) *MessageService {
    return &MessageService{
        msgRepo:     msgRepo,
        mailboxRepo: mailboxRepo,
        limits:      limits,
        spamFilter:  spam.NewSpamFilter(), // Создаём фильтр
    }
}

// Create создаёт новое письмо
func (s *MessageService) Create(msg *domain.Message) error {
    // Проверяем существование ящика
    mailbox, err := s.mailboxRepo.GetByID(msg.MailboxID)
    if err != nil {
        return err
    }
    if mailbox == nil {
        return ErrMailboxNotFound
    }

    // Проверяем, не истёк ли срок ящика
    if mailbox.IsExpired() {
        return ErrMailboxExpired
    }

    // Проверяем количество писем в ящике
    count, err := s.msgRepo.CountByMailboxID(msg.MailboxID)
    if err != nil {
        return err
    }
    if count >= s.limits.MaxMessagesPerMailbox {
        return ErrMailboxFull
    }

    // Проверяем размер письма
    messageSize := len(msg.BodyText) + len(msg.BodyHTML)
    if messageSize > s.limits.MaxMessageSize {
        return ErrMessageTooLarge
    }

    // Проверяем на спам
    isSpam, reason := s.spamFilter.Check(msg)
    if isSpam {
        log.Printf("Письмо помечено как спам: %s", reason)
        msg.IsSpam = true
    }

    return s.msgRepo.Create(msg)
}

// ... остальные методы без изменений
```

**Что изменилось:**
- Добавили поле `spamFilter` в структуру сервиса
- В методе `Create` проверяем письмо на спам
- Если письмо — спам, помечаем `msg.IsSpam = true`

---

## 4. Создаём планировщик задач

**Что такое планировщик:**  
Планировщик (scheduler) выполняет задачи по расписанию. Нам нужно периодически удалять устаревшие ящики и письма.

**Создаём файл `internal/service/scheduler.go`:**
```go
package service

import (
    "log"
    "time"

    "tempmail/internal/repository"
)

// Scheduler — планировщик фоновых задач
type Scheduler struct {
    mailboxRepo *repository.MailboxRepository
    interval    time.Duration // Интервал между запусками
    stopChan    chan struct{} // Канал для остановки
}

// NewScheduler создаёт новый планировщик
func NewScheduler(mailboxRepo *repository.MailboxRepository, interval time.Duration) *Scheduler {
    return &Scheduler{
        mailboxRepo: mailboxRepo,
        interval:    interval,
        stopChan:    make(chan struct{}),
    }
}

// Start запускает планировщик
func (s *Scheduler) Start() {
    log.Printf("Планировщик запущен, интервал: %s", s.interval)
    
    // Создаём тикер, который срабатывает каждый interval
    ticker := time.NewTicker(s.interval)
    defer ticker.Stop()

    // Выполняем очистку сразу при запуске
    s.cleanup()

    // Бесконечный цикл
    for {
        select {
        case <-ticker.C:
            // Тикер сработал — выполняем очистку
            s.cleanup()
        case <-s.stopChan:
            // Получен сигнал остановки
            log.Println("Планировщик остановлен")
            return
        }
    }
}

// Stop останавливает планировщик
func (s *Scheduler) Stop() {
    close(s.stopChan)
}

// cleanup удаляет устаревшие данные
func (s *Scheduler) cleanup() {
    log.Println("Запуск очистки устаревших данных...")

    // Удаляем истёкшие ящики (письма удалятся каскадно)
    deleted, err := s.mailboxRepo.DeleteExpired()
    if err != nil {
        log.Printf("Ошибка очистки: %v", err)
        return
    }

    if deleted > 0 {
        log.Printf("Удалено %d истёкших ящиков", deleted)
    } else {
        log.Println("Нет устаревших данных для удаления")
    }
}
```

**Разбор:**

- `time.NewTicker(interval)` — создаёт тикер, который отправляет сигнал в канал каждый `interval`.

- `defer ticker.Stop()` — останавливает тикер при выходе из функции.

- `select { ... }` — конструкция для работы с несколькими каналами. Ожидает, пока один из каналов не станет готов.

- `case <-ticker.C:` — срабатывает, когда тикер отправляет сигнал.

- `case <-s.stopChan:` — срабатывает, когда канал закрывается (вызван `Stop()`).

- `close(s.stopChan)` — закрывает канал. Все операции чтения из закрытого канала немедленно возвращаются.

---

## 5. Добавляем статистику очистки

**Создаём файл `internal/service/stats.go`:**
```go
package service

import (
    "sync"
    "time"
)

// Stats хранит статистику работы сервиса
type Stats struct {
    mu                sync.RWMutex // Мьютекс для безопасного доступа
    TotalMailboxes    int64        // Всего создано ящиков
    TotalMessages     int64        // Всего получено писем
    TotalSpamMessages int64        // Всего спам-писем
    DeletedMailboxes  int64        // Удалено ящиков
    LastCleanup       time.Time    // Время последней очистки
}

// GlobalStats — глобальная статистика
var GlobalStats = &Stats{}

// IncrementMailboxes увеличивает счётчик ящиков
func (s *Stats) IncrementMailboxes() {
    s.mu.Lock()         // Блокируем для записи
    defer s.mu.Unlock() // Разблокируем при выходе
    s.TotalMailboxes++
}

// IncrementMessages увеличивает счётчик писем
func (s *Stats) IncrementMessages(isSpam bool) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.TotalMessages++
    if isSpam {
        s.TotalSpamMessages++
    }
}

// AddDeletedMailboxes добавляет к счётчику удалённых ящиков
func (s *Stats) AddDeletedMailboxes(count int64) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.DeletedMailboxes += count
    s.LastCleanup = time.Now()
}

// GetStats возвращает копию статистики
func (s *Stats) GetStats() Stats {
    s.mu.RLock()         // Блокируем для чтения
    defer s.mu.RUnlock() // Разблокируем при выходе
    return Stats{
        TotalMailboxes:    s.TotalMailboxes,
        TotalMessages:     s.TotalMessages,
        TotalSpamMessages: s.TotalSpamMessages,
        DeletedMailboxes:  s.DeletedMailboxes,
        LastCleanup:       s.LastCleanup,
    }
}
```

**Разбор:**

- `sync.RWMutex` — мьютекс для чтения и записи. Позволяет нескольким горутинам читать одновременно, но только одной — писать.

- `s.mu.Lock()` — блокирует мьютекс для записи. Другие горутины будут ждать.

- `s.mu.RLock()` — блокирует мьютекс для чтения. Несколько читателей могут работать одновременно.

- `defer s.mu.Unlock()` — гарантирует разблокировку при выходе из функции.

- `var GlobalStats = &Stats{}` — глобальная переменная, доступная из любого места.

---

## 6. Обновляем планировщик для сбора статистики

**Обновляем `internal/service/scheduler.go`:**
```go
// cleanup удаляет устаревшие данные
func (s *Scheduler) cleanup() {
    log.Println("Запуск очистки устаревших данных...")

    deleted, err := s.mailboxRepo.DeleteExpired()
    if err != nil {
        log.Printf("Ошибка очистки: %v", err)
        return
    }

    // Обновляем статистику
    GlobalStats.AddDeletedMailboxes(deleted)

    if deleted > 0 {
        log.Printf("Удалено %d истёкших ящиков", deleted)
    } else {
        log.Println("Нет устаревших данных для удаления")
    }
}
```

---

## 7. Добавляем эндпоинт статистики

**Обновляем `internal/handler/routes.go`:**
```go
package handler

import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/cors"
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/recover"

    "tempmail/internal/service"
)

// SetupRoutes настраивает все маршруты приложения
func SetupRoutes(
    app *fiber.App,
    mailboxHandler *MailboxHandler,
    messageHandler *MessageHandler,
) {
    // ... middleware (без изменений)
    app.Use(logger.New())
    app.Use(recover.New())
    app.Use(cors.New(cors.Config{
        AllowOrigins: "*",
        AllowMethods: "GET,POST,PUT,DELETE,OPTIONS",
        AllowHeaders: "Content-Type,Authorization",
    }))

    // Группа маршрутов API v1
    api := app.Group("/api/v1")

    // Маршруты для почтовых ящиков
    mailbox := api.Group("/mailbox")
    mailbox.Post("/", mailboxHandler.Create)
    mailbox.Get("/:id", mailboxHandler.Get)
    mailbox.Delete("/:id", mailboxHandler.Delete)

    // Маршруты для писем
    mailbox.Get("/:id/messages", messageHandler.GetMessages)
    mailbox.Get("/:id/messages/:mid", messageHandler.GetMessage)
    mailbox.Delete("/:id/messages/:mid", messageHandler.DeleteMessage)

    // Маршрут для проверки здоровья
    app.Get("/health", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{
            "status": "ok",
        })
    })

    // Маршрут для статистики
    app.Get("/stats", func(c *fiber.Ctx) error {
        stats := service.GlobalStats.GetStats()
        return c.JSON(fiber.Map{
            "total_mailboxes":    stats.TotalMailboxes,
            "total_messages":     stats.TotalMessages,
            "total_spam":         stats.TotalSpamMessages,
            "deleted_mailboxes":  stats.DeletedMailboxes,
            "last_cleanup":       stats.LastCleanup.Format("2006-01-02 15:04:05"),
        })
    })
}
```

---

## 8. Обновляем главный файл

**Обновляем `cmd/api/main.go` для запуска планировщика:**
```go
package main

import (
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"

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

    // Создаём и запускаем планировщик (очистка каждые 5 минут)
    scheduler := service.NewScheduler(mailboxRepo, 5*time.Minute)
    go scheduler.Start()

    // Запускаем SMTP-сервер
    go func() {
        if err := smtpServer.Start(); err != nil {
            log.Printf("SMTP-сервер остановлен: %v", err)
        }
    }()

    // Запускаем HTTP-сервер
    go func() {
        addr := fmt.Sprintf(":%d", cfg.Server.HTTPPort)
        if err := app.Listen(addr); err != nil {
            log.Printf("HTTP-сервер остановлен: %v", err)
        }
    }()

    fmt.Printf("\nHTTP API: http://localhost:%d\n", cfg.Server.HTTPPort)
    fmt.Printf("SMTP: localhost:%d\n", cfg.Server.SMTPPort)
    fmt.Printf("Статистика: http://localhost:%d/stats\n", cfg.Server.HTTPPort)
    fmt.Println("\nНажмите Ctrl+C для остановки")

    // Ожидаем сигнал завершения
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    fmt.Println("\nОстановка серверов...")
    scheduler.Stop()
    smtpServer.Close()
    app.Shutdown()
}
```

---

## 9. Тестируем спам-фильтр

**Запускаем сервер:**
```bash
go run cmd/api/main.go
```

**Создаём ящик:**
```bash
curl -X POST http://localhost:8080/api/v1/mailbox -d '{"address": "test"}'
```

**Отправляем обычное письмо:**
```bash
swaks --to test@tempmail.dev \
      --from sender@example.com \
      --server localhost:2525 \
      --header "Subject: Привет" \
      --body "Обычное письмо"
```

**Отправляем спам-письмо:**
```bash
swaks --to test@tempmail.dev \
      --from sender@spam.com \
      --server localhost:2525 \
      --header "Subject: FREE MONEY! WINNER! CASINO!" \
      --body "Click here to win free money! Buy now! Limited time offer!"
```

**Проверяем письма:**
```bash
curl http://localhost:8080/api/v1/mailbox/{id}/messages
```

Спам-письмо будет иметь `"is_spam": true`.

**Проверяем статистику:**
```bash
curl http://localhost:8080/stats
```

---

## 10. Настройка интервала очистки через конфигурацию

**Обновляем `internal/config/config.go`:**
```go
// MailConfig — настройки почтовых ящиков
type MailConfig struct {
    Domain          string        `envconfig:"MAIL_DOMAIN" default:"tempmail.dev"`
    DefaultTTL      time.Duration `envconfig:"DEFAULT_TTL" default:"1h"`
    MaxTTL          time.Duration `envconfig:"MAX_TTL" default:"24h"`
    CleanupInterval time.Duration `envconfig:"CLEANUP_INTERVAL" default:"5m"` // Добавляем
}
```

**Обновляем `.env`:**
```
CLEANUP_INTERVAL=5m
```

**Обновляем `cmd/api/main.go`:**
```go
// Создаём и запускаем планировщик
scheduler := service.NewScheduler(mailboxRepo, cfg.Mail.CleanupInterval)
go scheduler.Start()
```

---

## Словарь терминов

| Термин | Объяснение |
|--------|------------|
| **Спам-фильтр** | Система для обнаружения нежелательных писем |
| **Планировщик** | Компонент, выполняющий задачи по расписанию |
| **Мьютекс** | Механизм синхронизации для защиты данных от одновременного доступа |
| **Тикер** | Таймер, срабатывающий с заданным интервалом |
| **Канал** | Механизм общения между горутинами |
| **select** | Конструкция для работы с несколькими каналами |

---

## Что мы узнали

- Как создать базовый спам-фильтр
- Как проверять письма по ключевым словам
- Как создать планировщик фоновых задач
- Как использовать тикеры и каналы
- Как безопасно работать с данными из нескольких горутин (мьютексы)
- Как собирать и отображать статистику

---

[Следующая глава: Docker и деплой](./07-docker-deploy.md)

[Вернуться к оглавлению](./README.md)

