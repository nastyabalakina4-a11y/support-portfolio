← [План](plan.md)

# 3. HTTPS (Let's Encrypt)

**Статус:** ✅ завершено

**Задача:** сервис должен открываться только по HTTPS, с настоящим доверенным сертификатом.

## Пошагово
- [x] Установить Nginx как reverse proxy перед n8n (уже был установлен с этапа хостинга сайтов)
- [x] Создать отдельный server block для поддомена n8n с `proxy_pass` на `http://127.0.0.1:5678`
- [x] Установить certbot: `sudo apt install certbot python3-certbot-nginx -y`
- [x] Выпустить сертификат: `sudo certbot --nginx -d n8n.viver-lab.com`
- [x] Автопродление настраивается certbot автоматически (systemd timer)
- [x] Обновить переменные окружения n8n: `N8N_PROTOCOL=https`, `N8N_HOST=n8n.viver-lab.com`, `WEBHOOK_URL=https://n8n.viver-lab.com/`
- [x] Убрать `N8N_SECURE_COOKIE=false` — больше не нужно, теперь настоящий HTTPS
- [x] Порт 5678 закрыт снаружи — привязан только к `127.0.0.1` в `docker-compose.yml`, доступ только через Nginx (80/443)

## Что нужно узнать
- [x] Certbot сам создаёт systemd-таймер для автопродления сертификата — отдельно настраивать не нужно
- [x] Поддомен проще path-based прокси — не пришлось указывать `N8N_PATH`, n8n работает в корне (`/`) своего поддомена

## Что сделано на практике
- Создан `/etc/nginx/sites-available/n8n` — server block с `server_name n8n.viver-lab.com`, `proxy_pass` на `http://127.0.0.1:5678` (с корректными заголовками `Upgrade`/`Connection` для WebSocket, который n8n использует)
- Подключён через симлинк в `sites-enabled`, конфиг проверен (`nginx -t`) и применён
- `certbot --nginx -d n8n.viver-lab.com` выпустил сертификат и сам донастроил Nginx на HTTPS (`Congratulations! You have successfully enabled HTTPS`)
- В `docker-compose.yml` порт изменён с `"5678:5678"` на `"127.0.0.1:5678:5678"` — теперь порт слушает только локально, снаружи недоступен напрямую
- Старое правило `ufw allow 5678/tcp` удалено
- Контейнер пересоздан (`docker compose up -d`) с новыми переменными окружения

## Результат
`https://n8n.viver-lab.com` открывается по HTTPS с доверенным сертификатом, вход работает. Прямой доступ по старому адресу `http://<IP>:5678` подтверждённо отключён (не открывается вообще, порт закрыт). Все пункты обязательной инфраструктурной части ТЗ (1-4) теперь выполнены полностью, без компромиссов на «пока без домена».

## Проблемы и решения
- `cat > /etc/nginx/... << EOF` с `sudo` не сработал (`Permission denied`) — перенаправление `>` выполняется до применения `sudo`. Решение: `sudo tee /etc/nginx/... > /dev/null << EOF`
