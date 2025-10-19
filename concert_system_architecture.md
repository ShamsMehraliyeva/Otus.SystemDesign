# Проектирование отказоустойчивой системы управления концертами

## 1. МАСШТАБИРОВАНИЕ СИСТЕМЫ

### 1.1 Выбор стратегии масштабирования

**Выбрано: Горизонтальное масштабирование**

**Обоснование:**
- Пиковые нагрузки при старте продаж билетов
- Экономически эффективно: добавление серверов по требованию
- Высокая отказоустойчивость: отказ одного узла не критичен
- Нет физических ограничений роста

**Применение:**
- Application серверы: автомасштабирование
- Database: Master + 2 Read реплик
- Cache: Redis Cluster + 3 реплики

**Вертикальное масштабирование** используется ограниченно только для Master БД.

### 1.2 Архитектура системы

```
Пользователи
    ↓
   CDN
    ↓
Load Balancer
    ↓
App Servers
    ↓
Redis Cluster (кэш) + Message Queue
    ↓
PostgreSQL Master-Slave
├── Master (запись)
└── Slaves x2 (чтение)
```

### 1.3 Шардинг базы данных

**Стратегия:** Географический шардинг, например:
- Shard 1: Москва и область
- Shard 2: Санкт-Петербург
- и т.д

**Дополнительно:** Партиционирование по времени (текущие/будущие/архивные события)

### 1.4 CAP-теорема

**Выбор: CP (Consistency + Partition Tolerance)**

**Обоснование:**
- Согласованность критична: нельзя продать одно место дважды
- Лучше временная недоступность, чем двойное бронирование
- Финансовые транзакции требуют точности

**Реализация:**
- Бронирование билетов: Strong Consistency, SELECT FOR UPDATE
- Просмотр концертов: Eventual Consistency
- Отзывы и рейтинги: Асинхронная репликация

---

## 2. ВЫСОКАЯ ДОСТУПНОСТЬ

### 2.1 Целевые показатели
- **SLA:** 99.9% (допустимый простой 8.76 часов/год)
- **RTO:** 15 минут
- **RPO:** 1 час

### 2.2 Архитектура отказоустойчивости

**Load Balancer:**
- Health checks каждые 5 секунд
- Автоматическое исключение неработающих серверов

**Application Servers:**
- Минимум 3 сервера в разных Availability Zones
- Stateless (сессии в Redis)
- Auto-scaling при нагрузке

**Database:**
- Master + 2 Read Slaves
- Автоматический failover
- Синхронная репликация на Standby Master

**Redis:**
- Cluster: мастера + 3 реплики
- Redis Sentinel для failover
- Автоматическая промоция при отказе

### 2.3 Сценарии отказов

#### Сценарий 1: Отказ Application Server
**Проблема:** Сервер приложений упал  
**Обнаружение:** Health check timeout (15 сек)  
**Действие:** Load balancer автоматически исключает сервер, запускается новый инстанс  
**Влияние:** Пользователи не замечают

#### Сценарий 2: Отказ Master БД
**Проблема:** Master база недоступна  
**Обнаружение:** Patroni (20-30 сек)
**Действие:** Автоматическая промоция Slave в Master, переключение приложений
**Влияние:** Запись недоступна 30-90 сек, чтение продолжает работать

#### Сценарий 3: Отказ Redis узла
**Проблема:** Redis Master недоступен  
**Обнаружение:** Sentinel (3-5 сек)  
**Действие:** Промоция реплики, переподключение клиентов  
**Влияние:** Cache miss для части данных, увеличение latency

#### Сценарий 4: Отказ Availability Zone
**Проблема:** Полная недоступность одной зоны  
**Обнаружение:** Множественные failures  
**Действие:** Трафик перенаправляется в другие зоны, auto-scaling компенсирует capacity  
**Влияние:** 1-2 минуты деградации, затем восстановление

### 2.4 Непрерывность работы

**Backup стратегия:**
- WAL streaming: RPO < 1 минута
- Инкрементальный backup: каждые 6 часов
- Полный backup: ежедневно
- Хранение: 90 дней + годовые архивы

**Disaster Recovery:**
- Primary Region: Москва (production)
- DR Region: Санкт-Петербург (hot standby)
- Failover time: 15 минут

---

## 3. ОПТИМИЗАЦИЯ ПРОИЗВОДИТЕЛЬНОСТИ

### 3.1 Узкие места

**База данных:**
- Медленные JOIN при поиске с фильтрами
- Блокировки при одновременном бронировании
- N+1 query problem

**Приложение:**
- Недостаточное кэширование
- Синхронная обработка запросов

### 3.2 Индексация

```sql
-- Поиск концертов по городу и дате
CREATE INDEX idx_events_city_date 
ON events(city_id, event_date) 
WHERE status = 'active';

-- Проверка доступности билетов
CREATE INDEX idx_tickets_availability 
ON tickets(event_id, status) 
WHERE status = 'available';

-- Предотвращение двойной продажи
CREATE UNIQUE INDEX idx_tickets_seat 
ON tickets(event_id, sector, row, seat);
```


### 3.3 Кэширование

**Уровни кэша:**

1. **CDN:** Статика (TTL: 24 часа)
2. **Redis:** Данные приложения


3. **Database query cache**

Cache-Aside для чтения, Write-Through для билетов

### 3.5 Профиль производительности

| Метрика | Целевое | Критическое |
|---------|---------|-------------|
| API Response (p95) | < 200ms | > 1000ms |
| API Response (p99) | < 500ms | > 2000ms |
| DB Query Time | < 50ms | > 500ms |
| Cache Hit Rate | > 85% | < 60% |

---

## 4. МОНИТОРИНГ И ПРОФИЛИРОВАНИЕ

### 4.1 Метрики

**Application:**
- Request rate (req/sec)
- Response time (p50, p95, p99)
- Error rate (4xx, 5xx)
- Throughput

**Database:**
- Query execution time
- Active connections
- Replication lag
- Deadlocks

**Infrastructure:**
- CPU, Memory, Disk I/O
- Network bandwidth

**Business:**
- Tickets sold/minute
- Booking success rate
- Revenue/hour

### 4.2 Инструменты

**Стек:**
- **Prometheus** - сбор метрик
- **Grafana Loki** - визуализация, дашборды, централизованное логирование
- **Alertmanager** - уведомления
- **Jaeger** - distributed tracing

### 4.3 Дашборды Grafana

**Dashboard 1: Application Health**
- Request rate
- Latency percentiles
- Error rate
- Active users

**Dashboard 2: Database**
- Query time
- Slow queries
- Connection pool
- Replication lag

**Dashboard 3: Business**
- Tickets sold (real-time)
- Revenue
- Conversion rate

### 4.4 Диагностика проблем

**Пример: Рост latency**

1. **Обнаружение:** Grafana alert (p95 > 1s)
2. **Первичный анализ:** Проверка CPU/Memory/Network
3. **Детальное исследование:** 
   - Jaeger traces для поиска медленных операций
   - Slow query log в БД
4. **Возможные причины:**
   - Медленный запрос без индекса
   - Cache miss
   - Пиковая нагрузка
5. **Действия:**
   - Краткосрочно: scale out, увеличение timeout
   - Долгосрочно: оптимизация запроса, добавление индекса

---

## 5. РЕЗУЛЬТАТЫ

### 5.1 Архитектурная диаграмма

1. Пользователи (Users)
   ↓
2. CDN (для статики)
   ↓
3. Load Balancer
   ↓
4. Application Servers (3-N штук)
   ↓
5. Redis Cluster (кэш)
6. Message Queue (RabbitMQ)
   ↓
7. PostgreSQL Master-Slave
   ↓
8. Monitoring (Prometheus + Grafana)


### 5.2 Ожидаемые показатели

**Производительность:**
- Throughput: 5,000 req/sec
- Latency p95: < 200ms
- Error rate: < 0.1%

**Доступность:**
- Uptime: 99.9%
- RTO: 15 минут
- RPO: 1 час

**Масштабируемость:**
- Concurrent users: 50,000+
- Auto-scaling: 3 → 50 серверов
- Database: шардинг по географии