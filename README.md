# Airflow baseline: стенд и текущая конфигурация

## Контекст задачи

**Фича:** Стенд и baseline-наблюдаемость Airflow  
**Эпик:** Оптимизация эффективного использования ресурсов Airflow

Цель baseline — зафиксировать текущую конфигурацию локального Airflow-стенда перед дальнейшими экспериментами по нагрузке и оптимизации ресурсов.

Стенд поднят локально через Docker Compose.

---

## 1. Runtime configuration

| Параметр | Значение | Источник |
|---|---:|---|
| Airflow version | `3.2.2` | `docker compose exec airflow-scheduler airflow version` |
| Executor | `CeleryExecutor` | `docker compose exec airflow-scheduler airflow config get-value core executor` |
| Backend DB | `PostgreSQL` | `docker compose exec airflow-scheduler airflow config get-value database sql_alchemy_conn` |
| SQLAlchemy connection | `postgresql+psycopg2://airflow:***@postgres/airflow` | `airflow config get-value database sql_alchemy_conn` |
| Broker | `Redis` | `docker compose exec airflow-scheduler airflow config get-value celery broker_url` |
| Broker URL | `redis://:@redis:6379/0` | `airflow config get-value celery broker_url` |

### Краткий вывод

Стенд использует `CeleryExecutor`, значит задачи Airflow выполняются не напрямую scheduler'ом, а через Celery worker. Для очереди задач используется Redis, а состояние Airflow хранится в PostgreSQL.

```text
Scheduler -> Redis broker -> Celery Worker
     |
     v
PostgreSQL metadata DB
```

---

## 2. Airflow services

Команда:

```bash
docker compose ps
```

| Service | Container | Image | Status | Ports |
|---|---|---|---|---|
| `airflow-apiserver` | `airflow-airflow-apiserver-1` | `apache/airflow:3.2.2` | `Up, healthy` | `0.0.0.0:8080->8080/tcp` |
| `airflow-dag-processor` | `airflow-airflow-dag-processor-1` | `apache/airflow:3.2.2` | `Up, healthy` | `8080/tcp` |
| `airflow-scheduler` | `airflow-airflow-scheduler-1` | `apache/airflow:3.2.2` | `Up, healthy` | `8080/tcp` |
| `airflow-triggerer` | `airflow-airflow-triggerer-1` | `apache/airflow:3.2.2` | `Up, healthy` | `8080/tcp` |
| `airflow-worker` | `airflow-airflow-worker-1` | `apache/airflow:3.2.2` | `Up, healthy` | `8080/tcp` |
| `postgres` | `airflow-postgres-1` | `postgres:16` | `Up, healthy` | `5432/tcp` |
| `redis` | `airflow-redis-1` | `redis:7.2-bookworm` | `Up, healthy` | `6379/tcp` |

### Количество компонентов

| Компонент | Количество |
|---|---:|
| API-server / UI | `1` |
| DAG processor | `1` |
| Scheduler | `1` |
| Triggerer | `1` |
| Worker | `1` |
| PostgreSQL | `1` |
| Redis | `1` |

### Краткий вывод

Все основные контейнеры Airflow находятся в состоянии `Up` и `healthy`. Для локального baseline-стенда поднят один экземпляр каждого ключевого компонента: scheduler, worker, API-server, DAG processor и triggerer.

---

## 3. Parallelism and execution limits

| Параметр | Значение | Источник | Назначение |
|---|---:|---|---|
| `parallelism` | `32` | `airflow config get-value core parallelism` | Глобальный лимит одновременно выполняемых task instances |
| `max_active_tasks_per_dag` | `16` | `airflow config get-value core max_active_tasks_per_dag` | Максимум активных задач в рамках одного DAG |
| `max_active_runs_per_dag` | `16` | `airflow config get-value core max_active_runs_per_dag` | Максимум активных запусков одного DAG |
| `worker_concurrency` | `16` | `airflow config get-value celery worker_concurrency` | Максимум задач, которые один Celery worker может выполнять параллельно |

### Краткий вывод

Так как используется один `airflow-worker` и `worker_concurrency = 16`, worker теоретически может взять до 16 задач одновременно.

Фактическая потенциальная ёмкость выполнения:

```text
worker_count × worker_concurrency = 1 × 16 = 16 задач одновременно
```

---

## 4. CPU/RAM usage snapshot

Команда:

```bash
docker stats --no-stream
```

| Container | CPU % | Memory usage | Memory limit | Memory % | PIDs |
|---|---:|---:|---:|---:|---:|
| `airflow-airflow-worker-1` | `0.28%` | `548.4 MiB` | `7.672 GiB` | `6.98%` | `24` |
| `airflow-airflow-triggerer-1` | `1.04%` | `283.1 MiB` | `7.672 GiB` | `3.60%` | `9` |
| `airflow-airflow-dag-processor-1` | `39.45%` | `351.3 MiB` | `7.672 GiB` | `4.47%` | `6` |
| `airflow-airflow-apiserver-1` | `0.11%` | `252.1 MiB` | `7.672 GiB` | `3.21%` | `15` |
| `airflow-airflow-scheduler-1` | `0.84%` | `208.6 MiB` | `7.672 GiB` | `2.65%` | `4` |
| `airflow-redis-1` | `0.14%` | `4.379 MiB` | `7.672 GiB` | `0.06%` | `6` |
| `airflow-postgres-1` | `0.90%` | `43.57 MiB` | `7.672 GiB` | `0.55%` | `13` |

### Краткий вывод

На момент снятия snapshot основную нагрузку по CPU показывал `airflow-dag-processor` — `39.45%`. Это ожидаемо для Airflow, так как DAG processor отвечает за обработку DAG-файлов. При дальнейшем росте количества DAG'ов именно этот компонент нужно наблюдать отдельно.

Наибольшее потребление RAM было у `airflow-worker` — `548.4 MiB`.

---

## 5. CPU/RAM limits

В выводе `docker stats` для всех контейнеров указан одинаковый memory limit:

```text
7.672 GiB
```


## 6. Итоговый baseline

| Блок | Значение |
|---|---|
| Airflow version | `3.2.2` |
| Executor | `CeleryExecutor` |
| Backend DB | `PostgreSQL` |
| Broker | `Redis` |
| Scheduler count | `1` |
| API-server count | `1` |
| DAG processor count | `1` |
| Worker count | `1` |
| Triggerer count | `1` |
| PostgreSQL count | `1` |
| Redis count | `1` |
| `parallelism` | `32` |
| `max_active_tasks_per_dag` | `16` |
| `max_active_runs_per_dag` | `16` |
| `worker_concurrency` | `16` |
| Individual CPU limits | `not set` |
| Individual RAM limits | `not set` |
| Highest CPU usage in snapshot | `airflow-dag-processor: 39.45%` |
| Highest RAM usage in snapshot | `airflow-worker: 548.4 MiB` |

---

## 7. Выводы

1. Локальный Airflow-стенд поднят через Docker Compose.
2. Metadata DB — PostgreSQL, брокер — Redis.
3. Все основные сервисы находятся в состоянии `healthy`.
4. Текущая worker capacity составляет 16 параллельных задач.
5. Глобальный `parallelism = 32`, но при одном worker и `worker_concurrency = 16` фактическая пропускная способность ограничена worker'ом.
6. Индивидуальные CPU/RAM limits для контейнеров не заданы.
7. В текущем snapshot наиболее заметную CPU-нагрузку создаёт `airflow-dag-processor`.
8. Для дальнейших стресс-тестов нужно отдельно наблюдать:
   - CPU/RAM `airflow-dag-processor`;
   - CPU/RAM `airflow-scheduler`;
   - CPU/RAM `airflow-worker`;
   - очередь Redis;
   - состояние PostgreSQL;
   - количество running/queued/scheduled task instances.

---

## 8. Команды, использованные для сбора baseline

```bash
docker compose exec airflow-scheduler airflow version
docker compose exec airflow-scheduler airflow config get-value core executor
docker compose exec airflow-scheduler airflow config get-value database sql_alchemy_conn
docker compose exec airflow-scheduler airflow config get-value celery broker_url
docker compose ps
docker compose exec airflow-scheduler airflow config get-value core parallelism
docker compose exec airflow-scheduler airflow config get-value core max_active_tasks_per_dag
docker compose exec airflow-scheduler airflow config get-value core max_active_runs_per_dag
docker compose exec airflow-scheduler airflow config get-value celery worker_concurrency
docker stats --no-stream
```

# Stress DAG Factory

## Описание

`stress_dag_factory.py` — фабрика DAG'ов для стресс-тестирования Airflow.
Все параметры нагрузки вынесены в Airflow Variables или переменные окружения.

---

# Архитектура

```
                 Airflow Variables
                        │
                        ▼
               get_config_value()
                        │
          если нет Variable
                        │
                        ▼
                Environment variables
                        │
          если нет env-переменной
                        │
                        ▼
             Значения по умолчанию
                        │
                        ▼
              create_stress_dag()
                        │
                        ▼
          Генерация DAG в цикле
                        │
                        ▼
        globals()[dag_id] = DAG
                        │
                        ▼
          Airflow обнаруживает DAG
```

---

# Настраиваемые параметры

| Параметр | Airflow Variable | Значение по умолчанию | Описание |
|----------|------------------|----------------------|----------|
| DAG_COUNT | STRESS_DAG_COUNT | 10 | Количество создаваемых DAG |
| TASKS_PER_DAG | STRESS_TASKS_PER_DAG | 10 | Количество задач в каждом DAG |
| TASK_DURATION | STRESS_TASK_DURATION | 5 | Продолжительность выполнения задачи (секунды) |
| POOL_NAME | STRESS_POOL_NAME | default_pool | Airflow Pool для задач |
| MAX_ACTIVE_RUNS | STRESS_MAX_ACTIVE_RUNS | 1 | Максимальное количество одновременно активных запусков одного DAG |
| WORKLOAD_TYPE | STRESS_WORKLOAD_TYPE | linear | Тип структуры DAG |

---

# Приоритет чтения настроек

Каждый параметр читается в следующем порядке:

1. Airflow Variable
2. Environment Variable
3. Значение по умолчанию

Схема работы:

```
Airflow Variable
       │
       ▼
если существует
       │
       ▼
использовать

иначе

Environment Variable
       │
       ▼
если существует
       │
       ▼
использовать

иначе

Default Value
```

---

# Типы нагрузки

## linear

Последовательное выполнение задач.

```
task_1
   │
   ▼
task_2
   │
   ▼
task_3
```

Следующая задача запускается только после успешного завершения предыдущей.

---

## parallel

Все задачи независимы.

```
task_1

task_2

task_3

task_4
```

Scheduler может отправить все задачи на выполнение одновременно (с учётом ограничений Airflow).

---

## fan_out

Одна задача запускает несколько последующих.

```
          ┌──► task_2
task_1 ───┤
          ├──► task_3
          └──► task_4
```

После завершения `task_1` становятся доступны все downstream-задачи.

---

## fan_in

Несколько задач сходятся в одну завершающую.

```
task_1 ───┐
          │
task_2 ───┼──► final_task
          │
task_3 ───┘
```

`final_task` будет выполнена только после успешного завершения всех upstream-зач.

---

# Структура DAG

Каждый DAG содержит:

- заданное количество задач BashOperator;
- общие параметры (`retries`, `retry_delay`);
- Pool;
- schedule;
- max_active_runs;
- выбранную структуру зависимостей.

---

# Конфигурация DAG

Используются следующие параметры Airflow:

| Параметр | Назначение |
|----------|------------|
| dag_id | Уникальный идентификатор DAG |
| start_date | Дата начала существования DAG |
| schedule | Расписание запуска |
| catchup | Создание пропущенных запусков |
| max_active_runs | Максимальное число одновременно выполняющихся DAG Run |
| default_args | Общие параметры задач |
| tags | Теги в интерфейсе Airflow |

---

# Работа функции create_stress_dag()

Функция создаёт один полностью настроенный DAG.

Алгоритм работы:

1. Создать объект DAG.
2. Создать необходимое количество задач BashOperator.
3. Добавить задачи в DAG.
4. Настроить зависимости между задачами.
5. Вернуть готовый объект DAG.

---

# Генерация нескольких DAG

После создания функции выполняется цикл:

```
for dag_number in range(...):
```

Для каждой итерации:

1. формируется уникальный `dag_id`;
2. вызывается `create_stress_dag()`;
3. полученный DAG регистрируется через

```
globals()[dag_id]
```

что позволяет Airflow обнаружить его при парсинге файла.

---

# Используемые Airflow-компоненты

Во время работы задействованы:

- DAG Processor — читает Python-файл и создаёт объекты DAG;
- Scheduler — анализирует расписание и зависимости;
- Redis (CeleryExecutor) — очередь задач;
- Worker — выполняет задачи;
- PostgreSQL — хранит состояние Airflow.

---

# Пример изменения конфигурации

Через Airflow Variables:

```bash
airflow variables set STRESS_DAG_COUNT 20
airflow variables set STRESS_TASKS_PER_DAG 50
airflow variables set STRESS_WORKLOAD_TYPE parallel
```

