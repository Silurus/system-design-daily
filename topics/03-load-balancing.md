# 03. Балансировка нагрузки

**Дата:** 2026-11-10  
**Статус:** пройдена (вопросы заданы, ответы на проверке)

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
| **Когда** | Высокий QPS, low latency, БД | Микросервисы, A/B тест, canary deploy |

## Health checks

```
GET /health → 200 OK (за 100 мс) → healthy
GET /health → timeout / 500          → unhealthy
                                       (3 подряд fail → out of rotation)
```

**Проблема:** `/health` возвращает 200, но сервер отвечает за 5 сек вместо 100 мс. Балансировщик считает его healthy, но пользователи получают timeout.

**Решение:** threshold на latency в health check (`response_time > 1s → unhealthy`) или active/passive health check через балансировщик (passive — смотрит на реальные ответы, не только `/health`).

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

## Примеры

**Cloudflare:**
- L7-балансировка на краю CDN → маршрутизация по географии, DDoS-защита
- Видит HTTP → отсекает вредоносные запросы до origin-сервера

**Uber:**
- L7 (Envoy) для service mesh — маршрутизация по версиям микросервисов (canary)
- L4 (AWS NLB) для БД — минимальная latency, нет парсинга HTTP

**Netflix:**
- Zuul (L7) — API gateway, аутентификация, rate limiting, маршрутизация
- Ribbon — client-side load balancing между микросервисами (least connections)

## Canary deploy через L7

```
L7 LB: 99% трафика → App v1.0
        1% трафика → App v1.1 (canary)
```

Возможно только на L7 — балансировщик видит HTTP и умеет делить трафик по весу.

**L4** этого не умеет — не видит HTTP, только TCP.

## Ключевые выводы

1. **L4** — быстро, низкая latency, для БД и TCP-трафика; не умеет маршрутизацию по HTTP.
2. **L7** — видит HTTP, SSL offload, canary deploy, sticky sessions по cookie; чуть медленнее.
3. **Least connections лучше round-robin** при разнородном времени обработки запросов.
4. **Health check по latency**, а не только по статус-коду — иначе «живой, но медленный» сервер загрязняет ротацию.
