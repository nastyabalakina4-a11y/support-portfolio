← [План](plan.md)

# 11. Хостинг основного сайта Viver Lab (viver-lab.com)

**Статус:** ✅ завершено (базовая часть) — сам контент сайта делает коллега, инфраструктура готова к приёму файлов

**Задача:** разместить статичный сайт (HTML/CSS/JS + картинки) на корневом домене `viver-lab.com`, который делает коллега, с безопасной передачей ей доступа для деплоя — без доступа к остальному серверу.

## Пошагово
- [x] Добавить DNS-записи в Cloudflare: A-запись `@` (корень) → IP сервера, CNAME `www` → `viver-lab.com`
- [x] Создать отдельную папку под сайт (`/var/www/viver-lab.com/html`)
- [x] Создать отдельный Nginx server block для `viver-lab.com` и `www.viver-lab.com`
- [x] Выпустить SSL-сертификат через certbot на оба адреса
- [x] Создать отдельного системного пользователя `deploy` — без `sudo`, с домашней папкой прямо в папке сайта
- [x] Настроить вход по SSH-ключу для коллеги (добавлен её публичный ключ в `authorized_keys` пользователя `deploy`)
- [x] Подготовить для коллеги файл `CLAUDE.md` с точной командой деплоя (`scp`) — кладётся в её проект, чтобы её собственный Claude Code сразу знал, куда и как заливать сайт

## Что нужно узнать
- [x] Для чисто статичного HTML-сайта (без сборки) отдельный build-контейнер не нужен — файлы можно просматривать локально в браузере или через staging-поддомен, без Docker
- [x] Добавление пользователя в группу `docker` эквивалентно root-доступу ко всему серверу (через Docker можно смонтировать любую директорию хоста) — категорически не подходит для ограниченного пользователя деплоя

## Что сделано на практике
```
# DNS в Cloudflare: A @ -> IP сервера, CNAME www -> viver-lab.com (оба DNS only)

sudo mkdir -p /var/www/viver-lab.com/html
sudo adduser deploy --home /var/www/viver-lab.com --no-create-home
sudo chown -R deploy:deploy /var/www/viver-lab.com
sudo chmod 755 /var/www/viver-lab.com/html
sudo mkdir -p /var/www/viver-lab.com/.ssh
sudo touch /var/www/viver-lab.com/.ssh/authorized_keys
sudo chmod 700 /var/www/viver-lab.com/.ssh
sudo chmod 600 /var/www/viver-lab.com/.ssh/authorized_keys
sudo chown -R deploy:deploy /var/www/viver-lab.com/.ssh

# Nginx: /etc/nginx/sites-available/viver-lab.com
#   server_name viver-lab.com www.viver-lab.com; root /var/www/viver-lab.com/html;

sudo certbot --nginx -d viver-lab.com -d www.viver-lab.com

# публичный ключ коллеги добавлен в authorized_keys пользователя deploy
```

Команда деплоя для коллеги (без sudo, без доступа к остальному серверу):
```
scp -P 2244 -r .\* deploy@<IP_сервера>:/var/www/viver-lab.com/html/
```

## Результат
`https://viver-lab.com` и `https://www.viver-lab.com` открываются по HTTPS. Коллега может самостоятельно заливать/обновлять файлы сайта по SSH-ключу через пользователя `deploy`, не имея доступа ни к чему другому на сервере (ни `sudo`, ни к n8n, ни к остальным папкам).

## Проблемы и решения
- Возник запрос добавить пользователя `deploy` в группу `docker`, чтобы коллега могла "создать контейнер для сборки сайта". Отказались — членство в группе `docker` равносильно root-доступу к хосту, что свело бы на нет весь смысл ограниченного пользователя. Так как сайт полностью статичный (HTML/CSS/JS), сборка вообще не нужна — обсуждение отложено до уточнения, что коллеге реально требуется (скорее всего, хватит либо локального просмотра файлов, либо отдельного staging-поддомена без Docker)
