_ssh -i /home/YOUR_USER/.ssh/YOUR_KEY -p YOUR_SSH_PORT YOUR_USER@YOUR_SERVER_IP_

Перевірити параметри серверу, що є на сервері, має бути докер і nginx, можливо щось видалити. Створити новий піддомен, встановити новий сервер nginx, запустити сайт, максимально попрактикувати навички в nginx, SSL/TLS - встановити сертифікати, і все налаштувати, що з цим може бути пов'язане.

# План роботи

Спочатку діагностика (тільки читання), потім очищення якщо треба, створення піддомену, потім налаштування nginx + SSL.

## Етап 1 — Діагностика сервера

### 1.1 Загальна інформація про систему

Показує: ядро Linux, версію, архітектуру (x86_64 чи arm64), ім'я хоста.

```bash
uname -a
```

Показує дистрибутив і версію ОС.

```bash
cat /etc/os-release
```

Показує характеристики процесора — скільки ядер, яка архітектура.

```bash
lscpu | grep -E "Architecture|CPU\(s\)|Model name"
```

Показує скільки RAM є, скільки використано, скільки swap.

```bash
free -h
```

Показує диски — розміри розділів, скільки місця зайнято і вільно.

```bash
df -h
```

### 1.2 Що зараз запущено і слухає мережу

Показує всі відкриті TCP-порти і які процеси їх слухають. Побачимо чи зайнятий порт 80, 443.

```bash
sudo ss -tlnp
```

Показує топ-20 процесів за споживанням пам'яті.

```bash
sudo ps aux --sort=-%mem | head -20
```

### 1.3 Перевірка nginx

Перевіряє чи встановлений nginx і яка версія.

```bash
which nginx && nginx -v
```

Статус служби nginx — запущена, зупинена, з помилками.

```bash
systemctl status nginx
```

Показує структуру конфігурації nginx.

```bash
ls /etc/nginx/
```

### 1.4 Перевірка Docker

Перевіряє чи встановлений Docker і версію.

```bash
which docker && docker --version
```

Показує всі контейнери (запущені і зупинені).

```bash
docker ps -a
```

Показує всі завантажені образи Docker.

```bash
docker images
```

Статус служби Docker.

```bash
systemctl status docker
```

### 1.5 Перевірка інших служб

Показує всі запущені служби systemd.

```bash
systemctl list-units --type=service --state=running
```

Показує заплановані задачі (cron) для root і поточного користувача.

```bash
sudo crontab -l; crontab -l
```

### 1.6 Продовжуємо діагностику — конфіг nginx

Головний конфіг nginx — основні параметри, які файли підключає.

```bash
cat /etc/nginx/nginx.conf
```

Які сайти зараз активні в nginx.

```bash
ls -la /etc/nginx/sites-enabled/
```

Вміст усіх активних конфігів сайтів.

```bash
cat /etc/nginx/sites-enabled/*
```

Альтернативна папка конфігів.

```bash
ls -la /etc/nginx/conf.d/
```

Що є в стандартній папці для сайтів.

```bash
ls /var/www/
```

Статус Apache.

```bash
systemctl status apache2
```

Які сайти налаштовані в Apache.

```bash
ls /etc/apache2/sites-enabled/
```

## Етап 2 — Створення піддомену your-subdomain.your-domain.com

### 2.1 DNS запис у Cloudflare (вручну в панелі)

Cloudflare → вибери домен → DNS → Add record:

| Поле         | Значення               |
| ------------ | ---------------------- |
| Type         | A                      |
| Name         | your-subdomain         |
| IPv4 address | YOUR_SERVER_IP         |
| Proxy status | DNS only (сіра хмарка) |
| TTL          | Auto                   |

> Важливо: спочатку ставимо DNS only. Це потрібно щоб Certbot міг отримати сертифікат напряму. Після отримання сертифікату — увімкнемо проксі.

Показує зовнішній IP сервера.

```bash
curl ifconfig.me
```

Перевіряємо що DNS резолвиться.

```bash
getent hosts your-subdomain.your-domain.com
```

### 2.2 Папка і HTML сторінка

Створює папку для файлів сайту.

```bash
mkdir -p /var/www/your-site
```

Перевірка:

```bash
ls -la /var/www/your-site/
```

### 2.3 Видаляємо старий nginx і встановлюємо Mainline

#### Крок 1 — Зберігаємо конфіги

```bash
cp -r /etc/nginx /root/nginx-backup
```

#### Крок 2 — Зупиняємо і видаляємо старий nginx

```bash
systemctl stop nginx
```

Видаляє nginx і всі його пакети. `--purge` — видаляє також конфігураційні файли.

```bash
apt remove --purge nginx nginx-common nginx-full nginx-core -y
```

Видаляє папку `/etc/nginx` повністю.

```bash
rm -rf /etc/nginx
```

Перевіряємо що nginx зник:

```bash
which nginx
```

#### Крок 3 — Встановлюємо Mainline з офіційного репо nginx.org

Завантажує GPG-ключ nginx.org.

```bash
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor > /usr/share/keyrings/nginx-archive-keyring.gpg
```

Додає офіційний репозиторій nginx Mainline для Debian 11 (bullseye).

```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/mainline/debian bullseye nginx" > /etc/apt/sources.list.d/nginx.list
```

Оновлює список пакетів.

```bash
apt update
```

Встановлює nginx Mainline.

```bash
apt install nginx -y
```

Перевіряємо версію:

```bash
nginx -v
```

Перевіряємо статус:

```bash
systemctl status nginx
```

Дивимось структуру папок нового nginx.

> Важливо: nginx.org версія не має `sites-available` і `sites-enabled` — конфіги кладуться в `conf.d/`. Ми додамо цю структуру вручну пізніше.

```bash
ls /etc/nginx/
```

#### Крок 4 — Переглядаємо nginx.conf

```bash
cat /etc/nginx/nginx.conf
```

Пояснення директив:

- `user nginx` — від якого користувача працюють worker-процеси
- `worker_processes auto` — по одному на кожне ядро CPU
- `error_log ... notice` — рівень логування (debug/info/notice/warn/error/crit)
- `worker_connections 1024` — максимум одночасних з'єднань на worker
- `sendfile on` — оптимізація передачі файлів
- `keepalive_timeout 65` — скільки секунд тримати з'єднання відкритим
- `include /etc/nginx/conf.d/*.conf` — підключає конфіги сайтів

#### Крок 5 — Створюємо конфіг сайту

```bash
cat > /etc/nginx/conf.d/your-site.conf << 'EOF'
server {
    listen 80;
    listen [::]:80;

    server_name your-subdomain.your-domain.com;

    root /var/www/your-site;
    index index.html;

    access_log /var/log/nginx/your-site_access.log main;
    error_log  /var/log/nginx/your-site_error.log warn;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

Перевіряємо синтаксис:

```bash
nginx -t
```

#### Крок 6 — Запускаємо nginx

```bash
systemctl start nginx
```

```bash
systemctl status nginx
```

Перевіряємо що сайт відповідає по HTTP:

```bash
curl -I http://your-subdomain.your-domain.com
```

## Етап 3 — SSL сертифікати

Перевіряємо Certbot:

```bash
certbot --version
```

Переглядаємо існуючі сертифікати:

```bash
ls /etc/letsencrypt/live/
```

Отримуємо сертифікат. Certbot автоматично:

- підтверджує домен через HTTP (http-01 challenge)
- отримує сертифікат
- модифікує конфіг nginx — додає SSL
- налаштовує редирект HTTP → HTTPS

```bash
certbot --nginx -d your-subdomain.your-domain.com
```

> Якщо помилка "nginx plugin does not appear to be installed":

```bash
apt install python3-certbot-nginx -y
```

Дивимось що Certbot додав до конфігу:

```bash
cat /etc/nginx/conf.d/your-site.conf
```

Перевіряємо HTTPS:

```bash
curl -I https://your-subdomain.your-domain.com
```

Перевіряємо редирект HTTP → HTTPS (має повернути 301):

```bash
curl -I http://your-subdomain.your-domain.com
```

### Вмикаємо Cloudflare проксі

1. Cloudflare → DNS → запис → Edit → хмарка **помаранчева (Proxied)** → Save
2. Cloudflare → SSL/TLS → Overview → режим **Full (strict)**

Різниця режимів:

- `Flexible` — Cloudflare↔сервер по HTTP. Небезпечно
- `Full` — HTTPS але без перевірки валідності сертифіката
- `Full (strict)` — HTTPS + валідний сертифікат. Найбезпечніше ✅

Перевіряємо через Cloudflare:

```bash
curl -I https://your-subdomain.your-domain.com
```

> Тепер `server: cloudflare` і `HTTP/2` — трафік йде через Cloudflare.

## Підсумок після Етапу 3

```
DNS запис        → your-subdomain.your-domain.com → YOUR_SERVER_IP
nginx            → встановлений з офіційного nginx.org
SSL сертифікат   → Let's Encrypt (діє 90 днів, автооновлення)
HTTP → HTTPS     → редирект працює (301)
Cloudflare       → Proxied + Full (strict)
HTTP/2           → увімкнений автоматично Cloudflare
```

## Етап 4 — server_tokens off

Навіщо? nginx повідомляє свою версію у кожній відповіді. Зловмисник може шукати вразливості для цієї версії.

```bash
nano /etc/nginx/nginx.conf
```

Додай після `keepalive_timeout 65;`:

```nginx
server_tokens off;
```

```bash
nginx -t && systemctl reload nginx
```

Перевіряємо напряму по IP (минаючи Cloudflare):

```bash
curl -I http://YOUR_SERVER_IP -H "Host: your-subdomain.your-domain.com"
```

> Тепер `Server: nginx` без версії.

## Етап 5 — Gzip стиснення

Навіщо? HTML, CSS, JS стискаються на 60-80%. Сторінка завантажується швидше.

```bash
nano /etc/nginx/nginx.conf
```

Замінити блок gzip на:

```nginx
gzip              on;
gzip_vary         on;       # Заголовок Vary: Accept-Encoding для CDN/проксі
gzip_proxied      any;      # Стискати навіть для проксованих запитів
gzip_comp_level   6;        # Рівень стиснення 1-9 (6 = оптимальний баланс)
gzip_buffers      16 8k;
gzip_http_version 1.1;
gzip_min_length   256;      # Не стискати файли менше 256 байт
gzip_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json
    application/xml
    image/svg+xml
    font/woff2;
```

```bash
nginx -t && systemctl reload nginx
```

Перевіряємо — шукаємо `Content-Encoding: gzip`:

```bash
curl -I -H "Accept-Encoding: gzip" https://YOUR_SERVER_IP -H "Host: your-subdomain.your-domain.com" -k
```

## Етап 6 — Заголовки безпеки

Навіщо? HTTP заголовки які браузер читає і застосовує додатковий захист.

```bash
nano /etc/nginx/sites-available/your-site.conf
```

Додати після `error_log`:

```nginx
# Security headers
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

Пояснення:

- `Strict-Transport-Security` (HSTS) — браузер завжди використовує HTTPS. `max-age=31536000` = 1 рік
- `X-Frame-Options: SAMEORIGIN` — захист від clickjacking (заборона iframe на чужих сайтах)
- `X-Content-Type-Options: nosniff` — браузер не вгадує тип файлу
- `X-XSS-Protection` — вбудований XSS фільтр браузера
- `Referrer-Policy` — передавати тільки домен (без шляху) при переході на інший сайт
- `Permissions-Policy` — забороняє геолокацію, мікрофон, камеру

```bash
nginx -t && systemctl reload nginx
```

Перевіряємо заголовки:

```bash
curl -I -k https://YOUR_SERVER_IP -H "Host: your-subdomain.your-domain.com"
```

## Етап 7 — Кешування статики

Навіщо? Браузер зберігає статичні файли локально. При повторному візиті не завантажує знову.

```bash
nano /etc/nginx/sites-available/your-site.conf
```

Додати після `location / {`:

```nginx
location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|webp|woff|woff2|ttf|eot)$ {
    expires 30d;                          # Кешувати 30 днів
    add_header Cache-Control "public, immutable";
    access_log off;                       # Не логувати запити до статики
}
```

Пояснення:

- `~*` — регулярний вираз без урахування регістру
- `expires 30d` — Cache-Control: max-age=2592000
- `public` — можна кешувати на CDN і проксі
- `immutable` — не перевіряти зміни до закінчення терміну
- `access_log off` — чистіший лог

```bash
nginx -t && systemctl reload nginx
```

Перевіряємо на реальному файлі:

```bash
touch /var/www/your-site/test.css
curl -I -k https://YOUR_SERVER_IP/test.css -H "Host: your-subdomain.your-domain.com"
rm /var/www/your-site/test.css
```

> Має бути: `Expires: ...` і `Cache-Control: max-age=2592000, public, immutable`

## Етап 8 — Rate limiting

Навіщо? Захищає від brute-force атак, DDoS, надмірного навантаження.

```bash
nano /etc/nginx/nginx.conf
```

Додати після `server_tokens off;`:

```nginx
# General zone: 20 requests/sec per IP, 10MB memory (~160k IPs)
limit_req_zone $binary_remote_addr zone=general:10m rate=20r/s;

# Strict zone: 5 requests/sec per IP (for sensitive endpoints)
limit_req_zone $binary_remote_addr zone=strict:10m rate=5r/s;
```

Пояснення:

- `$binary_remote_addr` — IP в бінарному форматі (економить пам'ять)
- `zone=general:10m` — назва зони, 10MB пам'яті (~160k IP)
- `rate=20r/s` — 20 запитів на секунду

Застосовуємо в конфігу сайту:

```bash
nano /etc/nginx/sites-available/your-site.conf
```

Всередині `location / {`:

```nginx
location / {
    limit_req zone=general burst=50 nodelay;
    limit_req_status 429;
    try_files $uri $uri/ =404;
}
```

Пояснення:

- `burst=50` — дозволяє короткочасний сплеск до 50 запитів
- `nodelay` — не затримувати в черзі, обробляти одразу
- `limit_req_status 429` — повертати 429 Too Many Requests (правильніше ніж 503)

```bash
nginx -t && systemctl reload nginx
```

Тест — 10 швидких запитів (всі мають повернути 200 бо burst=50):

```bash
for i in {1..10}; do curl -s -o /dev/null -w "%{http_code}\n" -k https://YOUR_SERVER_IP -H "Host: your-subdomain.your-domain.com"; done
```

## Етап 9 — Кастомні логи

Навіщо? Стандартний лог не показує реальний IP через Cloudflare. Додаємо корисні поля.

```bash
nano /etc/nginx/nginx.conf
```

Замінити `log_format main` на:

```nginx
# Розширений формат з часом відповіді
log_format main '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent" '
                '"$http_x_forwarded_for" '
                'rt=$request_time uct=$upstream_connect_time '
                'uht=$upstream_header_time urt=$upstream_response_time';

# Формат для сайтів за Cloudflare — показує реальний IP відвідувача
log_format cloudflare '$http_cf_connecting_ip - $remote_user [$time_local] '
                      '"$request" $status $body_bytes_sent '
                      '"$http_referer" "$http_user_agent" '
                      'rt=$request_time cf_ray=$http_cf_ray '
                      'country=$http_cf_ipcountry';
```

Пояснення нових полів:

- `$http_x_forwarded_for` — IP ланцюжок через проксі
- `rt=$request_time` — час обробки запиту
- `$http_cf_connecting_ip` — реальний IP від Cloudflare (замість IP проксі)
- `cf_ray` — унікальний ID запиту Cloudflare (для дебагу)
- `country` — країна відвідувача (Cloudflare визначає автоматично)

Застосовуємо формат у конфігу сайту:

```bash
nano /etc/nginx/sites-available/your-site.conf
```

```nginx
access_log /var/log/nginx/your-site_access.log cloudflare;
```

```bash
nginx -t && systemctl reload nginx
```

Перевіряємо лог:

```bash
curl -s -o /dev/null https://your-subdomain.your-domain.com
tail -5 /var/log/nginx/your-site_access.log
```

## Етап 10 — Автооновлення сертифікатів

Let's Encrypt сертифікати діють 90 днів. Certbot при встановленні створює systemd таймер.

Перевіряємо таймер:

```bash
systemctl status certbot.timer
```

> `active (waiting)` і `Run certbot twice daily` — таймер працює ✅

Тестуємо оновлення (без реального отримання):

```bash
certbot renew --dry-run
```

> `Congratulations, all simulated renewals succeeded!` — все налаштовано ✅

Якщо є прострочені сертифікати що заважають — вимкнути автооновлення для них:

```bash
mv /etc/letsencrypt/renewal/old-cert.conf /etc/letsencrypt/renewal/old-cert.conf.disabled
```

## Етап 11 — sites-available / sites-enabled

Навіщо? Зручна структура для керування сайтами через символічні посилання. nginx.org версія не має цих папок — додаємо вручну.

```bash
mkdir -p /etc/nginx/sites-available /etc/nginx/sites-enabled
```

Підключаємо в `nginx.conf` після `include /etc/nginx/conf.d/*.conf;`:

```nginx
include /etc/nginx/sites-enabled/*;
```

Переносимо конфіг:

```bash
mv /etc/nginx/conf.d/your-site.conf /etc/nginx/sites-available/your-site.conf
```

Створюємо симлінк (активуємо сайт):

```bash
ln -s /etc/nginx/sites-available/your-site.conf /etc/nginx/sites-enabled/your-site.conf
```

Як вимкнути сайт (не видаляючи конфіг):

```bash
rm /etc/nginx/sites-enabled/your-site.conf
```

Як увімкнути знову:

```bash
ln -s /etc/nginx/sites-available/your-site.conf /etc/nginx/sites-enabled/your-site.conf
```

```bash
nginx -t && systemctl reload nginx
```

## Етап 12 — Користувачі, групи, SSH

### Діагностика

Користувачі з реальним shell:

```bash
cat /etc/passwd | grep -v nologin | grep -v false
```

Хто має sudo:

```bash
grep sudo /etc/group
```

Параметри SSH:

```bash
grep -E "PermitRootLogin|PasswordAuthentication|PubkeyAuthentication|Port" /etc/ssh/sshd_config
```

SSH ключі користувачів:

```bash
ls -la /home/YOUR_USER/.ssh/
```

### Shell для користувача

Змінити shell з `/bin/sh` на `/bin/bash` (автодоповнення, історія, кольоровий промпт):

```bash
chsh -s /bin/bash YOUR_USER
```

Перевірка:

```bash
grep YOUR_USER /etc/passwd
```

### Групи і права на файли сайту

Переглянути всі групи:

```bash
cat /etc/group
```

Переглянути групи з користувачами:

```bash
cat /etc/group | grep -v "^.*:x:[0-9]*:$" | sort
```

Додати користувача і nginx в групу `www-data`:

```bash
usermod -aG www-data YOUR_USER
usermod -aG www-data nginx
```

Встановити правильні права на папку сайту:

```bash
chown -R www-data:www-data /var/www/your-site
chmod -R 750 /var/www/your-site
```

Пояснення `chmod 750`:

```
7 (rwx) — власник: читає, пише, заходить
5 (r-x) — група www-data: читає, заходить
0 (---) — інші: нічого
```

> Після зміни групи nginx потрібен перезапуск:

```bash
systemctl restart nginx
```

Перевіряємо права:

```bash
ls -la /var/www/
```

### Налаштування SSH

```bash
nano /etc/ssh/sshd_config
```

Змінити:

```
PermitRootLogin no          # Забороняє root по SSH
PubkeyAuthentication yes    # Тільки ключі
```

Додати в кінець:

```
AllowUsers user1 user2 user3   # Тільки ці користувачі можуть зайти по SSH
```

Перевіряємо синтаксис (як nginx -t але для SSH):

```bash
sshd -t
```

Перезапускаємо SSH:

```bash
systemctl restart ssh
```

> ⚠️ Не закривай поточне з'єднання поки не перевіриш що можеш зайти в новому вікні!

Тестуємо підключення в новому вікні:

```bash
ssh -i /home/YOUR_USER/.ssh/YOUR_KEY -p YOUR_SSH_PORT YOUR_USER@YOUR_SERVER_IP
```

Перевіряємо що root заблокований:

```bash
ssh -p YOUR_SSH_PORT root@YOUR_SERVER_IP
```

> Має повернути: `Permission denied (publickey)`

### Cloudflare vs nginx — що робить кожен

| Налаштування      | Cloudflare           | nginx            | Навіщо nginx якщо є CF              |
| ----------------- | -------------------- | ---------------- | ----------------------------------- |
| Gzip              | ✅                   | ✅               | CF не завжди передає стиснення далі |
| Заголовки безпеки | ✅ платно            | ✅               | CF може бути вимкнений              |
| Rate limiting     | ✅ платно            | ✅               | nginx безкоштовний                  |
| Кешування         | ✅ між CF і сервером | ✅ каже браузеру | Різні рівні кешування               |
| server_tokens     | ❌                   | ✅               | Тільки nginx                        |
| SSL сертифікат    | ✅ свій              | ✅ Let's Encrypt | Потрібен для Full (strict)          |
| Логи              | ✅ платно            | ✅               | Свій контроль                       |

> Принцип: не покладайся на один рівень захисту. Налаштував і там і там — сервер захищений незалежно від Cloudflare.
