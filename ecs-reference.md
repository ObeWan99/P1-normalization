# Справка: поля ECS

ECS (Elastic Common Schema) — стандарт названий полей. Нормализация: поле сырого журнала → поле ECS **с тем же смыслом**.
Ниже — выдержка; нужны не все поля. Полный стандарт: https://www.elastic.co/docs/reference/ecs/ecs-field-reference

## source / destination / host

| Группа | Смысл |
|---|---|
| `source.*` | инициатор действия или соединения |
| `destination.*` | цель соединения, если это не машина, где записано событие |
| `host.*` | машина, на которой записано событие |

Выбор группы определяет смысл поля в журнале, а не его название.

## Поля

### Время и событие
| Поле | Смысл | Пример |
|---|---|---|
| `@timestamp` | время события, UTC | `2025-03-12T08:15:00.000Z` |
| `event.action` | что произошло | `login_failed` |
| `event.dataset` | журнал-источник | `auth`, `web`, `system` |
| `event.category` | категория | `authentication`, `process`, `network` |
| `event.outcome` | результат | `success`, `failure` |
| `log.level` | важность по оценке источника | `info`, `high` |

### Адреса и машины
| Поле | Смысл |
|---|---|
| `source.ip`, `source.port` | адрес и порт инициатора |
| `destination.ip`, `destination.port` | адрес и порт цели |
| `host.ip` | адрес машины, где записано событие |
| `host.name` | имя машины, где записано событие |
| `host.os.name` | ОС этой машины |

### Пользователь
| Поле | Смысл |
|---|---|
| `user.name` | учётная запись |
| `user.domain` | домен учётной записи |

### Веб
| Поле | Смысл | Пример |
|---|---|---|
| `url.original` | адрес запроса целиком | `/search?q=test` |
| `url.path` | путь без параметров | `/search` |
| `http.request.method` | метод | `GET`, `POST` |
| `http.request.body.content` | тело запроса | |
| `http.response.status_code` | код ответа | `200`, `404` |
| `http.response.bytes` | размер ответа, байт | `20480` |
| `user_agent.original` | строка User-Agent | `Mozilla/5.0 …` |

### Процессы и файлы
| Поле | Смысл | Пример |
|---|---|---|
| `process.name` | имя программы | `bash` |
| `process.command_line` | команда с аргументами | `bash -c id` |
| `process.pid` | номер процесса | `4321` |
| `file.path` | путь к файлу | `/tmp/a.sh` |

## Шаги конвейера (Ingest Pipelines)

| Шаг | Действие |
|---|---|
| Set | записать значение в поле (`event.dataset`) |
| Date | разобрать время в `@timestamp`, формат `ISO8601` |
| Rename | переименовать поле |
| Lowercase | привести значение к нижнему регистру |
| Remove | удалить поле |
