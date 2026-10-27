# Справка: развёртывание Elasticsearch + Kibana

Нужно: Linux (или macOS, Windows + WSL 2), Docker с Compose, `git`, `curl`, `openssl`, 6 ГБ свободной памяти, 8 ГБ на диске.
Все команды — в папке репозитория.
Разбор каждой команды по частям — в **[P1-start.pdf](P1-start.pdf)**.

| Компонент | Роль | Адрес |
|---|---|---|
| Elasticsearch 9.4.8 | хранение и поиск событий | `localhost:9200` |
| Kibana 9.4.8 | веб-интерфейс | http://localhost:5601 |

## 1. Пароли

```bash
git clone https://github.com/ObeWan99/P1-normalization.git
cd P1-normalization
cp .env.example .env
openssl rand -hex 16        # значение для ENCRYPTION_KEY
nano .env                   # заполнить ELASTIC_PASSWORD, KIBANA_PASSWORD (не короче 6 символов), ENCRYPTION_KEY
source .env                 # повторять в каждом новом терминале
```

## 2. Elasticsearch

```bash
docker compose up -d elasticsearch
docker compose ps                                   # ждать (healthy), 1–2 мин
curl -u "elastic:$ELASTIC_PASSWORD" localhost:9200  # ✔ "number" : "9.4.8"
```

## 3. Пароль учётки kibana_system

Kibana подключается к Elasticsearch под `kibana_system`, а не под `elastic`.

```bash
curl -u "elastic:$ELASTIC_PASSWORD" -H 'Content-Type: application/json' \
  -X POST localhost:9200/_security/user/kibana_system/_password \
  -d "{\"password\":\"$KIBANA_PASSWORD\"}"
```
✔ `{}`

## 4. Kibana

```bash
docker compose up -d
```
Через 1–2 мин: http://localhost:5601, логин `elastic`, пароль `ELASTIC_PASSWORD`.

## 5. Шаблоны индексов (типы полей)

```bash
curl -u "elastic:$ELASTIC_PASSWORD" -H 'Content-Type: application/json' \
  -X PUT localhost:9200/_index_template/raw -d @setup/raw-template.json
curl -u "elastic:$ELASTIC_PASSWORD" -H 'Content-Type: application/json' \
  -X PUT localhost:9200/_index_template/norm -d @setup/norm-template.json
```
✔ `{"acknowledged":true}` дважды

## 6. Журналы (Bulk API)

```bash
for j in auth web system; do
  curl -s -u "elastic:$ELASTIC_PASSWORD" -H 'Content-Type: application/x-ndjson' \
    -X POST "localhost:9200/raw-$j/_bulk?refresh=true" --data-binary @data/raw-$j.ndjson \
    | grep -o '"errors":[a-z]*'
done
curl -u "elastic:$ELASTIC_PASSWORD" \
  "localhost:9200/_cat/indices/raw-*?v&h=index,docs.count&s=index"
```
✔ `"errors":false` трижды; `raw-auth` 20000, `raw-system` 15000, `raw-web` 15000

## 7. Образец нормализации: веб-журнал

```bash
curl -u "elastic:$ELASTIC_PASSWORD" -H 'Content-Type: application/json' \
  -X PUT localhost:9200/_ingest/pipeline/norm-web -d @pipelines/norm-web.json
curl -u "elastic:$ELASTIC_PASSWORD" -H 'Content-Type: application/json' \
  -X POST "localhost:9200/_reindex?refresh=true" \
  -d '{"source":{"index":"raw-web"},"dest":{"index":"norm-web","pipeline":"norm-web"}}'
```
✔ `{"acknowledged":true}`; затем `"created":15000`, `"failures":[]`

## 8. Kibana: представления данных

Часовой пояс (`UTC`) и период по умолчанию (10–16 марта 2025) Kibana берёт из файла `kibana/kibana.yml` при запуске — настраивать их не нужно.

Разделы ищутся через строку поиска вверху Kibana. **Data Views** → Create data view → Save data view to Kibana, пять раз:

| Name | Index pattern | Timestamp field |
|---|---|---|
| `Сырые журналы (raw-*)` | `raw-*` | `--- I don't want to use the time filter ---` |
| `Журнал входов (raw-auth)` | `raw-auth` | то же |
| `Журнал веб-портала (raw-web)` | `raw-web` | то же |
| `Журнал серверов (raw-system)` | `raw-system` | то же |
| `Нормализованные журналы (norm-*)` | `norm-*` | `@timestamp` |

## ✔ Проверка

Discover → список **Data view** слева вверху (по умолчанию там пустое **All logs**) → «Сырые журналы (raw-\*)» — **50,000** событий; «Нормализованные журналы (norm-\*)» — **15,000**.

## Ошибки

| Симптом | Решение |
|---|---|
| `permission denied ... docker.sock` | `sudo usermod -aG docker $USER`, перезайти |
| `port is already allocated` | порт 9200/5601 занят: `docker ps`, `docker stop <имя>` |
| `curl: (56) Recv failure: Connection reset by peer` | Elasticsearch ещё запускается: ждать `(healthy)` в `docker compose ps`, повторить |
| `passwords must be at least [6] characters long` | в `.env` удлинить `KIBANA_PASSWORD`, `source .env`, повторить шаг 3 (`ELASTIC_PASSWORD` не менять) |
| `401` / `unable to authenticate` | `source .env` в этом терминале; если пароль меняли после запуска — начать заново |
| HTML прокси в ответе `curl` | `unset http_proxy https_proxy` или `curl --noproxy '*'` |
| Elasticsearch не `healthy` | `docker compose logs elasticsearch --tail 30`; память `free -h` |
| Kibana «not ready yet» > 5 мин | повторить шаг 3, `docker compose restart kibana` |
| `"errors":true` при загрузке | нет шаблона (шаг 5) или повторная загрузка — начать заново |
| Discover: 0 событий | период справа вверху: 10–16 марта 2025 |

## Управление

| Команда | Действие |
|---|---|
| `docker compose stop` / `start` | остановить / запустить (данные сохраняются) |
| `docker compose down -v` | удалить всё вместе с данными; затем с шага 2 |
