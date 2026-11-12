# 04. Кэширование

**Дата:** 2026-11-12

---

## Суть

Кэш — это быстрое промежуточное хранилище между приложением и медленным источником данных (база, API, диск). Цель — снизить latency и уменьшить нагрузку на источник. Кэш работает по принципу **locality**: недавно запрошенные данные скорее всего запросят снова (temporal locality), а рядом лежащие данные часто нужны вместе (spatial locality).

**Ключевые метрики:**
- **Hit rate** (попадания) — процент запросов, найденных в кэше. Для эффективности нужен 80–90%+.
- **Latency** — задержка ответа. RAM (Redis/Memcached) даёт ~1 мс против ~10–50 мс к БД.
- **Eviction** — вытеснение старых записей при заполнении. Алгоритмы: LRU (Least Recently Used), LFU (Least Frequently Used), TTL (Time To Live).

---

## Стратегии кэширования

### 1. Cache-Aside (Lazy Loading)

**Как работает:**
1. Приложение сначала проверяет кэш.
2. **Cache miss** → читает из БД, кладёт результат в кэш, возвращает клиенту.
3. **Cache hit** → сразу отдаёт из кэша.

**Запись:** идёт напрямую в БД, кэш **не обновляется** автоматически. При следующем чтении — cache miss, данные перезагрузятся.

**Плюсы:**
- Простота: кэш заполняется только востребованным.
- Отказ кэша не ломает систему (только замедляет).

**Минусы:**
- Первый запрос после записи идёт в БД (stale read).
- **Cache stampede**: если популярная запись протухла, тысячи запросов одновременно полезут в БД.

**Когда использовать:** чтение >> записи, данные меняются редко (профили пользователей, настройки).

```csharp
public async Task<User> GetUserAsync(int userId)
{
    var cacheKey = $"user:{userId}";
    var cached = await _redis.StringGetAsync(cacheKey);
    
    if (cached.HasValue)
        return JsonSerializer.Deserialize<User>(cached);
    
    // Cache miss
    var user = await _db.Users.FindAsync(userId);
    await _redis.StringSetAsync(cacheKey, JsonSerializer.Serialize(user), 
        expiry: TimeSpan.FromMinutes(10));
    
    return user;
}
```

---

### 2. Write-Through

**Как работает:**
1. Запись идёт **одновременно** в кэш и в БД (синхронно).
2. Клиент получает подтверждение только после записи в оба места.
3. Чтение всегда из кэша (он всегда актуален).

**Плюсы:**
- Кэш и БД всегда синхронны (нет stale data).
- Высокий hit rate для свежезаписанных данных.

**Минусы:**
- **Высокая latency записи** (ждём БД + кэш).
- Пишем в кэш даже то, что никогда не прочитают (wasteful).

**Когда использовать:** критична консистентность + данные часто читаются сразу после записи (корзина покупок, инвентарь товаров).

```csharp
public async Task UpdateInventoryAsync(int productId, int quantity)
{
    var product = new Product { Id = productId, Quantity = quantity };
    
    // Пишем в оба места синхронно
    await _db.Products.UpdateAsync(product);
    await _redis.StringSetAsync($"product:{productId}", 
        JsonSerializer.Serialize(product), 
        expiry: TimeSpan.FromHours(1));
}
```

---

### 3. Write-Back (Write-Behind)

**Как работает:**
1. Запись идёт **только в кэш**, клиент сразу получает ответ.
2. Данные асинхронно (батчами или по таймеру) сбрасываются в БД в фоне.

**Плюсы:**
- **Минимальная latency записи** (только RAM).
- Можно батчить несколько записей в одну транзакцию БД (пропускная способность × 10).

**Минусы:**
- **Риск потери данных** при отказе кэша до flush (нужна репликация кэша).
- Сложность: нужна очередь, логика retry, порядок операций.

**Когда использовать:** запись >> чтения, допустима eventual consistency + есть репликация кэша (счётчики просмотров, метрики, логи событий).

```csharp
// Запись в кэш
await _redis.StringSetAsync($"view_count:{videoId}", count);
await _redis.ListLeftPushAsync("write_back_queue", $"video:{videoId}:{count}");

// Фоновый worker батчит и сбрасывает в БД
while (true)
{
    var batch = await _redis.ListRangeAsync("write_back_queue", 0, 99);
    await _db.BulkUpdateViewCountsAsync(batch);
    await _redis.ListTrimAsync("write_back_queue", 100, -1);
    await Task.Delay(5000); // каждые 5 сек
}
```

---

## Инвалидация кэша

**Проблема:** после изменения данных в БД кэш устаревает. Как синхронизировать?

### Стратегии:

1. **TTL (Time To Live)** — запись протухает через N секунд. Просто, но данные stale до истечения TTL.
2. **Явная инвалидация** — после записи в БД удаляем ключ из кэша. При следующем чтении — cache miss, загрузим актуальные данные.
   ```csharp
   await _db.Users.UpdateAsync(user);
   await _redis.KeyDeleteAsync($"user:{user.Id}"); // инвалидация
   ```
3. **Event-driven invalidation** — БД шлёт событие (CDC: Change Data Capture через Debezium/Maxwell), воркер чистит кэш.
4. **Cache-aside с версионностью** — ключ содержит версию данных (`user:{id}:v{version}`). Обновление = новая версия, старые ключи сами протухнут.

**Trade-off:** TTL прост, но даёт stale data; явная инвалидация точна, но требует координации при высокой нагрузке.

---

## Cache Stampede (гром толпы)

**Проблема:** популярный ключ (например, главная страница) протухает. Тысячи запросов одновременно обнаруживают cache miss и идут в БД → перегрузка БД, spike latency.

### Решения:

1. **Probabilistic early expiration** — случайно обновляем кэш **до** истечения TTL (за 10% времени до окончания). Не даёт всем промахнуться одновременно.
   ```csharp
   var timeLeft = _redis.KeyTimeToLive(key);
   if (timeLeft < ttl * 0.1 && Random.Shared.NextDouble() < 0.5)
       RefreshCache(key); // обновим фоново
   ```

2. **Lock на перезагрузку** — только один запрос идёт в БД, остальные ждут или получают stale данные.
   ```csharp
   if (!await _redis.LockTakeAsync($"lock:{key}", workerId, TimeSpan.FromSeconds(10)))
   {
       await Task.Delay(50); // подождём, кто-то уже загружает
       return await _redis.StringGetAsync(key); // повторная попытка
   }
   // Только мы пришли первыми — загружаем
   var data = await _db.GetAsync(key);
   await _redis.StringSetAsync(key, data, ttl);
   await _redis.LockReleaseAsync($"lock:{key}", workerId);
   ```

3. **Background refresh** — воркер обновляет топ-N горячих ключей за 1–2 минуты до истечения TTL. Пользователи всегда видят свежий кэш.

---

## Пример: Instagram Feed

**Задача:** лента постов пользователя (read-heavy, миллионы QPS).

**Решение:**
- **Стратегия:** cache-aside с TTL 5 минут.
- **Кэш:** Redis Cluster, ключ `feed:{user_id}`, значение — JSON массив из 50 последних постов.
- **Cache stampede:** топ-1000 популярных аккаунтов (селебрити) обновляются фоново воркером каждые 4 минуты (до протухания). Остальные — по требованию.
- **Инвалидация:** новый пост → append в кэш (если ключ есть) + обновление TTL. Если ключа нет — ничего не делаем, загрузится при следующем чтении.

**Результат:** hit rate 95%, latency p99 = 2 мс (против 50 мс к БД), нагрузка на Cassandra снижена в 20 раз.

---

## Вопросы для обсуждения

1. **Выбор стратегии.** Профили пользователей: 100 млн DAU, каждый в среднем открывает 50 профилей в день, а свой профиль меняет раз в месяц. Посчитай QPS чтения и записи. Какую стратегию выбрать — cache-aside или write-through — и почему? Какой TTL и как инвалидировать после правки профиля?

2. **Cache stampede.** Карточка видео на главной YouTube протухает в кэше. За секунду приходит 50 000 запросов, все промахиваются. БД держит 5 000 QPS. Что произойдёт? Какое из трёх решений (probabilistic refresh, lock, background worker) выбрать и почему?

3. **Write-back риски.** Счётчик лайков пишется в Redis (write-back), батч сбрасывается в PostgreSQL каждые 10 секунд. Redis-нода падает между flush. Сколько данных потеряно? Назови одно архитектурное решение, которое снизит потери до нуля (и какую цену оно добавит).

Задавай вопросы.
