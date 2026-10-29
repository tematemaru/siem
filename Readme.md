# SIEM Stack (Loki 3.7 + Alloy + Grafana 12 + Nginx)

Минимальный локальный SIEM-стенд для тренировки LogQL, имитации инцидентов и подготовки шаблонов для ИИ-агента.

## Стек

| Компонент | Роль | Версия |
|---|---|---|
| Loki | Хранилище и индексация логов | 3.7.1 |
| Grafana Alloy | Сбор логов (замена Promtail) | latest |
| Grafana | Визуализация и Explore | 12.2.8 |
| Nginx | Единая точка входа (только loopback) | alpine |

Все сервисы работают в одной Docker-сети `siem-net`. Наружу торчит только Nginx на `127.0.0.1:80` — доступ к Grafana возможен только через SSH-туннель.

## Структура проекта

```
monitoring-stack/
├── .env
├── docker-compose.yaml
├── loki/config.yaml
├── alloy/config.alloy
├── nginx/conf.d/default.conf
└── grafana/provisioning/datasources/loki.yaml
```

## Требования

- Docker Engine 24+ и Docker Compose v2
- Linux-хост (Debian/Ubuntu или RHEL/CentOS)
- Свободный порт `127.0.0.1:80`
- Доступ к Docker Hub для скачивания образов

## Быстрый старт

```bash
git clone <your-repo-url> monitoring-stack
cd monitoring-stack

docker compose down
docker compose up -d
docker compose ps
curl -s http://127.0.0.1/ | head -c 200
```

Ожидаемый результат:

- `docker compose ps` — все сервисы в статусе `Up` (кроме `loki-init`, он должен быть `Exited (0)`)
- `curl` возвращает HTML-страницу входа Grafana (или редирект `302`)

## Доступ к Grafana через SSH-туннель

С **локальной машины** (не с виртуалки):

```bash
ssh -N -L 3000:127.0.0.1:80 siem@<IP-виртуалки>
```

Открыть в браузере: `http://127.0.0.1:3000`

Логин: `admin`  
Пароль: `qazwsx` (из `.env`, смени при необходимости)

Остановить туннель: `pkill -f "ssh -N -L 3000"`

## Конфигурация

### `.env`

```env
GF_SECURITY_ADMIN_PASSWORD=qazwsx
GF_SERVER_ROOT_URL=http://127.0.0.1/
LOKI_RETENTION_DAYS=7
```

- `GF_SECURITY_ADMIN_PASSWORD` — пароль администратора Grafana
- `GF_SERVER_ROOT_URL` — публичный URL Grafana (для генерации ссылок в UI). При доступе через SSH-туннель — `http://127.0.0.1/`
- `LOKI_RETENTION_DAYS` — срок хранения логов (подставляется в `loki/config.yaml` при генерации)

### `docker-compose.yaml`

Ключевые моменты:

- `loki-init` — одноразовый контейнер на `busybox`, создаёт каталоги и выставляет владельца `10001:10001` (Loki внутри работает от этого UID)
- `loki` — запускается от `10001:10001`, монтирует `./loki/data` и `./loki/wal`
- `alloy` — монтирует `/var/log` и `/var/run/docker.sock` для сбора системных и контейнерных логов
- `nginx` — слушает только `127.0.0.1:80`, проксирует Grafana на `/`
- У `grafana` и `loki` **нет** секции `ports` — они доступны только внутри сети `siem-net`

### `loki/config.yaml`

- Схема `v13` + `tsdb` (актуально для Loki 3.x)
- `replication_factor: 1` — для одиночного инстанса
- `retention_enabled: true` + `delete_request_store: filesystem` — удаление старых чанков через compactor
- `allow_structured_metadata: true` — поддержка structured metadata

### `alloy/config.alloy`

Три источника логов, все пишут в `loki.write.default`:

1. **Docker-логи** (`discovery.docker` + `loki.source.docker`) — метки `application="docker"`, `source="docker"`, `container_name`
2. **`/var/log/auth.log`** — метки `source="auth"`, `job="authlog"`
3. **Системные логи** (`syslog`, `kern.log`, `dpkg.log`) — метки `application="system"`, `job="varlogs"`

`auth.log` намеренно исключён из третьего потока, чтобы не было дублей. На RHEL/CentOS замени путь на `/var/log/secure`.

### `nginx/conf.d/default.conf`

Проксирует Grafana на `/`. Loki API наружу не отдаётся — он нужен только Alloy внутри сети.

## Проверочные запросы в Grafana Explore

| Запрос | Что показывает |
|---|---|
| `{source="auth"} \|= "Failed password"` | Неудачные SSH-входы |
| `{source="auth"} \|= "Invalid user"` | Попытки входа с несуществующими пользователями |
| `{application="system"}` | syslog, kern.log, dpkg.log |
| `{source="docker"}` | Логи всех Docker-контейнеров |

Сгенерировать событие для проверки:

```bash
ssh wronguser@localhost
# введи любой пароль
```

Затем в Grafana: `{source="auth"} |= "wronguser"`.

## Диагностика

### Loki не стартует

```bash
docker compose logs loki --tail 100
```

Типичные причины:

- `permission denied` на `/loki/chunks` — проверь, что `loki-init` завершился успешно (`docker compose ps loki-init` → `Exited (0)`)
- `mkdir : no such file or directory` — та же причина, каталоги не созданы
- `too many unhealthy instances in the ring` — подожди 10–15 секунд после старта, ring стабилизируется. Если повторяется — проверь `curl http://127.0.0.1:3100/ring` (изнутри виртуалки)

### Alloy не отправляет логи

```bash
docker compose logs alloy --tail 100
```

Проверь синтаксис `config.alloy`: Alloy — строгий парсер River, одна опечатка валит процесс.

### Grafana не открывается через туннель

```bash
# на виртуалке
curl -I http://127.0.0.1/
ss -tlnp | grep ':80'
```

Если Nginx не слушает — проверь `docker compose ps nginx` и логи.

## Остановка и очистка

```bash
# остановить, сохранить данные
docker compose down

# остановить и удалить данные Loki
docker compose down -v
rm -rf loki/data loki/wal
```

## Расширение стенда

### Добавить Juice Shop

Добавь в `docker-compose.yaml`:

```yaml
  juice-shop:
    image: bkimminich/juice-shop:latest
    container_name: juice-shop
    restart: unless-stopped
    networks:
      - siem-net
```

Alloy подхватит его логи автоматически — запрос `{container_name="juice-shop"}` заработает без правок конфига. Если нужен доступ из браузера — добавь `location /juice-shop/` в `nginx/conf.d/default.conf` с `proxy_pass http://juice-shop:3000/;`.

### Добавить удалённую виртуалку с логами

На удалённой машине поставь Alloy и настрой `loki.write` на `http://<IP-виртуалки-со-стендом>:3100/loki/api/v1/push`. Порт `3100` на стенде должен быть доступен изнутри сети (сейчас он не проброшен наружу — придётся либо пробросить на loopback и ходить через SSH-туннель, либо открыть в security group).

## Лицензия

Учебный проект. Используй на свой страх и риск, не выставляй в публичный интернет без смены пароля и настройки TLS.