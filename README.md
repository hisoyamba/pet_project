
End-to-end ETL-пайплайн на стеке Apache Airflow + PostgreSQL + MinIO (S3) + Metabase, развёрнутый локально через Docker Compose. Проект демонстрирует подход Lakehouse: сырые данные из публичного API USGS приземляются в S3, обрабатываются через слои ODS/DM в PostgreSQL и визуализируются в Metabase.

📋 Содержание

Архитектура
Стек технологий
Структура репозитория
Слои данных
Быстрый старт
DAG'и
DDL и схемы
Подход Data Governance
Источник данных
Лицензия


🏗 Архитектура
┌──────────────┐     ┌─────────┐     ┌──────────────┐     ┌────────────┐     ┌──────────┐
│  USGS API    │────▶│ Airflow │────▶│ MinIO (S3)   │────▶│ PostgreSQL │────▶│ Metabase │
│ (earthquake) │     │  (ETL)  │     │  Data Lake   │     │  ODS / DM  │     │   (BI)   │
└──────────────┘     └─────────┘     └──────────────┘     └────────────┘     └──────────┘
                          │                                      ▲
                          │           DuckDB (compute)           │
                          └──────────────────────────────────────┘
Логика пайплайна:

Airflow по расписанию забирает CSV с данными о землетрясениях из API USGS.
Сырой файл складывается в S3-бакет (MinIO) — это «золотой» слой Data Lake.
DuckDB читает CSV из S3 и загружает данные в схему ods PostgreSQL.
На основе ODS формируются витрины в схеме dm (количество и средняя магнитуда по дням).
Metabase подключается к PostgreSQL и строит дашборды поверх витрин.



 Структура репозитория
.
├── dags/                   # DAG'и Airflow
│   ├── extract_earthquake_to_s3.py    # API → S3
│   ├── load_s3_to_postgres.py         # S3 → ODS (через DuckDB)
│   ├── build_dm_count_day.py          # ODS → DM (количество/день)
│   └── build_dm_avg_day.py            # ODS → DM (средняя магнитуда/день)
├── metabase/               # Конфигурация Metabase
├── docker-compose.yaml     # Инфраструктура: Airflow + Postgres + MinIO + Metabase
├── requirements.txt        # Python-зависимости
├── .gitignore
├── LICENSE
└── README.md

🗂 Слои данных
СхемаНазначениеИсточникТипизацияstgStagingПромежуточная зонаvarcharodsOperational Data Store — сырые данные «AS IS»S3 (CSV)varchar (минимально приводится)dmData Marts — агрегированные витрины для BIODSстрого типизировано
Выбор модели AS IS обусловлен природой данных: события землетрясений неизменяемы (immutable), поэтому исторические снимки не нужны и звезда/снежинка избыточны.



Установка
bash# 1. Клонировать репозиторий
git clone https://github.com/<your-username>/earthquake-dwh.git
cd earthquake-dwh

# 2. Создать виртуальное окружение
python3.12 -m venv venv && \
source venv/bin/activate && \
pip install --upgrade pip && \
pip install -r requirements.txt

# 3. Поднять инфраструктуру
docker-compose up -d
Доступы по умолчанию
СервисURLЛогин/ПарольAirflowhttp://localhost:8080airflow / airflowMinIOhttp://localhost:9001задаётся в .envMetabasehttp://localhost:3000настраивается при первом входеPostgreSQLlocalhost:5432задаётся в .env
Первоначальная настройка

MinIO: создать бакет, access key и secret key.
Airflow Variables: добавить ключи MinIO, имя бакета, параметры подключения.
Airflow Connections: создать подключение postgres_default к PostgreSQL.
PostgreSQL: выполнить DDL из раздела ниже для создания схем и таблиц.


🔄 DAG'и
DAGЧто делаетЗависимость отextract_earthquake_to_s3Тянет CSV из API USGS → MinIO—load_s3_to_postgresS3 → DuckDB → ods.fct_earthquakeextract_earthquake_to_s3build_dm_count_dayВитрина: количество землетрясений по днямload_s3_to_postgresbuild_dm_avg_dayВитрина: средняя магнитуда по днямload_s3_to_postgres
Все DAG'и идемпотентны — повторный запуск за ту же дату не приводит к дублям.

📐 DDL и схемы
Создание схем
sqlCREATE SCHEMA stg;
CREATE SCHEMA ods;
CREATE SCHEMA dm;
ODS — сырой слой
sqlCREATE TABLE ods.fct_earthquake (
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
DM — витрины
sql-- Количество землетрясений по дням
CREATE TABLE dm.fct_count_day_earthquake AS
SELECT
    time::date AS date,
    count(*)   AS earthquake_count
FROM ods.fct_earthquake
GROUP BY 1;

-- Средняя магнитуда по дням
CREATE TABLE dm.fct_avg_day_earthquake AS
SELECT
    time::date     AS date,
    avg(mag::float) AS avg_magnitude
FROM ods.fct_earthquake
GROUP BY 1;


</details>
