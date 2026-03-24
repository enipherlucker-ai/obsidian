#### 4.1 Что такое unit-тест

**Unit-тест (модульный тест)** — автоматизированная проверка наименьшей тестируемой единицы кода (функции, метода, класса) **в изоляции** от остальных компонентов системы (БД, API, файловой системы).

**«Unit» = единица** — одна функция, один метод, один класс. Unit-тест проверяет поведение этой единицы при разных входных данных, граничных случаях и ошибках.

#### 4.2 Зачем нужны unit-тесты

1. **Раннее обнаружение ошибок** — ошибка обнаруживается за секунды при запуске тестов, а не при деплое в продакшен.
2. **Защита от регрессий** — при изменении кода тесты гарантируют, что старый функционал не сломался.
3. **Документация поведения** — тесты показывают, как функция должна себя вести. Новый разработчик читает тесты и понимает контракт функции.
4. **Рефакторинг без страха** — можно менять внутреннюю реализацию, пока тесты проходят.
5. **Ускорение разработки** — ручная проверка «запустил → посмотрел → ОК» занимает больше времени.
6. **CI/CD** — тесты запускаются автоматически в пайплайне при каждом push/PR.

#### 4.3 Основные принципы (FIRST)

| Буква | Принцип | Описание |
|-------|---------|----------|
| **F** | Fast | Тесты выполняются быстро (секунды). Медленные тесты не запускают. |
| **I** | Independent / Isolated | Тесты не зависят друг от друга. Порядок запуска не важен. Каждый тест готовит своё окружение. |
| **R** | Repeatable | Результат одинаков при любом запуске, в любой среде. Нет зависимости от времени, сети, случайности. |
| **S** | Self-validating | Тест сам определяет pass/fail через assert. Не нужно смотреть логи глазами. |
| **T** | Timely | Тесты пишутся вовремя — вместе с кодом или до него (TDD). |

#### 4.4 Структура теста — паттерн AAA (Arrange-Act-Assert)

```python
def test_calculate_discount():
    # Arrange — подготовка данных
    price = 1000
    discount_percent = 15
    
    # Act — вызов тестируемого кода
    result = calculate_discount(price, discount_percent)
    
    # Assert — проверка результата
    assert result == 850
```

Название теста должно отражать: **что тестируется**, **при каких условиях**, **какой ожидается результат**:
`test_calculate_discount_with_15_percent_returns_850`

#### 4.5 Примеры на pytest

```python
import pytest

# Тестируемый код
def divide(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("Деление на ноль")
    return a / b

# Тесты
def test_divide_positive_numbers():
    assert divide(10, 2) == 5.0

def test_divide_negative_numbers():
    assert divide(-10, 2) == -5.0

def test_divide_returns_float():
    result = divide(7, 2)
    assert result == 3.5
    assert isinstance(result, float)

def test_divide_by_zero_raises():
    with pytest.raises(ValueError, match="Деление на ноль"):
        divide(10, 0)
```

#### 4.6 Фикстуры (fixtures)

**Фикстура** — подготовка окружения для тестов: создание данных, подключений, файлов. В pytest фикстуры объявляются через `@pytest.fixture` и передаются по имени:

```python
import pytest
import pandas as pd

@pytest.fixture
def sample_df():
    return pd.DataFrame({
        'name': ['Иван', 'Мария', 'Пётр'],
        'age': [30, 25, 35],
        'salary': [100000, 120000, 90000]
    })

def test_dataframe_shape(sample_df):
    assert sample_df.shape == (3, 3)

def test_average_age(sample_df):
    assert sample_df['age'].mean() == 30.0
```

Scope фикстуры: `function` (по умолчанию — каждый тест), `class`, `module`, `session`.

#### 4.7 Моки и изоляция

Для проверки логики без реальных внешних систем используют **моки** (mock objects):

```python
from unittest.mock import patch, MagicMock

def fetch_user_data(user_id):
    response = requests.get(f'https://api.example.com/users/{user_id}')
    return response.json()

def test_fetch_user_data():
    mock_response = MagicMock()
    mock_response.json.return_value = {'id': 1, 'name': 'Иван'}
    mock_response.status_code = 200

    with patch('requests.get', return_value=mock_response) as mock_get:
        result = fetch_user_data(1)
        assert result == {'id': 1, 'name': 'Иван'}
        mock_get.assert_called_once_with('https://api.example.com/users/1')
```

#### 4.8 Параметризация

Запуск одного теста с разными входными данными:

```python
@pytest.mark.parametrize("input_val, expected", [
    (0, 0),
    (1, 1),
    (-1, 1),
    (3.14, 3.14),
    (-5, 5),
])
def test_abs_value(input_val, expected):
    assert abs(input_val) == expected
```

#### 4.9 Сопутствующие вопросы

**Чем unit-тесты отличаются от интеграционных и e2e?**

| Тип | Что проверяет | Скорость | Изоляция |
|-----|---------------|----------|----------|
| Unit | Одна функция/класс | Очень быстро (мс) | Полная (моки) |
| Integration | Взаимодействие модулей (код + БД) | Средне (сек) | Частичная |
| E2E (end-to-end) | Весь путь пользователя | Медленно (мин) | Нет |

**Что такое TDD (Test-Driven Development)?**
Цикл: (1) Написать тест, который падает → (2) Написать минимальный код, чтобы тест прошёл → (3) Рефакторить. Повторить.

**Что такое покрытие (coverage)?**
Метрика, показывающая процент строк/ветвей кода, выполненных при запуске тестов. Инструмент: `pytest-cov`. 100% покрытие не гарантирует отсутствие багов, но низкое покрытие — сигнал риска.
