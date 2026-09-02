# Проект 2: Kafka — Асинхронный сервис нотификаций (на примере Додо Пицца)

## Бизнес-сценарий
Проектирование отказоустойчивой и масштабируемой системы отправки Push-уведомлений клиентам сети пиццерий при изменении статусов заказов на кухне (например, переход в статус `COOKING` — «Пицца в печи!»). Система построена на брокере сообщений Apache Kafka, что позволяет полностью изолировать Сервис Заказов от CRM-системы, сглаживать пиковые нагрузки в пятницу вечером и гарантировать доставку сообщений.

---

## 1. Архитектурные роли систем
*   **Producer (Поставщик данных)** — Бэкенд Сервиса Заказов. Принимает REST-запрос от фронтенда кухни, обновляет статус заказа в базе данных и публикует событие в брокер Kafka.
*   **Consumer (Потребитель данных)** — Сервис Нотификаций (CRM). Работает по Pull-модели, непрерывно запрашивая (Poll) брокер на наличие новых сообщений и отправляя Push на смартфоны клиентов через внешние шлюзы (Apple APNS / Google FCM).
*   **Основной топик (Topic)** — `order.status-changed`. Настроен ключ партиционирования по `order_id`, что гарантирует строгую последовательность обработки статусов одного конкретного заказа внутри одной партиции.

---

## 2. Асинхронная диаграмма последовательности (Sequence Diagram)

Диаграмма наглядно демонстрирует слабую связность (loose coupling) систем через шину данных Kafka. Сервис заказов не ждет ответа от CRM и моментально освобождает поток для работы кухни.

![Kafka Sequence Diagram](https://www.plantuml.com/plantuml/png/PLJjJYjL5Fq_Jx4TVzmskRLSVpQfkVJ3Yq5fq_R6c9YGGsUaaxOpTJxm6cFIEnNKWAByD562-W03KgW5oYkSyqQkiwSPAeIGdfcpzjfhhtNcl0n2uOVHS628AFJSQBXZ-qRGTzoHyCMGcjXqkiv8k65JM9RZxjABjPNzYlTcvRqXkg7daobFvLJEPQB6wiWa4L15z44xUY03vcyrbWjvfsAIzndycE4V-fuR6ZlU6y1E09ce6E2flCFVJFvB07CbDiQwUgOEr86S0ZEJCvFU_TZPxOMVsOE1zzMZfh-gdr2icIu_TtFoZ8ikGF2MvBLCj0hyBkH5VgcxsxTTox14A7P4O6i6HYGaBx6087MezXbRgnXVHdO4pARueY-eudiOMdVv4uGGAUSbydpBzj-3vs4K5BezuUxQLjgAY_wljjhwfA3UQd4gXkPRcO2jwhb1DBJzj0eG0m89NcNc_oPlq6og3Ken0yU6dKQvJF8lyC5jzOCM3YT1VQ2XsdosDhSXHgElaNlOorEvR71oLFWWpQH4ydUCVGKer80d9bE_KT_ZFrP79F-H2uY5WNAk9gR1LRfywMI9cersXufxBufiHL3ynh6-BQQ6K2xzBPlLHcFpewrNPbuFvUo9qCwYC9OBaCBo4hrkrjOxTKhPg5tlq9BasPBa6NswKM_LoR7AkjTgFNkEcVwGzseYpzctEOldQU0E7iwHSAmDfHmtsNQivzHbJtdvL7FMuS6szzVMgB59E77WqkGJd3XDBk-utuqE8MzOzjDvjpmyUNksdyfzhZE7vb_qoK2A--bdn3nJA4OKC_0VwY7madKtey4WRw1kTUXJ5DoaQ20y82WLWzefZyx3_wYL0NHCuBeco5yUPfbauV0nP5wUEHLu0k0rsCOo0GaVbiU6DbgrUckxtLdll6vlLpVMjrxLQmRhp5ZEr7To0OvRI1YlHXFhEqD_6IM-918SwIIx2x14e5Te8I_uUj5plcDcr9oU3XuqzreXU6VCsuo2Na7xe2MYDmyAmfdwMTkJw15k-QgAiGlZLp9Z6uoNI0ttwNy0)

---

## 3. Структура асинхронного сообщения
Сообщение спроектировано с четким разделением на технические метаданные (для трассировки и логирования) и бизнес-полезную нагрузку (`payload`).

<details>
<summary>📂 Нажмите, чтобы развернуть и посмотреть JSON сообщения</summary>

```json
{
  "event_id": "evt_883a-11ee-b9d1-0242ac120002",
  "event_type": "ORDER_COOKING",
  "created_at": "2026-09-01T19:24:00Z",
  "payload": {
    "order_id": "ord_994821",
    "client_id": "clnt_33019"
  }
}
```

</details>

*   `event_id` — Уникальный UUID события. Необходим для обеспечения идемпотентности консьюмера на принимающей стороне.
*   `event_type` — Маркер бизнес-события (`ORDER_CREATED`, `ORDER_COOKING`, `ORDER_DELIVERING`, `ORDER_CANCELLED`) для быстрой маршрутизации логики внутри CRM без полного парсинга объекта.

---

## 4. Обработка инфраструктурных сбоев
В случае недоступности внешних шлюзов нотификаций (Apple APNS / Google FCM) включается трехступенчатый паттерн обработки ошибок:
1.  **Immediate Retry**: CRM выполняет 3 мгновенных повтора отправки с интервалом в 2 секунды.
2.  **Retry Topic**: При затяжном сбое сообщение перекладывается в служебный топик `order.status-changed.retry`. Специальный консьюмер вычитывает его оттуда с задержкой в 5 минут, давая внешним шлюзам время восстановиться.
3.  **DLQ (Dead Letter Queue)**: Если после 10 попыток Push так и не ушел, сообщение отправляется в топик `order.status-changed.DLQ`. Основная очередь брокера при этом не блокируется. Сообщения из DLQ отправляются на мониторинг инженерам поддержки для ручного разбора.

---

## 5. Обработка дублей сообщений
Apache Kafka гарантирует доставку сообщений по принципу *At-Least-Once* (Минимум один раз). При кратковременных сбоях сети между брокером и CRM возможна повторная отправка одного и того же события.
*   **Решение** — паттерн **Idempotent Consumer**. Перед физической отправкой Push-уведомления CRM выполняет атомарную операцию проверки ключа `event_id` в распределенном кэше Redis.
*   Если ключ найден — сообщение признается дубликатом, логируется и сбрасывается (`Ack`) без отправки повторного Push клиенту.
*   Если ключа нет — CRM отправляет Push и записывает `event_id` в Redis со сроком жизни (TTL) 24 часа.

