# LEGO Application - Объединенная Docker Архитектура

## Обзор

Этот проект объединяет фронтенд (Vue.js) и бэкенд (FastAPI) приложения в единую Docker-архитектуру с использованием Nginx как обратного прокси-сервера.

## Структура проекта

```
LEGO/
├── LEGO_API/               # Backend (FastAPI)
│   ├── src/
│   ├── Dockerfile
│   └── requirements.txt
├── LEGO_frontend/          # Frontend (Vue.js)
│   ├── src/
│   ├── Dockerfile
│   └── package.json
├── docker-compose.yml      # Основной compose файл
├── nginx.conf              # Конфигурация Nginx
└── README.md
```

## 🚀 Быстрый старт (локальная разработка)

### Windows (PowerShell):

```powershell
# Запуск одной командой
.\start.ps1
```

### Linux/Mac:

```bash
# Запуск одной командой
./start.sh
```

### Ручной запуск:

```bash
# 1. Скопировать настройки
cp LEGO_API/.env .env

# 2. Запустить все сервисы
docker-compose up -d

# 3. Открыть в браузере
# http://localhost - приложение
# http://localhost/docs - API документация
```

## Варианты запуска

### Вариант 1: Единый docker-compose.yml (Рекомендуется)

Самый простой и удобный способ:

```bash
# Скопируйте .env файл из LEGO_API/ в корень проекта
cp LEGO_API/.env .env

# Запуск всех сервисов
docker-compose up -d

# Просмотр логов
docker-compose logs -f

# Остановка
docker-compose down
```

### Вариант 2: Использование нескольких compose файлов

```bash
# Создание общей сети
docker network create lego_network

# Запуск backend сервисов
docker-compose -f LEGO_API/docker-compose.yml up -d

# Запуск frontend сервисов
docker-compose -f LEGO_frontend/docker-compose.yml up -d

# Запуск общего nginx
docker-compose -f docker-compose.override.yml up -d
```

## Архитектура сервисов

### Сервисы в составе:

1. **PostgreSQL** (`postgres:5432`) - База данных
2. **RabbitMQ** (`rabbitmq:5672`, `15672`) - Брокер сообщений
3. **Backend API** (`backend:8000`) - FastAPI приложение
4. **Celery Worker** - Обработка фоновых задач
5. **Celery Beat** - Планировщик задач
6. **Frontend** (`frontend:80`) - Vue.js приложение
7. **Nginx** (`nginx:80`, `443`) - Обратный прокси и веб-сервер

## Nginx - Подробное объяснение

### Что такое Nginx и зачем он нужен?

**Nginx** (произносится "engine-x") - это высокопроизводительный веб-сервер и обратный прокси-сервер. В нашей архитектуре он выполняет роль "входной точки" для всех HTTP-запросов.

### Основные функции Nginx в нашем проекте:

#### 1. **Обратный прокси (Reverse Proxy)**

Nginx принимает все входящие запросы и перенаправляет их к соответствующим сервисам:

- Запросы к `/api/*` → Backend (FastAPI на порту 8000)
- Запросы к `/docs` → API документация
- Остальные запросы → Frontend (Vue.js на порту 80)

```nginx
# Пример маршрутизации
location /api/ {
    proxy_pass http://backend:8000/;  # Перенаправление в backend
}

location / {
    proxy_pass http://frontend:80;    # Перенаправление во frontend
}
```

#### 2. **Единая точка входа**

Вместо того чтобы обращаться к разным портам:

- ❌ `http://localhost:8000/users` (backend)
- ❌ `http://localhost:3000` (frontend)

Все запросы идут через один порт:

- ✅ `http://localhost/api/users` (API)
- ✅ `http://localhost/` (приложение)

#### 3. **Балансировка нагрузки**

```nginx
upstream backend {
    server backend1:8000;
    server backend2:8000;  # Можно добавить несколько инстансов
    keepalive 32;
}
```

#### 4. **Кеширование статических файлов**

```nginx
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
    proxy_cache_valid 200 1y;
    add_header Cache-Control "public, immutable";
}
```

#### 5. **Сжатие данных (Gzip)**

```nginx
gzip on;
gzip_types text/css application/javascript application/json;
```

#### 6. **SSL Termination (HTTPS)**

Nginx может обрабатывать SSL-сертификаты и шифрование, разгружая backend приложения.

#### 7. **Защита и ограничения**

```nginx
client_max_body_size 100M;  # Ограничение размера загружаемых файлов
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;  # Rate limiting
```

### Преимущества использования Nginx:

1. **Производительность**: Nginx эффективно обрабатывает тысячи одновременных соединений
2. **Безопасность**: Скрывает внутреннюю архитектуру приложения
3. **Масштабируемость**: Легко добавить новые сервисы
4. **Мониторинг**: Централизованное логирование всех запросов
5. **SEO**: Один домен для всего приложения

### Без Nginx (проблемы):

```
Пользователь → Frontend (localhost:3000)
             → Backend (localhost:8000) - CORS проблемы!
```

### С Nginx (решение):

```
Пользователь → Nginx (localhost:80) → Frontend (internal)
                                    → Backend (internal)
```

## Переменные окружения

Создайте `.env` файл в корне проекта:

```env
# База данных
DB_HOST=postgres
DB_PORT=5432
DB_NAME=your_database_name
DB_USER=your_db_user
DB_PASS=your_secure_password

# JWT (Генерируйте надежный ключ!)
SECRET_KEY=your_very_long_and_secure_secret_key_here
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# Email настройки
SMTP_HOST=smtp.your-provider.com
SMTP_PORT=587
SMTP_USER=your_email@domain.com
SMTP_PASSWORD=your_app_specific_password
EMAIL_FROM=noreply@yourdomain.com

# RabbitMQ
RABBITMQ_USER=your_rabbitmq_user
RABBITMQ_PASSWORD=your_rabbitmq_password

# Базовый URL (без порта для продакшена)
BASE_URL=https://yourdomain.com
```

## Мониторинг и отладка

### Полезные команды:

```bash
# Просмотр состояния контейнеров
docker-compose ps

# Просмотр логов конкретного сервиса
docker-compose logs -f nginx
docker-compose logs -f backend

# Вход в контейнер для отладки
docker-compose exec nginx sh
docker-compose exec backend bash

# Перезапуск конкретного сервиса
docker-compose restart nginx

# Пересборка после изменений
docker-compose up --build
```

### Порты доступа в разработке:

- **Приложение**: http://localhost
- **API документация**: http://localhost/docs
- **PostgreSQL**: localhost:5432 (только для подключения из IDE)
- **RabbitMQ Management**: http://localhost:15672 (admin interface)
- **Backend напрямую**: http://localhost:8000 (только для отладки)

## 🔒 Безопасность API

### В разработке (development):

- Backend доступен напрямую на порту 8000 для отладки
- Все порты открыты для удобства разработки

### В продакшене (production):

- ❌ **Backend НЕ доступен** напрямую пользователям
- ✅ Все запросы идут **только через Nginx**
- ✅ Внутренние сервисы изолированы в Docker сети
- ✅ Открыт только порт 80 (HTTP) и 443 (HTTPS)

```
Пользователь → Nginx (80/443) → Backend (внутренняя сеть)
             ↘ Frontend (внутренняя сеть)
```

**Почему это безопасно:**

1. Пользователи не могут обойти Nginx и обратиться к API напрямую
2. Nginx контролирует все входящие запросы
3. Можно настроить rate limiting, фильтрацию, аутентификацию
4. SSL/TLS терминируется на Nginx, разгружая backend

## Развертывание в продакшене

### Для продакшена используйте:

```bash
docker-compose -f docker-compose.prod.yml up -d
```

### Ключевые отличия prod версии:

1. **Закрыты внутренние порты** - нет прямого доступа к backend/database
2. **Оптимизированы Dockerfile** - alpine версии, меньший размер
3. **Автоперезапуск** - `restart: unless-stopped`
4. **Health checks** - мониторинг состояния сервисов
5. **Масштабирование** - можно запустить несколько Celery worker'ов
6. **Мониторинг** - опциональные Prometheus и Grafana

### Дополнительные настройки:

1. Настройте SSL сертификаты (Let's Encrypt)
2. Используйте внешние базы данных (AWS RDS, Azure Database)
3. Настройте резервное копирование
4. Настройте мониторинг и алерты
5. Используйте секреты Docker для sensitive данных

## Решение проблем

### Контейнеры не запускаются:

```bash
docker-compose down -v  # Удалить все volumes
docker system prune -a  # Очистить Docker
```

### Проблемы с сетью:

```bash
docker network ls
docker network prune
```

### Проблемы с правами доступа:

```bash
sudo chown -R $USER:$USER logs/
```
