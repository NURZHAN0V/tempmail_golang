# Глава 4: Создание REST API с Fiber

В этой главе мы создадим REST API для работы с почтовыми ящиками и письмами, используя веб-фреймворк Fiber.

---

## 1. Что такое REST API

**Простое объяснение:**  
REST API — это способ общения между программами через HTTP-запросы. Клиент (браузер, мобильное приложение) отправляет запрос на сервер, сервер обрабатывает его и возвращает ответ в формате JSON.

**Основные HTTP-методы:**

| Метод | Назначение | Пример |
|-------|------------|--------|
| GET | Получить данные | Получить список писем |
| POST | Создать новые данные | Создать почтовый ящик |
| PUT | Обновить данные полностью | Обновить все поля письма |
| PATCH | Обновить данные частично | Пометить письмо как прочитанное |
| DELETE | Удалить данные | Удалить почтовый ящик |

---

## 2. Что такое Fiber

**Fiber** — это быстрый веб-фреймворк для Go, вдохновлённый Express.js из мира Node.js. Он простой в использовании и очень производительный.

**Почему Fiber:**
- Простой синтаксис, похожий на Express
- Высокая производительность
- Встроенная поддержка JSON
- Хорошая документация

---

## 3. Создаём сервисный слой

**Что такое сервис:**  
Сервис — это слой бизнес-логики. Он использует репозитории для работы с данными и содержит логику приложения.

**Создаём файл `internal/service/mailbox_service.go`:**
```go
package service

import (
    "errors"
    "fmt"
    "math/rand"
    "time"

    "tempmail/internal/config"
    "tempmail/internal/domain"
    "tempmail/internal/repository"
)

// Ошибки сервиса
var (
    ErrMailboxNotFound = errors.New("почтовый ящик не найден")
    ErrMailboxExpired  = errors.New("срок действия ящика истёк")
    ErrInvalidTTL      = errors.New("недопустимое время жизни")
)

// MailboxService — сервис для работы с почтовыми ящиками
type MailboxService struct {
    repo   *repository.MailboxRepository // Репозиторий для работы с БД
    config config.MailConfig             // Настройки почты
}

// NewMailboxService создаёт новый сервис
func NewMailboxService(repo *repository.MailboxRepository, cfg config.MailConfig) *MailboxService {
    return &MailboxService{
        repo:   repo,
        config: cfg,
    }
}

// Create создаёт новый почтовый ящик
// Если address пустой — генерируется случайный
func (s *MailboxService) Create(address string, ttl time.Duration) (*domain.Mailbox, error) {
    // Если адрес не указан — генерируем случайный
    if address == "" {
        address = s.generateRandomAddress()
    } else {
        // Добавляем домен к адресу
        address = fmt.Sprintf("%s@%s", address, s.config.Domain)
    }

    // Проверяем TTL
    if ttl <= 0 {
        ttl = s.config.DefaultTTL
    }
    if ttl > s.config.MaxTTL {
        return nil, ErrInvalidTTL
    }

    // Проверяем, не занят ли адрес
    existing, err := s.repo.GetByAddress(address)
    if err != nil {
        return nil, err
    }
    if existing != nil {
        // Адрес занят — генерируем новый
        address = s.generateRandomAddress()
    }

    // Создаём ящик
    return s.repo.Create(address, ttl)
}

// GetByID возвращает ящик по ID
func (s *MailboxService) GetByID(id string) (*domain.Mailbox, error) {
    mailbox, err := s.repo.GetByID(id)
    if err != nil {
        return nil, err
    }
    if mailbox == nil {
        return nil, ErrMailboxNotFound
    }

    // Проверяем, не истёк ли срок
    if mailbox.IsExpired() {
        return nil, ErrMailboxExpired
    }

    return mailbox, nil
}

// Delete удаляет почтовый ящик
func (s *MailboxService) Delete(id string) error {
    // Проверяем существование
    _, err := s.GetByID(id)
    if err != nil {
        return err
    }

    return s.repo.Delete(id)
}

// generateRandomAddress генерирует случайный email-адрес
func (s *MailboxService) generateRandomAddress() string {
    // Символы для генерации
    const chars = "abcdefghijklmnopqrstuvwxyz0123456789"
    
    // Генерируем 10 случайных символов
    result := make([]byte, 10)
    for i := range result {
        result[i] = chars[rand.Intn(len(chars))]
    }

    return fmt.Sprintf("%s@%s", string(result), s.config.Domain)
}

// init вызывается при загрузке пакета
// Инициализируем генератор случайных чисел
func init() {
    rand.Seed(time.Now().UnixNano())
}
```

**Разбор:**

- `errors.New("...")` — создаёт новую ошибку с текстовым сообщением. Эти ошибки объявлены как переменные пакета, чтобы их можно было сравнивать.

- `var ErrMailboxNotFound = errors.New(...)` — объявляем переменную-ошибку. Начинается с `Err` по соглашению Go.

- `make([]byte, 10)` — создаёт срез байтов длиной 10. `make` используется для создания срезов, карт и каналов с заданным размером.

- `rand.Intn(len(chars))` — возвращает случайное число от 0 до len(chars)-1.

- `func init()` — специальная функция, которая вызывается автоматически при загрузке пакета. Используется для инициализации.

- `rand.Seed(time.Now().UnixNano())` — инициализирует генератор случайных чисел текущим временем в наносекундах.

---

## 4. Создаём сервис для сообщений

**Создаём файл `internal/service/message_service.go`:**
```go
package service

import (
    "errors"

    "tempmail/internal/config"
    "tempmail/internal/domain"
    "tempmail/internal/repository"
)

// Ошибки сервиса
var (
    ErrMessageNotFound    = errors.New("письмо не найдено")
    ErrMailboxFull        = errors.New("ящик переполнен")
    ErrMessageTooLarge    = errors.New("письмо слишком большое")
)

// MessageService — сервис для работы с письмами
type MessageService struct {
    msgRepo     *repository.MessageRepository
    mailboxRepo *repository.MailboxRepository
    limits      config.LimitsConfig
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

    return s.msgRepo.Create(msg)
}

// GetByMailboxID возвращает все письма ящика
func (s *MessageService) GetByMailboxID(mailboxID string) ([]*domain.Message, error) {
    // Проверяем существование ящика
    mailbox, err := s.mailboxRepo.GetByID(mailboxID)
    if err != nil {
        return nil, err
    }
    if mailbox == nil {
        return nil, ErrMailboxNotFound
    }

    return s.msgRepo.GetByMailboxID(mailboxID)
}

// GetByID возвращает письмо по ID
func (s *MessageService) GetByID(id string) (*domain.Message, error) {
    msg, err := s.msgRepo.GetByID(id)
    if err != nil {
        return nil, err
    }
    if msg == nil {
        return nil, ErrMessageNotFound
    }

    // Помечаем как прочитанное
    _ = s.msgRepo.MarkAsRead(id)

    return msg, nil
}

// Delete удаляет письмо
func (s *MessageService) Delete(id string) error {
    msg, err := s.msgRepo.GetByID(id)
    if err != nil {
        return err
    }
    if msg == nil {
        return ErrMessageNotFound
    }

    return s.msgRepo.Delete(id)
}
```

---

## 5. Создаём HTTP-обработчики

**Что такое обработчик:**  
Обработчик (handler) — это функция, которая принимает HTTP-запрос и возвращает HTTP-ответ.

**Создаём файл `internal/handler/mailbox_handler.go`:**
```go
package handler

import (
    "errors"
    "time"

    "github.com/gofiber/fiber/v2"

    "tempmail/internal/service"
)

// MailboxHandler — обработчик запросов для почтовых ящиков
type MailboxHandler struct {
    service *service.MailboxService
}

// NewMailboxHandler создаёт новый обработчик
func NewMailboxHandler(svc *service.MailboxService) *MailboxHandler {
    return &MailboxHandler{service: svc}
}

// CreateRequest — структура запроса на создание ящика
type CreateRequest struct {
    Address string `json:"address"` // Желаемый адрес (необязательно)
    TTL     string `json:"ttl"`     // Время жизни (например, "1h", "30m")
}

// MailboxResponse — структура ответа с данными ящика
type MailboxResponse struct {
    ID        string `json:"id"`
    Address   string `json:"address"`
    CreatedAt string `json:"created_at"`
    ExpiresAt string `json:"expires_at"`
    IsActive  bool   `json:"is_active"`
}

// Create обрабатывает POST /api/v1/mailbox
// @Summary Создать новый почтовый ящик
// @Description Создаёт временный почтовый ящик с указанным или случайным адресом
// @Tags mailbox
// @Accept json
// @Produce json
// @Param request body CreateRequest false "Параметры создания"
// @Success 201 {object} MailboxResponse
// @Failure 400 {object} ErrorResponse
// @Router /api/v1/mailbox [post]
func (h *MailboxHandler) Create(c *fiber.Ctx) error {
    // Парсим тело запроса
    var req CreateRequest
    
    // BodyParser читает JSON из тела запроса и заполняет структуру
    if err := c.BodyParser(&req); err != nil {
        // Если тело пустое — это нормально, создадим ящик с настройками по умолчанию
        req = CreateRequest{}
    }

    // Парсим TTL
    var ttl time.Duration
    if req.TTL != "" {
        var err error
        ttl, err = time.ParseDuration(req.TTL)
        if err != nil {
            return c.Status(fiber.StatusBadRequest).JSON(ErrorResponse{
                Error: "Неверный формат TTL. Используйте формат: 1h, 30m, 24h",
            })
        }
    }

    // Создаём ящик
    mailbox, err := h.service.Create(req.Address, ttl)
    if err != nil {
        // Проверяем тип ошибки
        if errors.Is(err, service.ErrInvalidTTL) {
            return c.Status(fiber.StatusBadRequest).JSON(ErrorResponse{
                Error: "TTL превышает максимально допустимое значение",
            })
        }
        return c.Status(fiber.StatusInternalServerError).JSON(ErrorResponse{
            Error: "Внутренняя ошибка сервера",
        })
    }

    // Возвращаем успешный ответ
    // Status(201) — код "Created" (создано)
    return c.Status(fiber.StatusCreated).JSON(MailboxResponse{
        ID:        mailbox.ID,
        Address:   mailbox.Address,
        CreatedAt: mailbox.CreatedAt.Format(time.RFC3339),
        ExpiresAt: mailbox.ExpiresAt.Format(time.RFC3339),
        IsActive:  mailbox.IsActive,
    })
}

// Get обрабатывает GET /api/v1/mailbox/:id
// @Summary Получить информацию о ящике
// @Tags mailbox
// @Produce json
// @Param id path string true "ID ящика"
// @Success 200 {object} MailboxResponse
// @Failure 404 {object} ErrorResponse
// @Router /api/v1/mailbox/{id} [get]
func (h *MailboxHandler) Get(c *fiber.Ctx) error {
    // Params получает параметр из URL
    id := c.Params("id")

    mailbox, err := h.service.GetByID(id)
    if err != nil {
        if errors.Is(err, service.ErrMailboxNotFound) {
            return c.Status(fiber.StatusNotFound).JSON(ErrorResponse{
                Error: "Почтовый ящик не найден",
            })
        }
        if errors.Is(err, service.ErrMailboxExpired) {
            return c.Status(fiber.StatusGone).JSON(ErrorResponse{
                Error: "Срок действия ящика истёк",
            })
        }
        return c.Status(fiber.StatusInternalServerError).JSON(ErrorResponse{
            Error: "Внутренняя ошибка сервера",
        })
    }

    return c.JSON(MailboxResponse{
        ID:        mailbox.ID,
        Address:   mailbox.Address,
        CreatedAt: mailbox.CreatedAt.Format(time.RFC3339),
        ExpiresAt: mailbox.ExpiresAt.Format(time.RFC3339),
        IsActive:  mailbox.IsActive,
    })
}

// Delete обрабатывает DELETE /api/v1/mailbox/:id
// @Summary Удалить почтовый ящик
// @Tags mailbox
// @Param id path string true "ID ящика"
// @Success 204 "Ящик удалён"
// @Failure 404 {object} ErrorResponse
// @Router /api/v1/mailbox/{id} [delete]
func (h *MailboxHandler) Delete(c *fiber.Ctx) error {
    id := c.Params("id")

    err := h.service.Delete(id)
    if err != nil {
        if errors.Is(err, service.ErrMailboxNotFound) {
            return c.Status(fiber.StatusNotFound).JSON(ErrorResponse{
                Error: "Почтовый ящик не найден",
            })
        }
        return c.Status(fiber.StatusInternalServerError).JSON(ErrorResponse{
            Error: "Внутренняя ошибка сервера",
        })
    }

    // 204 No Content — успешное удаление без тела ответа
    return c.SendStatus(fiber.StatusNoContent)
}
```

**Разбор:**

- `*fiber.Ctx` — контекст запроса Fiber. Содержит информацию о запросе и методы для формирования ответа.

- `c.BodyParser(&req)` — читает JSON из тела запроса и заполняет структуру. `&req` — передаём адрес, чтобы функция могла изменить структуру.

- `time.ParseDuration("1h")` — парсит строку в `time.Duration`. Понимает форматы: "1h" (час), "30m" (минуты), "1h30m" (час и 30 минут).

- `errors.Is(err, service.ErrInvalidTTL)` — проверяет, является ли ошибка указанной. Работает даже с обёрнутыми ошибками.

- `c.Status(201).JSON(...)` — устанавливает HTTP-статус и возвращает JSON.

- `c.Params("id")` — получает параметр из URL. Для маршрута `/mailbox/:id` вернёт значение после `/mailbox/`.

- `fiber.StatusCreated` — константа для HTTP-статуса 201 (Created).

- `time.RFC3339` — стандартный формат даты-времени: "2006-01-02T15:04:05Z07:00".

---

## 6. Создаём структуру ошибки

**Создаём файл `internal/handler/error.go`:**
```go
package handler

// ErrorResponse — стандартный формат ошибки API
type ErrorResponse struct {
    Error   string `json:"error"`             // Сообщение об ошибке
    Details string `json:"details,omitempty"` // Дополнительные детали (необязательно)
}
```

**Разбор:**
- `omitempty` — если поле пустое, оно не будет включено в JSON.

---

## 7. Создаём обработчик сообщений

**Создаём файл `internal/handler/message_handler.go`:**
```go
package handler

import (
    "errors"
    "time"

    "github.com/gofiber/fiber/v2"

    "tempmail/internal/service"
)

// MessageHandler — обработчик запросов для писем
type MessageHandler struct {
    service *service.MessageService
}

// NewMessageHandler создаёт новый обработчик
func NewMessageHandler(svc *service.MessageService) *MessageHandler {
    return &MessageHandler{service: svc}
}

// MessageResponse — структура ответа с данными письма
type MessageResponse struct {
    ID          string `json:"id"`
    MailboxID   string `json:"mailbox_id"`
    FromAddress string `json:"from_address"`
    Subject     string `json:"subject"`
    BodyText    string `json:"body_text,omitempty"`
    BodyHTML    string `json:"body_html,omitempty"`
    ReceivedAt  string `json:"received_at"`
    IsRead      bool   `json:"is_read"`
    IsSpam      bool   `json:"is_spam"`
}

// MessageListResponse — краткая информация о письме для списка
type MessageListResponse struct {
    ID          string `json:"id"`
    FromAddress string `json:"from_address"`
    Subject     string `json:"subject"`
    ReceivedAt  string `json:"received_at"`
    IsRead      bool   `json:"is_read"`
    IsSpam      bool   `json:"is_spam"`
}

// GetMessages обрабатывает GET /api/v1/mailbox/:id/messages
// @Summary Получить список писем
// @Tags messages
// @Produce json
// @Param id path string true "ID ящика"
// @Success 200 {array} MessageListResponse
// @Failure 404 {object} ErrorResponse
// @Router /api/v1/mailbox/{id}/messages [get]
func (h *MessageHandler) GetMessages(c *fiber.Ctx) error {
    mailboxID := c.Params("id")

    messages, err := h.service.GetByMailboxID(mailboxID)
    if err != nil {
        if errors.Is(err, service.ErrMailboxNotFound) {
            return c.Status(fiber.StatusNotFound).JSON(ErrorResponse{
                Error: "Почтовый ящик не найден",
            })
        }
        return c.Status(fiber.StatusInternalServerError).JSON(ErrorResponse{
            Error: "Внутренняя ошибка сервера",
        })
    }

    // Преобразуем в формат ответа
    response := make([]MessageListResponse, len(messages))
    for i, msg := range messages {
        response[i] = MessageListResponse{
            ID:          msg.ID,
            FromAddress: msg.FromAddress,
            Subject:     msg.Subject,
            ReceivedAt:  msg.ReceivedAt.Format(time.RFC3339),
            IsRead:      msg.IsRead,
            IsSpam:      msg.IsSpam,
        }
    }

    return c.JSON(response)
}

// GetMessage обрабатывает GET /api/v1/mailbox/:id/messages/:mid
// @Summary Получить письмо
// @Tags messages
// @Produce json
// @Param id path string true "ID ящика"
// @Param mid path string true "ID письма"
// @Success 200 {object} MessageResponse
// @Failure 404 {object} ErrorResponse
// @Router /api/v1/mailbox/{id}/messages/{mid} [get]
func (h *MessageHandler) GetMessage(c *fiber.Ctx) error {
    // mid — ID письма
    messageID := c.Params("mid")

    msg, err := h.service.GetByID(messageID)
    if err != nil {
        if errors.Is(err, service.ErrMessageNotFound) {
            return c.Status(fiber.StatusNotFound).JSON(ErrorResponse{
                Error: "Письмо не найдено",
            })
        }
        return c.Status(fiber.StatusInternalServerError).JSON(ErrorResponse{
            Error: "Внутренняя ошибка сервера",
        })
    }

    return c.JSON(MessageResponse{
        ID:          msg.ID,
        MailboxID:   msg.MailboxID,
        FromAddress: msg.FromAddress,
        Subject:     msg.Subject,
        BodyText:    msg.BodyText,
        BodyHTML:    msg.BodyHTML,
        ReceivedAt:  msg.ReceivedAt.Format(time.RFC3339),
        IsRead:      msg.IsRead,
        IsSpam:      msg.IsSpam,
    })
}

// DeleteMessage обрабатывает DELETE /api/v1/mailbox/:id/messages/:mid
// @Summary Удалить письмо
// @Tags messages
// @Param id path string true "ID ящика"
// @Param mid path string true "ID письма"
// @Success 204 "Письмо удалено"
// @Failure 404 {object} ErrorResponse
// @Router /api/v1/mailbox/{id}/messages/{mid} [delete]
func (h *MessageHandler) DeleteMessage(c *fiber.Ctx) error {
    messageID := c.Params("mid")

    err := h.service.Delete(messageID)
    if err != nil {
        if errors.Is(err, service.ErrMessageNotFound) {
            return c.Status(fiber.StatusNotFound).JSON(ErrorResponse{
                Error: "Письмо не найдено",
            })
        }
        return c.Status(fiber.StatusInternalServerError).JSON(ErrorResponse{
            Error: "Внутренняя ошибка сервера",
        })
    }

    return c.SendStatus(fiber.StatusNoContent)
}
```

**Разбор:**

- `make([]MessageListResponse, len(messages))` — создаём срез с заданной длиной. Это эффективнее, чем `append` в цикле, потому что память выделяется сразу.

- `for i, msg := range messages` — перебираем срез, `i` — индекс, `msg` — элемент.

- `response[i] = ...` — записываем в срез по индексу.

---

## 8. Настраиваем маршруты

**Создаём файл `internal/handler/routes.go`:**
```go
package handler

import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/cors"
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/recover"
)

// SetupRoutes настраивает все маршруты приложения
func SetupRoutes(
    app *fiber.App,
    mailboxHandler *MailboxHandler,
    messageHandler *MessageHandler,
) {
    // Подключаем middleware (промежуточные обработчики)
    
    // Logger — логирует все запросы
    app.Use(logger.New())
    
    // Recover — перехватывает паники и возвращает 500 вместо падения сервера
    app.Use(recover.New())
    
    // CORS — разрешает запросы с других доменов (для фронтенда)
    app.Use(cors.New(cors.Config{
        AllowOrigins: "*",                           // Разрешаем все домены
        AllowMethods: "GET,POST,PUT,DELETE,OPTIONS", // Разрешённые методы
        AllowHeaders: "Content-Type,Authorization",  // Разрешённые заголовки
    }))

    // Группа маршрутов API v1
    api := app.Group("/api/v1")

    // Маршруты для почтовых ящиков
    mailbox := api.Group("/mailbox")
    mailbox.Post("/", mailboxHandler.Create)       // POST /api/v1/mailbox
    mailbox.Get("/:id", mailboxHandler.Get)        // GET /api/v1/mailbox/:id
    mailbox.Delete("/:id", mailboxHandler.Delete)  // DELETE /api/v1/mailbox/:id

    // Маршруты для писем (вложены в mailbox)
    mailbox.Get("/:id/messages", messageHandler.GetMessages)           // GET /api/v1/mailbox/:id/messages
    mailbox.Get("/:id/messages/:mid", messageHandler.GetMessage)       // GET /api/v1/mailbox/:id/messages/:mid
    mailbox.Delete("/:id/messages/:mid", messageHandler.DeleteMessage) // DELETE /api/v1/mailbox/:id/messages/:mid

    // Маршрут для проверки здоровья сервера
    app.Get("/health", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{
            "status": "ok",
            "time":   c.Context().Time().Format("2006-01-02 15:04:05"),
        })
    })
}
```

**Разбор:**

- `app.Use(...)` — добавляет middleware. Middleware — это функции, которые выполняются до или после обработчика.

- `logger.New()` — логирует информацию о каждом запросе (метод, URL, время выполнения).

- `recover.New()` — перехватывает паники (критические ошибки) и возвращает 500 вместо падения сервера.

- `cors.New(...)` — настраивает CORS (Cross-Origin Resource Sharing). Позволяет браузеру делать запросы с других доменов.

- `app.Group("/api/v1")` — создаёт группу маршрутов с общим префиксом.

- `:id` и `:mid` — параметры в URL. Их значения доступны через `c.Params("id")`.

- `fiber.Map` — это `map[string]interface{}`, удобный тип для создания JSON на лету.

---

## 9. Обновляем главный файл

**Обновляем `cmd/api/main.go`:**
```go
package main

import (
    "fmt"
    "log"

    "github.com/gofiber/fiber/v2"

    "tempmail/internal/config"
    "tempmail/internal/handler"
    "tempmail/internal/repository"
    "tempmail/internal/service"
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

    // Запускаем сервер
    addr := fmt.Sprintf(":%d", cfg.Server.HTTPPort)
    fmt.Printf("\nСервер запущен на http://localhost%s\n", addr)
    fmt.Println("Нажмите Ctrl+C для остановки")
    
    // Listen блокирует выполнение и слушает входящие соединения
    if err := app.Listen(addr); err != nil {
        log.Fatal("Ошибка запуска сервера:", err)
    }
}
```

**Разбор:**

- `fiber.New(...)` — создаёт новое Fiber-приложение с настройками.

- `app.Listen(":8080")` — запускает HTTP-сервер на указанном порту. Эта функция блокирует выполнение (программа "зависает" здесь, обрабатывая запросы).

---

## 10. Тестируем API

**Запускаем сервер:**
```bash
go run cmd/api/main.go
```

**Тестируем с помощью curl:**

**Создаём почтовый ящик:**
```bash
curl -X POST http://localhost:8080/api/v1/mailbox
```

**Ответ:**
```json
{
  "id": "abc123-...",
  "address": "x7k2m9p4q1@tempmail.dev",
  "created_at": "2024-01-15T12:00:00Z",
  "expires_at": "2024-01-15T13:00:00Z",
  "is_active": true
}
```

**Создаём ящик с параметрами:**
```bash
curl -X POST http://localhost:8080/api/v1/mailbox \
  -H "Content-Type: application/json" \
  -d '{"address": "mybox", "ttl": "2h"}'
```

**Получаем информацию о ящике:**
```bash
curl http://localhost:8080/api/v1/mailbox/abc123-...
```

**Получаем список писем:**
```bash
curl http://localhost:8080/api/v1/mailbox/abc123-.../messages
```

**Удаляем ящик:**
```bash
curl -X DELETE http://localhost:8080/api/v1/mailbox/abc123-...
```

**Проверяем здоровье сервера:**
```bash
curl http://localhost:8080/health
```

---

## Словарь терминов

| Термин | Объяснение |
|--------|------------|
| **REST API** | Архитектурный стиль для создания веб-сервисов |
| **HTTP-метод** | Тип действия: GET, POST, PUT, DELETE |
| **Middleware** | Промежуточный обработчик, выполняется до/после основного |
| **CORS** | Механизм, позволяющий запросы с других доменов |
| **JSON** | Формат обмена данными (JavaScript Object Notation) |
| **Маршрут** | Соответствие URL и обработчика |
| **Контекст** | Объект с информацией о запросе и методами для ответа |

---

## Что мы узнали

- Что такое REST API и HTTP-методы
- Как использовать веб-фреймворк Fiber
- Как создавать сервисный слой с бизнес-логикой
- Как писать HTTP-обработчики
- Как настраивать маршруты и middleware
- Как работать с JSON-запросами и ответами
- Как обрабатывать ошибки в API

---

[Следующая глава: Реализация SMTP-сервера](./05-smtp-server.md)

[Вернуться к оглавлению](./README.md)

