# Установка и настройка

Документ описывает выполненную практику. Команды приведены для новой установки на подготовленной Ubuntu 26.04 amd64. Они не являются скриптом для безусловного повторного запуска на работающем сервере. Локальные пароли вводятся только на VM и не передаются в чат или репозиторий.

## 1. Подготовка VM

Проверены ОС, ресурсы и сеть:

```bash
cat /etc/os-release
nproc
free -h
df -h /
ip -br addr
dpkg --print-architecture
sudo ss -lntp
```

Исходно: 1 vCPU, 1,6 GiB RAM, swap 0, около 20G свободного места. На VM уже работали Nginx :80 и SSH :22.

В Windows проверены ping и HTTP:

```powershell
ping -n 4 192.168.1.53
curl.exe -I --connect-timeout 5 --max-time 15 http://192.168.1.53/
```

Получены 4 ответа без потерь и HTTP 200. В PowerShell использован curl.exe, поскольку curl вызывал Invoke-WebRequest.

После освобождения памяти Windows пользователь штатно выключил Ubuntu, выделил 4096 МБ RAM и 2 vCPU, создал снимок Before-Zabbix. После запуска nproc показал 2, free -h — около 3,3 GiB. Резерв crashkernel проверен отдельно и сохранён.

Проверка HTTPS из Ubuntu выявила неправильную дату. Диагностика и исправление времени описаны в [troubleshooting](troubleshooting.md). Перед установкой подтверждены правильная дата, синхронизация и HTTPS 200.

## 2. Подключение репозитория

Для лаборатории выбрана поддерживаемая ветка Zabbix 7.0 LTS. На момент практики репозиторий предлагал 7.0.32.

```bash
curl -fL --connect-timeout 10 --max-time 60 -o zabbix-release.deb https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu26.04_all.deb
sudo dpkg -i ./zabbix-release.deb
sudo apt update
```

Пакет zabbix-release добавляет источник и ключ проверки подписей. Сам сервер на этом шаге не устанавливается. Получены списки resolute/main для amd64 без ошибок подписей.

## 3. Проверка зависимостей и установка

Сначала выполнен расчёт без установки:

```bash
sudo apt install --simulate zabbix-server-pgsql zabbix-frontend-php php8.5-pgsql zabbix-nginx-conf zabbix-sql-scripts zabbix-agent2 postgresql
```

План: 71 новый пакет, 0 обновлений и 0 удалений, 78 существующих обновлений отложены. Выбраны Zabbix 7.0.32, PostgreSQL 18 и PHP 8.5. Этот расчёт проверяет зависимости, но не заменяет последующую проверку работы приложения.

Затем пользователь выполнил установку:

```bash
sudo apt install zabbix-server-pgsql zabbix-frontend-php php8.5-pgsql zabbix-nginx-conf zabbix-sql-scripts zabbix-agent2 postgresql
```

Роли пакетов:

| Пакет | Назначение |
|---|---|
| zabbix-server-pgsql | сервер Zabbix с PostgreSQL |
| zabbix-frontend-php | веб-интерфейс |
| php8.5-pgsql | поддержка PostgreSQL в PHP |
| zabbix-nginx-conf | настройки Nginx/PHP для интерфейса |
| zabbix-sql-scripts | схема и начальные данные базы |
| zabbix-agent2 | сбор метрик Linux |
| postgresql | сервер базы |

Установка подтверждена последующими проверками служб и интерфейса. Полный фактический журнал установки пользователь не предоставлял; пробный расчёт не выдаётся за журнал установки.

## 4. База данных

Проверка сервера:

```bash
pg_lsclusters
```

Результат: версия 18, кластер main, порт 5432, online.

Для свежей базы выполнены:

```bash
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb --owner=zabbix --encoding=UTF8 --template=template0 zabbix
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql --dbname=zabbix --set=ON_ERROR_STOP=on
```

Пароль задаётся интерактивно при создании роли; роль не наделяется правами суперпользователя. Импорт выполняется один раз в пустую базу. При ошибке следует остановиться и проверить причину, а не повторять импорт поверх частично заполненной базы. Путь соответствует пакетам 7.0; для других веток его следует проверять.

Проверено:

```bash
sudo -u zabbix psql --dbname=zabbix --command="SELECT mandatory, optional FROM dbversion;"
```

Получено mandatory = 7000000, optional = 7000030. Само существование этой таблицы не доказывает полноту импорта; дополнительно подтверждены запуск рабочих процессов, работа интерфейса, метрик и событий.

## 5. Подключение Zabbix Server к базе

В файле /etc/zabbix/zabbix_server.conf настроены:

```ini
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=REPLACE_WITH_LOCAL_DB_PASSWORD
```

Значение последней строки заменяется настоящим паролем роли zabbix локально. Для каждого параметра оставлена одна активная строка. Пример в репозитории содержит только заполнитель.

Проверки:

```bash
sudo systemctl restart zabbix-server
systemctl status zabbix-server --no-pager -l
sudo tail -n 50 /var/log/zabbix/zabbix_server.log
```

Подтверждён active (running), запуск configuration syncer и других рабочих процессов. Краткая ошибка связи с агентом сменилась connection restored; ненужные дополнительные компоненты дали Not supported, см. troubleshooting.

## 6. Nginx и PHP-FPM

Порт 80 оставлен существующему сайту. В /etc/zabbix/nginx.conf включены:

```nginx
listen 8080;
server_name 192.168.1.53;
```

Остальная поставляемая конфигурация, включая ограничения доступа к служебным каталогам, сохранена. Подключение PHP:

```nginx
fastcgi_pass unix:/var/run/php/zabbix.sock;
```

Команды:

```bash
sudo nginx -t
sudo systemctl restart php8.5-fpm
sudo ls -l /var/run/php/zabbix.sock
sudo systemctl reload nginx
```

nginx -t успешен по сообщению пользователя; сокет существует с владельцем и группой www-data, правами srw-rw----. После reload мастер открылся в Windows на http://192.168.1.53:8080/.

## 7. Мастер настройки

- Язык: English (en_US).
- Проверки требований: видимые строки на скриншоте OK, пользователь прошёл на следующий экран.
- База: PostgreSQL, localhost, 5432, zabbix, пользователь zabbix.
- Database schema: пусто, используется стандартная схема.
- Пароль: ранее заданный пароль роли PostgreSQL.
- Store credentials: Plain text; действующий файл с паролем в репозиторий не включён.
- Database TLS encryption: выключено для локального TCP-соединения внутри учебной VM.
- Имя установки: Ubuntu Monitoring Lab.
- Часовой пояс интерфейса: Europe/Moscow; ОС остаётся в UTC.

Пользователь завершил мастер, выполнил первый вход и сообщил о смене стандартного пароля Admin. Пароль веб-интерфейса и пароль базы — разные учётные данные. На панели сервер и frontend показали 7.0.32, server running = Yes.

## 8. Автозапуск

```bash
sudo systemctl enable zabbix-server zabbix-agent2 nginx php8.5-fpm postgresql
systemctl is-enabled zabbix-server zabbix-agent2 nginx php8.5-fpm postgresql
```

Получено пять enabled. Работа после реального reboot подтверждена в [проверках](validation.md).

## Источники

[Репозиторий Ubuntu 26.04](https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/), [установка из пакетов](https://www.zabbix.com/documentation/7.0/en/manual/installation/install_from_packages), [требования и совместимость](https://www.zabbix.com/documentation/7.0/en/manual/installation/requirements), [первый вход](https://www.zabbix.com/documentation/7.0/en/manual/quickstart/basic_config/login).
