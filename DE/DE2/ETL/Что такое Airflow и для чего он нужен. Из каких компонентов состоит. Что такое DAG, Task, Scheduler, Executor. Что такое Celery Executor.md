#### 13.1. Что такое Apache Airflow и для чего он нужен

**Apache Airflow** — это открытая платформа для **оркестрации рабочих процессов** (workflow orchestration), разработанная в Airbnb в 2014 году и переданная в Apache Software Foundation (top-level project с 2019 г.).

Основное назначение Airflow:

| Функция | Описание |
|---|---|
| **Оркестрация** | Управление порядком выполнения задач (зависимости, ветвление, параллелизм) |
| **Планирование (Scheduling)** | Автоматический запуск DAG-ов по расписанию (cron, timedelta, timetable) |
| **Мониторинг** | Web-интерфейс со статусами всех задач, Gantt-диаграммами, логами |
| **Retry / Alerting** | Автоматический перезапуск упавших задач, уведомления через e-mail, Slack, PagerDuty и т. д. |
| **Backfill** | Возможность перезапустить пайплайн за прошлые периоды |
| **Идемпотентность** | Архитектура поощряет написание задач, которые безопасно перезапускать |
| **Extensibility** | Огромная экосистема провайдеров (AWS, GCP, Azure, Snowflake, dbt, Spark …) |

Airflow **НЕ является** инструментом для потоковой обработки данных (streaming) — он предназначен для **batch-пайплайнов**. Для streaming используют Kafka Streams, Flink, Spark Structured Streaming.

Типичные сценарии использования:
- ETL/ELT пайплайны: извлечение из источников → трансформация → загрузка в DWH.
- ML-пайплайны: обучение моделей, валидация, деплой.
- Оркестрация dbt-моделей.
- Data quality checks.
- Периодическая генерация отчётов.

---

#### 13.2. Компоненты Airflow

Airflow состоит из следующих ключевых компонентов:

```
┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│  Web Server  │────▶│ Metadata DB │◀────│  Scheduler   │
│  (Flask/     │     │ (PostgreSQL/│     │              │
│   Gunicorn)  │     │  MySQL)     │     │              │
└──────────────┘     └──────┬──────┘     └──────┬───────┘
                            │                   │
                            │            ┌──────▼───────┐
                            │            │   Executor   │
                            │            │              │
                            │            └──────┬───────┘
                            │                   │
                            │         ┌─────────┼─────────┐
                            │         ▼         ▼         ▼
                            │    ┌────────┐┌────────┐┌────────┐
                            └───▶│Worker 1││Worker 2││Worker N│
                                 └────────┘└────────┘└────────┘
                                          ┌──────────┐
                                          │ Triggerer │
                                          └──────────┘
```

**1. Web Server (Веб-сервер)**
- Построен на Flask + Gunicorn.
- Предоставляет UI для мониторинга DAG-ов, просмотра логов, ручного запуска, управления переменными и подключениями.
- REST API для программного управления (с Airflow 2.0 — стабильный REST API).
- Аутентификация через RBAC (Flask-AppBuilder).

**2. Scheduler (Планировщик)**
- Центральный «мозг» Airflow.
- Периодически сканирует папку `dags_folder`, парсит Python-файлы, строит объекты DAG.
- Создаёт `DagRun` в соответствии с расписанием.
- Определяет, какие `TaskInstance` готовы к выполнению (все upstream-зависимости выполнены).
- Отправляет готовые задачи в Executor.
- Параметр `min_file_process_interval` (по умолчанию 30 сек.) — минимальный интервал между повторными парсингами одного и того же DAG-файла.
- С Airflow 2.0 Scheduler может работать в режиме HA (несколько экземпляров).

**3. Executor (Исполнитель)**
- Определяет **где и как** выполняются задачи.
- Executor не выполняет задачи сам — он делегирует выполнение воркерам или процессам.
- Подробнее о типах — в разделе 13.6.

**4. Metadata Database (База метаданных)**
- Хранит **всё** состояние Airflow: определения DAG, историю DagRun, статусы TaskInstance, переменные (Variables), подключения (Connections), пулы (Pools), XCom-данные, информацию о пользователях.
- Поддерживаемые СУБД: **PostgreSQL** (рекомендуется для production), **MySQL**, SQLite (только для разработки/тестирования).
- Миграции через Alembic (`airflow db migrate`).

**5. Worker (Воркер)**
- Процесс, который **фактически выполняет** задачу.
- В случае CeleryExecutor — это отдельный процесс `airflow celery worker`, слушающий очередь брокера.
- В случае LocalExecutor — дочерний процесс Scheduler-а.
- В случае KubernetesExecutor — Pod в Kubernetes.

**6. Triggerer (Триггерер)** *(с Airflow 2.2)*
- Отвечает за **deferrable operators** — операторы, которые освобождают слот воркера, пока ждут внешнего события.
- Использует asyncio для эффективного ожидания тысяч внешних условий в одном процессе.
- Заменяет неэффективный «poke» в сенсорах, при котором воркер блокируется.

---

#### 13.3. DAG (Directed Acyclic Graph)

**DAG** — направленный ациклический граф.

Математически:
- **Граф** — множество вершин (узлов) V и рёбер E ⊆ V × V.
- **Направленный** — каждое ребро имеет направление: (u, v) означает «от u к v».
- **Ациклический** — в графе нет циклов, т. е. невозможно, стартуя из вершины v, по направленным рёбрам вернуться обратно в v.

В контексте Airflow:
- **Вершины** = задачи (Tasks).
- **Рёбра** = зависимости между задачами.
- **Ацикличность** гарантирует, что пайплайн можно выполнить в определённом порядке (топологическая сортировка) и он когда-нибудь завершится.

Определение DAG в Python:

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data_team',
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email': ['alerts@company.com'],
}

with DAG(
    dag_id='etl_sales_pipeline',
    default_args=default_args,
    description='Ежедневная загрузка данных о продажах',
    schedule_interval='0 3 * * *',    # каждый день в 03:00 UTC
    start_date=datetime(2024, 1, 1),
    end_date=None,                     # без ограничения (работает бессрочно)
    catchup=False,                     # не запускать за пропущенные периоды
    tags=['etl', 'sales'],
    max_active_runs=1,                 # не более одного активного DagRun
) as dag:

    extract = PythonOperator(
        task_id='extract_data',
        python_callable=extract_function,
    )

    transform = PythonOperator(
        task_id='transform_data',
        python_callable=transform_function,
    )

    load = PythonOperator(
        task_id='load_to_dwh',
        python_callable=load_function,
    )

    extract >> transform >> load
```

Ключевые параметры DAG:

| Параметр | Описание |
|---|---|
| `dag_id` | Уникальный идентификатор DAG (строка) |
| `schedule_interval` | Расписание: cron-выражение (`'0 3 * * *'`), `timedelta(hours=1)`, пресеты (`'@daily'`, `'@hourly'`), `None` — ручной запуск |
| `timetable` | (Airflow 2.2+) Более гибкая замена `schedule_interval`, позволяет задавать нерегулярные расписания (напр., рабочие дни) |
| `start_date` | Дата начала: первый `execution_date` будет равен `start_date`, первый фактический запуск — `start_date + schedule_interval` |
| `end_date` | Дата окончания, после которой DAG перестаёт планироваться |
| `catchup` | `True` — Scheduler создаст DagRun за каждый пропущенный интервал от `start_date` до now. `False` — создаёт только последний |
| `max_active_runs` | Максимальное количество одновременно выполняющихся DagRun |
| `concurrency` | Максимальное количество одновременно выполняющихся задач внутри одного DagRun |
| `default_args` | Словарь параметров по умолчанию для всех задач в DAG |

---

#### 13.4. Task (Задача)

**Task** — это единица работы внутри DAG. Каждая задача представляет собой экземпляр **оператора** (Operator).

Ключевые свойства задачи:

| Свойство | Описание |
|---|---|
| `task_id` | Уникальный (в пределах DAG) идентификатор задачи |
| `operator` | Класс оператора: `PythonOperator`, `BashOperator`, `PostgresOperator` и т. д. |
| `dependencies` | Зависимости от других задач (upstream/downstream) |
| `trigger_rule` | Правило запуска задачи в зависимости от статуса upstream-задач |
| `retries` | Количество повторных попыток при ошибке |
| `retry_delay` | Задержка между повторными попытками |
| `pool` | Пул ресурсов (ограничивает параллелизм однотипных задач) |
| `queue` | Имя очереди (при CeleryExecutor — определяет, какой воркер возьмёт задачу) |
| `execution_timeout` | Максимальное время выполнения задачи (после — `AirflowTaskTimeout`) |
| `priority_weight` | Приоритет задачи в очереди (выше число = выше приоритет) |
| `sla` | SLA — если задача не выполнена в течение указанного времени с начала DagRun, отправляется уведомление |

**Определение зависимостей между задачами:**

```python
# Способ 1: оператор >> (bitshift) — самый распространённый
extract >> transform >> load

# Способ 2: оператор <<
load << transform << extract

# Способ 3: методы set_upstream / set_downstream
transform.set_upstream(extract)
load.set_upstream(transform)

# Способ 4: chain (для длинных цепочек)
from airflow.models.baseoperator import chain
chain(extract, transform, validate, load)

# Параллельные зависимости
extract >> [transform_a, transform_b] >> load

# cross_downstream — каждая задача из первого списка → каждая из второго
from airflow.models.baseoperator import cross_downstream
cross_downstream([extract_a, extract_b], [transform_x, transform_y])
```

**Trigger Rules** — правила запуска задачи:

| Правило | Описание |
|---|---|
| `all_success` | (по умолчанию) Все upstream задачи завершились успешно |
| `all_failed` | Все upstream задачи упали |
| `all_done` | Все upstream задачи завершились (с любым статусом) |
| `one_success` | Хотя бы одна upstream задача завершилась успешно |
| `one_failed` | Хотя бы одна upstream задача упала |
| `none_failed` | Ни одна upstream задача не упала (success или skip) |
| `none_skipped` | Ни одна upstream задача не была пропущена |
| `none_failed_min_one_success` | Ни одна не упала и хотя бы одна успешна |
| `always` | Задача запускается всегда, независимо от upstream |

---

#### 13.5. Scheduler (Планировщик) — глубже

Алгоритм работы Scheduler-а (упрощённо):

1. **Парсинг DAG-файлов**: Scheduler периодически обходит `dags_folder` и парсит каждый `.py` файл. Результат — объект `DAG` с метаинформацией. Параметр `min_file_process_interval` определяет минимальный интервал перепарсинга файла (по умолчанию 30 сек.).
2. **Создание DagRun**: Для каждого DAG Scheduler проверяет, пора ли создать новый `DagRun` в соответствии с `schedule_interval`. Если пора — создаёт запись в Metadata DB со статусом `running`.
3. **Планирование TaskInstance**: Для каждого активного `DagRun` Scheduler определяет, какие задачи готовы к выполнению:
   - Все upstream-зависимости выполнены.
   - `trigger_rule` удовлетворён.
   - Есть свободные слоты в пуле.
   - Не превышен `max_active_runs` / `concurrency`.
4. **Передача в Executor**: Готовые `TaskInstance` передаются в Executor для выполнения.
5. **Обновление статусов**: Scheduler получает от Executor обратную связь и обновляет статусы TaskInstance в Metadata DB.
6. **Heartbeat**: Scheduler периодически шлёт heartbeat в БД, чтобы другие компоненты знали, что он жив.

Ключевые настройки Scheduler-а:

```ini
[scheduler]
min_file_process_interval = 30     # мин. интервал перепарсинга файла (сек.)
dag_dir_list_interval = 300        # интервал сканирования dags_folder на новые файлы
parsing_processes = 2              # кол-во процессов-парсеров (DAG processor)
max_dagruns_to_create_per_loop = 10
scheduler_heartbeat_sec = 5
```

---

#### 13.6. Типы Executor-ов

| Executor | Описание | Когда использовать |
|---|---|---|
| **SequentialExecutor** | Выполняет задачи **последовательно**, одну за другой, в процессе Scheduler-а. Использует SQLite. | Только для разработки и тестирования. Нельзя использовать в production |
| **LocalExecutor** | Запускает каждую задачу в **отдельном процессе** на той же машине, где работает Scheduler. Поддерживает параллелизм (до `parallelism` задач). | Небольшие production-инсталляции, один сервер, до ~32 параллельных задач |
| **CeleryExecutor** | Использует **Celery** (распределённую очередь задач) для отправки задач на удалённые **воркеры** через брокер сообщений. | Production: горизонтальное масштабирование, много задач, несколько серверов |
| **KubernetesExecutor** | Запускает каждую задачу в **отдельном Pod** в Kubernetes. Pod создаётся per-task и уничтожается после завершения. | Cloud-native среды, изоляция задач (каждая в своём контейнере с ресурсными лимитами), auto-scaling |
| **CeleryKubernetesExecutor** | Гибридный: часть задач — через Celery (быстрый запуск для лёгких задач), часть — через Kubernetes (для тяжёлых задач, требующих изоляции). Задача отправляется в K8s, если указана `queue='kubernetes'`. | Гибридные сценарии: много мелких задач + редкие тяжёлые |
| **DaskExecutor** | Использует Dask distributed для распределённого выполнения. | Специфичные сценарии с Dask |

---

#### 13.7. Celery Executor — подробный разбор

**Celery** — распределённая система очередей задач для Python. В связке с Airflow она обеспечивает **горизонтальное масштабирование**: можно добавлять воркеры на новых серверах без изменения кода DAG-ов.

Архитектура CeleryExecutor:

```
┌────────────┐     ┌─────────────────┐     ┌──────────────┐
│  Scheduler │────▶│  Message Broker  │────▶│   Worker 1   │
│            │     │  (Redis /       │     │(celery worker)│
│            │     │   RabbitMQ)     │     └──────────────┘
│            │     │                 │────▶┌──────────────┐
│            │     │                 │     │   Worker 2   │
│            │     └────────┬────────┘     └──────────────┘
└────────────┘              │              ┌──────────────┐
                            │         ────▶│   Worker N   │
                            │              └──────────────┘
                            ▼
                    ┌───────────────┐     ┌──────────────┐
                    │ Result Backend│     │   Flower     │
                    │ (Redis/DB)    │     │ (мониторинг) │
                    └───────────────┘     └──────────────┘
```

**Компоненты CeleryExecutor:**

1. **Broker (Брокер сообщений)**
   - Очередь, через которую Scheduler передаёт задачи воркерам.
   - **Redis** — чаще всего используется: быстрый, простой в настройке, но данные в памяти (персистенция через RDB/AOF).
   - **RabbitMQ** — более надёжный, поддерживает AMQP, сложные routing-правила, но тяжелее в эксплуатации.
   - Конфигурация: `broker_url = 'redis://redis:6379/0'`

2. **Workers (Воркеры)**
   - Процессы `airflow celery worker`, запущенные на одной или нескольких машинах.
   - Каждый воркер подписывается на одну или несколько очередей брокера.
   - Можно запустить с указанием очереди: `airflow celery worker -q default,high_priority`
   - Воркер забирает задачу из очереди, выполняет её, обновляет статус в Metadata DB.
   - Параметр `worker_concurrency` определяет количество одновременных задач на одном воркере.

3. **Result Backend (Хранилище результатов)**
   - Хранит статус выполнения задач Celery (не путать с XCom!).
   - Обычно тот же Redis или Metadata DB.
   - Конфигурация: `result_backend = 'db+postgresql://....'`

4. **Flower (Мониторинг)**
   - Веб-интерфейс для мониторинга Celery-воркеров.
   - Показывает: список воркеров, количество задач в очередях, скорость обработки, статусы.
   - Запуск: `airflow celery flower --port=5555`

Конфигурация CeleryExecutor в `airflow.cfg`:

```ini
[core]
executor = CeleryExecutor

[celery]
broker_url = redis://redis:6379/0
result_backend = db+postgresql://airflow:airflow@postgres/airflow
worker_concurrency = 16
worker_autoscale = 16,4       # max,min — автомасштабирование потоков воркера

[celery_broker_transport_options]
visibility_timeout = 21600    # таймаут видимости задачи (сек.)
```

---

#### 13.8. Подвопросы

**Что такое `execution_date` / `logical_date`?**

`execution_date` (переименован в `logical_date` в Airflow 2.2) — это **логическая дата**, за которую запускается DagRun. Это **начало интервала данных**, а не фактическое время запуска!

Пример: DAG с `schedule_interval='@daily'` и `start_date=2024-01-01`:
- Первый DagRun: `logical_date = 2024-01-01`, **фактический запуск** = `2024-01-02 00:00` (по окончании интервала).
- Второй DagRun: `logical_date = 2024-01-02`, фактический запуск = `2024-01-03 00:00`.

Это контринтуитивно, но логика такова: DAG обрабатывает данные **за период** `[logical_date, logical_date + schedule_interval)`, и запускается **после** окончания этого периода, чтобы все данные за период были доступны.

В Airflow 2.2+ вместо `execution_date` рекомендуется использовать `data_interval_start` и `data_interval_end` — это более понятно.

**Что такое DagRun?**

`DagRun` — это конкретный экземпляр выполнения DAG для определённого `logical_date`. У каждого DagRun есть:
- `dag_id` — идентификатор DAG.
- `run_id` — уникальный идентификатор запуска (напр., `scheduled__2024-01-01T00:00:00+00:00` или `manual__2024-01-15T12:30:00+00:00`).
- `logical_date` — логическая дата.
- `state` — статус: `queued`, `running`, `success`, `failed`.
- `data_interval_start`, `data_interval_end` — интервал данных.

Один DAG может иметь несколько активных DagRun одновременно (ограничено `max_active_runs`).

**Что такое Backfill?**

**Backfill** — запуск DAG за прошлые периоды, для которых DagRun ещё не был создан.

Сценарии:
- DAG был создан 15 января с `start_date = 1 января` и `catchup=True` — Scheduler автоматически создаст DagRun за каждый день с 1 по 14 января.
- Ручной backfill через CLI: `airflow dags backfill -s 2024-01-01 -e 2024-01-31 my_dag`
- Полезно, если: изменилась логика трансформации и нужно пересчитать историю; данные были загружены заново; появился новый DAG для исторических данных.

Важно: для безопасного backfill задачи должны быть **идемпотентными** — повторный запуск с тем же `logical_date` должен давать тот же результат (например, `INSERT OVERWRITE` вместо `INSERT INTO`).
