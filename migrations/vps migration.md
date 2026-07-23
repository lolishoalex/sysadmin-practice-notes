# Міграція VPS (Virtual Private Server):

**Інструкція зі створення, завантаження та розгортання снапшоту на новому VPS**

## Генеруємо снапшот

**(може вплинути на роботу продакшену)**
Цієї можливості може не бути в панелі, тоді потрібно підключити.
Генеруємо посилання на снапшот.
Йдемо на новий VPS, в rescue mode:
Чекаємо листа: оновлюємо ключ і пароль.

Перевір що бачить rescue mode:

```bash
lsblk
```

Переконайтеся, що в системі встановлена утиліта qemu-img:

```bash
qemu-img --version
```

Якщо утиліта відсутня, встановіть необхідні пакети:

```bash
apt-get update && apt-get install -y qemu-utils qemu-block-extra
```

Потрібно примонтувати sda1:

```bash
mount /dev/sda1 /mnt
df -h /mnt
```

## Завантажуємо снапшот

```bash
wget -O /mnt/snapshot.qcow2 "LINK"
```

Перевіримо що файл завантажився повністю:

```bash
ls -lh /mnt/snapshot.qcow2
```

## Конвертуємо та розгортаємо снапшот:

```bash
qemu-img convert -p -f qcow2 -O raw /mnt/snapshot.qcow2 /dev/sda
```

Після того як конвертація та розгортання закінчилось — обов'язково переключитись в активний режим, потім назад в rescue mode!!!

Перевіряємо диск в rescue mode, як розгорнулось:

```bash
mount /dev/sda1 /mnt && ls /mnt
```

Перевіряємо потрібні налаштування:
Порт:

```bash
grep -i port /mnt/etc/ssh/sshd_config
```

Дозволені користувачі:

```bash
grep -i allowusers /mnt/etc/ssh/sshd_config
```

Автентифікація за ключем:

```bash
grep -i pubkey /mnt/etc/ssh/sshd_config
```

Перевіримо де лежить снапшот:

```bash
mount /dev/sdb1 /mnt && ls /mnt
```

Перевіримо користувачів:

```bash
cat /mnt/etc/passwd | grep USER
```

Перевіримо SSH порт:

```bash
grep -i port /mnt/etc/ssh/sshd_config
```

Перевіримо ключі користувача USER:

```bash
cat /mnt/home/USER/.ssh/authorized_keys
```

Групи користувача USER:

```bash
grep USER /mnt/etc/group
```

Перевіримо найважливіше — /etc/fstab (як монтуються диски) та мережевий інтерфейс:
fstab:

```bash
cat /mnt/etc/fstab
```

Це важливо! В fstab диск монтується за UUID.
Перевіримо чи збігається UUID у fstab з реальним UUID диска sdb1:

```bash
blkid /dev/sdb1
```

Перевіримо authorized_keys на снапшоті:

```bash
cat /mnt/home/USER/.ssh/authorized_keys
```

Зробимо chroot:

```bash
mount /dev/sdb1 /mnt
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
chroot /mnt
```

Перевстановимо GRUB:

```bash
grub-install /dev/sdb
update-grub
```

Тепер додамо свій SSH ключ щоб можна було зайти.
Відкрий свій публічний ключ — на своєму комп'ютері в іншому терміналі:

```bash
cat ~/.ssh/NAME.pub
```

Тепер всередині chroot створимо папку та додамо ключ:

```bash
mkdir -p /home/USER/.ssh
echo "KEY_CONTENT" >> /home/USER/.ssh/authorized_keys
chmod 700 /home/USER/.ssh
chmod 600 /home/USER/.ssh/authorized_keys
chown -R USER:USER /home/USER/.ssh
cat ~/.ssh/NAME.pub
```

Перевіримо що після перезавантаження порт буде працювати.
Перевіримо конфіг SSH:

```bash
cat /etc/ssh/sshd_config | grep -i port
```

Тепер GRUB перевстановлено і конфіг SSH правильний.
Виходь з chroot та перезавантажуй:
в панелі провайдера → "Reboot my VPS" і пробуємо

```bash
ssh -p PORT -i ~/.ssh/NAME USER@192.0.2.1
```

## Щоб система побачила весь простір, потрібно розширити дві речі: розділ (partition) і файлову систему всередині нього.

Розширення розділу (через growpart)
Встановіть утиліту (якщо її немає):
Для Ubuntu/Debian:

```bash
sudo apt update && sudo apt install -y cloud-guest-utils
```

Виконайте розширення (між sda та 1 має бути пробіл):

```bash
sudo growpart /dev/sda 1
```

Якщо NOCHANGE, Перевірка реального розміру диска:

```bash
lsblk /dev/sda
```

Потрібно виконати одну команду залежно від типу вашої файлової системи.
Щоб не гадати, спочатку перевірте тип:

```bash
df -T /
```

Якщо в колонці "Type" написано ext4, виконайте:

```bash
sudo resize2fs /dev/sda1
```

Як перевірити, що все готово?
Після виконання відповідної команди введіть:

```bash
df -h /
```

## Тепер ваш сервер повністю готовий до роботи.

### Як перевірити його працездатність:

Прописати в хостах
Відкривати в окремому профілі браузера

### Що змінюється після клонування і чому сервіс може "не відразу запрацювати".

Чек-лист після клонування VM:

- _Новий зовнішній IP_ —

всі DNS-записи, що вказують на старий IP, потрібно оновлювати. Це, до речі, типовий затик: VPS підняли, а сайт не відкривається, тому що DNS ще вказує на старий сервер. Або TTL не минув.

- _MAC-адреса мережевої карти інша_ —

на деяких системах через це інтерфейс отримує іншу назву (ens3 замість eth0, або навпаки), і старий конфіг `/etc/netplan/...yaml` або `/etc/network/interfaces` перестає працювати.

- _Machine ID_ —

`/etc/machine-id` залишився від старої машини. Це не критично, але в ідеалі — обнулити і згенерувати заново (systemd-machine-id-setup). Це впливає на journald, іноді на DHCP, іноді на софт, прив'язаний до ID.

- _SSH host keys_ —

ті самі, що у старої машини. Тобто дві VM тепер мають однакові ключі хоста, що погано для безпеки. Правильно — видалити `/etc/ssh/ssh*host*\*` і згенерувати заново.

- _Cron-задачі._

Якщо на старому VPS бігали ті самі cron, що й на новому — може бути дубль дій (наприклад, обидва пишуть в ту саму зовнішню БД, або обидва надсилають листи). Перед запуском нового треба думати, що відключити на старому.

Відкрий свій новий VPS і перевір, що там з machine-id, з SSH host keys, з конфігом мережі.
Просто подивись:

```bash
cat /etc/machine-id
ls -la /etc/ssh/ssh*host*\*
ip a
```

Порівняй зі старим VPS, якщо він ще живий. Якщо збігається — це потенційні ризики.
