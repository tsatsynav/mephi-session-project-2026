# Сессионный проект — Настройка базовых средств защиты ОС GNU/Linux

Сессионный проект по дисциплине **«Безопасность GNU/Linux»** (НИЯУ МИФИ).
Полный цикл настройки на базе дистрибутива **РЕД ОС 8** без графического интерфейса:
от установки ВМ до публикации web-страницы через nginx с настроенными DAC, MAC,
capabilities и политикой паролей.

---

## 👤 Сведения о студенте

| Поле | Значение |
|------|----------|
| **ФИО** | _Цацын Андрей Владимирович_ |
| **Бухгалтерский шифр** | `375143` |
| **GitHub** | [@tsatsynav](https://github.com/tsatsynav) |

---

## 🖥️ Окружение

| Параметр | Значение |
|----------|----------|
| **Гостевая ОС** | RED OS 8 |
| **Хост** | macOS, Apple Silicon M3 |
| **Гипервизор** | UTM (QEMU + Virtualization.framework) |
| **Архитектура** | ARM64 (aarch64) |
| **Имя хоста** | `mephi-2026.domain.local` |
| **Администратор** | `tsatsynav` (член `wheel`, sudo-доступ) |
| **Первый диск** | `/dev/vda` — 64 ГБ |
| **Второй диск** | `/dev/vdb` — 50 ГБ |

> Все привилегированные команды выполнялись через `sudo` от аккаунта `tsatsynav`.
> Переключение в полный root-shell (`sudo -i`, `su -`) не использовалось.

---

## 📁 Структура репозитория

```
mephi-session-project-2026/
├── README.md                          ← этот файл
├── mephi-screenshot.png               ← скриншот с текстом "Hello from Student: 375143"
│
├── history.out                        ← история команд
├── ping.out                           ← проверка сетевой связности
├── dnf.out                            ← история транзакций DNF
├── stat.out                           ← stat /data/mephi-2026 и /mephi-web
├── journalctl.out                     ← журнал nginx
├── getcap.out                         ← привилегии tcpdump
├── getenforce.out                     ← режим SELinux
├── curl.out                           ← результат curl http://localhost/
│
├── fstab                              ← копия /etc/fstab
├── passwd                             ← копия /etc/passwd
├── shadow                             ← копия /etc/shadow
├── group                              ← копия /etc/group
└── pwquality.conf                     ← копия /etc/security/pwquality.conf
```

---

## 🔧 Раздел 1. Установка дистрибутива

### 1.1. Настройка сети (DHCP)

Проверка:

```bash
$ ip -4 addr show
# inet 10.1.30.44/24 brd ... scope global dynamic ens33
```

### 1.2. Настройка имени хоста

```bash
sudo hostnamectl set-hostname mephi-2026.domain.local
echo "127.0.1.1 mephi-2026.domain.local mephi-2026" | sudo tee -a /etc/hosts

$ hostnamectl
   Static hostname: mephi-2026.domain.local
```

### 1.3. Проверка сетевой связности

```bash
ping -c 4 8.8.8.8 > ~/ping.out
```

**Результат:** 4 успешных ответа, 0% потерь — см. `ping.out`.

📎 Артефакт: `ping.out`

---

## 📦 Раздел 2. Управление программным обеспечением

### 2.1. Обновление дистрибутива

```bash
sudo dnf update -y
```

### 2.2. Установка пакетов из репозиториев

```bash
sudo dnf install -y nginx libcap-ng-utils
```

### 2.3. Установка локального RPM-пакета

```bash
sudo dnf download tcpdump --destdir=/tmp
sudo rpm -ivh /tmp/tcpdump*.rpm

$ which tcpdump
/usr/sbin/tcpdump
```

### Сохранение истории DNF

```bash
dnf history > ~/dnf.out
```

📎 Артефакт: `dnf.out`

---

## 💾 Раздел 3. Управление файловыми системами

> Работаем со вторым диском `/dev/vdb`, добавленным в UTM в Разделе 0.

### 3.1. Создание файловой системы

```bash
sudo fdisk /dev/vdb
# n → p → 1 → Enter → Enter → w

$ lsblk
vdb      50G disk
└─vdb1   50G part

sudo mkfs.ext4 -L MEPHI_WEB /dev/vdb1
```

### 3.2. Монтирование файловой системы

```bash
sudo mkdir -p /mephi-web

# Автомонтирование по метке тома — надёжнее, чем по имени устройства
echo "LABEL=MEPHI_WEB /mephi-web ext4 defaults 0 2" | sudo tee -a /etc/fstab

sudo findmnt --verify
sudo mount -a
sudo systemctl daemon-reload

$ df -h | grep mephi-web
/dev/vdb1   49G  24K  47G  1% /mephi-web
```

📎 Артефакт: `fstab`, `stat.out`

---

## 🛠️ Раздел 4. Управление сервисами

### 4.1. Управление nginx

```bash
sudo systemctl start nginx
sudo systemctl enable nginx

$ systemctl status nginx
   Active: active (running) since ...
```

### 4.2. Журналирование

```bash
sudo journalctl -u nginx -b > ~/journalctl.out
```

📎 Артефакт: `journalctl.out`

---

## 🔐 Раздел 5. Управление доступом

### 5.1. Дискреционное управление доступом (DAC)

**Задача:** организовать совместную работу 3 разработчиков и 2 кураторов
в директории `/data/mephi-2026`.

**Требования:**
- Разработчики (`user1`/`user2`/`user3`, UID 5501/5502/5503) — чтение и запись на **все** файлы.
- Кураторы (`curator1`/`curator2`, группа `curators` GID 4444) — только чтение.
- Остальные — без доступа.

#### Создание групп и пользователей

```bash
sudo groupadd -g 4444 curators
sudo groupadd developers

sudo useradd -m -u 5501 -G developers user1
sudo useradd -m -u 5502 -G developers user2
sudo useradd -m -u 5503 -G developers user3

sudo useradd -m -G curators curator1
sudo useradd -m -G curators curator2

```

#### Директория проекта

```bash
sudo mkdir -p /data/mephi-2026
sudo chown root:developers /data/mephi-2026
sudo chmod 2770 /data/mephi-2026
```

**Ключевая идея:**
- `2770` — бит **set-GID** (`2`) гарантирует, что новые файлы наследуют группу
  `developers`, а не группу создателя. Это даёт всем разработчикам доступ к файлам друг друга.
- Права `---` для остальных — изоляция проекта.

#### ACL для кураторов

```bash
sudo setfacl -m g:curators:rx /data/mephi-2026
sudo setfacl -m d:g:curators:rx /data/mephi-2026   # default ACL для новых файлов
sudo setfacl -m o::--- /data/mephi-2026
```

**Пояснение:** `d:` (default) — ACL, который наследуется новыми файлами и поддиректориями.
Так кураторы получают право чтения на любые новые файлы, но не могут их изменять.

#### Проверка

```bash
$ sudo getfacl /data/mephi-2026
user::rwx
group::rwx
group:curators:r-x
mask::rwx
other::---

# user1 создаёт файл, user2 может писать
$ sudo -u user1 touch /data/mephi-2026/testfile
$ sudo -u user2 touch /data/mephi-2026/testfile2

# Куратор читает, но не пишет
$ sudo -u curator1 cat /data/mephi-2026/testfile       # OK
$ sudo -u curator1 touch /data/mephi-2026/testfile3
touch: cannot touch '.../testfile3': Permission denied  # OK
```

📎 Артефакт: `stat.out`

---

### 5.2. Привилегии (уменьшение количества set-UID-программ)

**Задача:** `tcpdump` требует root для запуска. Заменить полный root на точечные
capabilities, чтобы `user1` мог его запускать без set-UID.

```bash
# Снять бит set-UID (если он есть)
sudo chmod u-s $(which tcpdump)

# Назначить привилегии: raw-сокеты + управление сетевыми интерфейсами
sudo setcap cap_net_raw,cap_net_admin=eip $(which tcpdump)

# Проверка
$ getcap /usr/sbin/tcpdump
/usr/sbin/tcpdump = cap_net_admin,cap_net_raw+eip
```

**Пояснение:**
- `cap_net_raw` — открывать raw-сокеты (захват трафика).
- `cap_net_admin` — управлять сетевыми интерфейсами.
- `eip` — effective, inheritable, permitted. Для capability-dumb программ (таких как
  `tcpdump`, не использующих `libcap`) обязательно нужен флаг `e`, иначе привилегии не активируются.

#### Проверка от обычного пользователя

```bash
$ sudo -u user1 tcpdump -c 1 -i any
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
14:48:08.598781 enp0s1 B  ARP, Request who-has 10.1.30.44 tell _gateway, length 42
1 packet captured
22 packets received by filter
0 packets dropped by kernel
```

Работает без root и без бита set-UID — только за счёт привилегий.

📎 Артефакт: `getcap.out`

---

### 5.3. Мандатное управление доступом (MAC)

#### Режим SELinux

```bash
$ getenforce
Enforcing

# Для сохранения после перезагрузки:
sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
```

#### Контекст SELinux для `/mephi-web`

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/mephi-web(/.*)?"
sudo restorecon -Rv /mephi-web

$ ls -Z /mephi-web
system_u:object_r:httpd_sys_content_t:s0 lost+found
```

#### Конфигурация nginx

```bash
echo 'server {
    listen 80;
    server_name localhost;
    root /mephi-web;
    index index.html;
}' | sudo tee /etc/nginx/conf.d/mephi.conf

sudo nginx -t
sudo systemctl restart nginx
```

📎 Артефакт: `getenforce.out`, `stat.out`

---

## 🔑 Раздел 6. Аутентификация

### 6.1. Ограничение локального входа для кураторов

```bash
sudo usermod -s /sbin/nologin curator1
sudo usermod -s /sbin/nologin curator2

$ grep curator /etc/passwd
curator1:x:1004:4444::/home/curator1:/sbin/nologin
curator2:x:1005:4444::/home/curator2:/sbin/nologin
```

### 6.2. Политика паролей

```bash
# Максимальный срок действия пароля — 90 дней
sudo chage -M 90 user1
sudo chage -M 90 user2
sudo chage -M 90 user3
sudo chage -M 90 curator1
sudo chage -M 90 curator2

# Минимальная длина пароля — 12 символов
sudo sed -i 's/^#\s*minlen.*/minlen = 12/' /etc/security/pwquality.conf
grep -q '^minlen' /etc/security/pwquality.conf || \
    echo 'minlen = 12' | sudo tee -a /etc/security/pwquality.conf

$ grep minlen /etc/security/pwquality.conf
minlen = 12

$ sudo chage -l user1
Maximum number of days between password change : 90
```

📎 Артефакты: `pwquality.conf`, `passwd`, `shadow`

---

## 🌐 Раздел 7. Тестирование

### 7.1. Создание web-страницы

```bash
echo "Hello from Student: 375143" | sudo tee /mephi-web/index.html

$ cat /mephi-web/index.html
Hello from Student: 375143
```

### 7.2. Проверка доступности

```bash
curl http://localhost/ > ~/curl.out

$ cat ~/curl.out
Hello from Student: 375143
```

**Ожидаемый результат достигнут.** Скриншот терминала с выводом сохранён как
`mephi-screenshot.png`.

📎 Артефакты: `curl.out`, `mephi-screenshot.png`

---

## 🌍 Раздел 8. Публикация

Репозиторий публичный, доступен по ссылке:

**https://github.com/tsatsynav/mephi-session-project-2026**

### Собранные артефакты

| Файл | Раздел | Что показывает |
|------|--------|----------------|
| `mephi-screenshot.png` | 7 | Скриншот с текстом `Hello from Student: 375143` |
| `history.out` | все | История выполненных команд |
| `ping.out` | 1.3 | Успешный `ping 8.8.8.8` |
| `dnf.out` | 2 | История транзакций DNF |
| `stat.out` | 3, 5 | stat `/data/mephi-2026` и `/mephi-web` |
| `journalctl.out` | 4.2 | Журнал nginx текущей загрузки |
| `getcap.out` | 5.2 | Привилегии `tcpdump` |
| `getenforce.out` | 5.3 | Режим SELinux (`Enforcing`) |
| `curl.out` | 7.2 | Ответ web-сервера |
| `fstab` | 3.2 | Автомонтирование по `LABEL=MEPHI_WEB` |
| `passwd` | 5.1, 6 | Учётные записи с UID/GID и оболочками |
| `shadow` | 6.2 | Политика паролей (90 дней) |
| `group` | 5.1 | Группы `curators` (GID 4444) и `developers` |
| `pwquality.conf` | 6.2 | `minlen = 12` |

---

## ✅ Итоги

В ходе проекта выполнено:

- [x] Установка **РЕД ОС 8** в режиме виртуализации на **UTM / Apple Silicon M3** 
- [x] Настройка DHCP, hostname `mephi-2026.domain.local`, проверка внешней связности
- [x] Обновление дистрибутива и установка `nginx` + `libcap-ng-utils`
- [x] Работа с локальным RPM-пакетом `tcpdump`
- [x] Создание и автомонтирование ext4 по метке тома
- [x] Запуск и автозагрузка `nginx`, журналирование через `journalctl`
- [x] **DAC:** UID/GID, `chmod 2770` с set-GID, ACL для кураторов (default ACL)
- [x] **Capabilities:** `setcap cap_net_raw,cap_net_admin=eip` вместо set-UID для `tcpdump`
- [x] **MAC:** SELinux Enforcing + контекст `httpd_sys_content_t` для `/mephi-web`
- [x] **Аутентификация:** `nologin` для кураторов, пароль 90 дней / 12 символов
- [x] Публикация web-страницы с шифром `375143` и проверка через `curl`
- [x] Публикация артефактов на GitHub

---