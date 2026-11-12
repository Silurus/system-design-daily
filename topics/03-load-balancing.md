# 03. Балансировка нагрузки

**Дата:** 2026-11-10  
**Статус:** пройдена

## Суть

**Load balancer** распределяет входящий трафик между N серверами. Два основных типа: **L4** (транспортный, видит IP:port) и **L7** (прикладной, видит HTTP). **Health checks** автоматически исключают упавшие инстансы из ротации.

## Алгоритмы

| Алгоритм | Как работает | Когда |
|---|---|---|
| Round-robin | По очереди | Однородная нагрузка |
| Least connections | На сервер с наименьшим числом активных соединений | Запросы с разным временем обработки |
| Weighted | Пропорционально мощности (новый/слабый сервер получает меньше) | Разнородные инстансы |
| IP hash | Один клиент → один сервер | Sticky sessions |

## L4 vs. L7

| | L4 (HAProxy, AWS NLB) | L7 (nginx, AWS ALB, Envoy) |
|---|---|---|
| **Видит** | IP:port, TCP/UDP | HTTP: URL, headers, cookies |
| **Latency** | ~1 мс (нет парсинга HTTP) | ~5–10 мс |
| **Маршрутизация** | Только по IP:port | По URL, cookie, header |
| **SSL termination** | Нет | Да |
| **Sticky sessions** | Только IP hash | По cookie (гибче) |
| **Когда** | Высокий QPS, low latency, БД, TCP | Микросервисы, A/B тест, canary deploy, gRPC |

## Health checks

```
GET /health → 200 OK (за 100 мс) → healthy
GET /health → timeout / 500          → unhealthy
                                       (3 подряд fail → out of rotation)
```

**Проблема:** `/health` возвращает 200, но сервер отвечает за 5 сек вместо 100 мс. Балансировщик считает его healthy, но пользователи получают timeout.

**Решение:** threshold на latency в health check (`response_time > 1s → unhealthy`) или **passive health check** (балансировщик смотрит на реальные запросы, а не только `/health`; если 5xx превышают порог → инстанс out of rotation).

## Схема: L7 балансировщик

```
         Clients
            │
     ┌──────▼──────┐
     │   L7 LB     │  ← SSL termination
     │  (nginx)    │  ← маршрутизация по /api, /static
     └──┬────┬─────┘
        │    │
  ┌─────▼┐ ┌─▼────┐ ┌──────┐
  │ App1 │ │ App2 │ │ App3 │  ← stateless инстансы
  └──────┘ └──────┘ └──────┘
              │
       ┌──────▼──────┐
       │  PostgreSQL │
       └─────────────┘
```

## Canary deploy через L7

```
L7 LB: 99% трафика → App v1.0
        1% трафика → App v1.1 (canary)
```

Возможно только на L7 — балансировщик видит HTTP и умеет делить трафик по весу. L4 этого не умеет.

## gRPC и балансировка

gRPC работает поверх **HTTP/2**, где один TCP-коннект переиспользуется для множества запросов (multiplexing). Обычный L7 делает **connection-level balancing**: клиент открывает одно соединение → все запросы идут на один бэкенд. Горизонтальное масштабирование не работает.

**Решения:**

| Вариант | Как | Минус |
|---|---|---|
| gRPC-aware L7 (Envoy, nginx grpc_pass) | Балансирует каждый gRPC stream | Сложнее настройка |
| Client-side balancing (gRPC built-in) | Клиент сам резолвит DNS и распределяет | Нет централизованного управления |
| L4 | Работает, но теряем HTTP-маршрутизацию | Нет canary, SSL offload |

> **Исправление:** AWS ALB поддерживает gRPC с 2020 года (HTTP/2 end-to-end).

**Kubernetes + kube-proxy + gRPC — стандартная ловушка:** kube-proxy работает на L4, клиент открывает одно HTTP/2-соединение на Service IP, и все запросы навсегда уходят на один под. Решение — Envoy или client-side balancing.

## WebSocket и балансировка

WebSocket начинается как HTTP upgrade, после чего TCP становится full-duplex каналом. Работает с обоими типами балансировщиков.

**Sticky sessions обязательны**: клиент должен переподключаться к тому же бэкенду, где хранится его состояние (или state вынесен в Redis).

| Тип sticky | Как | Минус |
|---|---|---|
| L4 IP hash | Весь трафик с одного IP → один бэкенд | При NAT/proxy все клиенты → один бэкенд |
| L7 по cookie | Балансировщик вставляет `Set-Cookie: lb=backend1` | Чуть сложнее конфигурация |

```nginx
location /ws {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 86400;  # 24h для long-lived connection
}
```

**Slack:** L7 (nginx) + sticky sessions по cookie + Redis pub/sub между бэкендами для маршрутизации сообщений между разными инстансами.

## Envoy

Envoy — L7-прокси на C++, написанный в Lyft в 2016 году. CNCF graduated project (как Kubernetes). Главное отличие от nginx — получает конфигурацию **по API (xDS)**, а не из файла, что делает его удобным для динамических окружений.

### xDS API (control plane → data plane)

```
        ┌───────────────┐
        │ Control plane │  (Istio istiod, Envoy Gateway…)
        │  смотрит в k8s│
        └──────┬────────┘
               │ xDS (gRPC-стрим, без перезапуска)
     ┌─────────┼─────────┐
     ▼         ▼         ▼
  [Envoy]   [Envoy]   [Envoy]   ← data plane, через них идёт трафик
```

| xDS API | Что настраивает |
|---|---|
| LDS | Порты (listeners) |
| RDS | Маршруты (`/orders` → orders-cluster) |
| CDS | Список кластеров (сервисов) |
| EDS | Конкретные IP подов |
| SDS | TLS-сертификаты |

Поды в k8s постоянно меняют IP — control plane отслеживает k8s API и через EDS мгновенно сообщает Envoy новые адреса без перезапуска.

### Что умеет Envoy

- **Протоколы:** HTTP/1.1, HTTP/2, HTTP/3, gRPC, TCP/UDP, Redis, MongoDB
- **Балансировка по запросам (не по соединениям):** round robin, least request, ring hash, Maglev
- **Устойчивость:** таймауты, повторы, circuit breaker, rate limiting
- **Outlier detection (пассивный health check):** если под начал отдавать 5xx или замедлился — временно исключается из ротации без активного `/health` probe
- **Безопасность:** mTLS между сервисами, сертификаты обновляются на лету
- **Наблюдаемость:** метрики по каждому upstream, distributed tracing, access-логи

### Sidecar-паттерн в Kubernetes

```
Pod A                               Pod B
┌──────────────────┐   mTLS   ┌──────────────────┐
│ [app]  [Envoy]   │ ──────── │  [Envoy]  [app]  │
└──────────────────┘          └──────────────────┘
     iptables перехватывает весь трафик → Envoy
```

- Приложение обращается к `http://orders` как обычно
- iptables перехватывает трафик и направляет через sidecar-Envoy
- mTLS, повторы, трассировка — на уровне инфраструктуры, без кода в приложении
- Не зависит от языка: C#, Go, Java получают одинаковые гарантии

### Цена sidecar

- **Latency:** ~1 мс на вызов (два прокси на каждый запрос)
- **Память:** десятки МБ на под → при тысячах подов заметные расходы
- **Сложность:** конфигурацию вручную не пишут, нужен control plane

## Envoy vs. Istio

**Envoy** — это просто L7-прокси с xDS API. Он ничего не знает о Kubernetes, не следит за подами, не выдаёт сертификаты. Просто принимает трафик и проксирует по правилам, которые ему прислали через xDS.

**Istio** — система управления, которая использует Envoy как исполнителя. Istio следит за k8s API, решает какой трафик куда направить, выдаёт сертификаты, выполняет ротацию — и всё это транслирует в xDS-конфигурацию для Envoy.

```
Аналогия: Envoy — водитель, который едет туда, куда ему говорят.
          Istio — диспетчер, который знает карту и даёт маршрут.
```

**Связь:**
- **Envoy без Istio**: можно использовать самостоятельно как Ingress, API Gateway или L7-балансировщик (но нужно писать xDS-конфиг руками — сложно)
- **Istio без Envoy**: невозможно — Envoy это исполнитель, через который идёт реальный трафик

```
Istio (istiod, control plane)   ← принимает решения, следит за k8s
    │ xDS API (gRPC-стрим)
    ▼
Envoy (sidecar, data plane)     ← несёт реальный трафик по указаниям
    │
    ▼
Другой сервис
```

## Istio

Istio — самый популярный **service mesh** для Kubernetes. Добавляет в k8s сеть управления между сервисами: безопасность, наблюдаемость, трафик — без изменений в коде.

### Компоненты

```
┌─────────────────────────────────────────────┐
│               Control Plane                 │
│                 istiod                      │
│  ┌──────────┐ ┌───────────┐ ┌───────────┐  │
│  │  Pilot   │ │  Citadel  │ │  Galley   │  │
│  │ (xDS,    │ │ (сертифи- │ │  (валида- │  │
│  │ routing) │ │  каты)    │ │   ция)    │  │
│  └──────────┘ └───────────┘ └───────────┘  │
└──────────────────┬──────────────────────────┘
                   │ xDS
   ┌───────────────┼──────────────────┐
   ▼               ▼                  ▼
[Pod A]          [Pod B]           [Pod C]
[app+Envoy]    [app+Envoy]       [app+Envoy]
       ↑ sidecar-inject автоматически
```

**istiod** (единый процесс управления):
- **Pilot** — формирует xDS-конфигурацию для Envoy: маршруты, балансировка
- **Citadel** — выдаёт и ротирует mTLS-сертификаты (SPIFFE/X.509)
- **Galley** — валидирует конфигурацию Istio перед применением

### Что даёт Istio

**Трафик:**
- Canary deploy через `VirtualService` (10% на v2 без изменений в Deployment)
- Retry, timeout, circuit breaker — декларативно в YAML
- Traffic mirroring: дублировать трафик на staging-сервис без влияния на продакшн

**Безопасность:**
- mTLS между всеми сервисами по умолчанию (zero-trust)
- `AuthorizationPolicy`: «orders может обращаться к payments, notifications — нет»
- Сертификаты ротируются автоматически

**Наблюдаемость:**
- Метрики latency, error rate, throughput по каждой паре сервисов без кода
- Интеграция с Prometheus + Grafana, Jaeger/Zipkin для distributed tracing
- Kiali — граф зависимостей сервисов в реальном времени

### Пример: canary через VirtualService

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: orders
spec:
  hosts:
  - orders
  http:
  - route:
    - destination:
        host: orders
        subset: v1
      weight: 90
    - destination:
        host: orders
        subset: v2
      weight: 10  # 10% трафика на новую версию
```

Без Istio такой же canary потребовал бы менять количество реплик (неточно) или писать логику в коде.

### Ambient mode (без sidecar)

С 2024 года Istio поддерживает **ambient mode**: sidecar-контейнер убирается из подов. Вместо него:
- На каждом **узле** — `ztunnel` (L4, mTLS, минимальная цена)
- Там, где нужна L7-логика — `waypoint proxy` (Envoy, создаётся per-namespace или per-service)

Это снижает расход памяти и упрощает операционку (не нужно перезапускать поды при обновлении sidecar).

### Когда Istio нужен, а когда — нет

| Ситуация | Решение |
|---|---|
| 3–5 микросервисов | Простой L7 балансировщик (nginx, ALB), повторы в коде |
| 10+ микросервисов, нужен mTLS, canary | **Istio** оправдан |
| Нужна только балансировка в k8s | Envoy Gateway или nginx Ingress без mesh |
| Высокая нагрузка, минимальная latency | Istio ambient mode или отказ от mesh |

> **Альтернативы:** Linkerd (собственный прокси на Rust, проще Istio), Consul Connect (от HashiCorp).
> **Исправление:** Linkerd не использует Envoy — у него свой прокси linkerd2-proxy на Rust.

## Примеры из реальных систем

**Cloudflare:** L7-балансировка на краю CDN → маршрутизация по географии, DDoS-защита

**Uber:** L7 (Envoy service mesh) для микросервисов + L4 (AWS NLB) для БД

**Netflix:** Zuul (L7) — API gateway; Ribbon — client-side balancing (устарел, заменён Spring Cloud LoadBalancer)

**Slack:** L7 (nginx) + sticky sessions по cookie + Redis pub/sub для WebSocket

**Google:** Envoy — основа их внутренней инфраструктуры; они и продвигали xDS как стандарт

## Ключевые выводы

1. **L4** — быстро, для БД и TCP-трафика; не умеет маршрутизацию по HTTP.
2. **L7** — видит HTTP, SSL offload, canary deploy, sticky sessions по cookie; чуть медленнее.
3. **Least connections лучше round-robin** при разнородном времени обработки запросов.
4. **Health check по latency**, не только по статус-коду.
5. **gRPC + k8s kube-proxy = ловушка**: все запросы идут на один под. Решение — Envoy или client-side balancing.
6. **WebSocket работает с L7** — sticky sessions по cookie точнее, чем IP hash.
7. **Envoy = data plane + xDS API** — получает конфигурацию от control plane без перезапуска.
8. **Istio нужен при 10+ микросервисах** — даёт mTLS, canary, наблюдаемость без кода в сервисах.
9. **Control plane / data plane** — ключевое разделение: Envoy несёт трафик, istiod принимает решения.
