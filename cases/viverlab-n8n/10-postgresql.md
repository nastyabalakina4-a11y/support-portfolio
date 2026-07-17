← [План](plan.md)

# 10. Миграция на PostgreSQL

**Статус:** ✅ завершено

**Задача:** перевести n8n с встроенной SQLite на полноценную PostgreSQL — требование обновлённого ТЗ (важно перед серьёзной нагрузкой от автоматизаций публикаций и аналитики).

## Пошагово
- [x] Добавить сервис `postgres` в `docker-compose.yml` (образ `postgres:16-alpine`, отдельный volume для данных БД)
- [x] Настроить переменные окружения n8n для подключения к PostgreSQL
- [x] Решить вопрос переноса данных — выбрано начать с чистой БД (в n8n был только тестовый workflow и один credential, переносить было нечего критичного)
- [x] Обновить скрипт бэкапа — теперь бэкапится и PostgreSQL (`pg_dump`), не только volume n8n
- [x] Проверить персистентность и процедуру обновления заново (`docker compose down`/`up`) уже с PostgreSQL

## Что нужно узнать
- [x] Официальная миграция данных SQLite → PostgreSQL не потребовалась — начали с чистой БД осознанно, так как данные были чисто тестовые
- [x] `pg_dump` — логический бэкап (SQL-дамp), не зависит от версии/формата файлов на диске, поэтому предпочтительнее прямого архивирования файлов БД PostgreSQL

## Что сделано на практике
Обновлён `~/n8n/docker-compose.yml` — добавлен сервис `postgres`:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=<пароль>
      - POSTGRES_DB=n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data

  n8n:
    # ...
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=<пароль>
    depends_on:
      - postgres

volumes:
  n8n_data:
  postgres_data:
```

Скрипт бэкапа дополнен дампом PostgreSQL:
```bash
sudo docker exec n8n-postgres-1 pg_dump -U n8n n8n | gzip > "$BACKUP_DIR/postgres-backup-$DATE.sql.gz"
```

Проверено практически:
- `docker compose up -d` выполнил миграции схемы PostgreSQL с нуля (видно в логах: `Starting migration ... Finished migration ...` для каждой из ~15 миграций n8n) — без ошибок
- `pg_dump` создал корректный дамп (46 КБ, валидный SQL-заголовок `PostgreSQL database dump`)
- После `docker compose down && docker compose up -d` вход в n8n прошёл без повторного запроса создания аккаунта — данные в PostgreSQL сохраняются между пересозданием контейнеров так же, как раньше в SQLite

## Результат
n8n работает на полноценной PostgreSQL вместо встроенной SQLite — соответствует требованию обновлённого ТЗ и лучше готово к нагрузке от будущих автоматизаций. Бэкапы теперь покрывают обе части хранения (конфиги/ключ шифрования + сама база данных).

## Проблемы и решения
_Проблем не возникло — миграция прошла чисто, поскольку решили начать с пустой базы вместо переноса тестовых данных._
