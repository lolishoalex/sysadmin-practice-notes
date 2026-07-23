# Оптимізація резервного копіювання на сервері

## Треба подивитися якими засобами робиться бекап на старому сервері, що там зберігається, коли і куди, чи все це робиться оптимально

### Дивимося що є на сервері зараз

1.1 Які cron-задачі існують (бекапи часто там)

Що робить: виводить список завдань планувальника для поточного користувача. Тільки читає.

```bash
crontab -l
```

Що робить: показує системний crontab-файл. Тільки читає.

```bash
cat /etc/crontab
```

Що робить: показує всі файли з додатковими cron-задачами в системній директорії. Тільки читає.

```bash
ls /etc/cron.d/ && cat /etc/cron.d/*
```

1.2 Які плагіни бекапу встановлені у WordPress
Що робить: виводить список всіх плагінів WP з їх статусом (активний/неактивний). Тільки читає.

```bash
wp plugin list --path=/www/example.com/html/ --allow-root
```

1.3 Чи є вже якісь бекап-файли на сервері
Що робить: шукає по всій файловій системі файли з розширенням .sql або .sql.gz (типові бекапи бази). grep -v proc виключає системні псевдофайли, head -30 обмежує вивід першими 30 результатами. Тільки читає.

```bash
find / -name "*.sql" -o -name "*.sql.gz" 2>/dev/null | grep -v proc | head -30
```

Що робить: шукає .zip-архіви, новіші за wp-config.php — часто плагіни бекапу зберігають архіви саме так. Тільки читає.

```bash
find / -name "*.zip" -newer /var/www/html/wp-config.php 2>/dev/null | grep -v proc | head -20
```

1.4 Загальна інформація про сервер і диск
Що робить: показує скільки місця зайнято/вільно на дисках. Тільки читає.

```bash
df -h
```

Що робить: показує використання оперативної пам'яті. Тільки читає.

```bash
free -h
```

### Наступні кроки діагностики

2.1 Подивимося скрипти бекапу
Що робить: показує вміст скрипта бекапу БД. Тільки читає.

```bash
cat /home/backups/db_backup.sh
```

Що робить: показує вміст системного бекап-скрипта. Тільки читає.

```bash
cat /home/backups/systembackup.sh
```

2.2 Що займає місце на диску
Що робить: показує скільки місця займає директорія з архівними бекапами. Тільки читає.

```bash
du -sh /home/olduser/
```

Що робить: скільки займають поточні бекапи БД. Тільки читає.

```bash
du -sh /www/example.com/html/db_backup/
```

Що робить: скільки займає директорія з бекап-скриптами (і можливо там є ще файли). Тільки читає.

```bash
du -sh /home/backups/
```

Що робить: показує розмір кожної директорії всередині /www/ — побачимо де найбільше місця. Тільки читає.

```bash
du -sh /www/* 2>/dev/null
```

Що робить: показує розмір кожної папки в корені сайту, від найбільшої. Тільки читає.

```bash
du -sh /www/example.com/html/* 2>/dev/null | sort -rh | head -20
```

Що робить: скільки займають системні бекапи що ніколи не видалялись. Тільки читає.

```bash
du -sh /www/example.com/html/system_backup/ 2>/dev/null
```

Що робить: перевіряємо чи є креди для автоматичного підключення mysqldump. Тільки читає.

```bash
cat /root/.my.cnf 2>/dev/null || echo "Файл не існує"
```

### Наступна діагностика

Що їсть 112G в wp-content?
Що робить: розбивка по папках всередині wp-content. Тільки читає.

```bash
du -sh /www/example.com/html/wp-content/* 2>/dev/null | sort -rh | head -10
```

Перевіряємо чи mysqldump взагалі працює без пароля
Що робить: пробує підключитись до MySQL і зробити дамп тільки структури (без даних, швидко). Якщо помилка — побачимо яка. Тільки читає, нічого не зберігає.

```bash
mysqldump example_db --no-data 2>&1 | head -5
```

Дивимось що в system_backup
Що робить: список файлів системного бекапу з датами і розмірами. Тільки читає.

```bash
ls -lh /www/example.com/html/system_backup/ | head -20
```

Перевіряємо користувача MySQL
Що робить: показує користувачів MySQL і метод їх аутентифікації. Тільки читає.

```bash
mysql -e "SELECT user, host, plugin FROM mysql.user;" 2>&1
```

Що робить: дивимось що саме там за 16G старих файлів перед тим як чистити. Тільки читає.

```bash
ls -lh /home/olduser/
```

Що робить: рахує скільки всього файлів накопичилось в system_backup. Тільки читає.

```bash
find /www/example.com/html/system_backup/ -name "*.tar" | wc -l
```

Що робить: показує найстаріші файли — переконаємось що це дійсно старі бекапи. Тільки читає.

```bash
find /www/example.com/html/system_backup/ -name "*.tar" | sort | head -5
```

Що робить: показує що саме займає місце в папці 2022. Тільки читає.

```bash
du -sh /home/olduser/2022/* 2>/dev/null | sort -rh
```

Що робить: спустошує файл до 0 байт. WordPress продовжить в нього писати — файл не зникне. Не впливає на роботу сайту.

```bash
> /www/example.com/html/wp-content/debug.log
```

Що робить: видаляє всі .tar файли старші 7 днів в папці system_backup. Свіжі (останні 7 днів) залишаються. Незворотна операція.

```bash
find /www/example.com/html/system_backup/ -name "*.tar" -mtime +7 -exec rm {} \;
```

Що робить: рахує скільки файлів залишилось і який розмір папки. Тільки читає.

```bash
find /www/example.com/html/system_backup/ -name "*.tar" | wc -l && du -sh /www/example.com/html/system_backup/
```

Видаляємо сміття з /home/olduser/

```bash
rm -rf /home/olduser/2022/
```

...

## Навчитися робити бекап бази з контентом на старому сервері

Створюємо правильну структуру папок

```bash
mkdir -p /home/backups/db && mkdir -p /home/backups/logs && mkdir -p /home/backups/system
```

Бекапимо старі скрипти

```bash
cp /home/backups/db_backup.sh /home/backups/db_backup.sh.old && cp /home/backups/systembackup.sh /home/backups/systembackup.sh.old
```

Що робить: перевіряє чи встановлений logrotate. Тільки читає.

```bash
which logrotate && logrotate --version 2>&1 | head -1
```

Новий db_backup.sh

```bash
cat > /home/backups/db_backup.sh << 'EOF'
#!/bin/bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

# --- Settings ---
DB_NAME="example_db"
BACKUP_DIR="/home/backups/db"
LOG_FILE="/home/backups/logs/db_backup.log"
KEEP_DAYS=4
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M")
DUMP_FILE="$BACKUP_DIR/${DB_NAME}-${TIMESTAMP}.sql"

mkdir -p "$BACKUP_DIR"
mkdir -p "$(dirname $LOG_FILE)"

echo "=== Start backup: $(date) ===" >> "$LOG_FILE"

mysqldump "$DB_NAME" > "$DUMP_FILE"

if [ ! -s "$DUMP_FILE" ]; then
    echo "ERROR: dump not created or empty!" >> "$LOG_FILE"
    exit 1
fi

echo "Dump created: $DUMP_FILE ($(du -sh $DUMP_FILE | cut -f1))" >> "$LOG_FILE"

gzip -q "$DUMP_FILE"
echo "Compressed: ${DUMP_FILE}.gz ($(du -sh ${DUMP_FILE}.gz | cut -f1))" >> "$LOG_FILE"

DELETED=$(find "$BACKUP_DIR" -maxdepth 1 -type f -mtime +${KEEP_DAYS} -name "${DB_NAME}-*" -print -exec rm {} \;)
if [ -n "$DELETED" ]; then
    echo "Old backups deleted:" >> "$LOG_FILE"
    echo "$DELETED" >> "$LOG_FILE"
fi

echo "=== Ended: $(date) ===" >> "$LOG_FILE"
echo "" >> "$LOG_FILE"
EOF
```

Новий systembackup.sh

```bash
cat > /home/backups/systembackup.sh << 'EOF'
#!/bin/bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

# --- Settings ---
BACKUP_DIR="/home/backups/system"
LOG_FILE="/home/backups/logs/systembackup.log"
KEEP_DAYS=7
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M")
ARCHIVE="$BACKUP_DIR/system-${TIMESTAMP}.tar.gz"

FOLDERS="/etc/nginx /etc/mysql /etc/php /etc/letsencrypt"

mkdir -p "$BACKUP_DIR"
mkdir -p "$(dirname $LOG_FILE)"

echo "=== Start backup: $(date) ===" >> "$LOG_FILE"

tar -czf "$ARCHIVE" $FOLDERS 2>> "$LOG_FILE"

if [ ! -s "$ARCHIVE" ]; then
    echo "ERROR: dump not created or empty!" >> "$LOG_FILE"
    exit 1
fi

echo "Dump created: $ARCHIVE ($(du -sh $ARCHIVE | cut -f1))" >> "$LOG_FILE"

DELETED=$(find "$BACKUP_DIR" -maxdepth 1 -type f -mtime +${KEEP_DAYS} -name "system-*" -print -exec rm {} \;)
if [ -n "$DELETED" ]; then
    echo "Old backups deleted:" >> "$LOG_FILE"
    echo "$DELETED" >> "$LOG_FILE"
fi

echo "=== Ended: $(date) ===" >> "$LOG_FILE"
echo "" >> "$LOG_FILE"
EOF
```

Конфіг logrotate
error: /etc/logrotate.conf:3 unknown option 'dayly' — підтверджено, помилка є і logrotate ігнорує цей рядок.
Виправляємо помилку в logrotate.conf

```bash
sed -i 's/^dayly$/daily/' /etc/logrotate.conf
```

Що робить: Створюємо конфіг для наших логів. Щотижня стискає старий лог, зберігає 4 тижні (місяць), потім видаляє. Не впливає на роботу сайту.

```bash
cat > /etc/logrotate.d/backups << 'EOF'
/home/backups/logs/*.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
    create 0640 root root
}
EOF
```

Що робить: робить скрипти виконуваними. Не запускає їх.

```bash
chmod +x /home/backups/db_backup.sh && chmod +x /home/backups/systembackup.sh
```

Що робить: показує файли і вміст обох скриптів. Тільки читає.

```bash
ls -la /home/backups/ && echo "---" && cat /home/backups/db_backup.sh && echo "---" && cat /home/backups/systembackup.sh
```

Що робить: запускає скрипт бекапу БД прямо зараз. Створить файл дампу в /home/backups/db/. Не впливає на роботу сайту — тільки читає БД.

```bash
bash /home/backups/db_backup.sh
```

Після виконання перевіряємо результат:

```bash
cat /home/backups/logs/db_backup.log && echo "---" && ls -lh /home/backups/db/
```

Помилок немає

Що робить: архівує /etc/nginx, /etc/mysql, /etc/php, /etc/letsencrypt. Це тільки конфіги — невеликі файли, відпрацює швидко. Не впливає на роботу сайту.

```bash
bash /home/backups/systembackup.sh
```

Після виконання перевіряємо результат:

```bash
cat /home/backups/logs/systembackup.log && echo "---" && ls -lh /home/backups/system/
```

Помилок немає

## Налаштувати збереження бекапу бази з контентом, вночі щодня, на старому сервері

Дивимось поточний crontab

```bash
crontab -l
```

І в /etc/crontab

db_backup.sh — запускається о 02:22 щодня
systembackup.sh — запускається о 03:20 щодня
Шляхи правильні, нічого міняти не треба.

### Налаштування серверу MariaDB

Percona XtraBackup є стандартом де-факто, але це варто робити для високозавантажених проектів.
MariaDB - це фактично форк XtraBackup адаптований спеціально для MariaDB, а у вас саме MariaDB 10.3

Перевіримо версію MariaDB точно

```bash
mysql --version
```

Перевіримо чи вже встановлений mariadb-backup

```bash
which mariabackup 2>/dev/null || echo "Не встановлений"
```

Перевіримо доступні пакети

```bash
apt-cache search mariadb-backup
```

Встановлюємо MariaDB Backup
Старий сервер на Debian 10 Buster — End of Life з 2022 року. Жоден сучасний репозиторій його не підтримує.
Є один варіант — встановити через archive.debian.org. Це офіційний архів Debian де зберігаються всі старі пакети назавжди. Не ідеально, але працює і безпечно.

## Налаштувати збереження такого ж плану бекапу бази з контентом і зображеннями, вночі щодня, на новому сервері + і ще перекидати цей бекап на старий сервер також щоночі

Швидка діагностика на новому сервері (203.0.113.20):

```bash
crontab -l
```

```bash
cat /etc/crontab
```

```bash
find / -name "*.sql" -o -name "*.sql.gz" 2>/dev/null | grep -v proc | head -30
```

```bash
df -h && free -h
```

```bash
du -sh /www/example.com/html/* 2>/dev/null | sort -rh | head -10
```

```bash
cat /home/backups/db_backup.sh 2>/dev/null || echo "Файл не існує"
```

```bash
cat /home/backups/systembackup.sh 2>/dev/null || echo "Файл не існує"
```

Всі команди тільки читають, нічого не змінюють.

План для нового сервера
Робимо все те саме але вже знаємо послідовність — буде швидше.

Блок 1 — Чистимо сміття (як на старому)

Спочатку переконаємось що в olduser те саме:
перевіряємо розмір папок

```bash
du -sh /home/olduser/2022/* 2>/dev/null | sort -rh
```

рахує скільки старих system_backup накопичилось

```bash
find /www/example.com/html/system_backup/ -name "*.tar" | wc -l
```

перевіряємо розмір debug.log

```bash
du -sh /www/example.com/html/wp-content/debug.log 2>/dev/null
```

```bash
cat /root/.my.cnf 2>/dev/null || echo "Файл не існує"
```

```bash
logrotate --version 2>&1 | head -1
```

Блок 1 — Чистимо сміття
Що робить: спустошує debug.log до 0 байт. Не впливає на роботу сайту.

```bash
> /www/example.com/html/wp-content/debug.log
```

Що робить: видаляє system_backup старші 7 днів. Залишає свіжі. Незворотна операція.

```bash
find /www/example.com/html/system_backup/ -name "*.tar" -mtime +7 -exec rm {} \;
```

Що робить: видаляє папку з дублікатами медіафайлів і старими бекапами. Незворотна операція. Не впливає на роботу сайту.

```bash
rm -rf /home/olduser/
```

_Після команди rm -rf /home/olduser/ сталася критична помилка серверу, сайт перестав працювати, детальне описання проблеми в Task 4.1_

Що робить: виводить останні повідомлення ядра про файлову систему та диск. Допоможе побачити чи є ще помилки після перезавантаження.

```bash
dmesg | grep -iE 'ext4|error|fail|bad block|sda' | tail -50
```

використання диска

```bash
df -h
```

Починаємо з видалення /home/olduser/ — але цього разу обережно, частинами.
Спочатку подивимось що там є (тільки читання):

```bash
du -sh /home/olduser/
```

Видаляємо

```bash
rm -rf /home/olduser/
```

Тепер створюємо структуру папок і нові скрипти. Все те саме що на старому сервері — одним блоком:
Структура папок

```bash
mkdir -p /home/backups/db && mkdir -p /home/backups/logs && mkdir -p /home/backups/system
```

Бекапимо старі скрипти

```bash
cp /home/backups/db_backup.sh /home/backups/db_backup.sh.old && cp /home/backups/systembackup.sh /home/backups/systembackup.sh.old
```

Новий db_backup.sh

```bash
cat > /home/backups/db_backup.sh << 'EOF'
...
```

Новий systembackup.sh

```bash
cat > /home/backups/systembackup.sh << 'EOF'
...
```

Права на виконання

```bash
chmod +x /home/backups/db_backup.sh && chmod +x /home/backups/systembackup.sh
```

Виправляємо logrotate і додаємо конфіг

```bash
sed -i 's/^dayly$/daily/' /etc/logrotate.conf
```

```bash
cat > /etc/logrotate.d/backups << 'EOF'
/home/backups/logs/*.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
    create 0640 root root
}
EOF
```

Перевіряємо все

```bash
ls -la /home/backups/ && echo "---" && logrotate --debug /etc/logrotate.conf 2>&1 | grep -E "error|backups"
```

Тест db_backup.sh

```bash
bash /home/backups/db_backup.sh
```

Після завершення:

```bash
cat /home/backups/logs/db_backup.log && echo "---" && ls -lh /home/backups/db/
```

Тест systembackup.sh

```bash
bash /home/backups/systembackup.sh
```

Після завершення:

```bash
cat /home/backups/logs/systembackup.log && echo "---" && ls -lh /home/backups/system/
```

Тепер перевіримо crontab — чи правильно налаштований розклад:

```bash
crontab -l && echo "---" && cat /etc/crontab
```

Мені треба щоб схема була така:
бекап створюється від рута - рут має класти бекап за допомогою скрипта в спеціально створеного користувача
потім сервер старий йде в того користувача і робить копію бекапу собі в бекапи через ссш - це бестпрактіс або ні? і в зв'язку з цим треба з'ясувати що у мене за користувач backup (він є і на новому і на старому серверах), що він робить лдя кого і коли, і чи можу я його використати тільки для цих цілей, поки що залишаємо в папці www все як є і на новому і на старому сервері, а також залишаємо в папці home все як є і на новому і на старому сервері, до з'ясування обставин про користувача backup, ми або створимо свого або будемо тим користуватись

Що робить: створює користувача backupuser без пароля і без інтерактивного логіну. Не впливає на роботу сайту.

```bash
adduser --disabled-password --gecos "" backupuser
```

Що робить: перевіряємо що користувач створився. Тільки читає.

```bash
cat /etc/passwd | grep backupuser
```

Він має підключатись тільки по SSH для копіювання файлів, але не мати повноцінного термінального доступу.
Що робить: обмежує shell користувача — він зможе підключатись по SSH але не зможе виконувати довільні команди. Не впливає на роботу сайту.

```bash
usermod -s /bin/rbash backupuser
```

Що робить: створює папки для бекапів і скриптів. Не впливає на роботу сайту.

```bash
mkdir -p /home/backupuser/backups/db && mkdir -p /home/backupuser/backups/system && mkdir -p /home/backupuser/backups/logs && mkdir -p /home/backupuser/scripts
```

Що робить: передає всі папки у власність backupuser. Не впливає на роботу сайту.

```bash
chown -R backupuser:backupuser /home/backupuser/
```

Що робить: показує структуру папок і права. Тільки читає.

```bash
ls -la /home/backupuser/
```

Тепер налаштовуємо SSH для backupuser — щоб старий сервер міг підключатись по SSH без пароля.
Що робить: створює папку .ssh і файл authorized_keys з правильними правами. Не впливає на роботу сайту.

```bash
mkdir -p /home/backupuser/.ssh && chmod 700 /home/backupuser/.ssh && touch /home/backupuser/.ssh/authorized_keys && chmod 600 /home/backupuser/.ssh/authorized_keys && chown -R backupuser:backupuser /home/backupuser/.ssh
```

Що робить: перевіряємо права на папку і файл. Тільки читає.

```bash
ls -la /home/backupuser/.ssh/
```

Тепер переходимо на старий сервер (198.51.100.10) — там генеруємо SSH ключ для backupuser, який потім додамо на новий сервер.
Спочатку створюємо backupuser на старому сервері:

```bash
adduser --disabled-password --gecos "" backupuser
```

Що робить: обмежує shell. Не впливає на роботу сайту.

```bash
usermod -s /bin/rbash backupuser
```

Що робить: створює структуру папок. Не впливає на роботу сайту.

```bash
mkdir -p /home/backupuser/backups/db && mkdir -p /home/backupuser/backups/system && mkdir -p /home/backupuser/backups/logs && mkdir -p /home/backupuser/scripts
```

Що робить: встановлює власника. Не впливає на роботу сайту.

```bash
chown -R backupuser:backupuser /home/backupuser/
```

Після виконання перевіряємо:

```bash
ls -la /home/backupuser/
```

Тепер генеруємо SSH ключ на старому сервері — саме звідси буде підключення до нового:
Що робить: генерує пару SSH ключів (публічний і приватний) для backupuser. -N "" означає без пароля на ключ. Нічого не змінює на сервері крім створення файлів ключів.

```bash
sudo -u backupuser ssh-keygen -t ed25519 -C "backupuser@oldserver" -f /home/backupuser/.ssh/id_ed25519 -N ""
```

Що робить: показує публічний ключ який треба буде додати на новий сервер. Тільки читає.

```bash
cat /home/backupuser/.ssh/id_ed25519.pub
```

Що робить: додає публічний ключ старого сервера до списку дозволених ключів для backupuser на новому сервері. Не впливає на роботу сайту.

```bash
echo "ssh-ed25519 AAAA...ПУБЛІЧНИЙ_КЛЮЧ... backupuser@oldserver" >> /home/backupuser/.ssh/authorized_keys
```

Перевіряємо:

```bash
cat /home/backupuser/.ssh/authorized_keys
```

Тепер перевіримо що SSH підключення працює — переходимо на старий сервер і тестуємо:
Що робить: підключається по SSH з старого сервера на новий як backupuser і виконує просту команду. Нічого не змінює.

```bash
sudo -u backupuser ssh -p 50022 backupuser@203.0.113.20 "echo 'SSH connection successful'"
```

Переносимо скрипти і папки під backupuser на новому сервері 203.0.113.20

Переносимо бекапи в папку backupuser

```bash
mv /home/backups/db/* /home/backupuser/backups/db/ 2>/dev/null; mv /home/backups/system/* /home/backupuser/backups/system/ 2>/dev/null; mv /home/backups/logs/* /home/backupuser/backups/logs/ 2>/dev/null
```

Переносимо скрипти

```bash
mv /home/backups/db_backup.sh /home/backupuser/scripts/ && mv /home/backups/systembackup.sh /home/backupuser/scripts/ && mv /home/backups/db_backup.sh.old /home/backupuser/scripts/ && mv /home/backups/systembackup.sh.old /home/backupuser/scripts/
```

Встановлюємо правильного власника

```bash
chown -R backupuser:backupuser /home/backupuser/
```

Перевіряємо структуру

```bash
ls -la /home/backupuser/ && echo "---" && ls -la /home/backupuser/backups/ && echo "---" && ls -la /home/backupuser/scripts/
```

Тепер оновлюємо шляхи в скриптах — вони ще вказують на старі папки /home/backups/:

```bash
sed -i 's|/home/backups/db|/home/backupuser/backups/db|g' /home/backupuser/scripts/db_backup.sh && sed -i 's|/home/backups/logs/db_backup.log|/home/backupuser/backups/logs/db_backup.log|g' /home/backupuser/scripts/db_backup.sh
```

```bash
sed -i 's|/home/backups/system|/home/backupuser/backups/system|g' /home/backupuser/scripts/systembackup.sh && sed -i 's|/home/backups/logs/systembackup.log|/home/backupuser/backups/logs/systembackup.log|g' /home/backupuser/scripts/systembackup.sh
```

Перевіряємо

```bash
grep "BACKUP_DIR\|LOG_FILE" /home/backupuser/scripts/db_backup.sh && echo "---" && grep "BACKUP_DIR\|LOG_FILE" /home/backupuser/scripts/systembackup.sh
```

Тепер оновлюємо crontab

```bash
crontab -l
```

Оновлюємо шлях в crontab

```bash
(crontab -l | sed 's|/home/backups/db_backup.sh|/home/backupuser/scripts/db_backup.sh|') | crontab -
```

Також оновлюємо /etc/crontab для systembackup

```bash
sed -i 's|/home/backups/systembackup.sh|/home/backupuser/scripts/systembackup.sh|' /etc/crontab
```

Перевіряємо:

```bash
crontab -l | grep backup && echo "---" && grep systembackup /etc/crontab
```

Тепер видаляємо стару порожню папку /home/backups/

```bash
rm -rf /home/backups/
```

Перевіряємо

```bash
ls /home/
```

Тепер те саме робимо на старому сервері (198.51.100.10)

## Розгортати свіжий бекап з нового серверу на старий сервер щодня, при цьому видаляючи попередній бекап, таким чином синхронізовуючи старий сервер з новим.

Тепер створюємо скрипт синхронізації на старому сервері (198.51.100.10) — він буде щодня забирати свіжий бекап з нового сервера.

Схема:
Новий сервер о 02:22 → створює бекап БД
Старий сервер о 04:00 → забирає свіжий бекап з нового по SSH

Створюємо скрипт

```bash
cat > /home/backupuser/scripts/sync_backup.sh << 'EOF'
#!/bin/bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

# --- Settings ---
REMOTE_USER="backupuser"
REMOTE_HOST="203.0.113.20"
REMOTE_PORT="50022"
REMOTE_DIR="/home/backupuser/backups/db/"
LOCAL_DIR="/home/backupuser/backups/db/"
LOG_FILE="/home/backupuser/backups/logs/sync_backup.log"
SSH_KEY="/home/backupuser/.ssh/id_ed25519"

mkdir -p "$LOCAL_DIR"
mkdir -p "$(dirname $LOG_FILE)"

echo "=== Sync started: $(date) ===" >> "$LOG_FILE"

# --- Sync backup from remote server ---
rsync -avz --delete \
    -e "ssh -p ${REMOTE_PORT} -i ${SSH_KEY} -o StrictHostKeyChecking=no" \
    ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR} \
    ${LOCAL_DIR} >> "$LOG_FILE" 2>&1

if [ $? -eq 0 ]; then
    echo "Sync completed successfully: $(date)" >> "$LOG_FILE"
else
    echo "ERROR: Sync failed!" >> "$LOG_FILE"
    exit 1
fi

echo "=== Sync finished: $(date) ===" >> "$LOG_FILE"
echo "" >> "$LOG_FILE"
EOF
```

Права на виконання

```bash
chmod +x /home/backupuser/scripts/sync_backup.sh && chown backupuser:backupuser /home/backupuser/scripts/sync_backup.sh
```

Тестуємо вручну

```bash
sudo -u backupuser bash /home/backupuser/scripts/sync_backup.sh
```

Після виконання:

```bash
cat /home/backupuser/backups/logs/sync_backup.log && echo "---" && ls -lh /home/backupuser/backups/db/
```

Додаємо в crontab на старому сервері — запускати о 04:00 (через 2 години після того як новий сервер створить бекап о 02:22)

```bash
(crontab -l; echo "00 04 * * * /home/backupuser/scripts/sync_backup.sh") | crontab -
```

Перевіряємо:

```bash
crontab -l | grep sync
```

Додаємо ще один скрипт який:
Знаходить найсвіжіший файл в /home/backupuser/backups/db/
Розпаковує його
Імпортує в базу example_db
Видаляє тимчасовий розпакований файл
Логує результат

Після цього старий сервер завжди матиме актуальну базу і сайт виглядатиме як новий.

Створюємо скрипт імпорту на старому сервері (198.51.100.10)

```bash
cat > /home/backupuser/scripts/import_backup.sh << 'EOF'
#!/bin/bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

# --- Settings ---
DB_NAME="example_db"
BACKUP_DIR="/home/backupuser/backups/db"
LOG_FILE="/home/backupuser/backups/logs/import_backup.log"

mkdir -p "$(dirname $LOG_FILE)"

echo "=== Import started: $(date) ===" >> "$LOG_FILE"

# --- Find the latest backup file ---
LATEST=$(find "$BACKUP_DIR" -maxdepth 1 -name "*.sql.gz" -type f | sort | tail -1)

if [ -z "$LATEST" ]; then
    echo "ERROR: No backup file found!" >> "$LOG_FILE"
    exit 1
fi

echo "Latest backup: $LATEST" >> "$LOG_FILE"

# --- Decompress and import ---
gunzip -c "$LATEST" | mysql "$DB_NAME"

if [ $? -eq 0 ]; then
    echo "Import completed successfully: $(date)" >> "$LOG_FILE"
else
    echo "ERROR: Import failed!" >> "$LOG_FILE"
    exit 1
fi

echo "=== Import finished: $(date) ===" >> "$LOG_FILE"
echo "" >> "$LOG_FILE"
EOF
```

Права на виконання

```bash
chmod +x /home/backupuser/scripts/import_backup.sh && chown backupuser:backupuser /home/backupuser/scripts/import_backup.sh
```

Тестуємо вручну

```bash
bash /home/backupuser/scripts/import_backup.sh
```

Після виконання:

```bash
cat /home/backupuser/backups/logs/import_backup.log
```

Додаємо в crontab на старому сервері — запускати о 04:30 (після синхронізації о 04:00)

```bash
(crontab -l; echo "30 04 * * * /home/backupuser/scripts/import_backup.sh") | crontab -
```

Перевіряємо весь crontab:

```bash
crontab -l
```

### Видалити старі папки /www/example.com/html/db_backup/ і system_backup/ на обох серверах

Що робить: видаляє старі папки бекапів всередині сайту. Не впливає на роботу сайту.

```bash
rm -rf /www/example.com/html/db_backup/ && rm -rf /www/example.com/html/system_backup/
```

Перевіряємо:

```bash
df -h && ls /www/example.com/html/ | grep backup
```

Тепер те саме на новому сервері (203.0.113.20)

### Синхронізація по сертифікатам

Зроблено всі ті самі роботи що і на новому сервері (Task 2).

Битий конфіг example.com.conf перейменований
Email змінено на admin@example.com
Postfix зупинений і вимкнений
_Сертифікат не оновлюється автоматично - не зможе поки DNS веде на новий сервер_

План дій із SSL-сертифікатами при перемиканні на старий сервер

У момент реального перемикання DNS на старий сервер просто запускається certbot renew --force-renewal — і Let's Encrypt видасть новий валідний сертифікат вже на старому сервері (бо тоді DNS вже вестиме саме туди).
Послідовність: переключити DNS → дочекатись поширення → перевірити що 80 порт відкритий → certbot renew --force-renewal → перезавантажити nginx → перевірити сайт.
Єдине що варто перевіряти періодично (раз на місяць) — що старий сервер живий і certbot на ньому досі робочий, щоб не виявити несподіванку саме в момент аварії.

### Бекапи і синхронізація зображень на новому і старому серверах

Робота на новому сервері - Бекап uploads:
Що робить: перевіряє чи backupuser може читати папку uploads. Тільки читає.

```bash
sudo -u backupuser ls -la /www/example.com/html/wp-content/uploads/ 2>&1 | head -5
```

Що робить: показує власника і права папки uploads. Тільки читає.

```bash
ls -la /www/example.com/html/wp-content/ | grep uploads
```

Що робить: показує до яких груп належить backupuser. Тільки читає.

```bash
groups backupuser
```

Створюємо папку для бекапу uploads

```bash
mkdir -p /home/backupuser/backups/uploads
```

встановлює власника

```bash
chown backupuser:backupuser /home/backupuser/backups/uploads
```

Створюємо скрипт бекапу
Логіка:

Знаходить усі папки з 4-значною назвою (роки) автоматично через find
Синхронізує кожен рік окремо, з паузою 10 секунд між ними
Якщо якийсь рік не синхронізувався — пише помилку в лог, але не зупиняє весь процес (інші роки продовжать обробку)
Наприкінці одним викликом синхронізує все що залишилось (файли в корені, плагінні папки), виключаючи вже оброблені роки
Логує час початку/кінця кожного етапу і фінальний розмір бекапу

```bash
cat > /home/backupuser/scripts/uploads_backup.sh << 'EOF'
#!/bin/bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

# --- Settings ---
SOURCE_DIR="/www/example.com/html/wp-content/uploads"
BACKUP_DIR="/home/backupuser/backups/uploads"
LOG_FILE="/home/backupuser/backups/logs/uploads_backup.log"
PAUSE_SECONDS=10

mkdir -p "$BACKUP_DIR"
mkdir -p "$(dirname $LOG_FILE)"

echo "=== Backup started: $(date) ===" >> "$LOG_FILE"

# --- Find all year folders automatically (any 4-digit folder name) ---
YEARS=$(find "$SOURCE_DIR" -maxdepth 1 -type d -regextype posix-extended -regex '.*/[0-9]{4}' -printf '%f\n')

for YEAR in $YEARS; do
    echo "--- Syncing $YEAR: $(date) ---" >> "$LOG_FILE"
    rsync -a "$SOURCE_DIR/$YEAR/" "$BACKUP_DIR/$YEAR/" >> "$LOG_FILE" 2>&1
    if [ $? -ne 0 ]; then
        echo "ERROR: Sync failed for year $YEAR!" >> "$LOG_FILE"
    fi
    sleep $PAUSE_SECONDS
done

# --- Sync everything else (root files + non-year folders) ---
echo "--- Syncing remaining files/folders: $(date) ---" >> "$LOG_FILE"
rsync -a --exclude='[0-9][0-9][0-9][0-9]/' "$SOURCE_DIR/" "$BACKUP_DIR/" >> "$LOG_FILE" 2>&1

if [ $? -eq 0 ]; then
    echo "Backup completed successfully: $(date)" >> "$LOG_FILE"
    echo "Backup size: $(du -sh $BACKUP_DIR | cut -f1)" >> "$LOG_FILE"
else
    echo "ERROR: Final sync step failed!" >> "$LOG_FILE"
    exit 1
fi

echo "=== Backup finished: $(date) ===" >> "$LOG_FILE"
echo "" >> "$LOG_FILE"
EOF
```

Права на виконання

```bash
chmod +x /home/backupuser/scripts/uploads_backup.sh && chown backupuser:backupuser /home/backupuser/scripts/uploads_backup.sh
```

Перевіримо що в файлі реально записалось:

```bash
cat /home/backupuser/scripts/uploads_backup.sh
```

Перевіримо права на файл:

```bash
ls -la /home/backupuser/scripts/uploads_backup.sh
```

Що робить: запускає бекап у фоні, продовжиться навіть якщо закриєш термінал. Не впливає на роботу сайту.

```bash
sudo -u backupuser nohup bash /home/backupuser/scripts/uploads_backup.sh > /tmp/uploads_backup_output.log 2>&1 &
```

Дивимось лог у реальному часі

```bash
tail -f /home/backupuser/backups/logs/uploads_backup.log
```

Перевіримо фінальний лог:

```bash
cat /home/backupuser/backups/logs/uploads_backup.log
```

Що робить: порівнює розмір оригінальної папки і бекапу. Тільки читає.

```bash
du -sh /www/example.com/html/wp-content/uploads/ && du -sh /home/backupuser/backups/uploads/
```

Що робить: рахує кількість файлів в оригіналі і в бекапі — числа мають співпасти. Тільки читає.

```bash
find /www/example.com/html/wp-content/uploads/ -type f | wc -l && find /home/backupuser/backups/uploads/ -type f | wc -l
```

Бекап реально вдалий — _незважаючи на аварію під час процесу_, дані скопіювались коректно (ми це перевірили ще до інциденту — 112G=112G, 975938=975938 файлів).

Тепер можемо додати в crontab. Подивимось поточний розклад на новому сервері:

```bash
crontab -l
```

Додаємо задачу uploads_backup о 01:30

```bash
(crontab -l; echo "30 01 * * * /home/backupuser/scripts/uploads_backup.sh") | crontab -
```

Перевіряємо:

```bash
crontab -l | grep uploads
```

### Cинхронізація на старий сервер

Перевіримо скільки файлів змінились на новому сервері порівняно зі старим — без передачі даних
Що робить: dry-run (-n) — нічого не копіює, лише порівнює метадані між серверами і показує кількість файлів що відрізняються. Тільки читає і порівнює.

```bash
rsync -avn --delete \
    -e "ssh -p 50022 -i /home/backupuser/.ssh/id_ed25519 -o StrictHostKeyChecking=no" \
    backupuser@203.0.113.20:/www/example.com/html/wp-content/uploads/ \
    /www/example.com/html/wp-content/uploads/ \
    2>/dev/null | grep -v "/$" | wc -l
```

Запускаємо реальну синхронізацію на старому сервері
Створимо скрипт-файл sync_uploads.sh

```bash
cat > /home/backupuser/scripts/sync_uploads.sh << 'EOF'
#!/bin/bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

# --- Settings ---
REMOTE_USER="backupuser"
REMOTE_HOST="203.0.113.20"
REMOTE_PORT="50022"
REMOTE_DIR="/www/example.com/html/wp-content/uploads"
LOCAL_DIR="/www/example.com/html/wp-content/uploads"
LOG_FILE="/home/backupuser/backups/logs/uploads_sync.log"
SSH_KEY="/home/backupuser/.ssh/id_ed25519"
PAUSE_SECONDS=10

mkdir -p "$(dirname $LOG_FILE)"

echo "=== Sync started: $(date) ===" >> "$LOG_FILE"

# --- Get list of year folders from remote server ---
YEARS=$(ssh -p "$REMOTE_PORT" -i "$SSH_KEY" -o StrictHostKeyChecking=no \
    "$REMOTE_USER@$REMOTE_HOST" \
    "find $REMOTE_DIR -maxdepth 1 -type d -regextype posix-extended -regex '.*/[0-9]{4}' -printf '%f\n'")

# --- Sync each year separately ---
for YEAR in $YEARS; do
    echo "--- Syncing $YEAR: $(date) ---" >> "$LOG_FILE"
    rsync -a \
        -e "ssh -p $REMOTE_PORT -i $SSH_KEY -o StrictHostKeyChecking=no" \
        "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/$YEAR/" \
        "$LOCAL_DIR/$YEAR/" >> "$LOG_FILE" 2>&1
    if [ $? -ne 0 ]; then
        echo "ERROR: Sync failed for year $YEAR!" >> "$LOG_FILE"
    fi
    sleep $PAUSE_SECONDS
done

# --- Sync everything else ---
echo "--- Syncing remaining: $(date) ---" >> "$LOG_FILE"
rsync -a --exclude='[0-9][0-9][0-9][0-9]/' \
    -e "ssh -p $REMOTE_PORT -i $SSH_KEY -o StrictHostKeyChecking=no" \
    "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/" \
    "$LOCAL_DIR/" >> "$LOG_FILE" 2>&1

if [ $? -eq 0 ]; then
    echo "Sync completed successfully: $(date)" >> "$LOG_FILE"
else
    echo "ERROR: Final sync step failed!" >> "$LOG_FILE"
    exit 1
fi

echo "=== Sync finished: $(date) ===" >> "$LOG_FILE"
echo "" >> "$LOG_FILE"
EOF
```

Tестуємо:

```bash
chmod +x /home/backupuser/scripts/sync_uploads.sh && chown backupuser:backupuser /home/backupuser/scripts/sync_uploads.sh
```

```bash
sudo -u backupuser bash /home/backupuser/scripts/sync_uploads.sh
```

### Копіювання uploads на зйомний диск як бекап

Що робить: показує розмір папки uploads на старому сервері. Тільки читає.

```bash
du -sh /www/example.com/html/wp-content/uploads/
```

Копіювання на локальному комп'ютері

```bash
nohup rsync -a --progress \
    -e "ssh -p 50022" \
    user@198.51.100.10:/www/example.com/html/wp-content/uploads/ \
    "/media/user/EXTERNAL_DISK/server_uploads_backup_$(date +%Y-%m-%d)/" \
    > /tmp/rsync_to_disk.log 2>&1 &
```

спостерігаємо прогрес:

```bash
tail -f /tmp/rsync_to_disk.log
```
