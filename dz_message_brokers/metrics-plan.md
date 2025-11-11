# План мониторинга и ключевые метрики


### 1. Latency (Задержка)

**Что измеряем**: Время, необходимое для обработки запроса.

**Метрики:**

| Метрика | Описание | Target | Critical |
|---------|----------|--------|----------|
| **API Gateway Response Time** | | | |
| - p50 (median) | 50% запросов быстрее | < 100ms | > 500ms |
| - p95 | 95% запросов быстрее | < 300ms | > 1000ms |
| - p99 | 99% запросов быстрее | < 500ms | > 2000ms |
| **gRPC Call Duration** | | | |
| - Ticket→Event Service | p95 | < 50ms | > 200ms |
| - Ticket→Venue Service | p95 | < 100ms | > 300ms |
| **Database Query Time** | | | |
| - Simple SELECT | p95 | < 10ms | > 50ms |
| - Complex JOIN | p95 | < 50ms | > 200ms |
| **RabbitMQ Message Processing** | | | |
| - Time in queue | avg | < 100ms | > 1000ms |
| - Processing time | avg | < 500ms | > 5000ms |

### 2. Throughput (Пропускная способность)

**Что измеряем**: Количество обрабатываемых запросов.

**Метрики:**

| Метрика | Описание | Target | Critical |
|---------|----------|--------|----------|
| **Requests per Second (RPS)** | | | |
| - API Gateway total | Все запросы | 1000 rps | - |
| - GET /api/events | Просмотр событий | 500 rps | - |
| - POST /api/tickets | Покупка билетов | 100 rps | < 10 rps |
| **Messages per Second** | | | |
| - RabbitMQ publish rate | Публикация | 200 msg/s | - |
| - RabbitMQ consume rate | Обработка | 200 msg/s | < 50 msg/s |
| **Business Metrics** | | | |
| - Tickets sold per minute | Продажи | - | < 10/min |
| - Active users (concurrent) | Онлайн | - | - |


### 3. Errors (Ошибки)

**Что измеряем**: Частота ошибок.

**Метрики:**

| Метрика | Описание | Target | Critical |
|---------|----------|--------|----------|
| **HTTP Error Rate** | | | |
| - 4xx errors (%) | Клиентские ошибки | < 5% | > 20% |
| - 5xx errors (%) | Серверные ошибки | < 0.1% | > 1% |
| - 401/403 (%) | Auth errors | < 2% | > 10% |
| - 429 (%) | Rate limit hits | < 1% | > 5% |
| **gRPC Error Rate** | | | |
| - Failed calls (%) | Неудачные вызовы | < 0.1% | > 1% |
| - Circuit breaker open | Сервис недоступен | 0 | > 0 |
| **Database Errors** | | | |
| - Connection errors | Не могу подключиться | 0 | > 0 |
| - Query timeouts | Долгие запросы | < 0.1% | > 1% |
| **RabbitMQ Errors** | | | |
| - Failed publishes | Не отправлено | 0 | > 0 |
| - DLQ message count | В Dead Letter Queue | 0 | > 10 |
| - Consumer errors | Ошибки обработки | < 0.1% | > 1% |



## Инструменты мониторинга

### 1. Prometheus (Метрики)

### 2. Grafana (Визуализация)
