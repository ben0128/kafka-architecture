# Kafka 網路協定互動圖

## 概述
此圖展示客戶端與 Broker 之間的通訊協定，包括請求/回應流程和元數據更新機制。

## Mermaid 圖表

```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant B1 as Broker 1
    participant B2 as Broker 2 (Leader)
    participant B3 as Broker 3
    participant ZK as ZooKeeper/KRaft

    %% 1. 客戶端啟動和引導
    Note over C,ZK: 1. 客戶端初始化和引導階段
    C->>LB: TCP 連接 (bootstrap.servers)
    LB->>B1: 轉發連接
    C->>B1: ApiVersionsRequest
    B1-->>C: ApiVersionsResponse (支援的 API 版本)
    
    %% 2. 元數據發現
    Note over C,B3: 2. 叢集元數據發現
    C->>B1: MetadataRequest (topics=[])
    B1->>ZK: 查詢叢集資訊
    ZK-->>B1: 叢集拓撲
    B1-->>C: MetadataResponse (brokers, topics, partitions)
    
    %% 3. Producer 流程
    Note over C,B3: 3. Producer 訊息發送流程
    C->>C: 選擇分區 (Partitioner)
    C->>B2: ProduceRequest (topic, partition, records)
    
    %% 副本同步 (如果 acks=all)
    B2->>B1: FetchRequest (副本同步)
    B2->>B3: FetchRequest (副本同步)
    B1-->>B2: FetchResponse (ACK)
    B3-->>B2: FetchResponse (ACK)
    
    B2-->>C: ProduceResponse (offset, timestamp)
    
    %% 4. Consumer 流程
    Note over C,B3: 4. Consumer 訊息消費流程
    C->>B1: FindCoordinatorRequest (group.id)
    B1-->>C: FindCoordinatorResponse (coordinator broker)
    
    C->>B2: JoinGroupRequest (group.id, member.id)
    B2-->>C: JoinGroupResponse (member.id, leader)
    
    C->>B2: SyncGroupRequest (assignments)
    B2-->>C: SyncGroupResponse (partition assignment)
    
    C->>B2: FetchRequest (topic, partition, offset)
    B2-->>C: FetchResponse (records batch)
    
    C->>B2: OffsetCommitRequest (offsets)
    B2-->>C: OffsetCommitResponse (success)
    
    %% 5. 心跳和健康檢查
    Note over C,B2: 5. 心跳和連接維護
    loop 每 heartbeat.interval.ms
        C->>B2: HeartbeatRequest
        B2-->>C: HeartbeatResponse
    end
    
    %% 6. 錯誤處理和重試
    Note over C,B3: 6. 錯誤處理和恢復
    C->>B2: ProduceRequest
    B2--xC: NetworkException
    C->>B1: MetadataRequest (刷新元數據)
    B1-->>C: MetadataResponse (更新的 leader)
    C->>B3: ProduceRequest (重試到新 leader)
    B3-->>C: ProduceResponse (成功)
```

## 協定層次結構

### TCP 層
```mermaid
graph TB
    A[TCP Socket] --> B[Kafka Protocol Frame]
    B --> C[Request/Response Header]
    C --> D[Request/Response Body]
```

### 協定格式
```
+------------------+
| Message Size (4) |
+------------------+
| Request Header   |
+------------------+
| Request Body     |
+------------------+
```

## 核心 API 協定

### 1. ApiVersions API
- **目的**: 協商客戶端和 Broker 支援的 API 版本
- **時機**: 連接建立後的第一個請求
- **版本協商**: 確保相容性

### 2. Metadata API
```mermaid
graph LR
    A[MetadataRequest] --> B[Topic Names]
    B --> C[Allow Auto Topic Creation]
    C --> D[Include Cluster Authorized Operations]
```

**回應內容**:
- Broker 清單和連接資訊
- Topic 分區配置
- Leader/Follower 分配

### 3. Produce API
```mermaid
graph TB
    A[ProduceRequest] --> B[Required Acks]
    B --> C[Timeout]
    C --> D[Topic Data]
    D --> E[Partition Data]
    E --> F[Record Batch]
```

**確認模式**:
- `acks=0`: 不等待確認
- `acks=1`: 等待 Leader 確認
- `acks=all`: 等待所有 ISR 確認

### 4. Fetch API
```mermaid
graph TB
    A[FetchRequest] --> B[Max Wait Time]
    B --> C[Min Bytes]
    C --> D[Max Bytes]
    D --> E[Fetch Data]
    E --> F[Partition Data]
```

**優化參數**:
- `fetch.min.bytes`: 最小批次大小
- `fetch.max.wait.ms`: 最大等待時間
- `max.partition.fetch.bytes`: 單分區最大資料量

## 連接管理

### 連接池
```mermaid
graph TB
    A[Client] --> B[Connection Pool]
    B --> C[Broker 1 Connection]
    B --> D[Broker 2 Connection]
    B --> E[Broker N Connection]
```

### 連接狀態
- **CONNECTING**: 正在建立連接
- **CONNECTED**: 已建立連接
- **DISCONNECTED**: 連接已斷開
- **AUTHENTICATION_FAILED**: 認證失敗

## 錯誤處理機制

### 可重試錯誤
- `NETWORK_EXCEPTION`: 網路異常
- `REQUEST_TIMED_OUT`: 請求超時
- `NOT_LEADER_FOR_PARTITION`: 不是分區 Leader
- `LEADER_NOT_AVAILABLE`: Leader 不可用

### 不可重試錯誤
- `INVALID_TOPIC_EXCEPTION`: 無效 Topic
- `AUTHORIZATION_FAILED`: 授權失敗
- `UNSUPPORTED_VERSION`: 不支援的版本

## 效能優化

### 批次處理
```mermaid
graph LR
    A[Multiple Records] --> B[Batch Compression]
    B --> C[Single Network Request]
    C --> D[Reduced Latency]
```

### 管道化 (Pipelining)
- 允許多個未完成的請求
- 提高網路利用率
- 配置 `max.in.flight.requests.per.connection`

### 壓縮
支援的壓縮算法:
- **gzip**: 高壓縮率，CPU 密集
- **snappy**: 平衡壓縮率和速度
- **lz4**: 快速壓縮
- **zstd**: 新一代高效壓縮

## 安全協定

### SASL 認證流程
```mermaid
sequenceDiagram
    participant C as Client
    participant B as Broker

    C->>B: SaslHandshakeRequest (mechanism)
    B-->>C: SaslHandshakeResponse (mechanisms)
    C->>B: SaslAuthenticateRequest (auth data)
    B-->>C: SaslAuthenticateResponse (result)
```

### SSL/TLS 加密
- **握手**: SSL 憑證驗證
- **加密**: 資料傳輸加密
- **效能**: 增加 CPU 開銷

## 監控和診斷

### 網路指標
- `network-io-rate`: 網路 I/O 速率
- `request-latency-avg`: 平均請求延遲
- `request-rate`: 請求速率
- `response-rate`: 回應速率

### 連接指標
- `connection-count`: 活躍連接數
- `connection-creation-rate`: 連接建立速率
- `connection-close-rate`: 連接關閉速率

## 協定版本演進

### 版本相容性
```mermaid
graph TB
    A[Client API Version] --> B{Broker 支援?}
    B -->|是| C[使用請求版本]
    B -->|否| D[降級到支援版本]
```

### 新功能支援
- **向後相容**: 新版本支援舊協定
- **功能檢測**: 透過 ApiVersions 檢測功能
- **優雅降級**: 不支援時使用替代方案

## 最佳實踐

### 客戶端配置
```properties
# 連接配置
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092
connections.max.idle.ms=540000
request.timeout.ms=30000

# 重試配置
retries=2147483647
retry.backoff.ms=100
delivery.timeout.ms=120000

# 批次配置
batch.size=16384
linger.ms=5
```

### 網路調優
```properties
# Socket 緩衝區
send.buffer.bytes=131072
receive.buffer.bytes=65536

# 壓縮配置
compression.type=snappy

# 並發控制
max.in.flight.requests.per.connection=5
```
