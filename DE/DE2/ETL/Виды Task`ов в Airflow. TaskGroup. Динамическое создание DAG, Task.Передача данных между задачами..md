#### 14.1. Основные типы операторов

Операторы делятся на три категории: **Action Operators** (выполняют действие), **Transfer Operators** (перемещают данные), **Sensors** (ожидают условие).

**Action Operators:**

| Оператор | Описание | Типичный сценарий |
|---|---|---|
| `PythonOperator` | Вызывает произвольную Python-функцию | Любая трансформация данных в Python |
| `BashOperator` | Выполняет bash-команду | Запуск shell-скриптов, CLI-утилит |
| `PostgresOperator` | Выполняет SQL-запрос в PostgreSQL | DDL/DML операции в БД |
| `MySqlOperator` | Выполняет SQL-запрос в MySQL | DDL/DML операции в MySQL |
| `EmptyOperator` | Ничего не делает (ранее `DummyOperator`) | Заглушка для маршрутизации, точки слияния веток |
| `BranchPythonOperator` | Вызывает функцию, возвращающую `task_id` следующей задачи | Условное ветвление пайплайна |
| `ShortCircuitOperator` | Если функция вернула `False`, пропускает все downstream задачи | Условная остановка пайплайна |
| `SparkSubmitOperator` | Отправляет Spark-приложение через `spark-submit` | Запуск Spark-джобов |
| `DockerOperator` | Запускает задачу внутри Docker-контейнера | Изоляция зависимостей, воспроизводимость |
| `KubernetesPodOperator` | Запускает задачу в отдельном Pod в Kubernetes | Cloud-native среды, per-task ресурсные лимиты |
| `EmailOperator` | Отправляет e-mail | Уведомления |

**Пример `BranchPythonOperator`:**

```python
from airflow.operators.python import BranchPythonOperator

def choose_branch(**context):
    day = context['logical_date'].weekday()
    if day < 5:  # будний день
        return 'process_weekday'
    else:
        return 'process_weekend'

branch = BranchPythonOperator(
    task_id='branch_by_day',
    python_callable=choose_branch,
)

branch >> [process_weekday, process_weekend] >> join
# join должен иметь trigger_rule='none_failed_min_one_success'
```

---

#### 14.2. Sensors (Сенсоры)

**Sensor** — это особый тип оператора, который **ожидает** выполнения некоторого условия перед тем, как завершиться успешно.

| Сенсор | Что ожидает |
|---|---|
| `FileSensor` | Появление файла по указанному пути |
| `HttpSensor` | HTTP-ответ с определённым условием (напр., status 200) |
| `ExternalTaskSensor` | Завершение задачи в другом DAG |
| `SqlSensor` | Результат SQL-запроса (непустой = True) |
| `S3KeySensor` | Появление объекта в S3-бакете |
| `TimeDeltaSensor` | Прошествие заданного времени после `logical_date` |
| `DateTimeSensor` | Наступление определённой даты/времени |

Ключевые параметры сенсора:

| Параметр | Описание |
|---|---|
| `poke_interval` | Интервал между проверками (по умолчанию 60 сек.) |
| `timeout` | Максимальное время ожидания (по умолчанию 7 дней) |
| `mode` | Режим ожидания: `'poke'` или `'reschedule'` |
| `soft_fail` | Если `True`, при timeout задача получает статус `skipped` вместо `failed` |
| `exponential_backoff` | Если `True`, интервал между проверками растёт экспоненциально |

**Режимы работы сенсоров:**

- **`mode='poke'`** (по умолчанию): Сенсор занимает **слот воркера** на всё время ожидания. Между проверками «спит» (`time.sleep`). Плюс: минимальная задержка. Минус: неэффективно расходует ресурсы.
- **`mode='reschedule'`**: Сенсор освобождает слот воркера между проверками. После неудачной проверки переводит себя в статус `up_for_reschedule` и возвращает слот. Scheduler перепланирует его через `poke_interval`. Плюс: экономит ресурсы. Минус: больше накладных расходов.
- **Deferrable Sensors** (Airflow 2.2+): Наиболее эффективный режим. Сенсор передаёт ожидание **Triggerer-у** (asyncio), полностью освобождая слот. Один Triggerer может ожидать тысячи условий.

```python
from airflow.sensors.filesystem import FileSensor

wait_for_file = FileSensor(
    task_id='wait_for_data_file',
    filepath='/data/incoming/sales_{{ ds }}.csv',  # Jinja-шаблон
    poke_interval=300,    # проверять каждые 5 минут
    timeout=3600,         # таймаут 1 час
    mode='reschedule',    # освобождать слот между проверками
)
```

---

#### 14.3. TaskGroup

**TaskGroup** — визуальная группировка задач в UI Airflow. Не влияет на выполнение — только на отображение.

```python
from airflow.utils.task_group import TaskGroup

with DAG('pipeline', ...) as dag:
    extract = PythonOperator(task_id='extract', ...)

    with TaskGroup('transform_group') as transform_group:
        clean = PythonOperator(task_id='clean', ...)
        validate = PythonOperator(task_id='validate', ...)
        enrich = PythonOperator(task_id='enrich', ...)
        clean >> validate >> enrich

    load = PythonOperator(task_id='load', ...)

    extract >> transform_group >> load
```

В UI DAG будет показан со свёрнутой группой `transform_group`, которую можно развернуть для просмотра деталей.

TaskGroup можно вкладывать друг в друга (nested TaskGroups).

**SubDAG vs TaskGroup:**
- `SubDagOperator` (устаревший) — создавал **отдельный DAG** внутри DAG, что приводило к проблемам: deadlock-и (SubDAG занимал слот родительского DAG), сложности с мониторингом, отдельный scheduler loop.
- `TaskGroup` — **рекомендуемая замена** SubDAG с Airflow 2.0. Это просто визуальная группировка, задачи остаются частью родительского DAG, нет проблем с deadlock-ами.

---

#### 14.4. Динамическое создание DAG и Task

**Динамическое создание DAG (через цикл):**

```python
# dynamic_dags.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

configs = [
    {'name': 'sales', 'source': 'postgres', 'table': 'sales'},
    {'name': 'users', 'source': 'mysql', 'table': 'users'},
    {'name': 'orders', 'source': 'api', 'endpoint': '/orders'},
]

def create_dag(config):
    dag_id = f"etl_{config['name']}"

    dag = DAG(
        dag_id=dag_id,
        schedule_interval='@daily',
        start_date=datetime(2024, 1, 1),
        catchup=False,
    )

    def extract_fn(**kwargs):
        print(f"Extracting from {config['source']}")

    def load_fn(**kwargs):
        print(f"Loading {config['name']}")

    with dag:
        extract = PythonOperator(task_id='extract', python_callable=extract_fn)
        load = PythonOperator(task_id='load', python_callable=load_fn)
        extract >> load

    return dag

for config in configs:
    globals()[f"etl_{config['name']}"] = create_dag(config)
```

Ключевой момент: DAG-объект должен оказаться в `globals()` модуля, чтобы Scheduler его обнаружил при парсинге.

**Dynamic Task Mapping (Airflow 2.3+):**

Позволяет создавать задачи **динамически во время выполнения** (а не при парсинге). Аналог `map` в функциональном программировании.

```python
from airflow.decorators import dag, task
from datetime import datetime

@dag(schedule='@daily', start_date=datetime(2024, 1, 1), catchup=False)
def dynamic_etl():

    @task
    def get_partitions():
        """Возвращает список партиций для обработки."""
        return ['partition_a', 'partition_b', 'partition_c', 'partition_d']

    @task
    def process_partition(partition: str):
        """Обработка одной партиции."""
        print(f"Processing {partition}")
        return f"result_{partition}"

    @task
    def aggregate(results):
        """Агрегация всех результатов."""
        print(f"Aggregating {len(results)} results")

    partitions = get_partitions()
    # .expand() создаёт N экземпляров задачи — по одному на каждый элемент
    results = process_partition.expand(partition=partitions)
    aggregate(results)

dynamic_etl()
```

Airflow автоматически создаст 4 экземпляра `process_partition` (по одному на каждую партицию) и дождётся завершения всех перед запуском `aggregate`.

---

#### 14.5. Передача данных между задачами: XCom

**XCom** (Cross-Communication) — механизм обмена **небольшими** данными между задачами.

```python
# Способ 1: явный push/pull
def push_data(**context):
    context['ti'].xcom_push(key='row_count', value=42)

def pull_data(**context):
    row_count = context['ti'].xcom_pull(task_ids='extract', key='row_count')
    print(f"Extracted {row_count} rows")

# Способ 2: return value (автоматический push с key='return_value')
def extract_fn():
    return {'rows': 100, 'source': 'postgres'}

def load_fn(**context):
    data = context['ti'].xcom_pull(task_ids='extract')
    print(data)  # {'rows': 100, 'source': 'postgres'}

# Способ 3: TaskFlow API (Airflow 2.0+) — автоматически через аннотации
@task
def extract():
    return {'rows': 100}

@task
def transform(data):
    data['rows'] *= 2
    return data

result = transform(extract())  # XCom передаётся неявно
```

**Ограничения XCom:**
- Данные сериализуются в JSON и хранятся в Metadata DB.
- **Ограничение по размеру**: PostgreSQL — практически неограничено (TEXT), MySQL — до 64 KB (BLOB), SQLite — подобно.
- **Не предназначен для больших данных!** Для передачи больших объёмов данных — используйте промежуточное хранилище (S3, GCS, HDFS) и передавайте через XCom только **путь** к данным.

**Custom XCom Backend:**

С Airflow 2.0 можно заменить стандартное хранилище XCom на собственное (например, S3):

```python
# custom_xcom_backend.py
from airflow.models.xcom import BaseXCom
import json, boto3

class S3XComBackend(BaseXCom):
    @staticmethod
    def serialize_value(value, *, key=None, task_id=None, dag_id=None,
                        run_id=None, map_index=None):
        s3 = boto3.client('s3')
        s3_key = f"xcom/{dag_id}/{run_id}/{task_id}/{key}.json"
        s3.put_object(
            Bucket='airflow-xcom',
            Key=s3_key,
            Body=json.dumps(value),
        )
        return s3_key  # в Metadata DB хранится только путь

    @staticmethod
    def deserialize_value(result):
        s3 = boto3.client('s3')
        response = s3.get_object(Bucket='airflow-xcom', Key=result.value)
        return json.loads(response['Body'].read())
```

Конфигурация: `xcom_backend = my_module.S3XComBackend`

**Variables и Connections:**

- **Variables** — глобальные key-value настройки, доступные из любого DAG. Хранятся в Metadata DB. Можно шифровать. Доступ: `Variable.get('key')` или Jinja: `{{ var.value.key }}`.
- **Connections** — хранят параметры подключения к внешним системам (host, login, password, port, schema, extra JSON). Управляются через UI или CLI. Используются Hooks и Operators.

---

#### 14.6. Написание собственного оператора

Для создания кастомного оператора нужно унаследовать `BaseOperator` и реализовать метод `execute()`.

```python
from airflow.models import BaseOperator
from airflow.utils.decorators import apply_defaults
from typing import Any, Sequence

class MyCustomOperator(BaseOperator):
    """
    Оператор для загрузки данных из REST API в PostgreSQL.
    """

    # Поля, поддерживающие Jinja-шаблонизацию
    template_fields: Sequence[str] = ('endpoint', 'target_table')
    template_ext: Sequence[str] = ('.sql',)

    def __init__(
        self,
        endpoint: str,
        target_table: str,
        api_conn_id: str = 'default_api',
        postgres_conn_id: str = 'default_postgres',
        **kwargs,
    ):
        super().__init__(**kwargs)
        self.endpoint = endpoint
        self.target_table = target_table
        self.api_conn_id = api_conn_id
        self.postgres_conn_id = postgres_conn_id

    def execute(self, context: dict) -> Any:
        """
        Основной метод — вызывается при выполнении задачи.
        context содержит: ti, ds, logical_date, dag_run и т. д.
        """
        from airflow.providers.http.hooks.http import HttpHook
        from airflow.providers.postgres.hooks.postgres import PostgresHook

        # Используем Hook для работы с API
        http_hook = HttpHook(method='GET', http_conn_id=self.api_conn_id)
        response = http_hook.run(self.endpoint)
        data = response.json()

        self.log.info(f"Получено {len(data)} записей из API")

        # Используем Hook для работы с PostgreSQL
        pg_hook = PostgresHook(postgres_conn_id=self.postgres_conn_id)
        pg_hook.insert_rows(
            table=self.target_table,
            rows=[tuple(row.values()) for row in data],
            target_fields=list(data[0].keys()),
        )

        self.log.info(f"Загружено {len(data)} записей в {self.target_table}")
        return len(data)  # return value попадёт в XCom
```

Использование:

```python
load_api_data = MyCustomOperator(
    task_id='load_api_data',
    endpoint='/api/v1/sales/{{ ds }}',  # Jinja: подставит logical_date
    target_table='raw_sales',
    api_conn_id='sales_api',
    postgres_conn_id='dwh_postgres',
)
```

**`template_fields`** — кортеж имён атрибутов, в которых Airflow выполнит Jinja-рендеринг перед вызовом `execute()`. Это позволяет использовать макросы: `{{ ds }}`, `{{ logical_date }}`, `{{ params.my_param }}` и т. д.

**Hooks vs Operators:**
- **Hook** — низкоуровневый интерфейс для подключения к внешней системе (API, БД, облачному сервису). Hook знает *как* подключиться и взаимодействовать. Примеры: `PostgresHook`, `S3Hook`, `HttpHook`.
- **Operator** — высокоуровневая абстракция, определяющая *что* делать. Operator использует Hook для подключения к системе. Примеры: `PostgresOperator`, `S3ToRedshiftOperator`.
- Если нужна нестандартная логика — пишите **свой Operator**, используя существующие **Hooks**. Если нет подходящего Hook — пишите и Hook тоже.
