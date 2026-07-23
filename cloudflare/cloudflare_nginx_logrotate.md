# Послаблення захисту Cloudflare, nginx-фікс, logrotate

Дата робіт: 9–23 липня 2026  
Новий сервер: `vps-new` (203.0.113.10)  
Старий сервер: `vps-old` (198.51.100.20)  
Cloudflare: план Free, SSL Full, домен `example.org`

---

## Контекст

Початкова задача: послабити захист Cloudflare, який заважає реальним читачам. У процесі аудиту знайдено і закрито чотири додаткові проблеми.

---

## Знайдені проблеми та рішення

### Проблема 1 — Rate limiter блокував читачів 🔴 → ✅

**Що було:** правило «30 запитів / 10 секунд з одного IP» на весь сайт, включно зі статикою. Одна сторінка з фото = 40–80 паралельних запитів → ліміт вичерпувався → сторінка не довантажувалась. Найбільше страждали нові відвідувачі і мобільні оператори (CGNAT).

**Доказова база:**
- Events: пачки блоків в одну секунду з країн основної аудиторії (домашні провайдери)
- Path у деталях: `/wp-content/uploads/...` (статика, не атаки)
- Власний досвід: «ламається ~1 раз з 10, зазвичай головна»
- Ретест: `/images/` і `/js/` — папок не існує (`ls` показав порожньо), файли 404 → ті блоки були сканерами, не читачами

**Рішення: правка Rate limiter у Cloudflare**

`Security → Security rules → Rate limiter → Edit`

Expression (було):
```
(starts_with(http.request.uri.path, "/"))
```

Expression (стало):
```
(starts_with(http.request.uri.path, "/")) and not
(starts_with(http.request.uri.path, "/wp-content/")) and not
(starts_with(http.request.uri.path, "/wp-includes/"))
```

Requests: 30 → **60** (страховка від CGNAT).  
Решта (Characteristics: IP, Action: Block, Duration: 10s, Place: First) — без змін.

**Перевірка після:** Events → `Service = Rate limiting rules` + `Country = <основна аудиторія>` → нуль по людях; тести в інкогніто і з телефона — чисто.

---

### Проблема 2 — Custom rule «Temp SQL» ловив своїх 🔴 → ✅

**Що було:** правило DDoS-2022 (підписи атаки) містило дві «пастки» на звичайну навігацію:
```
(http.request.uri eq "/" and http.referer eq "https://example.org/")
(http.request.uri eq "/uk/" and http.referer eq "https://example.org/uk/")
```
Тобто клік по логотипу → Managed Challenge.

**Рішення:** видалити тільки ці дві умови, решту 16 (сигнатури атакуючої кампанії, службовий маркер, старі UA) залишити.

`Security → Security rules → DDoS-2022 signatures → Edit expression`

Поточний повний вираз (16 умов, без пасток):
```
(http.user_agent eq "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36")........
```

Також правило **перейменовано**: `Temp SQL` → `DDoS-2022 signatures (attack campaign, old UA)`.

Action: Managed Challenge, Place: Custom after «User agent hackers» — не змінювались.

---

### Проблема 3 — `uri contains "//"` ловив Mastodon/oEmbed 🟡 → ✅

**Що було:** умова `(http.request.uri contains "//")` матчила запити з URL у query string (`?url=https://...`), зокрема oEmbed-запити від сервера Mastodon. Прев'ю статей у федіверсі не будувались.

**Рішення:** замінити одне поле в тому ж правилі:

- було: `(http.request.uri contains "//")`
- стало: `(http.request.uri.path contains "//")`

`uri.path` — тільки шлях без query string. Захист від сміттєвих шляхів зберігається, легітимні URL у параметрах більше не ловляться.

**Перевірка:**
```bash
curl -sI "https://example.org/en/wp-json/oembed/1.0/embed?url=https://example.org/en/1608" | head -3
# Очікування: HTTP/2 200
```

---

### Проблема 4 — Сайтмапи мертві (всі 404) 🔴 → ✅

**Що було:** всі карти сайту (`/uk/sitemap.xml` і т.д. — 6 мов, задекларовані в robots.txt) повертали 404. Причина: у файлі W3TC `/www/example.org/html/nginx.conf` блок для `.xml` шле запити в `try_files → /uk/index.php` (або `/it/index.php`), якого не існує (`/uk/` — віртуальний префікс WPML). WordPress і плагін карт при цьому справні — запит до них не долітав.

Наслідки: 1 153 670 помилок «Primary script unknown» у error.log (364 MB).

**Важливо:** W3TC **перезаписує** `/www/example.org/html/nginx.conf` при збереженні налаштувань Browser Cache — тому проста правка файла не є постійним рішенням. Потрібен страховий location у головному конфігу.

**Рішення: два кроки**

**Крок А — прибрати `xml` з W3TC-блоку** (мінімальна правка):
```bash
sudo sed -i 's/location ~ \\\.(html|htm|rtf|rtx|txt|xsd|xsl|xml)\$/location ~ \\.(html|htm|rtf|rtx|txt|xsd|xsl)$/' /www/example.org/html/nginx.conf
# Перевірка:
grep -n "html|htm" /www/example.org/html/nginx.conf
# Очікування: рядок БЕЗ |xml
```

**Крок Б — страховий location у головному конфігу** (імунітет до перезаписів W3TC):

У `/etc/nginx/sites-available/example.org-full.conf` **перед** рядком `include /www/example.org/html/nginx.conf;` вставити:

```nginx
    ## Sitemaps: route to WordPress, immune to W3TC rewrites
    location ~* "^/([a-z][a-z]/)?sitemap[^/]*\.xml(\.gz)?$" {
        try_files $uri /index.php?q=$uri&$args;
    }
```

Застосування:
```bash
sudo nginx -t && sudo systemctl reload nginx
```

**Перевірка:**
```bash
curl -sI https://example.org/uk/sitemap.xml | head -2   # → 200
curl -sI https://example.org/en/sitemap.xml | head -2   # → 200
curl -s https://example.org/uk/sitemap.xml | head -5    # → валідний XML
```

---

### Проблема 5 — `/nginx.conf` публічно читається 🔴 → ✅

**Що було:** `https://example.org/nginx.conf` → HTTP 200. Файл W3TC у webroot доступний назовні.

**Рішення:** додати deny-блок у головний конфіг **перед** `include`:

```nginx
    ## Deny access to config/backup/dump files in webroot
    location ~* \.(conf|bak|sql|old|orig|save|swp)$ {
        deny all;
        access_log off;
        log_not_found off;
    }
```

**Перевірка:**
```bash
curl -sI https://example.org/nginx.conf | head -2   # → 403
```

---

### Проблема 6 — Логи сайту ростуть без ротації 🟡 → ✅

**Що було:** `/www/example.org/logs/` — нестандартне місце логів nginx, невідоме системному logrotate. access.log: 4.7 GB, error.log: 426 MB, ростуть нескінченно.

**Рішення:** `/etc/logrotate.d/example`:

```bash
sudo tee /etc/logrotate.d/example << 'EOF'
/www/example.org/logs/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0644 www-data root
    sharedscripts
    postrotate
        invoke-rc.d nginx rotate >/dev/null 2>&1
    endscript
}
EOF
```

Перша ротація (примусова):
```bash
sudo logrotate --force /etc/logrotate.conf
```

**Результат:** access.log: 0 → росте; access.log.1: 4.7 GB (стискається завтра вночі автоматично). Далі — щодня автоматично через `/etc/cron.daily/logrotate`.

**Логіка `delaycompress`:** перша ротація миттєва (тільки перейменування), стискання відбувається наступного дня — не блокує роботу під час великого першого запуску.

---

## Що з'ясувалось попутно

### Стан захисту, який НЕ чіпали

- **Bot Fight Mode** — ловить ботів з датацентрів (Сінгапур, США, Німеччина), читачів не чіпає → залишили
- **Managed rules** — аналогічно → залишили
- **Custom rule «Bypass»** — Skip для Heartbeat редактора (POST `/wp-admin/admin-ajax.php` кожні 2 хв від хостинг-провайдера) → норма, залишили
- **Challenge Passage = 30 хвилин** → не змінювали (ситуація вирішена на рівні правил)
- **SSL Full, Always Use HTTPS, acme-challenge** — інваріанти з попередніх задач → не чіпали

### Знахідки про інфраструктуру

- **W3TC перезаписує nginx.conf** при збереженні Browser Cache — фолбек змінився `/uk/` → `/it/` між 10 і 22 липня без нашої участі. Саме тому страховий location у головному конфігу (який W3TC не чіпає) є обов'язковим рішенням.
- **Сайтмапна проблема народилась на новому сервері** після клонування (W3TC додав xml після якогось збереження налаштувань). На старому сервері xml у W3TC-блоці не було.
- **error.log 364 MB** — майже повністю від «Primary script unknown» по сайтмапах. Після фіксу потік зупинився. Залишкові ~10k рядків/день — 404 картинок старих статей (2014), свідомо залишені як «детектор битих зображень».
- **access.log format: cf_custom** — пише реальний IP з CF-Connecting-IP. Поле шляху — **\$9** (не \$7 і не \$8 через дату з пробілом). Команда для аналізу:
```bash
sudo tail -20000 /www/example.org/logs/access.log | awk '{print $1, $9}' | grep -E ' /(uk/)?$' | sort | uniq -c | sort -rn | head
```
- **Логи доступу вимкнені у `/var/log/nginx/access.log`** (0 байт з 2021) — свідомо, але реальний лог є у `/www/example.org/logs/access.log`.
- **Zabbix-agent** встановлений на обох серверах (є в logrotate.d).
- **Биті зображення старих статей (2014)** — файли в `/wp-content/uploads/` без річних підпапок, загублені при якомусь із давніх переїздів. Масштаб не рахували, задача відкладена.

---

## Що зроблено на кожному сервері

| Зміна | Новий (203.0.113.10) | Старий (198.51.100.20) |
|---|---|---|
| Rate limiter | ✅ | не потрібно (Cloudflare — один) |
| DDoS-2022 / uri.path / перейменування | ✅ | не потрібно (Cloudflare — один) |
| Прибрати xml з W3TC-блоку | ✅ | не потрібно (там xml не було) |
| Страховий location для сайтмапів | ✅ | ✅ |
| deny-блок для .conf/.bak/.sql | ✅ | ✅ |
| logrotate конфіг | ✅ | ✅ |
| Бекапи конфігів у /root | ✅ (три файли) | ✅ (один файл) |

---

## Бекапи конфігів у /root

**Новий сервер:**
```
/root/nginx.conf.w3tc.bak-2026-07-22        # W3TC-файл «до»
/root/example-full.conf.bak-2026-07-22      # головний конфіг «до»
/root/example-full.conf.bak-2026-07-22-v2   # головний конфіг «після» (фінальний)
```

**Старий сервер:**
```
/root/example-full.conf.bak-2026-07-23      # головний конфіг «до» правок
```

**Відкат (якщо щось):**
```bash
# Новий сервер
sudo cp /root/example-full.conf.bak-2026-07-22-v2 /etc/nginx/sites-available/example.org-full.conf
sudo cp /root/nginx.conf.w3tc.bak-2026-07-22 /www/example.org/html/nginx.conf
sudo nginx -t && sudo systemctl reload nginx

# Старий сервер
sudo cp /root/example-full.conf.bak-2026-07-23 /etc/nginx/sites-available/example.org-full.conf
sudo /usr/sbin/nginx -t && sudo systemctl reload nginx
```

---

## Перевірочні команди після змін

З домашнього комп'ютера (новий сервер):
```bash
curl -sI https://example.org/uk/sitemap.xml | head -2   # → 200
curl -sI https://example.org/en/sitemap.xml | head -2   # → 200
curl -sI https://example.org/nginx.conf | head -2        # → 403
curl -sI https://example.org/ | head -2                  # → 301
```

З домашнього комп'ютера (старий сервер напряму):
```bash
curl -skI --resolve example.org:443:198.51.100.20 "https://example.org/uk/sitemap.xml" | head -2  # → 200
curl -skI --resolve example.org:443:198.51.100.20 "https://example.org/nginx.conf" | head -2       # → 403
curl -skI --resolve example.org:443:198.51.100.20 "https://example.org/" | head -2                 # → 301
```

На сервері:
```bash
sudo wc -l /www/example.org/logs/error.log   # темп після фіксу: ~10k/день замість ~19k
sudo ls -lh /www/example.org/logs/            # після ротації: .log маленький, .1 великий
grep -n "html|htm" /www/example.org/html/nginx.conf  # без |xml
```

Cloudflare Events:
```
Service = Rate limiting rules + Country = <основна аудиторія> → нуль по людях
Service = Custom rules + Country = <основна аудиторія> + Action = Managed Challenge → нуль
```

---

## Google Search Console

Сайтмапи підтверджені після фіксу:
- `https://example.org/fr/sitemap.xml` → **Успішно** (23 лип. 2026)
- Решта 5 мов — перейдуть в «Успішно» самостійно за 1–2 тижні

Всі шість карт вже присутні в GSC (подані в різний час протягом кількох років). Нових подань не потрібно.
