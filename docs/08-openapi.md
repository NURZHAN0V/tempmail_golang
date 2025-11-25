# Глава 8: Документация API с OpenAPI (Swagger)

В этой главе мы добавим автоматическую генерацию документации API с помощью Swagger/OpenAPI.

---

## 1. Что такое OpenAPI и Swagger

**OpenAPI** — это стандарт для описания REST API. Описание хранится в JSON или YAML файле и содержит информацию обо всех эндпоинтах, параметрах, ответах.

**Swagger** — это набор инструментов для работы с OpenAPI:
- **Swagger UI** — веб-интерфейс для просмотра и тестирования API
- **Swagger Editor** — редактор спецификаций
- **swag** — генератор OpenAPI из комментариев в Go-коде

---

## 2. Устанавливаем swag

**Что делаем:**  
Устанавливаем инструмент `swag`, который генерирует OpenAPI-спецификацию из комментариев в коде.

**Команда:**
```bash
go install github.com/swaggo/swag/cmd/swag@latest
```

**Устанавливаем библиотеки для Fiber:**
```bash
go get github.com/swaggo/swag
go get github.com/gofiber/swagger
```

**Проверяем установку:**
```bash
swag --version
```

---

## 3. Добавляем главные аннотации

**Что такое аннотации:**  
Аннотации — это специальные комментарии, которые `swag` читает и преобразует в OpenAPI-спецификацию.

**Обновляем `cmd/api/main.go`:**
```go
package main

// @title TempMail API
// @version 1.0
// @description Сервис временных email-адресов с REST API
// @termsOfService http://swagger.io/terms/

// @contact.name API Support
// @contact.email support@tempmail.dev

// @license.name MIT
// @license.url https://opensource.org/licenses/MIT

// @host localhost:8080
// @BasePath /api/v1

// @schemes http https

import (
    "fmt"
    "log"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/swagger"

    "tempmail/internal/config"
    "tempmail/internal/handler"
    "tempmail/internal/repository"
    "tempmail/internal/service"
    smtpserver "tempmail/internal/smtp"

    _ "tempmail/docs" // Импортируем сгенерированную документацию
)

func main() {
    // ... остальной код без изменений
}
```

**Разбор аннотаций:**

- `@title` — название API
- `@version` — версия API
- `@description` — описание API
- `@host` — хост, на котором работает API
- `@BasePath` — базовый путь для всех эндпоинтов
- `@schemes` — поддерживаемые протоколы (http, https)
- `@contact.name`, `@contact.email` — контактная информация
- `@license.name`, `@license.url` — лицензия

---

## 4. Документируем эндпоинты почтовых ящиков

**Обновляем `internal/handler/mailbox_handler.go`:**
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

// CreateRequest — запрос на создание ящика
// @Description Параметры для создания почтового ящика
type CreateRequest struct {
    Address string `json:"address" example:"mybox"`    // Желаемый адрес (без домена)
    TTL     string `json:"ttl" example:"1h"`           // Время жизни (1h, 30m, 24h)
}

// MailboxResponse — ответ с данными ящика
// @Description Информация о почтовом ящике
type MailboxResponse struct {
    ID        string `json:"id" example:"550e8400-e29b-41d4-a716-446655440000"`
    Address   string `json:"address" example:"mybox@tempmail.dev"`
    CreatedAt string `json:"created_at" example:"2024-01-15T12:00:00Z"`
    ExpiresAt string `json:"expires_at" example:"2024-01-15T13:00:00Z"`
    IsActive  bool   `json:"is_active" example:"true"`
}

// Create создаёт новый почтовый ящик
// @Summary Создать почтовый ящик
// @Description Создаёт новый временный почтовый ящик. Если адрес не указан, генерируется случайный.
// @Tags mailbox
// @Accept json
// @Produce json
// @Param request body CreateRequest false "Параметры создания (необязательно)"
// @Success 201 {object} MailboxResponse "Ящик успешно создан"
// @Failure 400 {object} ErrorResponse "Неверные параметры запроса"
// @Failure 500 {object} ErrorResponse "Внутренняя ошибка сервера"
// @Router /mailbox [post]
func (h *MailboxHandler) Create(c *fiber.Ctx) error {
    var req CreateRequest

    if err := c.BodyParser(&req); err != nil {
        req = CreateRequest{}
    }

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

    mailbox, err := h.service.Create(req.Address, ttl)
    if err != nil {
        if errors.Is(err, service.ErrInvalidTTL) {
            return c.Status(fiber.StatusBadRequest).JSON(ErrorResponse{
                Error: "TTL превышает максимально допустимое значение",
            })
        }
        return c.Status(fiber.StatusInternalServerError).JSON(ErrorResponse{
            Error: "Внутренняя ошибка сервера",
        })
    }

    return c.Status(fiber.StatusCreated).JSON(MailboxResponse{
        ID:        mailbox.ID,
        Address:   mailbox.Address,
        CreatedAt: mailbox.CreatedAt.Format(time.RFC3339),
        ExpiresAt: mailbox.ExpiresAt.Format(time.RFC3339),
        IsActive:  mailbox.IsActive,
    })
}

// Get возвращает информацию о ящике
// @Summary Получить информацию о ящике
// @Description Возвращает информацию о почтовом ящике по его ID
// @Tags mailbox
// @Produce json
// @Param id path string true "ID почтового ящика" example("550e8400-e29b-41d4-a716-446655440000")
// @Success 200 {object} MailboxResponse "Информация о ящике"
// @Failure 404 {object} ErrorResponse "Ящик не найден"
// @Failure 410 {object} ErrorResponse "Срок действия ящика истёк"
// @Failure 500 {object} ErrorResponse "Внутренняя ошибка сервера"
// @Router /mailbox/{id} [get]
func (h *MailboxHandler) Get(c *fiber.Ctx) error {
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

// Delete удаляет почтовый ящик
// @Summary Удалить почтовый ящик
// @Description Удаляет почтовый ящик и все связанные письма
// @Tags mailbox
// @Param id path string true "ID почтового ящика" example("550e8400-e29b-41d4-a716-446655440000")
// @Success 204 "Ящик успешно удалён"
// @Failure 404 {object} ErrorResponse "Ящик не найден"
// @Failure 500 {object} ErrorResponse "Внутренняя ошибка сервера"
// @Router /mailbox/{id} [delete]
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

    return c.SendStatus(fiber.StatusNoContent)
}
```

**Разбор аннотаций эндпоинтов:**

- `@Summary` — краткое описание (отображается в списке)
- `@Description` — полное описание
- `@Tags` — группа эндпоинтов (для организации в UI)
- `@Accept` — принимаемый формат (json, xml)
- `@Produce` — формат ответа
- `@Param` — параметр запроса:
  - `name` — имя параметра
  - `path/query/body` — где находится параметр
  - `type` — тип данных
  - `true/false` — обязательный или нет
  - `"описание"` — описание параметра
- `@Success` — успешный ответ (код, тип, описание)
- `@Failure` — ответ с ошибкой
- `@Router` — путь и HTTP-метод

---

## 5. Документируем эндпоинты сообщений

**Обновляем `internal/handler/message_handler.go`:**
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

// MessageResponse — полная информация о письме
// @Description Полная информация о письме включая содержимое
type MessageResponse struct {
    ID          string `json:"id" example:"550e8400-e29b-41d4-a716-446655440001"`
    MailboxID   string `json:"mailbox_id" example:"550e8400-e29b-41d4-a716-446655440000"`
    FromAddress string `json:"from_address" example:"sender@example.com"`
    Subject     string `json:"subject" example:"Добро пожаловать!"`
    BodyText    string `json:"body_text,omitempty" example:"Текст письма..."`
    BodyHTML    string `json:"body_html,omitempty" example:"<p>HTML письма...</p>"`
    ReceivedAt  string `json:"received_at" example:"2024-01-15T12:30:00Z"`
    IsRead      bool   `json:"is_read" example:"false"`
    IsSpam      bool   `json:"is_spam" example:"false"`
}

// MessageListResponse — краткая информация о письме для списка
// @Description Краткая информация о письме (без содержимого)
type MessageListResponse struct {
    ID          string `json:"id" example:"550e8400-e29b-41d4-a716-446655440001"`
    FromAddress string `json:"from_address" example:"sender@example.com"`
    Subject     string `json:"subject" example:"Добро пожаловать!"`
    ReceivedAt  string `json:"received_at" example:"2024-01-15T12:30:00Z"`
    IsRead      bool   `json:"is_read" example:"false"`
    IsSpam      bool   `json:"is_spam" example:"false"`
}

// GetMessages возвращает список писем
// @Summary Получить список писем
// @Description Возвращает список всех писем в почтовом ящике (без содержимого)
// @Tags messages
// @Produce json
// @Param id path string true "ID почтового ящика" example("550e8400-e29b-41d4-a716-446655440000")
// @Success 200 {array} MessageListResponse "Список писем"
// @Failure 404 {object} ErrorResponse "Ящик не найден"
// @Failure 500 {object} ErrorResponse "Внутренняя ошибка сервера"
// @Router /mailbox/{id}/messages [get]
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

// GetMessage возвращает письмо по ID
// @Summary Получить письмо
// @Description Возвращает полную информацию о письме включая содержимое. Автоматически помечает письмо как прочитанное.
// @Tags messages
// @Produce json
// @Param id path string true "ID почтового ящика" example("550e8400-e29b-41d4-a716-446655440000")
// @Param mid path string true "ID письма" example("550e8400-e29b-41d4-a716-446655440001")
// @Success 200 {object} MessageResponse "Информация о письме"
// @Failure 404 {object} ErrorResponse "Письмо не найдено"
// @Failure 500 {object} ErrorResponse "Внутренняя ошибка сервера"
// @Router /mailbox/{id}/messages/{mid} [get]
func (h *MessageHandler) GetMessage(c *fiber.Ctx) error {
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

// DeleteMessage удаляет письмо
// @Summary Удалить письмо
// @Description Удаляет письмо из почтового ящика
// @Tags messages
// @Param id path string true "ID почтового ящика" example("550e8400-e29b-41d4-a716-446655440000")
// @Param mid path string true "ID письма" example("550e8400-e29b-41d4-a716-446655440001")
// @Success 204 "Письмо успешно удалено"
// @Failure 404 {object} ErrorResponse "Письмо не найдено"
// @Failure 500 {object} ErrorResponse "Внутренняя ошибка сервера"
// @Router /mailbox/{id}/messages/{mid} [delete]
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

---

## 6. Обновляем структуру ошибки

**Обновляем `internal/handler/error.go`:**
```go
package handler

// ErrorResponse — стандартный формат ошибки API
// @Description Ответ с информацией об ошибке
type ErrorResponse struct {
    Error   string `json:"error" example:"Почтовый ящик не найден"`
    Details string `json:"details,omitempty" example:"ID: 550e8400-..."`
}
```

---

## 7. Добавляем маршрут Swagger UI

**Обновляем `internal/handler/routes.go`:**
```go
package handler

import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/cors"
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/recover"
    "github.com/gofiber/swagger"

    "tempmail/internal/service"
)

// SetupRoutes настраивает все маршруты приложения
func SetupRoutes(
    app *fiber.App,
    mailboxHandler *MailboxHandler,
    messageHandler *MessageHandler,
) {
    // Middleware
    app.Use(logger.New())
    app.Use(recover.New())
    app.Use(cors.New(cors.Config{
        AllowOrigins: "*",
        AllowMethods: "GET,POST,PUT,DELETE,OPTIONS",
        AllowHeaders: "Content-Type,Authorization",
    }))

    // Swagger UI
    app.Get("/swagger/*", swagger.HandlerDefault)

    // API v1
    api := app.Group("/api/v1")

    // Mailbox routes
    mailbox := api.Group("/mailbox")
    mailbox.Post("/", mailboxHandler.Create)
    mailbox.Get("/:id", mailboxHandler.Get)
    mailbox.Delete("/:id", mailboxHandler.Delete)

    // Message routes
    mailbox.Get("/:id/messages", messageHandler.GetMessages)
    mailbox.Get("/:id/messages/:mid", messageHandler.GetMessage)
    mailbox.Delete("/:id/messages/:mid", messageHandler.DeleteMessage)

    // Health check
    // @Summary Проверка здоровья
    // @Description Возвращает статус сервера
    // @Tags system
    // @Produce json
    // @Success 200 {object} map[string]string "Статус сервера"
    // @Router /health [get]
    app.Get("/health", func(c *fiber.Ctx) error {
        return c.JSON(fiber.Map{
            "status": "ok",
        })
    })

    // Stats
    // @Summary Статистика сервиса
    // @Description Возвращает статистику работы сервиса
    // @Tags system
    // @Produce json
    // @Success 200 {object} map[string]interface{} "Статистика"
    // @Router /stats [get]
    app.Get("/stats", func(c *fiber.Ctx) error {
        stats := service.GlobalStats.GetStats()
        return c.JSON(fiber.Map{
            "total_mailboxes":   stats.TotalMailboxes,
            "total_messages":    stats.TotalMessages,
            "total_spam":        stats.TotalSpamMessages,
            "deleted_mailboxes": stats.DeletedMailboxes,
            "last_cleanup":      stats.LastCleanup.Format("2006-01-02 15:04:05"),
        })
    })
}
```

---

## 8. Генерируем документацию

**Запускаем swag init:**
```bash
swag init -g cmd/api/main.go -o docs
```

**Разбор параметров:**
- `-g cmd/api/main.go` — главный файл с общими аннотациями
- `-o docs` — папка для сгенерированных файлов

**Результат:**  
В папке `docs` появятся файлы:
- `docs.go` — Go-код для инициализации
- `swagger.json` — OpenAPI-спецификация в JSON
- `swagger.yaml` — OpenAPI-спецификация в YAML

---

## 9. Полный swagger.json

После генерации файл `docs/swagger.json` будет содержать:

```json
{
  "swagger": "2.0",
  "info": {
    "description": "Сервис временных email-адресов с REST API",
    "title": "TempMail API",
    "termsOfService": "http://swagger.io/terms/",
    "contact": {
      "name": "API Support",
      "email": "support@tempmail.dev"
    },
    "license": {
      "name": "MIT",
      "url": "https://opensource.org/licenses/MIT"
    },
    "version": "1.0"
  },
  "host": "localhost:8080",
  "basePath": "/api/v1",
  "paths": {
    "/mailbox": {
      "post": {
        "description": "Создаёт новый временный почтовый ящик. Если адрес не указан, генерируется случайный.",
        "consumes": ["application/json"],
        "produces": ["application/json"],
        "tags": ["mailbox"],
        "summary": "Создать почтовый ящик",
        "parameters": [
          {
            "description": "Параметры создания (необязательно)",
            "name": "request",
            "in": "body",
            "schema": {
              "$ref": "#/definitions/handler.CreateRequest"
            }
          }
        ],
        "responses": {
          "201": {
            "description": "Ящик успешно создан",
            "schema": {
              "$ref": "#/definitions/handler.MailboxResponse"
            }
          },
          "400": {
            "description": "Неверные параметры запроса",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          },
          "500": {
            "description": "Внутренняя ошибка сервера",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          }
        }
      }
    },
    "/mailbox/{id}": {
      "get": {
        "description": "Возвращает информацию о почтовом ящике по его ID",
        "produces": ["application/json"],
        "tags": ["mailbox"],
        "summary": "Получить информацию о ящике",
        "parameters": [
          {
            "type": "string",
            "example": "550e8400-e29b-41d4-a716-446655440000",
            "description": "ID почтового ящика",
            "name": "id",
            "in": "path",
            "required": true
          }
        ],
        "responses": {
          "200": {
            "description": "Информация о ящике",
            "schema": {
              "$ref": "#/definitions/handler.MailboxResponse"
            }
          },
          "404": {
            "description": "Ящик не найден",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          },
          "410": {
            "description": "Срок действия ящика истёк",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          },
          "500": {
            "description": "Внутренняя ошибка сервера",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          }
        }
      },
      "delete": {
        "description": "Удаляет почтовый ящик и все связанные письма",
        "tags": ["mailbox"],
        "summary": "Удалить почтовый ящик",
        "parameters": [
          {
            "type": "string",
            "example": "550e8400-e29b-41d4-a716-446655440000",
            "description": "ID почтового ящика",
            "name": "id",
            "in": "path",
            "required": true
          }
        ],
        "responses": {
          "204": {
            "description": "Ящик успешно удалён"
          },
          "404": {
            "description": "Ящик не найден",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          },
          "500": {
            "description": "Внутренняя ошибка сервера",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          }
        }
      }
    },
    "/mailbox/{id}/messages": {
      "get": {
        "description": "Возвращает список всех писем в почтовом ящике (без содержимого)",
        "produces": ["application/json"],
        "tags": ["messages"],
        "summary": "Получить список писем",
        "parameters": [
          {
            "type": "string",
            "example": "550e8400-e29b-41d4-a716-446655440000",
            "description": "ID почтового ящика",
            "name": "id",
            "in": "path",
            "required": true
          }
        ],
        "responses": {
          "200": {
            "description": "Список писем",
            "schema": {
              "type": "array",
              "items": {
                "$ref": "#/definitions/handler.MessageListResponse"
              }
            }
          },
          "404": {
            "description": "Ящик не найден",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          },
          "500": {
            "description": "Внутренняя ошибка сервера",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          }
        }
      }
    },
    "/mailbox/{id}/messages/{mid}": {
      "get": {
        "description": "Возвращает полную информацию о письме включая содержимое. Автоматически помечает письмо как прочитанное.",
        "produces": ["application/json"],
        "tags": ["messages"],
        "summary": "Получить письмо",
        "parameters": [
          {
            "type": "string",
            "example": "550e8400-e29b-41d4-a716-446655440000",
            "description": "ID почтового ящика",
            "name": "id",
            "in": "path",
            "required": true
          },
          {
            "type": "string",
            "example": "550e8400-e29b-41d4-a716-446655440001",
            "description": "ID письма",
            "name": "mid",
            "in": "path",
            "required": true
          }
        ],
        "responses": {
          "200": {
            "description": "Информация о письме",
            "schema": {
              "$ref": "#/definitions/handler.MessageResponse"
            }
          },
          "404": {
            "description": "Письмо не найдено",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          },
          "500": {
            "description": "Внутренняя ошибка сервера",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          }
        }
      },
      "delete": {
        "description": "Удаляет письмо из почтового ящика",
        "tags": ["messages"],
        "summary": "Удалить письмо",
        "parameters": [
          {
            "type": "string",
            "example": "550e8400-e29b-41d4-a716-446655440000",
            "description": "ID почтового ящика",
            "name": "id",
            "in": "path",
            "required": true
          },
          {
            "type": "string",
            "example": "550e8400-e29b-41d4-a716-446655440001",
            "description": "ID письма",
            "name": "mid",
            "in": "path",
            "required": true
          }
        ],
        "responses": {
          "204": {
            "description": "Письмо успешно удалено"
          },
          "404": {
            "description": "Письмо не найдено",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          },
          "500": {
            "description": "Внутренняя ошибка сервера",
            "schema": {
              "$ref": "#/definitions/handler.ErrorResponse"
            }
          }
        }
      }
    }
  },
  "definitions": {
    "handler.CreateRequest": {
      "description": "Параметры для создания почтового ящика",
      "type": "object",
      "properties": {
        "address": {
          "description": "Желаемый адрес (без домена)",
          "type": "string",
          "example": "mybox"
        },
        "ttl": {
          "description": "Время жизни (1h, 30m, 24h)",
          "type": "string",
          "example": "1h"
        }
      }
    },
    "handler.ErrorResponse": {
      "description": "Ответ с информацией об ошибке",
      "type": "object",
      "properties": {
        "details": {
          "type": "string",
          "example": "ID: 550e8400-..."
        },
        "error": {
          "type": "string",
          "example": "Почтовый ящик не найден"
        }
      }
    },
    "handler.MailboxResponse": {
      "description": "Информация о почтовом ящике",
      "type": "object",
      "properties": {
        "address": {
          "type": "string",
          "example": "mybox@tempmail.dev"
        },
        "created_at": {
          "type": "string",
          "example": "2024-01-15T12:00:00Z"
        },
        "expires_at": {
          "type": "string",
          "example": "2024-01-15T13:00:00Z"
        },
        "id": {
          "type": "string",
          "example": "550e8400-e29b-41d4-a716-446655440000"
        },
        "is_active": {
          "type": "boolean",
          "example": true
        }
      }
    },
    "handler.MessageListResponse": {
      "description": "Краткая информация о письме (без содержимого)",
      "type": "object",
      "properties": {
        "from_address": {
          "type": "string",
          "example": "sender@example.com"
        },
        "id": {
          "type": "string",
          "example": "550e8400-e29b-41d4-a716-446655440001"
        },
        "is_read": {
          "type": "boolean",
          "example": false
        },
        "is_spam": {
          "type": "boolean",
          "example": false
        },
        "received_at": {
          "type": "string",
          "example": "2024-01-15T12:30:00Z"
        },
        "subject": {
          "type": "string",
          "example": "Добро пожаловать!"
        }
      }
    },
    "handler.MessageResponse": {
      "description": "Полная информация о письме включая содержимое",
      "type": "object",
      "properties": {
        "body_html": {
          "type": "string",
          "example": "\u003cp\u003eHTML письма...\u003c/p\u003e"
        },
        "body_text": {
          "type": "string",
          "example": "Текст письма..."
        },
        "from_address": {
          "type": "string",
          "example": "sender@example.com"
        },
        "id": {
          "type": "string",
          "example": "550e8400-e29b-41d4-a716-446655440001"
        },
        "is_read": {
          "type": "boolean",
          "example": false
        },
        "is_spam": {
          "type": "boolean",
          "example": false
        },
        "mailbox_id": {
          "type": "string",
          "example": "550e8400-e29b-41d4-a716-446655440000"
        },
        "received_at": {
          "type": "string",
          "example": "2024-01-15T12:30:00Z"
        },
        "subject": {
          "type": "string",
          "example": "Добро пожаловать!"
        }
      }
    }
  },
  "schemes": ["http", "https"]
}
```

---

## 10. Добавляем команду генерации

### Linux/macOS (Makefile)

**Добавляем в `Makefile`:**
```makefile
.PHONY: swagger
swagger:
	swag init -g cmd/api/main.go -o docs
	@echo "Swagger документация сгенерирована в docs/"
```

**Использование:**
```bash
make swagger
```

### Windows (PowerShell)

**Создаём файл `scripts/swagger.ps1`:**
```powershell
# Генерация Swagger документации
Write-Host "Генерация Swagger..." -ForegroundColor Green
swag init -g cmd/api/main.go -o docs
Write-Host "Готово! Файлы в папке docs/" -ForegroundColor Green
```

**Использование:**
```powershell
.\scripts\swagger.ps1
```

**Или напрямую без скрипта:**
```powershell
swag init -g cmd/api/main.go -o docs
```

---

## 11. Проверяем работу

**Запускаем сервер:**
```bash
go run cmd/api/main.go
```

**Открываем Swagger UI:**
```
http://localhost:8080/swagger/index.html
```

**Скачиваем swagger.json:**
```
http://localhost:8080/swagger/doc.json
```

---

## Словарь терминов

| Термин | Объяснение |
|--------|------------|
| **OpenAPI** | Стандарт описания REST API |
| **Swagger** | Набор инструментов для работы с OpenAPI |
| **Swagger UI** | Веб-интерфейс для документации API |
| **Аннотация** | Специальный комментарий для генерации документации |
| **Спецификация** | Файл с описанием API (swagger.json) |

---

## Что мы узнали

- Что такое OpenAPI и Swagger
- Как устанавливать и использовать swag
- Как писать аннотации для эндпоинтов
- Как генерировать swagger.json
- Как подключать Swagger UI к Fiber

---

[Вернуться к оглавлению](./README.md)

