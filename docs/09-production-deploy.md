# Глава 9: Развёртывание на продакшн

В этой главе мы развернём наш сервис временных email-адресов на реальном сервере с собственным доменным именем. Вы узнаете, как купить домен, настроить DNS, получить SSL-сертификат и запустить приложение в продакшене.

---

## 1. Покупка доменного имени

**Что такое домен:**  
Доменное имя — это адрес вашего сайта в интернете (например, `tempmail.dev`). Это то, что пользователи будут вводить в браузере.

**Где купить домен:**

Популярные регистраторы доменов:
- **Namecheap** (https://www.namecheap.com) — простой интерфейс, хорошая поддержка
- **GoDaddy** (https://www.godaddy.com) — крупный регистратор
- **Cloudflare Registrar** (https://www.cloudflare.com/products/registrar) — низкие цены
- **Reg.ru** (https://www.reg.ru) — для русскоязычных пользователей

**Процесс покупки:**

1. Перейдите на сайт регистратора
2. Введите желаемое доменное имя (например, `tempmail`)
3. Выберите доменную зону (`.dev`, `.com`, `.io` и т.д.)
4. Добавьте в корзину и оформите заказ
5. Заполните данные владельца домена
6. Оплатите заказ

**Важно:**  
- `.dev` домены требуют HTTPS (обязательны SSL-сертификаты)
- Некоторые доменные зоны дороже других
- Проверьте доступность домена перед покупкой

---

## 2. Выбор и настройка VPS-сервера

**Что такое VPS:**  
VPS (Virtual Private Server) — это виртуальный сервер, на котором вы можете установить своё программное обеспечение. Это ваш собственный компьютер в интернете.

**Рекомендуемые провайдеры:**

- **DigitalOcean** (https://www.digitalocean.com) — простой интерфейс, хорошая документация
- **Linode** (https://www.linode.com) — надёжность и производительность
- **Vultr** (https://www.vultr.com) — низкие цены, множество локаций
- **Hetzner** (https://www.hetzner.com) — хорошее соотношение цена/качество

**Минимальные требования для сервера:**

- **CPU:** 1 ядро
- **RAM:** 2 GB
- **Диск:** 20 GB SSD
- **ОС:** Ubuntu 22.04 LTS (рекомендуется)

**Настройка сервера после покупки:**

**Шаг 1: Подключение по SSH**

```bash
# Подключитесь к серверу (замените IP_ADDRESS на IP вашего сервера)
ssh root@IP_ADDRESS

# Если используется ключ вместо пароля
ssh -i /path/to/private/key root@IP_ADDRESS
```

**Шаг 2: Обновление системы**

```bash
# Обновляем список пакетов
apt update

# Устанавливаем обновления
apt upgrade -y

# Перезагружаем сервер (если требуется)
reboot
```

**Шаг 3: Создание пользователя**

```bash
# Создаём нового пользователя (замените username на ваше имя)
adduser username

# Добавляем пользователя в группу sudo (для выполнения команд от администратора)
usermod -aG sudo username

# Переключаемся на нового пользователя
su - username
```

**Шаг 4: Настройка SSH-ключа (опционально, но рекомендуется)**

```bash
# На локальном компьютере создаём SSH-ключ (если ещё нет)
ssh-keygen -t ed25519 -C "your_email@example.com"

# Копируем публичный ключ на сервер
ssh-copy-id username@IP_ADDRESS

# Проверяем подключение
ssh username@IP_ADDRESS
```

**Шаг 5: Установка Docker и Docker Compose**

```bash
# Устанавливаем необходимые пакеты
sudo apt install -y ca-certificates curl gnupg lsb-release

# Добавляем официальный GPG-ключ Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Добавляем репозиторий Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Обновляем список пакетов
sudo apt update

# Устанавливаем Docker
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Добавляем пользователя в группу docker (чтобы не использовать sudo)
sudo usermod -aG docker username

# Проверяем установку
docker --version
docker compose version

# Выходим и заходим заново, чтобы изменения вступили в силу
exit
```

---

## 3. Настройка DNS-записей

**Что такое DNS:**  
DNS (Domain Name System) — это система, которая преобразует доменные имена в IP-адреса. Когда пользователь вводит `tempmail.dev`, DNS находит IP-адрес вашего сервера.

**Типы DNS-записей:**

| Тип | Назначение | Пример |
|-----|------------|--------|
| **A** | Указывает на IP-адрес сервера | `@ A 192.0.2.1` |
| **MX** | Указывает почтовый сервер | `@ MX 10 mail.tempmail.dev` |
| **CNAME** | Алиас для другого домена | `www CNAME tempmail.dev` |
| **TXT** | Произвольный текст (для SPF, DKIM) | `@ TXT "v=spf1 ..."` |

**Настройка DNS-записей:**

**Шаг 1: Найдите DNS-панель у вашего регистратора**

Обычно это раздел "DNS Management", "Управление DNS" или "DNS Settings" в панели управления доменом.

**Шаг 2: Добавьте A-запись (основная запись)**

```
Тип: A
Имя: @ (или пусто, означает корневой домен)
Значение: IP_АДРЕС_ВАШЕГО_СЕРВЕРА
TTL: 3600 (или Auto)
```

**Пример:**
```
A    @    192.0.2.1    3600
```

**Шаг 3: Добавьте A-запись для поддомена mail (для SMTP)**

```
Тип: A
Имя: mail
Значение: IP_АДРЕС_ВАШЕГО_СЕРВЕРА (тот же)
TTL: 3600
```

**Пример:**
```
A    mail    192.0.2.1    3600
```

**Шаг 4: Добавьте MX-запись (почтовый сервер)**

```
Тип: MX
Имя: @
Значение: mail.tempmail.dev
Приоритет: 10
TTL: 3600
```

**Пример:**
```
MX   @    mail.tempmail.dev    10    3600
```

**Шаг 5: Добавьте TXT-запись для SPF (защита от спама)**

SPF (Sender Policy Framework) указывает, какие серверы могут отправлять письма от имени вашего домена.

```
Тип: TXT
Имя: @
Значение: v=spf1 mx a:mail.tempmail.dev ~all
TTL: 3600
```

**Пример:**
```
TXT  @    "v=spf1 mx a:mail.tempmail.dev ~all"    3600
```

**Важно:**  
- Изменения DNS распространяются от нескольких минут до 48 часов
- Проверить DNS можно через `dig` или онлайн-сервисы (например, https://dnschecker.org)

**Проверка DNS:**

```bash
# На сервере или локально установите dig
sudo apt install -y dnsutils

# Проверяем A-запись
dig tempmail.dev A

# Проверяем MX-запись
dig tempmail.dev MX

# Проверяем TXT-запись
dig tempmail.dev TXT
```

---

## 4. Установка и настройка Nginx

**Что такое Nginx:**  
Nginx — это веб-сервер и reverse proxy. Он будет принимать HTTP/HTTPS-запросы и перенаправлять их на наше приложение, работающее на порту 8080.

**Зачем нужен Nginx:**
- SSL/TLS-терминация (обработка HTTPS)
- Балансировка нагрузки
- Кэширование статических файлов
- Безопасность

**Установка Nginx:**

```bash
# Устанавливаем Nginx
sudo apt install -y nginx

# Проверяем статус
sudo systemctl status nginx

# Включаем автозапуск
sudo systemctl enable nginx
```

**Создание конфигурации для нашего приложения:**

**Создаём файл `/etc/nginx/sites-available/tempmail`:**

```bash
sudo nano /etc/nginx/sites-available/tempmail
```

**Содержимое файла (пока без SSL):**

```nginx
server {
    listen 80;
    server_name tempmail.dev www.tempmail.dev;

    # Логирование
    access_log /var/log/nginx/tempmail-access.log;
    error_log /var/log/nginx/tempmail-error.log;

    # Максимальный размер тела запроса (для загрузки файлов)
    client_max_body_size 10M;

    # Проксирование на наше приложение
    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        
        # Заголовки для корректной работы
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Таймауты
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

**Активация конфигурации:**

```bash
# Создаём символическую ссылку
sudo ln -s /etc/nginx/sites-available/tempmail /etc/nginx/sites-enabled/

# Удаляем дефолтную конфигурацию (опционально)
sudo rm /etc/nginx/sites-enabled/default

# Проверяем конфигурацию на ошибки
sudo nginx -t

# Перезагружаем Nginx
sudo systemctl reload nginx
```

**Проверка работы:**

```bash
# Проверяем, что Nginx слушает порт 80
sudo netstat -tlnp | grep :80

# Или используем ss
sudo ss -tlnp | grep :80
```

---

## 5. Получение SSL-сертификата (Let's Encrypt)

**Что такое SSL/TLS:**  
SSL/TLS — это протоколы шифрования, которые защищают данные между браузером и сервером. HTTPS — это HTTP с SSL/TLS.

**Let's Encrypt:**  
Let's Encrypt — это бесплатный центр сертификации, который выдаёт SSL-сертификаты. Сертификаты действительны 90 дней и могут быть автоматически продлены.

**Установка Certbot:**

Certbot — это инструмент для автоматического получения и обновления SSL-сертификатов.

```bash
# Устанавливаем Certbot
sudo apt install -y certbot python3-certbot-nginx

# Проверяем установку
certbot --version
```

**Получение SSL-сертификата:**

```bash
# Получаем сертификат (замените tempmail.dev на ваш домен)
sudo certbot --nginx -d tempmail.dev -d www.tempmail.dev

# Certbot спросит:
# - Email для уведомлений (введите ваш email)
# - Согласие с условиями (Y)
# - Рассылка от EFF (можно отказаться, N)
```

**Что делает Certbot:**
- Автоматически получает сертификат
- Настраивает Nginx для HTTPS
- Настраивает автоматическое обновление

**Проверка автоматического обновления:**

```bash
# Проверяем, что автообновление настроено
sudo systemctl status certbot.timer

# Тестируем обновление (dry-run)
sudo certbot renew --dry-run
```

**Обновление конфигурации Nginx (Certbot делает это автоматически):**

После получения сертификата файл `/etc/nginx/sites-available/tempmail` будет выглядеть так:

```nginx
server {
    listen 80;
    server_name tempmail.dev www.tempmail.dev;
    
    # Перенаправление HTTP на HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name tempmail.dev www.tempmail.dev;

    # Пути к SSL-сертификатам (Certbot добавляет автоматически)
    ssl_certificate /etc/letsencrypt/live/tempmail.dev/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tempmail.dev/privkey.pem;
    
    # Настройки SSL (рекомендуемые)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # HSTS (HTTP Strict Transport Security)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Логирование
    access_log /var/log/nginx/tempmail-access.log;
    error_log /var/log/nginx/tempmail-error.log;

    # Максимальный размер тела запроса
    client_max_body_size 10M;

    # Проксирование на приложение
    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

**Проверка HTTPS:**

```bash
# Проверяем, что сертификат установлен
curl -I https://tempmail.dev

# Или откройте в браузере
# https://tempmail.dev
```

---

## 6. Подготовка приложения к продакшену

**Обновляем конфигурацию приложения:**

**Создаём файл `.env.prod` на сервере:**

```bash
# Переходим в директорию проекта
cd ~/tempmail

# Создаём файл с продакшен-конфигурацией
nano .env.prod
```

**Содержимое `.env.prod`:**

```env
# HTTP и SMTP порты
HTTP_PORT=8080
SMTP_PORT=25

# База данных
DB_HOST=postgres
DB_PORT=5432
DB_NAME=tempmail
DB_USER=tempmail_user
DB_PASSWORD=ВАШ_СЛОЖНЫЙ_ПАРОЛЬ

# Redis (если используется)
REDIS_HOST=redis
REDIS_PORT=6379

# Почтовые настройки
MAIL_DOMAIN=tempmail.dev
DEFAULT_TTL=1h
MAX_TTL=24h
CLEANUP_INTERVAL=5m

# Другие настройки
LOG_LEVEL=info
```

**Важно:**  
- Используйте сложные пароли в продакшене
- Не коммитьте `.env.prod` в Git (добавьте в `.gitignore`)
- Храните пароли в секретах (например, Docker secrets)

**Обновляем `docker-compose.prod.yml`:**

**Создаём файл `docker-compose.prod.yml` в корне проекта:**

```yaml
version: '3.8'

services:
  # Наше приложение
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: tempmail-app
    restart: unless-stopped
    env_file:
      - .env.prod
    networks:
      - tempmail-network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    # Не публикуем порты наружу (Nginx будет проксировать)
    expose:
      - "8080"
    # Для SMTP нужен порт 25
    ports:
      - "25:25"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: tempmail-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations/001_init.up.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - tempmail-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # Redis
  redis:
    image: redis:7-alpine
    container_name: tempmail-redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    networks:
      - tempmail-network
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  tempmail-network:
    driver: bridge

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
```

**Важные изменения для продакшена:**

- `restart: unless-stopped` — автоматический перезапуск при падении
- `expose` вместо `ports` для внутренних портов
- Порты публикуются только для SMTP (25) и через Nginx для HTTP
- Настроены healthcheck для мониторинга
- Логирование с ротацией

---

## 7. Настройка файрвола

**Что такое файрвол:**  
Файрвол (брандмауэр) — это система безопасности, которая контролирует входящий и исходящий сетевой трафик.

**Настройка UFW (Uncomplicated Firewall):**

UFW — это простой интерфейс для настройки файрвола в Ubuntu.

```bash
# Проверяем статус
sudo ufw status

# Разрешаем SSH (ОБЯЗАТЕЛЬНО перед включением файрвола!)
sudo ufw allow 22/tcp

# Разрешаем HTTP
sudo ufw allow 80/tcp

# Разрешаем HTTPS
sudo ufw allow 443/tcp

# Разрешаем SMTP
sudo ufw allow 25/tcp

# Включаем файрвол
sudo ufw enable

# Проверяем статус
sudo ufw status verbose
```

**Важно:**  
- Всегда разрешайте SSH (порт 22) перед включением файрвола
- Если заблокируете SSH, придётся обращаться к провайдеру

**Проверка открытых портов:**

```bash
# Смотрим открытые порты
sudo netstat -tlnp
# Или
sudo ss -tlnp
```

---

## 8. Развёртывание приложения

**Шаг 1: Клонируем проект на сервер**

```bash
# Устанавливаем Git (если не установлен)
sudo apt install -y git

# Клонируем репозиторий (замените URL на ваш)
git clone https://github.com/yourusername/tempmail.git
cd tempmail
```

**Или загружаем файлы через SCP:**

```bash
# На локальном компьютере
scp -r . username@IP_ADDRESS:~/tempmail
```

**Шаг 2: Создаём `.env.prod` файл**

```bash
nano .env.prod
# Вставьте конфигурацию из предыдущего раздела
```

**Шаг 3: Собираем Docker-образ**

```bash
# Собираем образ
docker compose -f docker-compose.prod.yml build

# Или если используете docker-compose (старая версия)
docker-compose -f docker-compose.prod.yml build
```

**Шаг 4: Запускаем приложение**

```bash
# Запускаем в фоновом режиме
docker compose -f docker-compose.prod.yml up -d

# Проверяем статус
docker compose -f docker-compose.prod.yml ps

# Смотрим логи
docker compose -f docker-compose.prod.yml logs -f app
```

**Шаг 5: Проверяем работу**

```bash
# Проверяем health check
curl http://localhost:8080/health

# Проверяем через Nginx (должен работать HTTPS)
curl https://tempmail.dev/health

# Проверяем API
curl https://tempmail.dev/api/v1/mailbox
```

---

## 9. Настройка автоматического обновления (опционально)

**Создаём скрипт для автоматического обновления:**

**Создаём файл `scripts/update.sh`:**

```bash
#!/bin/bash

# Скрипт для обновления приложения на продакшене

set -e

echo "Начинаем обновление..."

# Переходим в директорию проекта
cd ~/tempmail

# Получаем последние изменения из Git
git pull origin main

# Пересобираем образ
docker compose -f docker-compose.prod.yml build

# Останавливаем старые контейнеры
docker compose -f docker-compose.prod.yml down

# Запускаем новые контейнеры
docker compose -f docker-compose.prod.yml up -d

# Очищаем неиспользуемые образы
docker image prune -f

echo "Обновление завершено!"
```

**Делаем скрипт исполняемым:**

```bash
chmod +x scripts/update.sh
```

**Использование:**

```bash
./scripts/update.sh
```

---

## 10. Настройка мониторинга и логирования

**Просмотр логов:**

```bash
# Логи приложения
docker compose -f docker-compose.prod.yml logs -f app

# Логи базы данных
docker compose -f docker-compose.prod.yml logs -f postgres

# Логи Nginx
sudo tail -f /var/log/nginx/tempmail-access.log
sudo tail -f /var/log/nginx/tempmail-error.log
```

**Настройка ротации логов:**

**Создаём файл `/etc/logrotate.d/tempmail`:**

```bash
sudo nano /etc/logrotate.d/tempmail
```

**Содержимое:**

```
/var/log/nginx/tempmail-*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

**Проверка использования ресурсов:**

```bash
# Использование дискового пространства
df -h

# Использование памяти
free -h

# Загрузка процессора
top
# Или
htop  # (если установлен: sudo apt install htop)

# Использование ресурсов контейнерами
docker stats
```

---

## 11. Резервное копирование базы данных

**Важность бэкапов:**  
База данных содержит все письма и ящики. Регулярные бэкапы критически важны для восстановления данных.

**Создаём скрипт для бэкапа:**

**Создаём файл `scripts/backup.sh`:**

```bash
#!/bin/bash

# Скрипт для создания резервной копии базы данных

set -e

# Параметры
BACKUP_DIR=~/backups
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME=tempmail
DB_USER=tempmail_user
CONTAINER_NAME=tempmail-postgres

# Создаём директорию для бэкапов
mkdir -p $BACKUP_DIR

# Создаём бэкап
echo "Создание резервной копии базы данных..."
docker exec $CONTAINER_NAME pg_dump -U $DB_USER $DB_NAME | gzip > $BACKUP_DIR/backup_$DATE.sql.gz

# Удаляем старые бэкапы (старше 7 дней)
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +7 -delete

echo "Резервная копия создана: $BACKUP_DIR/backup_$DATE.sql.gz"
```

**Делаем скрипт исполняемым:**

```bash
chmod +x scripts/backup.sh
```

**Настройка автоматических бэкапов через cron:**

```bash
# Редактируем crontab
crontab -e

# Добавляем строку (бэкап каждый день в 3:00)
0 3 * * * /home/username/tempmail/scripts/backup.sh >> /home/username/tempmail/logs/backup.log 2>&1
```

**Восстановление из бэкапа:**

**Создаём скрипт `scripts/restore.sh`:**

```bash
#!/bin/bash

# Скрипт для восстановления базы данных из бэкапа

set -e

if [ -z "$1" ]; then
    echo "Использование: $0 <путь_к_бэкапу>"
    exit 1
fi

BACKUP_FILE=$1
DB_NAME=tempmail
DB_USER=tempmail_user
CONTAINER_NAME=tempmail-postgres

echo "Восстановление базы данных из $BACKUP_FILE..."

# Распаковываем и восстанавливаем
gunzip -c $BACKUP_FILE | docker exec -i $CONTAINER_NAME psql -U $DB_USER -d $DB_NAME

echo "База данных восстановлена!"
```

**Использование:**

```bash
# Создать бэкап
./scripts/backup.sh

# Восстановить из бэкапа
./scripts/restore.sh ~/backups/backup_20231209_030000.sql.gz
```

---

## 12. Безопасность продакшен-сервера

**Рекомендации по безопасности:**

1. **Регулярные обновления:**
```bash
# Обновляем систему раз в неделю
sudo apt update && sudo apt upgrade -y
```

2. **Отключение входа по паролю (только SSH-ключи):**
```bash
# Редактируем конфигурацию SSH
sudo nano /etc/ssh/sshd_config

# Изменяем:
# PasswordAuthentication no
# PermitRootLogin no

# Перезагружаем SSH
sudo systemctl restart sshd
```

3. **Установка fail2ban (защита от брутфорса):**
```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

4. **Использование сильных паролей для БД:**
- Генерируйте случайные пароли
- Храните их в безопасном месте
- Не коммитьте в Git

5. **Ограничение доступа к базе данных:**
- База доступна только внутри Docker-сети
- Не публикуем порт PostgreSQL наружу

---

## 13. Тестирование продакшен-окружения

**Проверочный список:**

1. **HTTP/HTTPS работает:**
```bash
curl https://tempmail.dev/health
```

2. **API доступен:**
```bash
curl https://tempmail.dev/api/v1/mailbox
```

3. **SMTP принимает письма:**
```bash
# С другого сервера
swaks --to test@tempmail.dev \
      --from sender@example.com \
      --server mail.tempmail.dev \
      --header "Subject: Test" \
      --body "Hello from production!"
```

4. **SSL-сертификат валиден:**
```bash
curl -v https://tempmail.dev 2>&1 | grep -i "SSL\|TLS"
```

5. **Бэкапы создаются:**
```bash
./scripts/backup.sh
ls -lh ~/backups/
```

---

## Словарь терминов

| Термин | Объяснение |
|--------|------------|
| **VPS** | Виртуальный приватный сервер |
| **DNS** | Система доменных имён, преобразует домены в IP-адреса |
| **A-запись** | DNS-запись, указывающая на IP-адрес |
| **MX-запись** | DNS-запись для почтового сервера |
| **SPF** | Протокол для защиты от подделки email |
| **SSL/TLS** | Протоколы шифрования для HTTPS |
| **Let's Encrypt** | Бесплатный центр сертификации |
| **Certbot** | Инструмент для получения SSL-сертификатов |
| **Nginx** | Веб-сервер и reverse proxy |
| **Reverse proxy** | Сервер, который перенаправляет запросы на другие серверы |
| **UFW** | Простой файрвол для Ubuntu |
| **Cron** | Планировщик задач в Linux |
| **Health check** | Проверка работоспособности сервиса |

---

## Что мы узнали

- Как купить и настроить доменное имя
- Как выбрать и настроить VPS-сервер
- Как настроить DNS-записи (A, MX, TXT)
- Как установить и настроить Nginx как reverse proxy
- Как получить бесплатный SSL-сертификат через Let's Encrypt
- Как развернуть приложение на продакшене через Docker
- Как настроить файрвол для безопасности
- Как настроить автоматические бэкапы базы данных
- Как мониторить и логировать работу приложения
- Основы безопасности продакшен-сервера

---

[Вернуться к оглавлению](./README.md)
