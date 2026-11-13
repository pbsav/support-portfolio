# Проверки мониторинга и восстановления

Все результаты ниже относятся к данным пользователя, полученным в этой практике. Скриншоты собраны в [каталоге доказательств](../evidence/README.md).

## 1. CPU

Monitoring → Latest data → Hosts: Zabbix server → Name: CPU utilization → Apply.

На первом скриншоте метрики: 10,173%, последняя проверка 17 секунд назад. Затем открыт Graph за 30 минут.

На графике (13 ноября, около 19:13–19:43 по Москве) данные начинаются примерно с 19:26:

| Показатель | Значение |
|---|---|
| Последнее | 9,9565% |
| Минимум | 7,7411% |
| Среднее по имеющимся данным | 9,2635% |
| Максимум | 10,2117% |

Пустой участок не означает нулевую нагрузку. Шкала автоматически ограничена примерно 7–11%, поэтому визуальная амплитуда не равна сильной загрузке CPU. Контролируемая нагрузка и точность метрики под нагрузкой отдельно не проверялись.

## 2. RAM

Фильтр Name: memory.

| Метрика Zabbix | Значение |
|---|---|
| Total memory | 3,32 GB |
| Available memory | 2,51 GB |
| Available memory in % | 75,5264% |
| Memory utilization | 24,4736% |

Проверка в Ubuntu:

```bash
free -h
```

На скриншоте: total 3,3Gi, used 839Mi, free 1,5Gi, shared 122Mi, buff/cache 1,4Gi, available 2,5Gi; swap 0B.

Сопоставлены total и available. Available учитывает память, которую можно предоставить приложениям, включая освобождаемую часть кеша; это не то же самое, что free. Значения согласуются с округлением и разным моментом измерения. Обозначение GB в интерфейсе Zabbix сохранено в таблицах как показано на скриншотах.

## 3. Диск и inode

Monitoring → Latest data → очистить Name → Apply → подфильтр filesystem: /.

| Метрика Zabbix | Значение |
|---|---|
| Space: Total | 24,44 GB |
| Space: Used | 4,08 GB |
| Space: Available | 19,09 GB |
| Space: Used, in % | 17,6025% |
| Inodes: Free, in % | 90,2348% |
| Option: Read-only | 0 |

Проверки:

```bash
df -h /
df -i /
```

Ubuntu: /dev/sda2, Size 25G, Used 4.1G, Avail 20G, Use 18%; inode total 1638400, used 159993, free 1478407, IUse 10%.

Значения сопоставимы с округлением. Used + Available не обязаны равняться Total из-за резерва ФС. Не следует рассчитывать занятость как простой Used / Total и выдавать её за формулу метрики. Inode проверены отдельно, поскольку свободное место не гарантирует возможность создавать новые файлы при исчерпании inode.

## 4. HTTP-сценарий

Data collection → Hosts → Zabbix server → Web → Create web scenario.

| Параметр | Значение |
|---|---|
| Name | Nginx HTTP availability |
| Update interval | 30s |
| Attempts | 1 |
| Agent (HTTP User-Agent) | Zabbix |
| Enabled | да |
| HTTP proxy, variables, headers | не заданы |

В Steps добавлен шаг:

| Параметр | Значение |
|---|---|
| Name | Nginx home page |
| URL | http://192.168.1.53/ |
| Timeout | 5s |
| Required status codes | 200 |
| Retrieve mode | Body |
| Follow redirects | выключено |
| Required string | пусто |

Оба уровня формы сохранены: сначала Add шага, затем Add сценария. В Monitoring → Web сценарий получил OK, Last check 10s. Это проверка HTTP со стороны Zabbix Server на той же VM; она не заменяет независимую внешнюю проверку из Windows.

## 5. Триггер

Data collection → Hosts → Zabbix server → Triggers → Create trigger.

Name: Nginx: HTTP check failed. Severity: Average.

```text
last(/Zabbix server/web.test.fail[Nginx HTTP availability])<>0
```

web.test.fail возвращает 0 при успехе и номер неуспешного шага при ошибке. Проверяется последнее значение. Одна ошибка открывает проблему; отдельное условие отсутствия данных не добавлялось.

Настройки:

- OK event generation: Expression.
- PROBLEM event generation mode: Single.
- OK event closes: All problems.
- Allow manual close: выключено.

После сохранения пользователь подтвердил исходное состояние OK. Триггер сообщает об ошибке HTTP-проверки; сам по себе он не доказывает остановку процесса nginx.

## 6. Контролируемый сбой Nginx

Перед опытом зафиксированы успешный HTTP-сценарий и OK триггера. Пользователь выполнил:

```bash
sudo systemctl stop nginx
systemctl is-active nginx
curl -I --connect-timeout 5 --max-time 10 http://192.168.1.53/
```

Фактический результат: inactive и curl (7), Failed to connect к порту 80.

Nginx обслуживает и тестовый сайт, и веб-интерфейс. Во время остановки интерфейс :8080 тоже недоступен, но Zabbix Server — отдельная служба и может продолжать HTTP-проверки. Целевая выдержка в инструкции — около 90 секунд; точная длительность остановки процесса по журналу не измерялась.

Восстановление:

```bash
sudo systemctl start nginx
systemctl is-active nginx
curl -I --connect-timeout 5 --max-time 10 http://192.168.1.53/
```

Получено active и HTTP/1.1 200 OK. Заголовок Date на скриншоте: 13 ноября 2026, 16:59:50 GMT.

После восстановления: Monitoring → Problems → Show: History → host Zabbix server → период Last 1 hour.

| Поле события | Подтверждённый результат |
|---|---|
| Имя | Nginx: HTTP check failed |
| Severity | Average |
| Начало | 19:59:01 Europe/Moscow |
| Recovery time | 20:00:02 Europe/Moscow |
| Duration | 1m 1s |
| Status | RESOLVED |

Время 19:59–20:00 показано веб-интерфейсом в UTC+3; терминальный HTTP-заголовок использует GMT/UTC. Длительность события 61 секунда определяется проверками Zabbix, а не равна гарантированно точному времени простоя.

Внешние сообщения не отправлялись: уведомления пропущены по решению пользователя. Запись и закрытие события — подтверждённый результат, доставка оповещений не проверена.

## 7. Перезагрузка VM

После проверки enable выполнено:

```bash
sudo reboot
```

После входа:

```bash
systemctl is-active zabbix-server zabbix-agent2 nginx php8.5-fpm
pg_lsclusters
timedatectl
```

Подтверждены четыре active, PostgreSQL 18/main online, правильная дата 2026-11-13, synchronized: yes, NTP active. Системное и RTC-время различались на одну секунду.

После загрузки в интерфейсе:

- HTTP-сценарий: OK, проверка 27 секунд назад.
- CPU utilization: 7,2308%, проверка 16 секунд назад.

Это подтверждает возобновление проверок после перезагрузки. RAM и диск проверены до reboot; повторная отдельная проверка их свежести после reboot не проводилась.

## Что проверено и что не проверено

Проверены запуск, базовые метрики, HTTP-код, обнаружение контролируемого сбоя и автоматическое восстановление, автозапуск и свежесть CPU/HTTP после reboot.

Не проверены внешние уведомления, мониторинг выключенной VM извне, HTTPS интерфейса, содержимое приложения, резервное восстановление базы, длительная или производственная нагрузка. Снимок Before-Zabbix создан, но откат к нему не тестировался.

[Описание web.test.fail и примеры триггеров](https://www.zabbix.com/documentation/7.0/en/manual/web_monitoring/items).
