# ДЗ. Установка и настройка защищённого дистрибутива РЕД ОС 8

**Студент:** *Логинов Константин Олегович*  
**Уникальный номер:** *374926*  

---

## Раздел 1. Установка дистрибутива

**Цель:** установить РЕД ОС 8 без графического интерфейса, настроить сеть по DHCP, задать имя хоста `mephi-2026.domain.local`, проверить сетевую связность.

**Выполненные команды:**

```bash
# Проверка сетевой связности
ping -c 4 8.8.8.8 > ping.out
cat ping.out
```

---

## Раздел 2. Управление программным обеспечением

**Цель:** обновить пакеты дистрибутива, установить nginx и libcap-ng-utils, скачать и установить локальный RPM-пакет tcpdump, сохранить историю транзакций dnf.

**Выполненные команды:**

```bash
# Обновление дистрибутива
dnf update -y

# Установка пакетов из репозитория
dnf install -y nginx libcap-ng-utils

# Скачивание и установка локального RPM-пакета tcpdump
dnf download --destdir /tmp tcpdump
rpm -i /tmp/tcpdump-4.99.5-1.red80.x86_64.rpm

# Сохранение истории транзакций
dnf history > dnf.out
```

---

## Раздел 3. Управление файловыми системами

**Цель:** создать на втором диске `/dev/sdb` раздел на всё пространство, отформатировать как ext4 с меткой `MEPHI_WEB`, настроить автомонтирование по метке и выполнить ручное монтирование.

**Выполненные команды:**

```bash
# Разметка второго диска
parted -s /dev/sdb mklabel gpt
parted -s /dev/sdb mkpart primary ext4 0% 100%
partprobe /dev/sdb
lsblk

# Создание файловой системы ext4 с меткой MEPHI_WEB
mkfs.ext4 -L MEPHI_WEB /dev/sdb1

# Создание точки монтирования
mkdir /mephi-web

# Настройка автоматического монтирования по метке
echo 'LABEL=MEPHI_WEB /mephi-web ext4 defaults 0 0' >> /etc/fstab
systemctl daemon-reload

# Ручное монтирование раздела
mount /dev/sdb1 /mephi-web

# Проверка результата
mount | grep mephi
cat /etc/fstab
```

---

## Раздел 4. Управление сервисами

**Цель:** запустить веб-сервер nginx, включить его автозапуск при загрузке, сохранить журнал сообщений сервиса.

**Выполненные команды:**

```bash
# Запуск nginx и включение автозапуска
systemctl enable --now nginx

# Проверка статуса
systemctl status nginx

# Сохранение журнала сообщений nginx за текущую загрузку
journalctl -u nginx -b > journalctl.out
```

---

## Раздел 5. Управление доступом

### 5.1. Дискреционное управление доступом (DAC)

**Цель:** создать каталог `/data/mephi-2026` для файлов проекта. Разработчики user1, user2, user3 (UID 5501–5503) должны иметь полный доступ ко всем файлам друг друга. Кураторы curator1, curator2 (группа `curators`, GID 4444) — только чтение. Остальные — без доступа.

**Выполненные команды:**

```bash
# Создание каталога проекта
mkdir -p /data/mephi-2026

# Создание групп
groupadd -g 5500 developers
groupadd -g 4444 curators

# Создание разработчиков
useradd -u 5501 -g developers user1
useradd -u 5502 -g developers user2
useradd -u 5503 -g developers user3

# Создание кураторов
useradd -g curators curator1
useradd -g curators curator2

# Установка паролей
passwd user1
passwd user2
passwd user3
passwd curator1
passwd curator2

# Настройка прав на каталог проекта
chown root:developers /data/mephi-2026
chmod 2770 /data/mephi-2026

# Настройка ACL для кураторов
dnf install -y acl
setfacl -m g:curators:r-x,m::rwx /data/mephi-2026
setfacl -d -m u::rwx,g::rwx,o::---,g:curators:r-x,m::rwx /data/mephi-2026

# Проверка
getfacl /data/mephi-2026
sudo -u user1 touch /data/mephi-2026/file1
sudo -u user2 sh -c 'echo hello > /data/mephi-2026/file2'
sudo -u user3 sh -c 'echo world > /data/mephi-2026/file3'
sudo -u user1 cat /data/mephi-2026/file3
sudo -u curator1 cat /data/mephi-2026/file1
sudo -u curator1 sh -c 'echo world >> /data/mephi-2026/file1'  # отказ
stat /data/mephi-2026 > stat.out
```

### 5.2. Привилегии (уменьшение set-UID-программ)

**Цель:** настроить tcpdump так, чтобы его мог запускать обычный пользователь без set-UID — через capabilities.

**Выполненные команды:**

```bash
# Проверка текущих прав
ls -l /usr/sbin/tcpdump
getcap /usr/sbin/tcpdump

# Снятие set-UID
chmod u-s /usr/sbin/tcpdump

# Выдача capabilities
setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump

# Проверка
getcap /usr/sbin/tcpdump
sudo -u user1 tcpdump
getcap /usr/sbin/tcpdump > getcap.out
```

### 5.3. Мандатное управление доступом (MAC)

**Цель:** убедиться, что SELinux в режиме Enforcing; разрешить nginx отдавать файлы из нестандартной директории `/mephi-web`.

**Выполненные команды:**

```bash
# Проверка режима SELinux
getenforce
getenforce > getenforce.out

# Настройка контекста SELinux для /mephi-web
semanage fcontext -a -t httpd_sys_content_t "/mephi-web(/.*)?"
restorecon -Rv /mephi-web

# Проверка контекста
ls -Z /mephi-web/
```

---

## Раздел 6. Аутентификация

**Цель:** запретить кураторам локальный вход через консоль, оставив всем остальным возможность входа. Настроить смену пароля каждые 90 дней и минимальную длину пароля 12 символов.

**Выполненные команды:**

```bash
# Проверка наличия модуля pam_access
ls /usr/lib64/security/pam_access.so

# Добавление pam_access.so в /etc/pam.d/login
nano /etc/pam.d/login
# account    required     pam_access.so

# Настройка запрета локального входа для кураторов
nano /etc/security/access.conf
# -:curator1,curator2:LOCAL
# +:ALL:ALL

# Настройка срока действия пароля — 90 дней
chage -M 90 user1
chage -M 90 user2
chage -M 90 user3
chage -M 90 curator1
chage -M 90 curator2

# Настройка минимальной длины пароля
nano /etc/security/pwquality.conf
# minlen = 12

# Проверка
sudo -u user1 passwd   # короткий пароль отвергнут
```

---

## Раздел 7. Тестирование

**Цель:** создать web-страницу `/mephi-web/index.html` с текстом `Hello from Student: 374926`, проверить, что nginx отдаёт её.

**Выполненные команды:**

```bash
# Настройка nginx на нестандартную директорию
cd /etc/nginx/
nano nginx.conf
# root /mephi-web;

# Перезапуск nginx
systemctl restart nginx

# Создание web-страницы
echo 'Hello from Student: 374926' > /mephi-web/index.html
cat /mephi-web/index.html
ls -Z /mephi-web/index.html

# Проверка доступности
curl http://localhost/
curl http://localhost/ > curl.out
```

**Результат:**

```
Hello from Student: 374926
```
