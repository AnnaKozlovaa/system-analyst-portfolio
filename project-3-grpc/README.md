# Проект 3: gRPC — Микросервис расчета стоимости поездки (на примере Яндекс Такси)

## Бизнес-сценарий
Проектирование высоконагруженного синхронного взаимодействия внутренних систем. При открытии пользователем приложения такси Сервис Заказов (`Order Service`) должен мгновенно запросить у Сервиса Ценообразования (`Pricing Service`) расчет стоимости поездки сразу по всей сетке доступных тарифов. Выбор gRPC обусловлен необходимостью обеспечения минимальной задержки (low latency) и бинарного сжатия данных по протоколу HTTP/2.

---

## 1. Диаграмма последовательности (Sequence Diagram)
Взаимодействие происходит по синхронной схеме «запрос-ответ». Сервис Ценообразования агрегирует данные от внутреннего Гео-сервиса (количество машин в радиусе, коэффициент повышенного спроса Surge) и возвращает сформированный бинарный пакет обратно.

![gRPC Sequence Diagram](https:////www.plantuml.com/plantuml/png/VLHTJpDL4BxVNp6fDsrIXDVNzwQ9vXKYQABIb7f7OawxXtfYsQrdjuXtjE27CGJG2nDY_F32EnCAhPH2bxymvn_vJBQRNG9uiJbdFkQPcMVExEieLZRkxtOzrO_3eB_RqjQBlZ11JrcrIwlXVbtvlWawz6AbeAyZoBHDJmLnmUwfTXnQAl4vt_AK9pntmnAfY3wDWDeAzqbCYQZTeJiWleFNp1rmuWOymNd9KzmIlYJ-YozW6l2qH6zyP3gVnnjMwy3hvOcfz5xzGzfGiVvAVKtBz77OCbrDg-lhaZCpF8pWF_CynBz83dpNad8xA6zQNrlQqdRFj7L5K4XKzlym_fOs4P_m2Ce55iuvp2V0h5i4Wfu7g0kbvwjOjLIa0VepS9Ab8Wz2puNx7XJDWREXm-M6zasK8a3qTE0JIFKyePUMtirQhX7_veP2hFjHQW7aaDonAAGmualeHvo8I72ym6NgZY2K-PJ7a2QI_LGW-QpYBM0XfdJKKkRkqy6H6m0ZGPbt4EK-9Rw7OW8knZo3M6wylzKaUDn2w3JgQnLvAMOFk0k2lKz2tCCzRLD5DQEeHk9FKd9ABGasVT1irfVVGhrdGezqm7Dut20iZCDMVwVY2OwmaYEjgMwxtqNITMLrERIcOu8g-JgAJQ1Y4q9eTuqEui-CNwWm6vQtk0Wcw7_40gXEX5OUIotkctn-GZa9sp1DtQ4CcOIb6IAFtP4xWlRRR64mXgs-xMWFEMYfM3__1F1hK04azqCwLsX_pR8WXJj1YeJlw1rwkqfy_bG2s5wy-HBS_GvFy8fWw1VykK7MKQNOSA68Fv1kuBwJvCIZxCNZktdk9QH4304hLKo42K2apYXuZe2RxG3EabgYfHxvfEBdMK8JXHBpnodJRUYe5mQHBblzPLz7Cj4gMTtJ6AXFJMNDpivcJsPUyNpzQDJPuybtwcds2SULx0jVyHHdjlhWKYeVvqA-HC2KVLqiQVRR-3S3dCbZ5WCenkuKznFeLEP_X130OierUlbgXU1-o2EPjkZ4Jm1gz6ebi-KzvWltMWU-_FR_0m00)

---

## 2. Интерфейсный контракт (`pricing.proto`)
Вместо текстового JSON/YAML используется двоичный протокол Protocol Buffers v3. Поля размечаются целочисленными индексами для оптимизации сериализации данных. Полная спецификация вынесена в отдельный файл `pricing.proto` в текущей папке.

<details>
<summary>📂 Нажмите, чтобы развернуть и посмотреть .proto спецификацию</summary>

```protobuf
syntax = "proto3";

package pricing;

service PricingService {
  rpc CalculateFare (FareRequest) returns (FareResponse);
}

message LatLng {
  double latitude = 1;
  double longitude = 2;
}

message FareRequest {
  LatLng origin = 1;
  LatLng destination = 2;
  string promo_code = 3;
  string client_id = 4;
}

message TariffOption {
  string tariff_class = 1;
  int64 total_fare = 2;
  string currency = 3;
  int32 estimated_arrival_min = 4;
}

message FareResponse {
  string request_id = 1;
  repeated TariffOption tariff_options = 2;
}
```

</details>

---

## 3. Обработка ошибок на базе gRPC Status Codes
В gRPC отсутствует понятие HTTP-статусов. Системные и бизнес-исключения маппятся на 17 нативных gRPC кодов.

### Сценарий 1: Ошибка бизнес-валидации данных
Если Сервис Заказов передает некорректные или физически несуществующие географические координаты, Сервис Ценообразования прерывает расчет и возвращает ошибку **`INVALID_ARGUMENT` (Code 3)**. Для детального информирования клиентской стороны используется расширение `google.rpc.BadRequest`:

```json
{
  "code": 3,
  "status": "INVALID_ARGUMENT",
  "message": "Validation failed for incoming coordinates",
  "details": [
    {
      "@type": "://googleapis.com",
      "field_violations": [
        {
          "field": "origin.latitude",
          "description": "Latitude must be between -90.0 and 90.0. Provided: 150.0"
        }
      ]
    }
  ]
}
```

### Сценарий 2: Инфраструктурный сбой внутренних зависимостей
Если внутренний Гео-сервис распределения машин или база данных динамического спроса недоступны, Сервис Ценообразования возвращает код **`UNAVAILABLE` (Code 14)**. На стороне `Order Service` проектируется архитектурная политика отказоустойчивости (Circuit Breaker) — быстрый повтор запроса (Retry) или переключение на локальный показ пользователю статичных базовых тарифов региона.

