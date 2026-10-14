# ppad-orders-airflow-lab

Лаба по пайплайну заказов: генератор → Airflow → PostgreSQL (STG/DDS/CDM) → Metabase.

Ниже — запуск окружения (Docker Compose), доступы и обслуживание.

## Что нужно

- Git
- Python 3.9+
- Docker Desktop (Linux containers) или Docker Engine + Compose
- примерно 5 ГБ RAM под Docker и немного места под образы

## Быстрый старт

Клон:

```bash
git clone https://github.com/Malyshenko-Alexander/ppad-orders-airflow-lab.git
cd ppad-orders-airflow-lab
```

Linux / macOS:

```bash
bash scripts/start.sh
```

Windows (PowerShell):

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\start.ps1
```

Скрипт:

1. генерирует/проверяет JSON-снимки
2. создаёт `.env` и `private/passwords.json` (если их ещё нет)
3. поднимает postgres, airflow, metabase
4. ждёт healthcheck, гоняет DAG `orders_pipeline`
5. настраивает Metabase и прогоняет проверки

Первый раз качает образы — нужен интернет. На Windows/macOS сначала запусти Docker Desktop.

## Адреса и логины

| Сервис | URL / адрес | Логин | Пароль |
| --- | --- | --- | --- |
| Airflow | http://localhost:8080 | `admin` | `AIRFLOW_PASSWORD` в `.env` |
| Metabase | http://localhost:3333 | `admin@orders-lab.invalid` | `METABASE_ADMIN_PASSWORD` в `.env` |
| PostgreSQL | `127.0.0.1:15432`, БД `warehouse` | `warehouse` | `WAREHOUSE_PASSWORD` в `.env` |
| BI (только чтение CDM) | тот же Postgres | `bi_reader` | `BI_PASSWORD` в `.env` |

`.env` и `private/` в git не попадают. Шаблон переменных — `.env.example`, его с `replace` копировать не надо: пароли пишет `scripts/configure.py`.

В Metabase после старта: коллекция «Лабораторная — заказы», дашборд «Заказы интернет-магазина».

## Как устроено окружение

Три контейнера из `compose.yaml`:

- **postgres** — порт `15432`, лимит ~512 МБ  
  базы: `warehouse` (данные лабы), `airflow`, `metabase`
- **airflow** — порт `8080`, лимит ~3 ГБ, standalone
- **metabase** — порт `3333`, лимит ~1.5 ГБ

Внутри сети Compose хост БД — `postgres:5432`. С хоста машины — только `127.0.0.1:15432`.

Инициализация ролей/схем (`scripts/init-postgres.sh` + SQL) срабатывает только на **пустом** томе. Смена пароля в `.env` сама по себе пароль в уже созданной БД не меняет.

DAG `orders_pipeline` (файл `dags/orders_pipeline.py`):

`stg → dds → check_dds → cdm → check_cdm`

Расписание выключено, запуск руками. Повтор того же набора не должен плодить дубли.

## Полезные команды

Из корня репо:

```bash
docker compose ps
docker compose exec -T airflow airflow dags trigger orders_pipeline
docker compose exec -T airflow airflow dags list-runs orders_pipeline -o json
docker compose exec -T airflow airflow dags list-import-errors -o json
docker compose logs --tail 80 airflow
docker compose logs --tail 80 metabase
docker compose logs --tail 80 postgres
docker stats --no-stream
```

Проверки:

```bash
python -m unittest discover -s tests -v
docker compose exec -T airflow python /opt/lab/scripts/verify_warehouse.py
python scripts/verify_bi.py
```

Отчёты пишутся в `reports/`.

Остановка / старт с сохранением данных:

```bash
docker compose stop
docker compose start
```

Убрать контейнеры и сеть, тома оставить:

```bash
docker compose down
```

Не добавляй `-v`, если БД ещё нужна. Код, DAG и SQL монтируются с диска — после правок образ пересобирать не надо.

## Подключение к БД с хоста

- host: `127.0.0.1`
- port: `15432`
- database: `warehouse`
- user/password: `warehouse` / значение из `.env`

Админ Postgres внутри контейнера: `postgres` / `POSTGRES_PASSWORD`. Для обычной работы лабы он не обязателен.

## Другой объём данных

Стандарт: 100 покупателей, 30 товаров, 1000 заказов, 30 дней.

Большой прогон — только в **отдельной копии** проекта (другие тома / другое имя Compose), иначе конфликт портов и `dataset_id`:

```bash
COMPOSE_PROJECT_NAME=orders-lab-large bash scripts/start.sh --orders 10000 --customers 1000
```

PowerShell:

```powershell
$env:COMPOSE_PROJECT_NAME='orders-lab-large'
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\start.ps1 -Orders 10000 -Customers 1000
```

Не подменяй manifest в уже заполненном warehouse.

Только генерация JSON без Docker:

```bash
python -m orders_lab.generator --catalog data/reference/catalog.json
```

Путь к `manifest.json` потом можно передать в `python scripts/configure.py --manifest ...`.

## Если что-то не поднимается

| Симптом | Куда смотреть |
| --- | --- |
| Docker недоступен | `docker info`, запущен ли Desktop |
| Порт занят | 8080 / 3333 / 15432 — другой контейнер или сервис |
| Airflow не видит DAG | `dags list-import-errors`, логи airflow |
| DAG `failed` | лог упавшей task в UI Airflow |
| Metabase долго «молчит» | логи + память; первый старт бывает долгим |
| Новый пароль из `.env` не подходит к БД | том уже инициализирован старым паролем |

## Структура (коротко)

```
compose.yaml          # сервисы
dags/                 # Airflow DAG
orders_lab/           # генератор и загрузчики слоёв
sql/                  # DDL warehouse / CDM
scripts/              # start, configure, verify, metabase bootstrap
data/                 # каталог и эталонные снимки
tests/                # юнит-тесты
reports/              # результаты прогонов (после запуска)
```
