# ЛР 1. Loki + Zabbix + Grafana

**Задача:** подключить к Nextcloud логирование (Loki + Promtail) и мониторинг (Zabbix), вывести данные в Grafana.

**Файлы:** [`docker-compose.yml`](docker-compose.yml), [`promtail_config.yml`](promtail_config.yml), [`template.yml`](template.yml)

## Часть 1. Логирование

Запуск стека и проверка контейнеров:

```bash
docker compose up -d
docker ps
```

![compose up](screenshots/01_compose_up.png)

![docker ps](screenshots/02_docker_ps.png)

Nextcloud инициализирован на http://localhost:8080:

![Nextcloud](screenshots/03_nextcloud_install.png)

Логи пишутся в `/var/www/html/data/nextcloud.log`:

```bash
docker exec -it nextcloud bash -c "ls -la /var/www/html/data"
docker exec -it nextcloud bash -c "tail -n 5 /var/www/html/data/nextcloud.log"
```

![nextcloud.log](screenshots/04_nextcloud_log.png)

Promtail подхватил лог-файл (`Seeked /opt/nc_data/nextcloud.log`):

```bash
docker logs promtail 2>&1 | grep Seeked
```

![promtail](screenshots/05_promtail_seeked.png)

## Часть 2. Мониторинг

В Zabbix (http://localhost:8082, Admin / zabbix) через *Data collection → Templates → Import* импортирован шаблон `template.yml`:

![template](screenshots/07_zabbix_template.png)

Имя `nextcloud` добавлено в доверенные домены Nextcloud:

```bash
docker exec -u www-data -it nextcloud php occ config:system:set trusted_domains 1 --value="nextcloud"
```

![trusted_domains](screenshots/08_trusted_domains.png)

Создан хост `nextcloud` (группа Applications, шаблон Test ping template):

![host](screenshots/09_zabbix_host.png)

В *Monitoring → Latest data* получено значение `healthy`:

![latest data](screenshots/10_latest_data_healthy.png)

## Часть 3. Визуализация

Установлен и включён плагин Zabbix для Grafana:

```bash
docker exec -it grafana bash -c "grafana cli plugins install alexanderzobnin-zabbix-app"
docker restart grafana
```

![plugin install](screenshots/13_grafana_plugin_install.png)

![plugin enabled](screenshots/14_grafana_plugin_enabled.png)

Подключён источник данных Loki (`http://loki:3100`):

![loki ds](screenshots/15_loki_datasource.png)

Подключён источник данных Zabbix (`http://zabbix-front:8080/api_jsonrpc.php`):

![zabbix ds](screenshots/16_zabbix_datasource.png)

Логи Nextcloud в Explore (`job = nextcloud_logs`):

![explore loki query](screenshots/17_explore_loki_query.png)

![explore loki logs](screenshots/17_explore_loki_logs.png)

Данные Zabbix в Explore:

![explore zabbix](screenshots/18_explore_zabbix.png)

## Задание

### Запросы LogQL

Все логи:

```logql
{job="nextcloud_logs"}
```

![all](screenshots/19a_query_all.png)

Только неудачные входы:

```logql
{job="nextcloud_logs"} |= "Login failed"
```

![login failed](screenshots/19b_query_login_failed.png)

![login failed result](screenshots/19b_query_login_failed_result.png)

Разбор JSON, вывод времени и сообщения:

```logql
{job="nextcloud_logs"} | json | line_format "{{.time}} {{.message}}"
```

![json](screenshots/19c_query_json.png)

Количество строк лога за 5 минут:

```logql
count_over_time({job="nextcloud_logs"}[5m])
```

![count](screenshots/19d_count_over_time.png)

### Дашборды

Статус Nextcloud (Zabbix, Stat + value mappings):

![dashboard zabbix](screenshots/20_dashboard_zabbix.png)

Логи Nextcloud (Loki, Table):

![dashboard loki](screenshots/22_dashboard_loki.png)

## Ответы на вопросы

**1. Чем SLO отличается от SLA?**
SLA — договор с клиентом о гарантированном уровне сервиса со штрафами за нарушение. SLO — внутренняя целевая метрика команды (например, 99,9 % успешных запросов за месяц). SLO обычно строже SLA.

**2. Чем отличается инкрементальный бэкап от дифференциального?**
Инкрементальный копирует изменения с последнего бэкапа любого типа: копии маленькие, но для восстановления нужна вся цепочка. Дифференциальный копирует изменения с последнего полного бэкапа: копии растут, но для восстановления нужны только полный и последний дифференциальный.

**3. В чем разница между мониторингом и observability?**
Мониторинг отслеживает заранее заданные метрики и сообщает, что что-то сломалось. Observability позволяет по метрикам, логам и трассировкам понять, почему сломалось, в том числе при непредвиденных проблемах. Мониторинг — часть observability.

## Вывод

Логи Nextcloud собираются в Loki, доступность отслеживается Zabbix, данные выведены на дашборды Grafana.
