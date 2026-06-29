# Тест-кейс 3: Настройка NextCloud с СУБД PostgreSQL и ПО для веб-сервера Nginx

## Информация о тест-кейсе
| Параметр | Значение |
|----------|----------|
| **Название тест-кейса** | Настройка NextCloud с СУБД PostgreSQL и ПО для веб-сервера Nginx |
| **Инструкция** | [Настройка NextCloud с СУБД PostgreSQL и ПО для веб-сервера Nginx](https://redos.red-soft.ru/base/redos-8_0/8_0-administation/8_0-redos-cloud/8_0-nextcloud/8_0-nextcloud-with-psql-and-ngnix/) |
| **Окружение** | Версия ОС: РЕД ОС 8, Конфигурация: Сервер графический, Редакция: Стандартная |
| **Версия ПО** | nextcloud-28.0.4-1, postgresql14-server-14.23-1, nginx-1.24.0-1, php-8.1.34-1 |
| **Выполняется на** | ВМ1 (server, IP: 192.168.3.10, FQDN: server.redosipa.test) |
| **Дата проверки** | 29.06.2026 |
| **Тестировщик** | Кильдюшов А. Д. |

---

## 1. Предварительные условия

1. ВМ1 (server) установлена и обновлена (выполнено `dnf up -y`)
2. Статический IP-адрес назначен: **192.168.3.10**
3. Имя хоста: **server.redosipa.test**
4. Снимок ВМ1 создан перед началом тестирования
5. Доступ к учетной записи root
6. Брандмауэр настроен (порты 80, 443, 5432)

---

## 2. Процедура тестирования

### Шаг 2.1: Настройка SELinux

**Действие:** Выполнить:
```bash
# setsebool -P httpd_can_network_connect 1
# setsebool -P httpd_graceful_shutdown 1
# setsebool -P httpd_can_network_connect_db 1
# setsebool -P daemons_dump_core 1
```

**Ожидаемый результат:** Команды выполняются без ошибок. SELinux разрешает подключения Nginx к сети и БД.

---

### Шаг 2.2: Установка NextCloud и зависимостей

**Действие:** Выполнить:
```bash
# dnf install nextcloud nextcloud-postgresql nextcloud-nginx php
```

**Ожидаемый результат:** Пакеты установлены успешно с зависимостями.

---

### Шаг 2.3: Установка PostgreSQL 14

**Действие:** Выполнить:
```bash
# dnf install postgresql14-server
```

**Ожидаемый результат:** Пакет `postgresql14-server` установлен.

---

### Шаг 2.4: Инициализация и запуск PostgreSQL

**Действие:**
```bash
# /bin/postgresql-14-setup initdb
# systemctl enable postgresql-14 --now
# systemctl status postgresql-14
```

**Ожидаемый результат:**
- Инициализация завершена: `Initializing database ... OK`
- Служба запущена и активна: `active (running)`

---

### Шаг 2.5: Создание базы данных и пользователя

**Действие:**
```bash
# su - postgres
# psql
```

В psql выполнить:
```sql
CREATE ROLE nextcloud WITH NOSUPERUSER LOGIN PASSWORD 'nextcloudpassword';
CREATE DATABASE nextcloud WITH OWNER nextcloud;
GRANT ALL PRIVILEGES ON DATABASE nextcloud TO nextcloud;
```

Проверить:
```sql
\l
```

Выйти:
```sql
\q
# exit
```

**Ожидаемый результат:**
- Роль `nextcloud` создана
- База `nextcloud` создана с владельцем `nextcloud`
- Права `GRANT` применены

---

### Шаг 2.6: Настройка доступа к PostgreSQL

**Действие:**
1. Открыть `/var/lib/pgsql/14/data/postgresql.conf`, установить:
```
listen_addresses = '*'
```
2. Открыть `/var/lib/pgsql/14/data/pg_hba.conf`, добавить:
```
host    all             all             127.0.0.1/32      md5
host    all             all             192.168.3.10/32   md5
```
3. Перезапустить PostgreSQL:
```bash
# systemctl restart postgresql-14
```

**Ожидаемый результат:** PostgreSQL перезапущен, доступ разрешён с localhost и 192.168.3.10.

---

### Шаг 2.7: Запуск и настройка Nginx

**Действие:**
```bash
# systemctl enable nginx --now
# systemctl status nginx
```

**Ожидаемый результат:** Nginx запущен и активен: `active (running)`.

---

### Шаг 2.8: Настройка конфигурации Nginx

**Действие:** Открыть `/etc/nginx/nginx.conf`, в секции `server` добавить:
```nginx
    proxy_connect_timeout 600s;
    proxy_send_timeout 600s;
    proxy_read_timeout 600s;
    fastcgi_send_timeout 600s;
    fastcgi_read_timeout 600s;
    sendfile on;
```

Проверить конфигурацию:
```bash
# nginx -t
```

Перезапустить Nginx:
```bash
# systemctl restart nginx
```

**Ожидаемый результат:**
- `nginx: the configuration file /etc/nginx/nginx.conf syntax is ok`
- `nginx: configuration file /etc/nginx/nginx.conf test is successful`
- Nginx перезапущен без ошибок

---

### Шаг 2.9: Настройка PHP

**Действие:** Открыть `/etc/php.ini`, установить:
```ini
max_execution_time = 3600
max_input_time = 3600
max_input_vars = 1000
memory_limit = 512M
upload_tmp_dir = /tmp/
```

**Ожидаемый результат:** Параметры PHP настроены для работы NextCloud.

---

### Шаг 2.10: Подготовка NextCloud к установке

**Действие:**
```bash
# touch /usr/share/nextcloud/config/CAN_INSTALL
# chown -R apache:apache /usr/share/nextcloud
# chcon -R -t httpd_sys_rw_content_t /usr/share/nextcloud
```

Открыть `/etc/nextcloud/config.php`, добавить:
```php
'overwrite.cli.url' => 'http://192.168.3.10/nextcloud',
```

Добавить запись в hosts:
```bash
# echo "192.168.3.10 redsite.ru" >> /etc/hosts
```

**Ожидаемый результат:** NextCloud готов к веб-установке.

---

### Шаг 2.11: Веб-установка NextCloud

**Действие:**
1. Открыть браузер, перейти по адресу: `http://192.168.3.10/nextcloud`
2. Создать учётную запись администратора (логин + пароль)
3. В поле "Хранилище и база данных" выбрать **PostgreSQL**
4. Указать данные:
   - Учётная запись БД: `nextcloud`
   - Пароль БД: `nextcloudpassword`
   - Имя БД: `nextcloud`
   - Хост БД: `localhost:5432`
5. Нажать "Установить"

**Ожидаемый результат:**
- Открывается страница установки NextCloud
- Форма установки отображается корректно
- Поле выбора PostgreSQL доступно

> **СКРИНШОТ 1:** Страница установки NextCloud с заполненными полями PostgreSQL

---

### Шаг 2.12: Установка рекомендуемых приложений

**Действие:** После завершения установки NextCloud нажать "Установить рекомендуемые приложения".

**Ожидаемый результат:**
- Приложения устанавливаются
- Появляется страница "Рекомендуемые приложения"

> **СКРИНШОТ 2:** Страница "Рекомендуемые приложения"

---

### Шаг 2.13: Проверка работы NextCloud

**Действие:** Дождаться завершения установки приложений.

**Ожидаемый результат:**
- Открывается дашборд NextCloud
- Виджеты отображаются корректно
- Навигация работает

> **СКРИНШОТ 3:** Дашборд NextCloud

---

## 3. Ожидаемый результат тест-кейса

1. NextCloud установлен с СУБД PostgreSQL 14
2. Nginx настроен как веб-сервер
3. PHP настроен для работы NextCloud
4. Веб-интерфейс NextCloud доступен по адресу `192.168.3.10/nextcloud`
5. Администратор создан и может войти
6. Дашборд отображается корректно с виджетами
7. SELinux настроен для работы веб-приложения

---

## 4. Статус

| Статус | Дата | Примечание |
|--------|------|------------|
| Passed | 29.06.2026 | NextCloud установлен и работает |

---

## 5. Фактический результат

- NextCloud установлен с PostgreSQL 14
- Nginx настроен как веб-сервер (порт 80)
- PHP настроен: max_execution_time=3600, memory_limit=512M
- Веб-интерфейс доступен по адресу 192.168.3.10/nextcloud
- Администратор создан и может войти
- Дашборд отображается корректно с виджетами
- SELinux настроен для работы веб-приложения
- PostgreSQL доступен с localhost и 192.168.3.10

---

## 6. Баг-репорты

| ID | Описание | Влияние | Статус |
|----|----------|---------|--------|
| 1 | При установке PostgreSQL требовалась ручная настройка `pg_hba.conf` — добавление `127.0.0.1/32` и `192.168.3.10/32`. | Не повлияло. Доступ настроен корректно. | Решено |


---

## 7. Откат

ВМ1 откачена к чистому снимку для следующих тестов.

---

## Скриншоты

| № | Название файла | Описание |
|---|----------------|----------|
| 1 | скрин_1_установка.png | Страница установки NextCloud с PostgreSQL |
| 2 | скрин_2_приложения.png | Страница "Рекомендуемые приложения" |
| 3 | скрин_3_дашборд.png | Дашборд NextCloud |
