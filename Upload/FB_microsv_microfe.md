# Kiến Trúc Hệ Thống Mạng Xã Hội

---

## TASK 1 — Architecture Ownership & Communication Diagram


```mermaid
graph TD
    %% -- Styles --
    classDef client fill:#f8f9fa,stroke:#dee2e6,stroke-width:2px,color:#212529
    classDef infra fill:#e9ecef,stroke:#adb5bd,stroke-width:2px,color:#495057
    classDef mfe fill:#d1ecf1,stroke:#17a2b8,stroke-width:2px,color:#0c5460
    classDef gateway fill:#fff3cd,stroke:#ffc107,stroke-width:2px,color:#856404
    classDef service fill:#d4edda,stroke:#28a745,stroke-width:2px,color:#155724
    classDef db fill:#f8d7da,stroke:#dc3545,stroke-width:2px,color:#721c24
    classDef cache fill:#f5c6cb,stroke:#dc3545,stroke-width:2px,color:#721c24,shape:cylinder
    classDef broker fill:#c3e6cb,stroke:#28a745,stroke-width:2px,stroke-dasharray: 5 5,color:#155724
    classDef domainBoundary fill:#ffffff,stroke:#6c757d,stroke-width:2px,stroke-dasharray: 5 5

    %% -- Clients & Edge --
    Client([Client Mobile / Web]):::client
    
    subgraph "Infrastructure Layer"
        Cloudflare[Cloudflare DNS / CDN / WAF]:::infra
        NginxLB[Nginx Load Balancer]:::infra
    end
    
    Client -->|HTTPS| Cloudflare
    Cloudflare -->|HTTPS| NginxLB
    
    %% -- Frontend Layer --
    subgraph "Micro-frontend Layer (Next.js)"
        NewsfeedMFE[Newsfeed MFE]:::mfe
        ChatMFE[Chat MFE]:::mfe
        AdsMFE[Ads MFE]:::mfe
        SearchMFE[Search MFE]:::mfe
    end
    
    NginxLB --> NewsfeedMFE
    NginxLB --> ChatMFE
    NginxLB --> AdsMFE
    NginxLB --> SearchMFE
    
    %% -- API Gateway --
    APIGateway{API Gateway}:::gateway
    
    NewsfeedMFE -->|HTTPS| APIGateway
    ChatMFE -->|HTTPS| APIGateway
    AdsMFE -->|HTTPS| APIGateway
    SearchMFE -->|HTTPS| APIGateway
    
    %% -- DOMAIN BOUNDARIES (Backend Services & Data) --
    
    subgraph "DOMAIN 1 — Identity & Social"
        UserService[User Service]:::service
        UserDB[(User DB)]:::db
        UserService -->|OWNS| UserDB
        
        GraphService[Social Graph Service]:::service
        GraphDB[(Graph DB)]:::db
        GraphService -->|OWNS| GraphDB
    end

    subgraph "DOMAIN 2 — Content"
        ContentService[Content Service]:::service
        PgBouncer[PgBouncer\nConnection Pooling]:::infra
        
        subgraph "Content Database"
            ContentRouter{Database Router\nShard Key}:::infra
            ContentShard1[(Content Shard 1\nPrimary + Replica)]:::db
            ContentShard2[(Content Shard 2\nPrimary + Replica)]:::db
        end
        
        ContentService -->|USES| PgBouncer
        PgBouncer --> ContentRouter
        ContentRouter --> ContentShard1
        ContentRouter --> ContentShard2
    end
    
    subgraph "DOMAIN 3 — Engagement"
        NewsfeedService[Newsfeed Service]:::service
        NewsfeedCache[(Feed DB / Redis Cache)]:::cache
        NewsfeedService -->|OWNS| NewsfeedCache
        
        NotifService[Notification Service]:::service
        NotifDB[(Notification DB)]:::db
        NotifService -->|OWNS| NotifDB
        
        ChatService[Chat Service]:::service
        ChatDB[(Chat DB)]:::db
        ChatService -->|OWNS| ChatDB
    end

    subgraph "DOMAIN 4 — Discovery"
        SearchService[Search Service]:::service
        Elasticsearch[(Elasticsearch\nInverted Index)]:::db
        SearchService -->|OWNS| Elasticsearch
    end

    subgraph "DOMAIN 5 — Monetization"
        AdsService[Ads Service]:::service
        AdsDB[(Ads DB)]:::db
        AdsService -->|OWNS| AdsDB
    end

    %% -- API Gateway Routing --
    
    APIGateway -->|gRPC| AdsService
    APIGateway -->|gRPC| UserService
    APIGateway -->|gRPC| ContentService
    APIGateway -->|gRPC| GraphService
    APIGateway -->|gRPC| SearchService
    APIGateway -->|gRPC| NewsfeedService
    APIGateway -->|gRPC| NotifService
    APIGateway -->|gRPC| ChatService


    %% -- Shared Infrastructure & Event Bus --
    subgraph "Shared Infrastructure / Event Bus"
        KafkaCluster[[Kafka Cluster\n3 Brokers]]:::broker
        RedisSentinel[[Redis Sentinel\n3 Nodes / Cache & Session]]:::cache
    end
    
    %% -- Internal Communication (gRPC & EDA) --
    ContentService ==>|gRPC\nCheckFriendship| GraphService
    
    ContentService -.->|Publish: PostCreated| KafkaCluster
    KafkaCluster -.->|Event: PostCreated| NewsfeedService
    KafkaCluster -.->|Event: PostCreated| NotifService
    KafkaCluster -.->|Event: PostCreated| SearchService
```


---

## TASK 2 — Service Communication Diagram

**Synchronous (gRPC)** và **Asynchronous (EDA / Event)**.

```mermaid
graph LR
    %% -- Styles --
    classDef sync fill:#ffffff,stroke:#0d6efd,stroke-width:3px,color:#0a58ca
    classDef async fill:#ffffff,stroke:#fd7e14,stroke-width:3px,stroke-dasharray: 5 5,color:#c53030
    classDef service fill:#f8f9fa,stroke:#6c757d,stroke-width:2px,color:#212529
    classDef broker fill:#fff3cd,stroke:#ffc107,stroke-width:2px,color:#856404
    classDef db fill:#f8d7da,stroke:#dc3545,stroke-width:2px,color:#721c24

    subgraph "SYNCHRONOUS COMMUNICATION"
        ContentSync[Content Service]:::service
        GraphSync[Social Graph Service]:::service
        GraphDB[(Graph DB)]:::db
        
        ContentSync ==>|gRPC\nCheckFriendship| GraphSync:::sync
        GraphSync -->|OWNS| GraphDB
    end

    subgraph "ASYNCHRONOUS COMMUNICATION"
        ContentAsync[Content Service]:::service
        Kafka[Kafka Cluster]:::broker
        NFAsync[Newsfeed Service]:::service
        NotifAsync[Notification Service]:::service
        SearchAsync[Search Service]:::service
        
        ContentAsync -.->|Publish: PostCreated| Kafka:::async
        Kafka -.->|Event| NFAsync:::service
        Kafka -.->|Event| NotifAsync:::service
        Kafka -.->|Event| SearchAsync:::service
    end
```

### Communication Matrix


| Caller | Callee | Method/Event | Communication | Reason |
|---|---|---|---|---|
| API Gateway | User Service | User APIs | gRPC | Synchronous request |
| API Gateway | Content Service | Content APIs | gRPC | Synchronous request |
| Content Service | Social Graph Service | `CheckFriendship()` | gRPC | Immediate response |
| Content Service | Kafka | `PostCreated` | EDA | Asynchronous event |
| Kafka | Newsfeed Service | `PostCreated` | EDA | Update feed |
| Kafka | Notification Service | `PostCreated` | EDA | Create notification |
| Kafka | Search Service | `PostCreated` | EDA | Update search index |

---

## TASK 3 — Micro-frontend Detail

```mermaid
graph TD
    %% -- Styles --
    classDef mfe block,fill:#e1f5fe,stroke:#0288d1,stroke-width:2px,color:#01579b
    classDef gateway fill:#ffecb3,stroke:#ffc107,stroke-width:2px,color:#f57f17
    classDef note fill:#f8f9fa,stroke:#6c757d,stroke-width:1px,stroke-dasharray: 3 3,color:#495057
    
    subgraph "Micro-frontend Composition & Feature Ownership"
        
        subgraph "NEWSFEED MFE"
            NF["<b>Newsfeed Team</b></br>- Bảng tin<br/>- Bài viết<br/>- Bình luận<br/>- Like / Reaction<br/>- Chia sẻ"]:::mfe
        end
        
        subgraph "CHAT MFE"
            CH["<b>Chat Team</b></br>- Danh sách chat<br/>- Tin nhắn<br/>- Gửi file / media<br/>- Trạng thái online<br/>- Cài đặt chat"]:::mfe
        end

        subgraph "ADS MFE"
            AD["<b>Ads Team</b></br>- Tạo quảng cáo<br/>- Quản lý chiến dịch<br/>- Đối tượng quảng cáo<br/>- Thống kê hiệu quả<br/>- Thanh toán"]:::mfe
        end

        subgraph "SEARCH MFE"
            SE["<b>Search Team</b></br>- Ô tìm kiếm<br/>- Kết quả người dùng<br/>- Kết quả bài viết<br/>- Kết quả nhóm<br/>- Lọc & sắp xếp"]:::mfe
        end
        
    end

    APIGateway{API Gateway}:::gateway
    
    NF -->|HTTPS| APIGateway
    CH -->|HTTPS| APIGateway
    AD -->|HTTPS| APIGateway
    SE -->|HTTPS| APIGateway
    
    Note[Mỗi MFE có thể sử dụng một hoặc nhiều Backend Services thông qua API Gateway.]:::note
    APIGateway -.-> Note
```
