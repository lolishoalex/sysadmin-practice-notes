# Task 5 — Діагностика нового сервера під навантаження, зроблені зміни, аварійний рубильник, ТЗ нового сервера

Дата робіт: 10–15 липня
Сервер: новий, vps-new-01 (203.0.113.10), Debian 10, новинний сайт example.com за Cloudflare (план Free, SSL Full)
Контекст: діагностика проводилась ПІСЛЯ закриття дискового інциденту (Task 4.1): хостинг-провайдер визнав проблему сховища (тікет від 18.06), мігрував VPS на здоровий хост (~09.07), fsck пройдено, dmesg чистий.

---

## 1. Результати діагностики (9 блоків)

### 1.1 Залізо і загальний стан

- CPU: 16 ядер (Haswell virtual), RAM: 61 GB (використано ~7 GB), диск: 350 GB, scheduler `none` (оптимально для virtio)
- Load average ~1.0–1.2 на 16 ядрах = завантаження ~7%. Сервер сильно недовантажений
- vmstat: `wa=0` (нуль очікування диска), свопу немає (swappiness 60 — неактуально без swap)
- `vm.overcommit_memory=1` — поставлено свідомо для Redis, не чіпати
- Мережа: 0 errors / 0 dropped на eth0, conntrack 521/262144 — величезний запас

### 1.2 Ядро (sysctl)

Майже стокове, тюнили тільки: `net.core.somaxconn=1024`, `vm.overcommit_memory=1`.
BBR недоступний (модуль не завантажений, у ядрі 4.19 є) — активний cubic.
dirty_ratio 20 / dirty_background_ratio 10 (дефолти; з 61 GB RAM це до ~12 GB відкладеного запису — потенційні I/O-лавини при бекапах).
Висновок: нічого не горить, тюнінг ядра → у ТЗ нового сервера.

### 1.3 Nginx 1.14.2

- worker_processes 16, worker_connections 16384, multi_accept, epoll, worker_rlimit_nofile 64000 — налаштовано добре
- Воркери реально мають ліміт 64000/64000 (перевірено на живих процесах). Master має soft 1024 — косметика, не впливає
- sendfile/tcp_nopush/tcp_nodelay on, gzip on, open_file_cache налаштований
- **fastcgi_cache ПРАЦЮЄ**: зона site_cache 100m/512m у /www/example.com/cache/, TTL 200-х = 1 хвилина, background_update on, cache_lock on
- Умови $skip_cache: не-GET, wp-admin/xmlrpc/feed/sitemap/index.php, кукі залогінених — коректні
- ВАЖЛИВО ПРО ДІАГНОСТИКУ: `curl -I` шле HEAD → умова `$request_method != GET` → завжди BYPASS. Кеш перевіряти ТІЛЬКИ справжнім GET: `curl -s -o /dev/null -D - URL`
- `/etc/hosts` на сервері: example.com → 127.0.1.1, тому curl З СЕРВЕРА йде повз Cloudflare. Перевірки "як бачить світ" — тільки з зовнішньої машини
- access_log вимкнений (свідомо) — статистика запитів тільки в Cloudflare Analytics
- Конфіг захаращений (кілька server-блоків, дублі gzip, мертві listen) — розібрати на новому сервері, не тут
- W3TC-правила в /www/example.com/html/nginx.conf: try_files веде 404 статики у PHP — зайве навантаження, у ТЗ

### 1.4 PHP-FPM 7.4

- Активний пул wp.conf (unix-сокет php7.4-fpm-wp.sock): pm=dynamic, max_children=52, backlog=4096
- Реальне споживання ~130 MB/процес → стеля 52×130 ≈ 6,8 GB (запас величезний)
- Статус пулу: черга 0, max children reached: 0, пік 47/52 активних
- OPcache on, дефолти (128 MB, 10000 файлів) — на межі для 40+ плагінів, у ТЗ
- max_execution_time = 1200 (20 хв!) — кандидат на зменшення на новому сервері
- Пул www.conf (max_children=5) — дефолтний, практично не використовується

### 1.5 MariaDB 10.3

- innodb_buffer_pool_size = 3G при базі site_prod 3,7G (+ site_test 1,5G)
- Buffer pool hit rate = 99,997% (208K дискових читань на 6,3 млрд запитів) — БД працює з пам'яті
- max_connections 300 (пік 47), thread_cache 256 — запас великий
- innodb_flush_log_at_trx_commit = 2 — свідомий компроміс, ок для новинника
- ПРОБЛЕМИ (не критичні, ніхто не відчуває): query_cache УВІМКНЕНИЙ (16M; під високою конкурентністю — глобальний лок, шкодить); tmp_table_size/max_heap_table_size = 16M → 33% тимчасових таблиць на диску (170849 з 517200); slow_query_log OFF (сліпота щодо повільних запитів; вмикається на льоту без рестарту: SET GLOBAL slow_query_log=ON; SET GLOBAL long_query_time=2;)

### 1.6 WordPress

- РЕАЛЬНИЙ ШЛЯХ САЙТУ: /www/example.com/html/ (НЕ /var/www/html — та папка порожня). Префікс таблиць: нестандартний, xx_
- Redis object cache: ПРАЦЮЄ ЗРАЗКОВО — Connected, hit rate 94% (51,1M hits / 3,3M misses), 2,17G з ліміту 4G, allkeys-lru
- W3TC: pgcache=on (engine redis), dbcache=on (engine redis), objectcache=off, minify=off → сторінки і SQL кешуються ДВІЧІ-ТРИЧІ (nginx+W3TC pgcache; Redis object + W3TC dbcache + MariaDB query_cache). Не шкодить диску (все в Redis), але надлишково — розплутати на новому сервері
- WP_DEBUG був true (роками, з часів запуску!) — ВИПРАВЛЕНО 15.07 (див. розділ 2)
- WP-Cron від відвідувачів (DISABLE_WP_CRON немає), action_scheduler щохвилини
- debug.log зовні закритий (403)

### 1.7 Cloudflare (стан на 15.07)

- План Free, SSL Full, Always Use HTTPS, Always Online ON, Browser Cache TTL 30 min
- Статика кешується (MISS→HIT підтверджено), HTML — DYNAMIC (не кешується)
- Analytics за добу: 261K запитів, кешовано лише 47K (18%) — 214K долітали до VPS
- Page Rules (3/3 використано): tools.example.com (вимкнене), wp-admin bypass, *preview=true* bypass
- Пов'язана задача (окремий чат): rate limiter 30 req/10s на весь сайт включно зі статикою — правка ще НЕ зроблена. Правило: одна зміна Cloudflare за раз, доба між змінами

---

## 2. Зроблені зміни (15.07)

### 2.1 WP_DEBUG вимкнено — ОБИДВА сервери

Причина: роки дебаг-режиму на проді; постійний запис у debug.log на кожному запиті (8,2 MB за 2 тижні на новому); зайвий I/O; ризик витоку.
Зроблено на новому (203.0.113.10) і старому (198.51.100.20):
- бекап: wp-config.php.bak-ДАТА (лежить поруч з wp-config.php)
- define( 'WP_DEBUG', true ) → false (WP_DEBUG_LOG/DISPLAY не чіпали — неактивні при false)
- debug.log обнулено (truncate -s 0)
Перевірено: сайт 200, лог не росте.
Відкат: `sudo cp -a /www/example.com/html/wp-config.php.bak-ДАТА /www/example.com/html/wp-config.php`

### 2.2 /etc/hosts на старому сервері

sudo бурчав `unable to resolve host srv-new` — hostname був відсутній у hosts (закоментований разом з example.com, що для домену ПРАВИЛЬНО: старий сервер має резолвити example.com через DNS на новий — потрібно для rsync-синхронізацій і certbot).
Додано рядок: `127.0.1.1 srv-new` (тільки ім'я хоста, домен не чіпали). sudo -v — тихо.

### 2.3 Cloudflare Cache Rule — створено, протестовано, ВИМКНЕНО (у резерві)

Правило "Cache HTML for anonymous" (перейменувати на EMERGENCY ONLY — див. розділ 3): кешування HTML для анонімів, Edge TTL 10 min (Ignore cache-control), Browser TTL Respect origin.
Тестування пройдено повністю: cf-cache-status HIT на головній/рубриках (сторінки віддавав Cloudflare без походу на сервер), preview → DYNAMIC, /.well-known/acme-challenge/ → DYNAMIC (шлях certbot у безпеці), залогінені бачать сайт нормально (адмін-бар, wp-admin).
ЧОМУ ВИМКНЕНО: кеш за TTL суперечить бізнес-вимозі новинника — нова стаття має з'являтись на головній/рубриках за секунди, а TTL давав затримку до ~11 хв (10 CF + 1 nginx). Ручний purge для журналістів — не варіант. Правильне рішення (кеш за подіями публікації) — розділ 4.1.
Статус: Disabled, вираз збережено. Це аварійний рубильник.

---

## 3. АВАРІЙНИЙ РУБИЛЬНИК — інструкція

### Коли вмикати

- L7-флуд: сайт гальмує/лежить, у Cloudflare Analytics різкий стрибок Uncached-запитів, на сервері росте load/активні php-fpm
- Вірусний наплив легітимного трафіку (гучна подія, посилання великих медіа)
- Будь-яка ситуація "сервер не тягне HTML-трафік"
Cloudflare і так закриває L3/L4 DDoS і тупих ботів. Рубильник закриває те, що CF пропускає: легітимні на вигляд запити до динамічного HTML (розумний L7-флуд, наплив слави).

### Як увімкнути (1 клік)

dash.cloudflare.com → example.com → Caching → Cache Rules → правило → три крапки → Enable.
Ефект за ~1 хвилину (прогрів кешу).

### Ціна ввімкнення (прийнятна в аварії)

Нові статті з'являтимуться на головній/рубриках із затримкою до 10 хв; правки статей — до 10 хв. Попередити редакцію. Терміновий показ конкретної URL: Caching → Purge Cache → Custom purge.

### Перевірка після ввімкнення (з домашнього ПК, НЕ з сервера)

```bash
curl -s -o /dev/null -D - https://example.com/uk/ | grep -iE "cf-cache-status|x-fastcgi-cache"
```
Двічі: очікуємо MISS → HIT. ВАЖЛИВО: тільки так, НЕ `curl -I` (HEAD дасть хибний BYPASS).

```bash
curl -s -o /dev/null -D - https://example.com/.well-known/acme-challenge/test | grep -iE "HTTP|cf-cache-status"
```
Очікуємо 404 + DYNAMIC/BYPASS (не HIT).
Браузером залогіненою: адмін-бар на місці, wp-admin працює.

### Як вимкнути після аварії

Те саме меню → Disable. За бажанням Purge Everything (необов'язково — протухне саме за ≤10 хв).
Контроль: cf-cache-status повернувся в DYNAMIC, x-fastcgi-cache: HIT.

### Читання cf-cache-status

HIT — віддано з кешу CF; MISS/EXPIRED — кешується, сходили на origin; DYNAMIC — не підпадає під кешування; BYPASS — підпадало, але обійшли. Тривога тільки одна: HIT на wp-admin/preview/acme-challenge.

---

## 4. Відкладені рішення (свідомо НЕ робимо на цьому сервері)

### 4.1 Purge-інтеграція W3TC ↔ Cloudflare (єдиний пункт з самостійною цінністю — переживе міграцію)

Мета: кеш HTML за ПОДІЯМИ замість годинника. W3TC → Performance → Extensions → Cloudflare + API Token (My Profile → API Tokens, ЄДИНЕ право Zone → Cache Purge на зону сайту; токен = секрет, нікому не показувати). Після Publish/Update W3TC сам чистить у CF статтю+головну+рубрики.
Послідовність: (1) налаштувати інтеграцію → (2) протестувати на публікаціях → (3) ТІЛЬКИ ТОДІ Enable Cache Rule на постійно. Без кроку 1 правило не вмикати (граблі 15.07). Живе у WordPress+Cloudflare → автоматично переїде на новий сервер.
Рішення 15.07: не робимо, повернутись у разі скарг/потреби.

### 4.2 MariaDB

tmp_table_size/max_heap_table_size 16M→64–128M, query_cache OFF, buffer_pool 3G→6–8G, innodb_io_capacity — усе в ТЗ нового сервера. slow_query_log за потреби вмикається на льоту без рестарту.

### 4.3 WP-Cron → системний cron

`define('DISABLE_WP_CRON', true);` + crontab `*/1 * * * * ... wp cron event run --due-now`. У ТЗ нового сервера.

### 4.4 Знято з порядку денного повністю

TTL nginx fastcgi_cache (1 хв — ПРАВИЛЬНО для новинника, з'ясовано 15.07); noatime (relatime достатньо, fstab на цьому диску не чіпати); scheduler (вже none); somaxconn/netdev_backlog/ліміти FPM (запаси великі); hdparm/iostat-заміри (wa=0 і hit rate 99,997% зробили їх непотрібними).

---

## 5. ТЗ нового сервера (закласти з першого дня)

### Архітектура кешів — ОДИН шар на функцію

- Page cache: ТІЛЬКИ nginx fastcgi_cache (TTL 1 хв для новинника або довший + purge-інтеграція)
- Object/data cache: ТІЛЬКИ Redis (ліміт 4G, allkeys-lru — перенести як є, конфіг себе виправдав)
- W3TC: pgcache OFF, dbcache OFF, objectcache OFF (лишити тільки browser cache rules або замінити легшим плагіном)
- MariaDB query_cache: OFF (query_cache_type=0, query_cache_size=0)
- Cloudflare: Cache Rule для HTML + purge-інтеграція з першого дня

### MariaDB (сучасна версія)

innodb_buffer_pool_size ≥ 1,5× розміру бази (зараз 6–8G); tmp_table_size = max_heap_table_size = 64–128M; slow_query_log ON, long_query_time 1–2s; innodb_flush_log_at_trx_commit=2 (свідомо); не тягнути тестову базу без потреби

### PHP-FPM (сучасна версія PHP — після перевірки сумісності плагінів)

pm.max_children з розрахунку RAM/130MB із запасом під БД; max_execution_time 300 замість 1200; OPcache: memory 256M, max_accelerated_files 20000, opcache.validate_timestamps=1; окремий пул для сайту (схема wp.conf себе виправдала)

### Nginx (сучасна версія)

Перенести: worker auto, connections 16384, rlimit_nofile, epoll, multi_accept, open_file_cache, fastcgi_cache зі skip-умовами (вони правильні). Виправити: чистий конфіг без мертвих server-блоків; try_files статики БЕЗ фолбека в PHP (404 на картинці не повинен запускати WordPress); access_log — вирішити свідомо (зараз off)

### Ядро/система

BBR (modprobe tcp_bbr + sysctl); vm.dirty_background_bytes/dirty_bytes в абсолютних значеннях (напр. 256M/1G) замість відсотків; невеликий swap-файл (2–4G) як подушка від OOM або свідома відмова; fs.protected_* лишити; somaxconn 1024–4096

### WordPress

WP_DEBUG=false з першого дня; DISABLE_WP_CRON + системний cron; Redis drop-in; ревізія 40+ плагінів (кандидати на видалення: неактивні плагіни, дублі функціоналу); xx_options autoload перевірити після переїзду

### Незмінні інваріанти (з Task 2 — діють ЗАВЖДИ)

SSL mode = Full (НЕ Flexible); Always Use HTTPS on; ЖОДНИХ правил CF/nginx, що чіпають /.well-known/acme-challenge/; DNS другого домену організації → актуальний origin; вебрут certbot = вебрут nginx

---

## 6. Фонові задачі-нагадування

- Тижневий чек-лист після міграції хоста (Task 4.2) — до ~16.07, далі за бажанням
- ~7 серпня: `sudo certbot certificates` — переконатись, що сертифікат оновився (post-hook надішле лист про reload nginx)
- Задача rate limiter — у своєму чаті, не раніше ніж через добу після будь-якої іншої зміни CF
- Разово перейменувати Cache Rule на EMERGENCY ONLY-назву (розділ 3)

## 7. Ключові "граблі", задокументовані для майбутнього

1. `curl -I` (HEAD) завжди дає BYPASS на fastcgi_cache — тестувати кеш тільки GET-ом
2. curl з сервера йде повз Cloudflare (/etc/hosts) — зовнішні перевірки тільки з зовнішньої машини
3. Кеш HTML за TTL несумісний з новинною головною — тільки purge за подіями
4. Cache Rules: винятки через "and not" дають DYNAMIC (не BYPASS) — це нормально
5. Disable ≠ Delete: вимкнене правило CF зберігає вираз — завжди Disable
6. Rate limiting у CF рахує запити ДО кешу — кешування не лікує блокування користувачів лімітером
7. wp-cli з порожньої директорії показує оточення, а не сайт — реальний шлях /www/example.com/html/
8. sudo перед `cmd1 && cmd2` діє тільки на cmd1
