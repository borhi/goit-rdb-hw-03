# goit-rdb-hw-03 — PostgreSQL: staging, DDL, data-quality audit та EDA

Домашнє завдання 3: імпорт NYC Taxi у PostgreSQL (`pgserver`), типізація через staging → clean, аудит якості даних, розподіли, outlier-check, 10 DQL/EDA-запитів і reflection.

## Джерело даних

- **Набір:** NYC TLC Yellow Taxi Trip Records, **січень 2024**
- **Офіційна сторінка:** https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page
- **Файл:** https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet
- **Словник даних:** https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf

## Вибірка

- Завантажується лише один місячний Parquet-файл (~50 МБ), не весь рік.
- Беруться 19 колонок: `VendorID`, `tpep_pickup_datetime`, `tpep_dropoff_datetime`, `passenger_count`, `trip_distance`, `RatecodeID`, `store_and_fwd_flag`, `PULocationID`, `DOLocationID`, `payment_type`, `fare_amount`, `extra`, `mta_tax`, `tip_amount`, `tolls_amount`, `improvement_surcharge`, `total_amount`, `congestion_surcharge`, `Airport_fee`.
- З місяця береться випадкова вибірка **100 000 рядків** (`df.sample(n=100_000, random_state=42)`), відсортована за часом посадки. Випадкова вибірка, на відміну від `head()`, рівномірно покриває всі дні й години місяця.
- Результат зберігається в `data/hw3_taxi_sample.csv` (≈ 10 МБ, у стисненому вигляді ≈ 3 МБ). Це менше за 100 МБ, тому файл лежить у репозиторії.

## Структура

```
goit-rdb-hw-03/
├── README.md
├── goit_rdb_hw_03.ipynb        # основний notebook
├── .gitignore
└── data/
    └── hw3_taxi_sample.csv   # вибірка, яку notebook завантажує через COPY FROM STDIN
```

Parquet-файл (`data/yellow_tripdata_2024-01.parquet`) у git не комітиться (див. `.gitignore`). Notebook завантажує його сам.

## Запуск

Потрібен Python 3.10+. PostgreSQL окремо встановлювати не потрібно: `pgserver` запускає вбудований сервер.

```bash
git clone <repo-url> goit-rdb-hw-03
cd goit-rdb-hw-03
python -m venv .venv && source .venv/bin/activate
pip install jupyter
jupyter notebook goit_rdb_hw_03.ipynb
```

Далі виконайте **Kernel → Restart & Run All**. Перша комірка сама встановить залежності (`pgserver`, `psycopg2-binary`, `sqlalchemy`, `pandas`, `pyarrow`).

Що відбувається під час запуску:

1. Стартує `pgserver` у `/tmp/hw3_pg`, виводиться версія PostgreSQL.
2. Якщо `data/yellow_tripdata_2024-01.parquet` відсутній, він завантажується з офіційного джерела.
3. Формується `data/hw3_taxi_sample.csv` (seed 42).
4. CSV імпортується через `COPY FROM STDIN` у `hw3_taxi_staging` (усі колонки мають тип `TEXT`).
5. `INSERT INTO ... SELECT` очищає дані та переносить їх у типізовану `hw3_taxi_trips`.
6. Виконуються аудит, EDA-запити та reflection.

Усі таблиці створюються через `DROP ... IF EXISTS`, тому notebook можна перезапускати скільки завгодно.

У Google Colab notebook теж працює: шляхи відносні, файли з'являться в `/content/data/`.

## Примітки

- SQLAlchemy підключається явно через `postgresql+psycopg2://`, бо `COPY FROM STDIN` використовує `copy_expert` з psycopg2, а нові версії SQLAlchemy за замовчуванням обирають psycopg3.
- Час у джерелі — локальний нью-йоркський без offset, тому в clean-шарі він локалізується через `AT TIME ZONE 'America/New_York'` у `TIMESTAMPTZ`.
