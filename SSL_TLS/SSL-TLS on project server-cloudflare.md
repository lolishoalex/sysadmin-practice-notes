# SSL/TLS на сервері та Cloudflare

---

## Теорія

### Що таке SSL/TLS

**SSL/TLS** — протокол шифрування, який перетворює звичайний HTTP на HTTPS.  
Це «захищений конверт» для будь-якого трафіку між двома точками в інтернеті (наприклад, між браузером і сайтом).

- **SSL** (Secure Sockets Layer) — стара назва, протокол 1990-х. Усі версії SSL вже небезпечні й офіційно застаріли.
- **TLS** (Transport Layer Security) — сучасний наступник. Актуальні версії — **TLS 1.2** і **TLS 1.3**.

---

### Що таке сертифікат

**Сертифікат** — файл із даними, який сервер показує браузеру всередині протоколу TLS. Це структурований документ:

```
Кому видано:  example.org
Ким видано:   Let's Encrypt
Діє:          з 9 травня по 7 серпня
Публічний ключ: 30:82:01:0a:02:82:01:01:00:c4:...
Підпис CA:    af:3e:11:9c:...
```

---

### Як відбувається HTTPS-з'єднання

1. Браузер відкриває з'єднання з сервером
2. Сервер показує свій сертифікат («паспорт сайту», публічна частина)
3. Браузер перевіряє: чи підписаний сертифікат довіреним центром сертифікації (CA), чи не прострочений, чи відповідає домену
4. Якщо все гаразд — сторони домовляються про ключі шифрування, весь подальший трафік іде зашифрованим

---

### CA — Certificate Authority

**CA** — це «нотаріус інтернету». Організація, якій довіряють браузери та операційні системи.  
Коли CA підписує сертифікат, він підтверджує: «домен справді належить цим людям».  
Без CA вся система TLS не працювала б — неможливо було б відрізнити справжній сертифікат від підробленого.

#### Як CA перевіряє, що ви — це ви

**DV (Domain Validation)** — найпростіший рівень. Перевіряється лише факт володіння доменом.

Способи підтвердження:

- **HTTP-01**: покладіть файл із конкретним вмістом за адресою `https://example.org/.well-known/acme-challenge/<рядок>`
- **DNS-01**: додайте TXT-запис у DNS із конкретним значенням
- **TLS-ALPN-01**: налаштуйте спеціальну відповідь на 443 порту

Якщо це зроблено — ви реально контролюєте домен.  
Так працюють **Let's Encrypt**, **Google Trust Services**, **ZeroSSL**. Жодних паспортів, жодних компаній. Сертифікат можна отримати за хвилину автоматично.

---

### Let's Encrypt

**Let's Encrypt** — безкоштовний центр сертифікації (CA), який видає TLS-сертифікати автоматично, без оплати, без реєстрації, без паперів.  
Запущений у 2015 році. Станом на 2026 рік обслуговує більше половини всіх сайтів в інтернеті.

---

### Ланцюжок сертифікатів: Root → Intermediate → Leaf

**Навіщо три рівні?**  
Один кореневий сертифікат на все було б небезпечно: якщо він скомпрометований — катастрофа на роки. Тому CA будує трирівневу ієрархію.

| Рівень                | Опис                                                                                                                                                                        |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Root**              | Самопідписаний (вершина довіри). Вбудований у браузери та ОС. Зберігається на спеціальному залізі, відключеному від мережі. Використовується лише для підпису Intermediate. |
| **Intermediate**      | Підписаний Root. Робоча ланка — ним підписують кінцеві сертифікати щодня. Якщо скомпрометований — відкликається, Root у безпеці. Не вбудований у браузери.                  |
| **Leaf** (end-entity) | Підписаний Intermediate. Для конкретного домену. Короткий термін дії (90 днів).                                                                                             |

---

### Строк дії сертифіката

У кожного сертифіката є дві дати:

- `notBefore` — з якого моменту діє
- `notAfter` — до якого моменту діє

Після `notAfter` сертифікат прострочений — браузер йому більше не довіряє.

**Що побачить користувач при прострочений сертифікаті:**

```
Ваше з'єднання не захищене
NET::ERR_CERT_DATE_INVALID
```

---

## Практика

### Два SSL-з'єднання при роботі через Cloudflare

Коли сайт стоїть за Cloudflare, є **два окремих SSL-з'єднання**:

```
[Браузер] ──HTTPS (Cloudflare cert)──> [Cloudflare] ──HTTPS (Let's Encrypt cert)──> [VPS, nginx]
```

- **Браузер ↔ Cloudflare** — тут працює сертифікат Cloudflare (**Edge Certificate**), який бачать відвідувачі.
- **Cloudflare ↔ сервер (origin)** — тут працює сертифікат, встановлений на VPS (**Origin Certificate**, Let's Encrypt тощо).

Коли ви відкриваєте `https://example.org` у браузері — ви бачите сертифікат Cloudflare, а не той, що на сервері. Сертифікат на сервері «ховається» за Cloudflare і використовується лише для шифрування трафіку між Cloudflare та origin.

**«Головніший»** у цій схемі — Edge Certificate (його бачать користувачі). Origin-сертифікат важливий для безпеки, але публічно невидимий.

---

### Шаг 1. Що бачить публіка — сертифікат Cloudflare

```bash
echo | openssl s_client -servername example.org -connect example.org:443 2>/dev/null | openssl x509 -noout -issuer -subject -dates -ext subjectAltName
```

Приклад виводу:

```
issuer=C = US, O = Google Trust Services, CN = WE1
subject=CN = example.org
notBefore=May 13 04:26:48 2026 GMT
notAfter=Aug 11 05:24:14 2026 GMT
X509v3 Subject Alternative Name:
    DNS:example.org, DNS:*.example.org
```

_(Edge Certificate від Cloudflare, виданий через Google Trust Services)_

Якщо хочете детальніше (ланцюжок та все інше):

```bash
echo | openssl s_client -servername example.org -connect example.org:443 -showcerts 2>/dev/null
```

---

### Шаг 2. Що стоїть на самому сервері (origin)

Дивимося сертифікат, який віддає Nginx/Apache на VPS, в обхід Cloudflare — підключаємося до `127.0.0.1` прямо з сервера.

#### 2.1. Сертифікат, який сервер віддає на 443 порту

```bash
echo | openssl s_client -servername example.org -connect 127.0.0.1:443 2>/dev/null | openssl x509 -noout -issuer -subject -dates -ext subjectAltName
```

Приклад виводу:

```
issuer=C = US, O = Let's Encrypt, CN = R12
subject=CN = example.org
notBefore=May 9 20:53:42 2026 GMT
notAfter=Aug 7 20:53:41 2026 GMT
X509v3 Subject Alternative Name:
    DNS:example.org, DNS:www.example.org
```

#### 2.2. Який веб-сервер слухає 443

```bash
sudo ss -ltnp | grep ':443'
```

---

### Шаг 3. Знайти файли сертифікатів на диску

#### 3.1. Де конфіги і які шляхи до сертифікатів вони використовують

Для Nginx:

```bash
sudo grep -rE 'ssl_certificate|ssl_certificate_key' /etc/nginx/ 2>/dev/null
```

#### 3.2. Стандартні місця, де зазвичай лежать сертифікати

```bash
sudo ls -la /etc/letsencrypt/live/ 2>/dev/null
sudo ls -la /etc/ssl/certs/ 2>/dev/null | head -50
sudo ls -la /etc/ssl/private/ 2>/dev/null
```

---

### Шаг 4. Детальна інформація про знайдений сертифікат

Якщо знайшли шлях до файлу сертифіката (наприклад, `/etc/letsencrypt/live/example.org/fullchain.pem`):

```bash
sudo openssl x509 -in /etc/letsencrypt/live/example.org/fullchain.pem -noout -issuer -subject -dates -ext subjectAltName
```

Повна інформація про сертифікат:

```bash
sudo openssl x509 -in /etc/letsencrypt/live/example.org/fullchain.pem -noout -text | less
```

---

### Шаг 5. Хто і як керує сертифікатами

#### 5.1. Чи є certbot (Let's Encrypt)

```bash
which certbot && certbot --version
```

#### 5.2. Які сертифікати certbot обслуговує

```bash
sudo certbot certificates
```

Приклад виводу:

```
Found the following certs:
  Certificate Name: example.org
    Domains: example.org www.example.org
    Expiry Date: 2026-08-07 20:53:41+00:00 (VALID: 77 days)
    Certificate Path: /etc/letsencrypt/live/example.org/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/example.org/privkey.pem
```

**Важливо:** у папці `/etc/letsencrypt/renewal/` лежать конфіги автооновлення для кожного сертифіката. Якщо один із них зламаний — certbot ругатиметься та може не оновити навіть робочий сертифікат.

#### 5.3. Чи є автооновлення в cron / systemd timer

```bash
sudo systemctl list-timers --all | grep -i certbot
sudo ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.weekly/ 2>/dev/null | grep -iE 'cert|letsencrypt|ssl'
sudo crontab -l 2>/dev/null
```

> **Примітка:** сучасний certbot зазвичай налаштовує оновлення через **systemd-таймер**, а не через cron. Тому `crontab -l` може бути порожнім — і це нормально, якщо таймер працює.

#### 5.4. Історія certbot — коли останній раз оновлювався

```bash
sudo ls -la /var/log/letsencrypt/ 2>/dev/null | tail -20
```

---

### Шаг 6. Cloudflare-сторона — що там налаштовано

Повний список сертифікатів у Cloudflare через термінал не переглянути — це панель Cloudflare. Але можна з'ясувати деякі речі.

#### 6.1. Що віддає Cloudflare публічно (детальніше)

```bash
echo | openssl s_client -servername example.org -connect example.org:443 2>/dev/null | openssl x509 -noout -text | grep -E 'Issuer:|Subject:|Not Before|Not After|DNS:'
```

#### 6.2. Перевірити, чи справді сайт стоїть за Cloudflare

```bash
dig +short example.org
curl -sI https://example.org | grep -iE 'server:|cf-ray:|cf-cache'
```

Якщо `dig` не знайдено:

```bash
getent hosts example.org
```

Адреси вигляду `2a06:98c1::/32` — це IPv6-діапазон Cloudflare. Якщо DNS повертає такі IP — сайт проксується через Cloudflare (помаранчева хмаринка в DNS).

#### 6.3. Керування сертифікатами Cloudflare (панель)

Список сертифікатів і режим SSL — лише через панель Cloudflare:

- **SSL/TLS → Overview** — поточний режим (Flexible / Full / Full Strict)
- **SSL/TLS → Edge Certificates** — що бачать користувачі (Universal / Advanced / Custom)
- **SSL/TLS → Origin Server** — Origin Certificate, який Cloudflare генерує для установки на ваш сервер

Через API Cloudflare це теж можна отримати, але потрібен API-токен.

---

### Шаг 7. Режим SSL у Cloudflare — підсумкова логіка

| Режим             | Що відбувається                                                                                                          |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **Flexible**      | Cloudflare ходить до origin по HTTP — без сертифіката. Не рекомендується.                                                |
| **Full**          | Cloudflare ходить до origin по HTTPS, але приймає навіть самопідписаний сертифікат.                                      |
| **Full (Strict)** | Cloudflare ходить до origin по HTTPS і вимагає валідний сертифікат. Якщо origin-сертифікат невалідний — сайт зламається. |

**Публічно «головний»** — той, що віддає Cloudflare (Шаг 1). Origin-сертифікат валідний чи ні — користувач не побачить, але якщо режим Full (Strict), то невалідний origin-сертифікат зламає сайт.

Перевірити редирект Cloudflare:

```bash
curl -sI http://example.org/ | head -20
```

Якщо приходить `301/302` на `https://` — у Cloudflare увімкнено «Always Use HTTPS» (це добре).

Перевірити, як Cloudflare ходить до origin (емуляція його запиту):

```bash
curl -sI -k --resolve example.org:443:127.0.0.1 https://example.org/ | head -20
```

Переглянути список віртуальних хостів:

```bash
sudo ls -la /etc/nginx/sites-enabled/
sudo grep -rE '^\s*server_name' /etc/nginx/sites-enabled/
```

---

## Підсумкова картина SSL/TLS

### Edge Certificate (те, що бачать користувачі)

| Параметр    | Значення                                              |
| ----------- | ----------------------------------------------------- |
| Видав       | Google Trust Services (WE1) для Cloudflare            |
| Покриває    | `example.org` + wildcard `*.example.org`              |
| Термін      | ~90 днів                                              |
| Тип         | Cloudflare Universal SSL (безкоштовний, автоматичний) |
| Керується   | Cloudflare автоматично, участі не потребує            |
| Де дивитися | Панель Cloudflare → SSL/TLS → Edge Certificates       |

### Origin Certificate (те, що на VPS)

| Параметр    | Значення                            |
| ----------- | ----------------------------------- |
| Видав       | Let's Encrypt                       |
| Покриває    | `example.org`, `www.example.org`    |
| Термін      | ~90 днів                            |
| Файли       | `/etc/letsencrypt/live/example.org` |
| Керується   | certbot на VPS                      |
| Де дивитися | SSH на сервер, утиліта certbot      |

### Схема захисту

```
[Користувач] ──HTTPS (Cloudflare cert)──> [Cloudflare] ──HTTPS (Let's Encrypt cert)──> [VPS, nginx]
```

---

## Повідомлення від Cloudflare на пошту

Cloudflare автоматично слідкує за виданням нових SSL-сертифікатів для домену і надсилає листи-сповіщення. Це **інформаційний лист**, а не сигнал тривоги.

Можна вимкнути сповіщення (але не рекомендується):  
`SSL/TLS → Edge Certificates → Certificate Transparency Monitoring`

---

## Критично важливо знати

Edge cert оновлюєт Cloudflare сам.
Origin cert оновлюєт certbot.

### Права на приватний ключ

`.key`-файл повинен мати права `chmod 600`, власник `root`.  
Якщо `644` — це діра в безпеці.

### Строк дії Let's Encrypt — 90 днів

Оновлювати треба заздалегідь, зазвичай за 30 днів. Certbot робить це автоматично, але лише якщо він реально запущений за розкладом. Буває, що certbot встановлений, але cron/timer вимкнений — і через 90 днів все падає.

### Post-hook: автоматичний перезапуск nginx після оновлення

Приклад правильно налаштованого post-hook у конфізі certbot:

```
post_hook = nginx -s reload; echo "Nginx reloaded by certbot" | mail -s "Nginx report" admin@example.com
```

Після успішного оновлення certbot автоматично перезавантажить nginx (щоб підхопив новий сертифікат) і надішле email-сповіщення.

### Стара версія certbot

Certbot `0.31.0` — дуже стара версія (2018–2019). Актуальні версії — `2.x`.  
Зараз ще працює, але при наступних серйозних змінах з боку Let's Encrypt може перестати. Наприклад, у 2022 році при зміні ланцюжка ISRG / DST Root у старих certbot були проблеми.  
Оновлення certbot потребує окремого уроку.

---

## Виправлення типових помилок

### 1. Зламаний порожній файл конфігу certbot

```bash
# Переіменуйте битий конфіг (не видаляйте)
sudo mv /etc/letsencrypt/renewal/example.org.conf /etc/letsencrypt/renewal/example.org.conf.disabled
```

На сайт не впливає, certbot перестане ругатися бо не читає файли з .disabled

### 2. Зміна email у конфігах certbot

Файли для редагування:

- `/etc/letsencrypt/cli.ini`
- `/etc/letsencrypt/renewal/example.org.conf`

### 3. Перевірка, що автооновлення certbot реально спрацює

```bash
sudo certbot renew --dry-run
```

`--dry-run` — імітація оновлення без реальних змін. Якщо без помилок — все гаразд.

### 4. Проблема з DNS: домен вказує на старий IP

Якщо DNS одного з доменів у сертифікаті вказує на старий сервер — Let's Encrypt піде не на ваш VPS, а кудись ще. Оновлення впаде.  
Перевіряйте DNS перед кожним оновленням, якщо відбувалася міграція.

---

## Умови, на яких тримається поточна схема оновлення сертифікатів

> Це «крихко, але працює» в інфраструктурі — нормально. Головне — розуміти, на чому тримається, щоб не зламати випадково.

### Умова 1: DNS для всіх доменів у сертифікаті вказує на ваш VPS

**Якщо зміниться:** Let's Encrypt піде не на VPS, а в Cloudflare або деінде — оновлення впаде.

### Умова 2: nginx обробляє запити редиректом, а не блокуванням

**Якщо зміниться:** наприклад, WAF-правила, fail2ban, закритий 80-й порт — Let's Encrypt отримає таймаут або 403, оновлення впаде.

### Умова 3: Cloudflare не блокує `/.well-known/acme-challenge/`

**Якщо зміниться:** WAF-правило, «Under Attack Mode», агресивне кешування, бот-захист — валідація впаде.

### Умова 4: Режим SSL у Cloudflare = Full або Full (Strict)

**Якщо зміниться на Flexible:** Cloudflare ходитиме до origin по HTTP. nginx зробить редирект на HTTPS → цикл редиректів → Let's Encrypt заплутається.

### Умова 5: webroot для certbot відповідає поточному document root nginx

**Якщо зміниться:** certbot писатиме challenge-файли в стару папку, а nginx шукатиме в новій → 404 на challenge → оновлення впаде.
