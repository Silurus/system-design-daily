# Ресурсы для подготовки к System Design

Полезные материалы и ссылки для изучения проектирования систем.

---

## Структура интервью

**Источник:** [NowInterview — Структура интервью](https://nowinterview.ru/learn/system-design/in-a-hurry/framework)

### Рекомендуемое распределение времени (45 мин)

1. **Требования (~5 мин)**
   - Функциональные: «Пользователи могут...» (топ-3, не больше)
   - Нефункциональные: масштаб, latency, CAP, безопасность (3–5 самых важных)
   - Back-of-the-envelope расчёты — **только если влияют на дизайн**, не для галочки

2. **API и сущности (~5 мин)**
   - Базовые эндпоинты (1 на функциональное требование)
   - Список сущностей без детализации (User, Post, Follow...)

3. **Высокоуровневый дизайн (~10–15 мин)**
   - Прямоугольники (компоненты) + стрелки (потоки данных)
   - Один эндпоинт за раз, проговаривай ход мыслей
   - Колонки БД рядом с компонентом, без типов данных
   - **Не усложняй рано** — отметь «тут можно кэш/очередь», двигайся дальше

4. **Погружение в детали (~15–25 мин)**
   - Закрыть нефункциональные требования
   - Узкие места, граничные случаи
   - Senior+ должен сам находить места для улучшения
   - **Баланс:** не «переговаривай» интервьюера, оставь пространство для вопросов

### Чек-лист NFR (нефункциональные требования)

1. **CAP:** availability vs consistency (partition tolerance — данность)
2. **Ограничения среды:** мобильный клиент, батарея, 3G
3. **Масштаб:** всплески трафика, праздники, чтение vs запись
4. **Latency:** насколько быстро должен быть ответ
5. **Durability:** потеря данных критична (банк) или терпима (соцсеть)
6. **Безопасность:** защита данных, контроль доступа, GDPR
7. **Отказоустойчивость:** избыточность, failover, восстановление
8. **Регуляторика:** PCI DSS, HIPAA, региональные требования

---

## Паттерны System Design

**Источник:** [NowInterview — Паттерны](https://nowinterview.ru/learn/system-design/in-a-hurry/patterns)

### 1. Обновления в реальном времени
**Когда:** чаты, уведомления, live-дашборды  
**Протоколы:**
- HTTP polling — простейший, начни с него
- Server-Sent Events (SSE) — однонаправленный push от сервера
- WebSocket — двусторонний, сложнее инфра

**Серверная сторона:**
- Pub/Sub (Redis, Kafka) — разделить publisher и subscriber
- Stateful servers + consistent hashing — для тяжёлой обработки (Google Docs)

### 2. Управление длительными задачами
**Когда:** кодирование видео, генерация отчётов, любая задача >5 сек  
**Как:**
1. Клиент → API: валидация + task_id → очередь (Kafka/RabbitMQ)
2. Воркеры достают из очереди, выполняют
3. Статус задачи: polling `/task/{id}` или SSE/WebSocket

**Не добавляй очередь преждевременно:** если задача <2 сек — синхронно проще.

**Ключевые нюансы:**
- Повторы (retry)
- Dead Letter Queue (DLQ) для сбойных задач
- Отслеживание статуса в БД или Redis

### 3. Управление конкуренцией
**Когда:** race condition (последний билет, аукцион, инвентарь)  
**Решения:**
- **Pessimistic locking** — `SELECT FOR UPDATE` (блокирует строку)
- **Optimistic concurrency control** — версия записи, CAS
- **Распределённые блокировки** — Redis SETNX, ZooKeeper (избегай без нужды)
- **Очередь** — сериализация операций через queue

**Правило:** начинай с ACID-транзакций в одной БД, распределённые блокировки — только если шардирование неизбежно.

### 4. Масштабирование чтения
**Когда:** соотношение read:write 10:1 и выше (Instagram, Twitter)  
**Порядок действий:**
1. **Индексы** (B-tree, hash) — внутри БД
2. **Read replicas** — горизонтальное масштабирование (eventual consistency)
3. **Кэш** (Redis, Memcached) — горячие данные
4. **CDN** — статика и медиа

**Нюансы:**
- Инвалидация кэша (TTL, purge, cache tags)
- Replication lag — delay между master и replica
- Hot keys — миллионы запросов к одному ключу → sharding кэша

### 5. Масштабирование записи
**Когда:** >10k writes/sec, одна БД не справляется  
**Решения:**
- **Sharding** — горизонтальное партиционирование (по user_id, geo, hash)
- **Vertical partitioning** — разные типы данных в разные БД
- **Очередь** — сгладить пики нагрузки
- **Batching** — группировать несколько записей в одну транзакцию

**Ключ шардирования:**
- ✓ Равномерное распределение (hash, consistent hashing)
- ✗ Плохие ключи: timestamp (весь новый трафик на один shard), boolean (два shard — дисбаланс)

### 6. Работа с большими файлами
**Когда:** видео, фото, документы >10 МБ  
**Решение:**
- **Presigned URL** — клиент загружает напрямую в S3, сервер не посредник
- **Multipart upload** — файл разбит на части (5 МБ каждая), загрузка параллельно
- **CDN** — скачивание через CloudFront/Cloudflare

**Поток загрузки:**
1. Клиент → API: `POST /upload/init` → presigned URL (временный, с правами на PUT)
2. Клиент → S3: multipart upload
3. S3 → webhook → API: `upload_complete` → обновить БД (статус `completed`)

**Нюансы:**
- Жизненный цикл: удаление неоконченных upload через 7 дней
- Синхронизация: метаданные в БД vs объект в S3 (eventual consistency)

### 7. Многошаговые процессы
**Когда:** онбординг, заказ (резервирование → оплата → отгрузка), long-running workflow  
**Решения:**
- **Temporal** / AWS Step Functions — durable execution, автоматические retry
- **Event sourcing** — каждый шаг генерирует событие → следующий шаг
- **Saga pattern** — компенсационные транзакции при откате

**Правило:** не изобретай велосипед — Temporal обрабатывает 99% случаев.

---

## Важные цифры (2026)

**Источник:** [NowInterview — Важные цифры](https://nowinterview.ru/learn/system-design/core-concepts/numbers-to-know)

### Современное железо (AWS примеры)

| Ресурс | Значение | Инстанс |
|---|---|---|
| **RAM** | 512 ГБ | M6i.32xlarge |
| **RAM (memory-optimized)** | 4 ТБ | X1e.32xlarge |
| **RAM (ultra)** | 24 ТБ | U-24tb1.metal |
| **SSD** | 60 ТБ | i3en.24xlarge |
| **HDD** | 336 ТБ | D3en.12xlarge |
| **Сеть внутри дата-центра** | 10–20 Гбит/с | Стандарт |
| **Сеть cross-zone** | 100 Мбит/с – 1 Гбит/с | Средний |
| **Latency внутри региона** | 1–2 мс | — |
| **Latency между регионами** | 50–150 мс | — |

### Устаревшие правила (НЕ используй)

❌ «Делите БД на шарды по 100 ГБ» — современный PostgreSQL держит 10+ ТБ на одной машине  
❌ «Избегайте больших объектов в памяти» — 512 ГБ RAM позволяют держать огромные структуры  
❌ «Храните только горячие данные» — S3 практически бесконечен, стоимость упала в 10×

### Правильные ориентиры (2026)

- **PostgreSQL:** 10k–50k QPS на одной машине с индексами
- **Redis:** 100k–500k ops/sec на одной ноде
- **S3:** практически бесконечен, 5500 PUT/sec на prefix
- **Cassandra:** линейное масштабирование записи, 1 млн writes/sec при 100 нодах

**Вывод:** не усложняй дизайн преждевременно. Начинай с одной БД, добавляй sharding только когда упрёшься в лимиты железа.

---

## Разборы задач (примеры)

**Источник:** [NowInterview — Разборы задач](https://nowinterview.ru/learn/system-design/problem-breakdowns/allowedgram)

Полные разборы с требованиями, API, диаграммами, trade-offs и ожиданиями по уровням (Middle/Senior/Staff+):

- **Instagram** — fan-out on write, multipart upload, CDN
- **Bit.ly** — URL shortener, base62 encoding, collision handling
- **Rate Limiter** — sliding window, token bucket, distributed counters
- **LeetCode** — code execution, sandboxing, real-time judging
- **Ticketmaster** — inventory management, pessimistic locking, race conditions
- **Google Docs** — CRDT, OT, WebSocket, stateful servers
- **Payment System** — idempotency, 2PC, event sourcing, ACID
- **Distributed Cache** — consistent hashing, replication, eviction policies

---

## Технологии (шпаргалка)

**Источник:** [NowInterview — Технологии](https://nowinterview.ru/learn/system-design/in-a-hurry/technologies)

Детальные статьи по каждой технологии (когда использовать, trade-offs, примеры):

- **Redis** — кэш, сессии, leaderboards, sorted set для timeline
- **Kafka** — event streaming, fan-out, log aggregation
- **Cassandra** — массивная запись, временные ряды, eventual consistency
- **DynamoDB** — managed NoSQL, auto-scaling, single-digit ms latency
- **PostgreSQL** — ACID, сложные запросы, JSON-поля
- **Elasticsearch** — полнотекстовый поиск, агрегации, логи
- **ClickHouse** — аналитика, OLAP, колоночное хранилище
- **API Gateway** — rate limiting, auth, routing, circuit breaker
- **Temporal** — durable workflows, retry, saga, long-running tasks

---

## Как учиться

1. **Структура** (5 мин): изучи framework интервью — не тратишь время на ненужное
2. **Паттерны** (1 неделя): разбери все 7 паттернов — 80% задач сводятся к ним
3. **Технологии** (2 недели): прочти про Redis, Kafka, Cassandra, PostgreSQL — когда что использовать
4. **Разборы задач** (2 недели): реши 5–10 задач, записывай на доску, засекай 45 мин
5. **Mock interviews** — найди партнёра, по очереди интервьюируйте друг друга

**Главное:** не заучивай решения — понимай trade-offs. Интервьюер всегда спросит «А почему не X вместо Y?»

---

## Полезные ссылки

- [NowInterview — полный курс System Design](https://nowinterview.ru/learn/system-design/in-a-hurry/introduction) (русский)
- [Excalidraw](https://excalidraw.com/) — рисование диаграм на интервью
- [System Design Primer](https://github.com/donnemartin/system-design-primer) (английский, GitHub)
- [ByteByteGo](https://bytebytego.com/) — книги и курсы Alex Xu (английский)