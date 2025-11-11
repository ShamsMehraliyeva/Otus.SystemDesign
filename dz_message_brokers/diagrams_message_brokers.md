# Архитектурные диаграммы системы

## 1. System Context Diagram (Контекстная диаграмма)

Диаграмма показывает взаимодействие системы с внешними акторами.

```mermaid
graph TB
    User[👤 Пользователь<br/>Customer]
    Admin[👤 Администратор<br/>Admin]
    System[🎵 Concert Management System<br/>Система управления концертами]
    Payment[💳 Payment Gateway<br/>Платежная система]
    Email[📧 Email Service<br/>Email сервис]

    User -->|Просмотр концертов<br/>Покупка билетов| System
    Admin -->|Управление концертами<br/>Управление залами| System
    System -->|Обработка платежей| Payment
    System -->|Отправка email<br/>уведомлений| Email
    System -->|Real-time<br/>уведомления| User

   
```


## 2. Network Protocol Usage Map

Карта использования протоколов в системе.

```mermaid
graph LR
    subgraph External["External Actors"]
        Browser[🖥️ Browser]
        Mobile[📱 Mobile App]
    end

    subgraph APILayer["API Layer"]
        Gateway[API Gateway]
    end

    subgraph Services["Microservices"]
        ES[Event Service]
        TS[Ticket Service]
        VS[Venue Service]
        NS[Notification Service]
    end

    Browser -->|REST/HTTP2<br/>HTTPS| Gateway
    Mobile -->|REST/HTTP2<br/>HTTPS| Gateway

    Gateway -->|REST/HTTP2| ES
    Gateway -->|REST/HTTP2| TS
    Gateway -->|REST Wrapper<br/>HTTP2| VS
    Gateway -->|WebSocket<br/>SignalR| NS

    TS -->|gRPC/HTTP2<br/>TLS| ES
    TS -->|gRPC/HTTP2<br/>TLS| VS

    NS -.->|WebSocket<br/>Push| Browser
    NS -.->|WebSocket<br/>Push| Mobile

    classDef rest fill:#4CAF50,stroke:#2E7D32,color:#fff
    classDef grpc fill:#2196F3,stroke:#1565C0,color:#fff
    classDef ws fill:#FFC107,stroke:#F57F17,color:#000

    class Browser,Mobile,Gateway,ES,TS rest
    class VS grpc
    class NS ws
```

## 3. Data Flow Diagram: Message Broker Patterns

Поток данных через RabbitMQ.

```mermaid
graph TB
    subgraph Publishers["📤 Publishers (Producers)"]
        EventSvc[Event Service]
        TicketSvc[Ticket Service]
    end

    subgraph RabbitMQ["🐰 RabbitMQ Message Broker"]

        subgraph PubSub["Pattern 1: Pub/Sub (Fanout)"]
            FanoutEx[Exchange: events.fanout<br/>Type: fanout]
            Q1[Queue:<br/>notifications.queue]
            Q2[Queue:<br/>analytics.queue]

            FanoutEx -->|Binding| Q1
            FanoutEx -->|Binding| Q2
        end

        subgraph P2P["Pattern 2: Point-to-Point"]
            DirectQ[Queue:<br/>payment.processing<br/>Direct Queue]
        end

        DLX[Dead Letter Exchange<br/>dlx.exchange]
        DLQ[Dead Letter Queue<br/>failed.messages]

        DLX --> DLQ
        Q1 -.->|Failed msgs| DLX
        Q2 -.->|Failed msgs| DLX
        DirectQ -.->|Failed msgs| DLX

    end

    subgraph Consumers["📥 Consumers"]
        NotifSvc[Notification Service<br/>Subscriber]
        AnalyticsSvc[Analytics Service<br/>Subscriber]
        Worker1[Payment Worker 1]
        Worker2[Payment Worker 2]
        Worker3[Payment Worker 3]
    end

    EventSvc -->|Publish:<br/>EventCreated<br/>EventUpdated| FanoutEx
    TicketSvc -->|Publish:<br/>TicketPurchased| FanoutEx
    TicketSvc -->|Publish:<br/>ProcessPayment| DirectQ

    Q1 -->|Consume| NotifSvc
    Q2 -->|Consume| AnalyticsSvc

    DirectQ -->|Consume<br/>Round Robin| Worker1
    DirectQ -->|Consume<br/>Round Robin| Worker2
    DirectQ -->|Consume<br/>Round Robin| Worker3

    NotifSvc -.->|ACK/NACK| Q1
    AnalyticsSvc -.->|ACK/NACK| Q2
    Worker1 -.->|ACK/NACK| DirectQ

    style FanoutEx fill:#FF9800,stroke:#E65100,color:#fff
    style DirectQ fill:#2196F3,stroke:#1565C0,color:#fff
    style DLX fill:#F44336,stroke:#C62828,color:#fff
    style DLQ fill:#F44336,stroke:#C62828,color:#fff
```

**Преимущества:**
- ✅ Низкая latency для клиента
- ✅ Decoupling - сервисы не знают друг о друге
- ✅ Отказоустойчивость - если consumer down, сообщения в очереди
- ✅ Легко добавлять новых consumers