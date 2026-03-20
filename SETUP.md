# Инструкция по установке Laravel Octane + RoadRunner Boilerplate

Этот boilerplate предназначен для быстрого развертывания Laravel-проекта с архитектурой **Laravel Octane + RoadRunner + PostgreSQL + Redis + pgAdmin**.

## Для каких приложений подходит эта архитектура

### ✅ 1. Высоконагруженные API

**Идеальный кейс для RoadRunner**

* REST / GraphQL API с высоким RPS
* Микросервисы
* Real-time приложения (WebSocket через RoadRunner)

**Почему RoadRunner:** Persistent workers — приложение загружается один раз и обрабатывает тысячи запросов без повторного bootstrap.

---

### ✅ 2. Blade / Livewire / Inertia приложения

* Blade templates + Server-side rendering
* Livewire
* Inertia + Vue/React

**Почему подходит:** Laravel Octane полностью совместим с Blade, Livewire и Inertia. Получаете прирост производительности без изменения кода.

---

### ✅ 3. SaaS-панели и админки

* CRM / ERP системы
* Internal tools
* Корпоративные порталы

---

### ⚠️ Важно: особенности Octane

При работе с Laravel Octane нужно учитывать, что **приложение живёт между запросами**:

* Не храните состояние в статических свойствах классов
* Используйте `Octane::concurrently()` для параллельных задач
* Будьте осторожны с singleton-сервисами — они переиспользуются между запросами
* Подробнее: [Laravel Octane Documentation](https://laravel.com/docs/octane)

---

## 🚀 Процесс установки для разработки

### 1. Создание проекта Laravel

Создайте новый проект Laravel:

```bash
composer create-project laravel/laravel my-app
```

```bash
cd my-app
```

### 2. Установка Laravel Octane

```bash
composer require laravel/octane
```

```bash
php artisan octane:install --server=roadrunner
```

Эта команда устанавливает и настраивает Octane для RoadRunner. Файл воркера создаётся/используется внутри пакета Octane (`vendor/laravel/octane/bin/roadrunner-worker`), поэтому `worker.php` в корне проекта не требуется.

### 3. Копирование файлов boilerplate

Скопируйте следующие файлы и папки из данного boilerplate в корень вашего нового проекта Laravel:

* Папку `docker/` (включая все подпапки и файлы)
* Файлы `docker-compose.yml`, `docker-compose.prod.yml`, `docker-compose.prod.local.yml`
* Файл `Makefile`
* Файл `.rr.yaml`
* Файл `.dockerignore`
* Файл `.env.production.example`

### 4. Настройка окружения (.env)

Откройте файл `.env` в корне Laravel и выполните следующие изменения:

**a) Настройте подключение к БД:**

```dotenv
DB_CONNECTION=pgsql
DB_HOST=laravel-postgres-rr
DB_PORT=5432
DB_DATABASE=laravel
DB_USERNAME=postgres
DB_PASSWORD=root
```

**b) Настройте Redis и драйверы Laravel (обязательно для этого boilerplate):**

```dotenv
REDIS_CLIENT=phpredis
REDIS_HOST=laravel-redis-rr
REDIS_PORT=6379
REDIS_PASSWORD=

SESSION_DRIVER=redis
CACHE_STORE=redis
QUEUE_CONNECTION=redis
```

**c) Настройте Octane:**

```dotenv
OCTANE_SERVER=roadrunner
```

**d) Добавьте в конец `.env` секцию из файла `.env.docker`:**

```dotenv
# --- pgAdmin Web Interface ---
PGADMIN_DEFAULT_EMAIL=admin@example.com
PGADMIN_DEFAULT_PASSWORD=admin

# --- Docker ports ---
APP_PORT=8000
DB_FORWARD_PORT=5432
REDIS_FORWARD_PORT=6379
PGADMIN_PORT=8080

# --- Base path to the application inside the container ---
APP_BASE_PATH=/var/www/laravel

# --- Xdebug Configuration ---
XDEBUG_MODE=off
XDEBUG_START=no
XDEBUG_CLIENT_HOST=host.docker.internal

# --- RoadRunner Configuration ---
RR_HTTP_NUM_WORKERS=2
RR_LOG_LEVEL=debug
```

### 5. Инициализация проекта

Запустите команду, которая соберет контейнеры, установит все зависимости и выполнит миграции:

```bash
make setup
```

После завершения:
- **Приложение:** http://localhost:8000
- **pgAdmin:** http://localhost:8080
- **Vite HMR:** http://localhost:5173

### 6. Работа с RoadRunner в разработке

В отличие от PHP-FPM, RoadRunner **не перечитывает файлы автоматически**. После изменения PHP-кода нужно перезагрузить воркеры:

```bash
# Перезагрузить воркеры (быстро, без перезапуска контейнера)
make rr-reload

# Посмотреть статус воркеров
make rr-workers
```

> **Совет:** Для автоматической перезагрузки при изменении файлов можно использовать `fswatch` или аналогичные утилиты:
> ```bash
> fswatch -o app/ routes/ config/ | xargs -n1 -I{} make rr-reload
> ```

### 7. Xdebug

Для включения Xdebug измените в `.env`:

```dotenv
XDEBUG_MODE=debug
XDEBUG_START=yes
```

Затем пересоберите контейнер:

```bash
make rebuild
make up
```

---

## 🏭 Развертывание на Production

Этот boilerplate рассчитан на деплой через **Dokploy**.

### Настройка `.env.production`

Создайте production-файл из шаблона:

```bash
cp .env.production.example .env.production
```

Проверьте и заполните в `.env.production` production-значения:

```dotenv
APP_ENV=production
APP_DEBUG=false
APP_KEY=base64:your-generated-key
APP_URL=https://your-domain.com
APP_BASE_PATH=/var/www/laravel

DB_CONNECTION=pgsql
DB_HOST=laravel-postgres-rr
DB_PORT=5432
DB_DATABASE=your_db
DB_USERNAME=your_user
DB_PASSWORD=<strong_password>

REDIS_CLIENT=phpredis
REDIS_HOST=laravel-redis-rr
REDIS_PORT=6379
REDIS_PASSWORD=

SESSION_DRIVER=redis
CACHE_STORE=redis
QUEUE_CONNECTION=redis

OCTANE_SERVER=roadrunner
RR_HTTP_NUM_WORKERS=8
RR_LOG_LEVEL=warn
```

`APP_DEBUG=false` обязателен для production.  
`.env.production` не должен попадать в Git.

### Деплой в Dokploy

После создания проекта в Dokploy:

1. Откройте вкладку **Environment** -> **Environment Settings**.
2. Добавьте все необходимые переменные из `.env.production`.
3. В настройках сервиса укажите:
   - В поле **Post-deployment** (или аналогичном): `php artisan migrate --force && php artisan optimize:clear`.
   - Проверьте имя приложения и сервисов в соответствии с `docker-compose.prod.yml`.

---

## 📋 Полный список Make-команд

| Команда | Описание |
|---------|----------|
| `make help` | Показать справку |
| `make setup` | Полная инициализация проекта |
| `make up` | Запустить контейнеры (dev) |
| `make up-prod` | Запустить контейнеры (prod local, из `.env.production`) |
| `make down` | Остановить контейнеры |
| `make down-prod` | Остановить контейнеры (prod local) |
| `make restart` | Перезапустить контейнеры |
| `make build` | Собрать образы |
| `make rebuild` | Пересобрать без кэша |
| `make logs` | Логи всех сервисов |
| `make logs-app` | Логи RoadRunner |
| `make logs-queue` | Логи queue worker (prod local) |
| `make logs-scheduler` | Логи scheduler (prod local) |
| `make status` | Статус контейнеров |
| `make shell` | Войти в контейнер приложения |
| `make shell-postgres` | PostgreSQL CLI |
| `make rr-reload` | Перезагрузить воркеры RoadRunner |
| `make rr-workers` | Статус воркеров |
| `make artisan CMD="..."` | Artisan-команда |
| `make migrate` | Запустить миграции |
| `make fresh` | Пересоздать БД + сиды |
| `make test-php` | Запустить тесты |
| `make validate` | Проверить доступность сервисов |
| `make info` | Информация о проекте |
| `make clean` | Удалить контейнеры и тома |
| `make clean-all` | Полная очистка |
