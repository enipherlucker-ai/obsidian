### 6.1. Основные понятия

**Поток (thread)** — наименьшая единица исполнения внутри процесса. Потоки одного процесса **разделяют** общее адресное пространство (память, файловые дескрипторы, глобальные переменные).

**Процесс (process)** — экземпляр программы с **собственным** изолированным адресным пространством. Процессы общаются через IPC (Inter-Process Communication): каналы (pipes), очереди, разделяемую память, сокеты.

| Характеристика | Поток (Thread) | Процесс (Process) |
|----------------|---------------|--------------------|
| Память | Общая (heap, глобальные переменные) | Изолированная (копия при `fork`, отдельная при `spawn`) |
| Создание | Быстрое (~микросекунды) | Медленное (~миллисекунды) |
| Переключение контекста | Дешёвое | Дорогое |
| Сбой одного | Может уронить весь процесс | Не влияет на другие процессы |
| Обмен данными | Через общие переменные | Через IPC (сериализация) |

### 6.2. GIL (Global Interpreter Lock)

В CPython существует **GIL** — глобальная блокировка, которая позволяет одновременно исполнять байткод Python **только одному потоку** в рамках одного процесса.

**Последствия:**
- **I/O-bound задачи** — GIL **освобождается** при системных вызовах (сеть, диск), поэтому потоки реально работают параллельно при ожидании ввода-вывода.
- **CPU-bound задачи** — потоки **не дают ускорения** (и даже могут замедлить из-за overhead на переключение и борьбу за GIL). Нужны **процессы**.

> **Примечание:** В Python 3.13 появился экспериментальный режим free-threaded (no-GIL) CPython, но на практике в продакшне пока используется стандартный CPython с GIL.

### 6.3. Когда использовать threading

**I/O-bound** задачи — когда программа большую часть времени ждёт:
- HTTP-запросы к внешним API
- Чтение/запись файлов, работа с БД
- Загрузка/отправка данных по сети
- WebSocket-соединения

```python
import threading
import requests
import time

urls = [
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
    "https://httpbin.org/delay/1",
]

def fetch(url: str, results: list, idx: int) -> None:
    response = requests.get(url)
    results[idx] = response.status_code

# --- Последовательно ---
start = time.perf_counter()
results_seq = [requests.get(u).status_code for u in urls]
print(f"Последовательно: {time.perf_counter() - start:.2f}s")  # ~4 сек

# --- С потоками ---
start = time.perf_counter()
results_thr = [None] * len(urls)
threads = [threading.Thread(target=fetch, args=(u, results_thr, i)) for i, u in enumerate(urls)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(f"С потоками:       {time.perf_counter() - start:.2f}s")  # ~1 сек
```

### 6.4. Когда использовать multiprocessing

**CPU-bound** задачи — когда процессор загружен вычислениями:
- Математические расчёты, обработка данных
- Компрессия, шифрование
- Обработка изображений, видео
- Обучение ML-моделей (если не на GPU)

```python
import multiprocessing
import time
import math

def heavy_computation(n: int) -> float:
    """Тяжёлая CPU-bound функция."""
    return sum(math.sqrt(i) * math.sin(i) for i in range(n))

numbers = [5_000_000] * 4

# --- Последовательно ---
start = time.perf_counter()
results_seq = [heavy_computation(n) for n in numbers]
print(f"Последовательно: {time.perf_counter() - start:.2f}s")  # ~8 сек (условно)

# --- С процессами ---
start = time.perf_counter()
with multiprocessing.Pool(processes=4) as pool:
    results_mp = pool.map(heavy_computation, numbers)
print(f"С процессами:    {time.perf_counter() - start:.2f}s")  # ~2 сек (на 4 ядрах)
```

### 6.5. concurrent.futures — унифицированный интерфейс

Модуль `concurrent.futures` предоставляет одинаковый API для потоков и процессов через `Executor`:

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import requests
import math

# --- ThreadPoolExecutor (I/O-bound) ---
urls = ["https://httpbin.org/get"] * 10

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(requests.get, url) for url in urls]
    responses = [f.result() for f in futures]

# --- ProcessPoolExecutor (CPU-bound) ---
def cpu_task(n: int) -> float:
    return sum(math.factorial(i % 20) for i in range(n))

with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(executor.map(cpu_task, [1_000_000] * 4))
```

Преимущество: можно переключиться между потоками и процессами, заменив лишь имя класса Executor.

### 6.6. asyncio — третий подход (кооперативная многозадачность)

`asyncio` — это **однопоточная** событийная петля (event loop) с кооперативной многозадачностью. Корутины добровольно отдают управление через `await`.

```python
import asyncio
import aiohttp

async def fetch(session: aiohttp.ClientSession, url: str) -> int:
    async with session.get(url) as resp:
        return resp.status

async def main():
    urls = ["https://httpbin.org/delay/1"] * 10
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
    print(results)  # 10 запросов за ~1 секунду

asyncio.run(main())
```

| Подход | Тип задач | Параллелизм | Overhead | GIL |
|--------|-----------|-------------|----------|-----|
| `threading` | I/O-bound | Конкурентность (не параллельность из-за GIL) | Низкий | Да, но освобождается при I/O |
| `multiprocessing` | CPU-bound | Настоящий параллелизм (отдельные процессы) | Высокий (fork/spawn, IPC) | Каждый процесс — свой GIL |
| `asyncio` | I/O-bound (много соединений) | Кооперативная многозадачность (1 поток) | Минимальный | Не актуален (1 поток) |

### 6.7. Гонки данных (Race Conditions) и примитивы синхронизации

Когда потоки разделяют данные, возникает проблема **race condition** — результат зависит от порядка выполнения потоков.

```python
import threading

counter = 0

def increment():
    global counter
    for _ in range(1_000_000):
        counter += 1  # НЕ атомарная операция: read → modify → write

threads = [threading.Thread(target=increment) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)  # Ожидаем 4_000_000, получаем меньше (race condition!)
```

**Примитивы синхронизации:**

| Примитив | Назначение |
|----------|------------|
| `threading.Lock` | Взаимное исключение — только один поток в критической секции |
| `threading.RLock` | Реентерабельный Lock — один поток может захватить повторно |
| `threading.Semaphore` | Ограничить число одновременных потоков (например, макс. 5 соединений) |
| `threading.Event` | Сигнализация между потоками (set/wait) |
| `threading.Condition` | Ожидание с условием (notify/wait) |
| `queue.Queue` | Потокобезопасная очередь (producer-consumer) |
| `multiprocessing.Queue` | Межпроцессная очередь (сериализация через pickle) |
| `multiprocessing.Value / Array` | Разделяемая память между процессами |

```python
import threading

counter = 0
lock = threading.Lock()

def safe_increment():
    global counter
    for _ in range(1_000_000):
        with lock:
            counter += 1

threads = [threading.Thread(target=safe_increment) for _ in range(4)]
for t in threads:
    t.start()
for t in threads:
    t.join()

print(counter)  # 4_000_000 — корректно
```

### 6.8. Сводная таблица

| Критерий | `threading` | `multiprocessing` | `asyncio` |
|----------|------------|-------------------|-----------|
| Модель памяти | Общая | Изолированная | Общая (1 поток) |
| Обход GIL | Нет | Да | Не нужен |
| Идеален для | I/O-bound | CPU-bound | Высококонкурентный I/O |
| Сложность отладки | Средняя (race conditions) | Средняя (IPC, сериализация) | Средняя (async-стек, дебаг) |
| Масштабируемость | Сотни потоков | Десятки процессов (по числу ядер) | Тысячи-десятки тысяч корутин |
