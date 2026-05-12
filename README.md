# Earthquake DWH

End-to-end ETL-пайплайн на стеке **Apache Airflow + PostgreSQL + MinIO (S3) + Metabase**, развёрнутый локально через Docker Compose. Проект демонстрирует подход **Lakehouse**: сырые данные из публичного API USGS приземляются в S3, обрабатываются через слои ODS/DM в PostgreSQL и визуализируются в Metabase.

## Содержание

- [Архитектура](#архитектура)
- [Структура репозитория](#структура-репозитория)
- [Слои данных](#слои-данных)
- [Установка](#установка)
- [Доступы по умолчанию](#доступы-по-умолчанию)
- [Первоначальная настройка](#первоначальная-настройка)
- [DAG'и](#dagи)
- [DDL и схемы](#ddl-и-схемы)

## Архитектура

```
┌──────────────┐     ┌─────────┐     ┌──────────────┐     ┌────────────┐     ┌──────────┐
│  USGS API    │────▶│ Airflow │────▶│ MinIO (S3)   │────▶│ PostgreSQL │────▶│ Metabase │
│ (earthquake) │     │  (ETL)  │     │  Data Lake   │     │  ODS / DM  │     │   (BI)   │
└──────────────┘     └─────────┘     └──────────────┘     └────────────┘     └──────────┘
                          │                                      ▲
                          │           DuckDB (compute)           │
                          └──────────────────────────────────────┘
```


## Структура репозитория

```
.
├── dags/                                   # DAG'и Airflow
│   ├── extract_earthquake_to_s3.py         # API → S3
│   ├── load_s3_to_postgres.py              # S3 → ODS (через DuckDB)
│   ├── build_dm_count_day.py               # ODS → DM (количество/день)
│   └── build_dm_avg_day.py                 # ODS → DM (средняя магнитуда/день)
├── metabase/                               # Конфигурация Metabase
├── docker-compose.yaml                     # Инфраструктура: Airflow + Postgres + MinIO + Metabase
├── requirements.txt                        # Python-зависимости
├── .gitignore
├── LICENSE
└── README.md
```

## Слои данных

| Схема | Назначение                                    | Источник           | Типизация                       |
|-------|-----------------------------------------------|--------------------|---------------------------------|
| `stg` | Staging                                       | Промежуточная зона | varchar                         |
| `ods` | Operational Data Store — сырые данные «AS IS» | S3 (CSV)           | varchar (минимально приводится) |
| `dm`  | Data Marts — агрегированные витрины для BI    | ODS                | строго типизировано             |


## Установка

```bash
# 1. Клонировать репозиторий
git clone https://github.com/<your-username>/earthquake-dwh.git
cd earthquake-dwh

# 2. Создать виртуальное окружение
python3.12 -m venv venv && \
source venv/bin/activate && \
pip install --upgrade pip && \
pip install -r requirements.txt

# 3. Поднять инфраструктуру
docker-compose up -d
```

## Доступы по умолчанию

| Сервис     | URL                   | Логин/Пароль                   |
|------------|-----------------------|--------------------------------|
| Airflow    | http://localhost:8080 | `airflow` / `airflow`          |
| MinIO      | http://localhost:9001 | задаётся в `.env`              |
| Metabase   | http://localhost:3000 | настраивается при первом входе |
| PostgreSQL | `localhost:5432`      | задаётся в `.env`              |

## Первоначальная настройка

1. **MinIO**: создать бакет, access key и secret key.
2. **Airflow Variables**: добавить ключи MinIO, имя бакета, параметры подключения.
3. **Airflow Connections**: создать подключение `postgres_default` к PostgreSQL.
4. **PostgreSQL**: выполнить DDL из раздела ниже для создания схем и таблиц.

## DAG'и

| DAG                          | Что делает                                | Зависимость от             |
|------------------------------|-------------------------------------------|----------------------------|
| `extract_earthquake_to_s3`   | Тянет CSV из API USGS → MinIO             | —                          |
| `load_s3_to_postgres`        | S3 → DuckDB → `ods.fct_earthquake`        | `extract_earthquake_to_s3` |
| `build_dm_count_day`         | Витрина: количество землетрясений по дням | `load_s3_to_postgres`      |
| `build_dm_avg_day`           | Витрина: средняя магнитуда по дням        | `load_s3_to_postgres`      |

Все DAG'и **идемпотентны** — повторный запуск за ту же дату не приводит к дублям.

## DDL и схемы

Создание схем:

```sql
CREATE SCHEMA stg;
CREATE SCHEMA ods;
CREATE SCHEMA dm;
```

ODS — сырой слой:

```sql
CREATE TABLE ods.fct_earthquake (
    time             varchar,
    latitude         varchar,
    longitude        varchar,
    depth            varchar,
    mag              varchar,
    mag_type         varchar,
    nst              varchar,
    gap              varchar,
    dmin             varchar,
    rms              varchar,
    net              varchar,
    id               varchar,
    updated          varchar,
    place            varchar,
    type             varchar,
    horizontal_error varchar,
    depth_error      varchar,
    mag_error        varchar,
    mag_nst          varchar,
    status           varchar,
    location_source  varchar,
    mag_source       varchar
);
```

DM — витрины:

```sql
-- Количество землетрясений по дням
CREATE TABLE dm.fct_count_day_earthquake AS
SELECT
    time::date AS date,
    count(*)   AS earthquake_count
FROM ods.fct_earthquake
GROUP BY 1;

-- Средняя магнитуда по дням
CREATE TABLE dm.fct_avg_day_earthquake AS
SELECT
    time::date      AS date,
    avg(mag::float) AS avg_magnitude
FROM ods.fct_earthquake
GROUP BY 1;
```
