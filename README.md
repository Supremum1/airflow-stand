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

---

## 2. Airflow services

Команда:

```bash
docker compose ps
```

| Service | Container | Image | Status | Ports |
|---|---|---|---|---|
| `airflow-apiserver` | `airflow-airflow-apiserver-1` | `apache/airflow:3.2.2` | `Up, healthy` | `8080/tcp` |
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


