# Ручна синхронізація медіафайлів: новий → старий сервер

Виконувати на **старому сервері** (`198.51.100.10`) від root.

---

## Крок 1 — Суха перевірка (скільки файлів відрізняється)

```bash
rsync -avn \
    -e "ssh -p 50022 -i /home/backupuser/.ssh/id_ed25519 -o StrictHostKeyChecking=no" \
    backupuser@203.0.113.20:/www/example.com/html/wp-content/uploads/ \
    /www/example.com/html/wp-content/uploads/ \
    2>/dev/null | grep -v "/$"
```

**На що дивитись:** якщо є реальні файли (рядки з назвами файлів, не службові) — є різниця, треба синхронізувати. Якщо тільки 3 службові рядки (receiving, sent, total size) — сервери вже синхронізовані, далі нічого робити не треба.

---

## Крок 2 — Запускаємо синхронізацію

```bash
bash /home/backupuser/scripts/sync_uploads.sh
```

Спостерігаємо прогрес в другому терміналі:
```bash
tail -f /home/backupuser/backups/logs/uploads_sync.log
```
`Ctrl+C` щоб вийти з перегляду логу — синхронізація продовжиться у фоні.

---

## Крок 3 — Перевіряємо лог після завершення

```bash
tail -20 /home/backupuser/backups/logs/uploads_sync.log
```

**На що дивитись:**
- Має бути `Sync completed successfully`
- Жодного `ERROR` в рядках з назвами років
- Є час початку і завершення

```bash
grep "ERROR" /home/backupuser/backups/logs/uploads_sync.log
```
**На що дивитись:** порожній результат = без помилок ✅

---

## Крок 4 — Фінальна суха перевірка (переконуємось що різниці більше немає)

```bash
rsync -avn \
    -e "ssh -p 50022 -i /home/backupuser/.ssh/id_ed25519 -o StrictHostKeyChecking=no" \
    backupuser@203.0.113.20:/www/example.com/html/wp-content/uploads/ \
    /www/example.com/html/wp-content/uploads/ \
    2>/dev/null | grep -v "/$"
```

**На що дивитись:** тільки 3 службові рядки без назв файлів = сервери синхронізовані ✅

Якщо ще є файли — запустити Крок 2 ще раз.

---

## Крок 5 — Перевіряємо стан серверів після синхронізації

```bash
dmesg | grep -iE "ext4-fs error|bad block|remount-ro" | tail -10
```
**На що дивитись:** порожній результат = файлова система здорова ✅

```bash
uptime
```
**На що дивитись:** сервер не перезавантажувався під час синхронізації ✅

```bash
df -h
```
**На що дивитись:** місце на диску не вичерпалось ✅

```bash
curl -sI https://example.com/ | head -3
```
**На що дивитись:** `HTTP/1.1 301` — живий сайт на новому сервері досі відповідає ✅

---

## Примітки

- Синхронізацію запускати **вручну**, не через cron — для повного контролю
- Найкраще запускати **вночі** (після 00:00) коли трафік мінімальний
- Перший крок (суха перевірка) завжди обов'язковий — щоб розуміти обсяг роботи перед запуском
- Якщо суха перевірка показує тисячі файлів — краще запускати через `nohup` у фоні:
  ```bash
  nohup bash /home/backupuser/scripts/sync_uploads.sh > /tmp/sync_output.log 2>&1 &
  ```