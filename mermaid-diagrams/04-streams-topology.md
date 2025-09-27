# Kafka Streams 處理拓撲圖

## 概述
此圖展示 Kafka Streams 的處理拓撲結構，包括 Source、Processor、Sink 節點及其連接關係。

## Mermaid 圖表

```mermaid
graph TB
    %% 定義樣式
    classDef source fill:#e8f5e8,stroke:#2e7d32,stroke-width:3px,color:#000
    classDef processor fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    classDef sink fill:#fce4ec,stroke:#c2185b,stroke-width:3px,color:#000
    classDef store fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#000
    classDef topic fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000

    %% 輸入 Topics
    subgraph "輸入 Topics"
        IT1[orders-topic]:::topic
        IT2[users-topic]:::topic
        IT3[products-topic]:::topic
    end

    %% Source Processors
    subgraph "Source Processors"
        S1[Orders Source]:::source
        S2[Users Source]:::source
        S3[Products Source]:::source
    end

    %% Stream Processing Nodes
    subgraph "Stream Processing Topology"
        %% 基本轉換
        P1[Filter<br/>過濾無效訂單]:::processor
        P2[Map<br/>格式化用戶資料]:::processor
        P3[MapValues<br/>計算產品價格]:::processor
        
        %% 聚合處理
        P4[GroupByKey<br/>按用戶分組]:::processor
        P5[Aggregate<br/>計算用戶總消費]:::processor
        P6[Window<br/>時間窗口聚合]:::processor
        
        %% Join 操作
        P7[Join<br/>訂單-用戶關聯]:::processor
        P8[LeftJoin<br/>訂單-產品關聯]:::processor
        
        %% 分支處理
        P9[Branch<br/>訂單分類]:::processor
        P10[Filter<br/>高價值訂單]:::processor
        P11[Filter<br/>一般訂單]:::processor
    end

    %% State Stores
    subgraph "State Stores"
        SS1[User Aggregates<br/>State Store]:::store
        SS2[Window Store<br/>時間窗口狀態]:::store
        SS3[Join State Store<br/>Join 狀態]:::store
    end

    %% Sink Processors
    subgraph "Sink Processors"
        SINK1[High Value Orders<br/>Sink]:::sink
        SINK2[Regular Orders<br/>Sink]:::sink
        SINK3[User Statistics<br/>Sink]:::sink
        SINK4[Windowed Results<br/>Sink]:::sink
    end

    %% 輸出 Topics
    subgraph "輸出 Topics"
        OT1[high-value-orders]:::topic
        OT2[regular-orders]:::topic
        OT3[user-statistics]:::topic
        OT4[windowed-analytics]:::topic
    end

    %% 連接關係 - 輸入到 Source
    IT1 --> S1
    IT2 --> S2
    IT3 --> S3

    %% Source 到處理節點
    S1 --> P1
    S2 --> P2
    S3 --> P3

    %% 基本處理流程
    P1 --> P7
    P2 --> P4
    P3 --> P8

    %% Join 操作
    P7 --> P8
    P8 --> P9

    %% 聚合流程
    P4 --> P5
    P5 --> P6

    %% 分支處理
    P9 --> P10
    P9 --> P11

    %% 連接到 Sink
    P10 --> SINK1
    P11 --> SINK2
    P5 --> SINK3
    P6 --> SINK4

    %% Sink 到輸出 Topic
    SINK1 --> OT1
    SINK2 --> OT2
    SINK3 --> OT3
    SINK4 --> OT4

    %% State Store 連接
    P5 -.-> SS1
    P6 -.-> SS2
    P7 -.-> SS3
    P8 -.-> SS3
```

## 拓撲組件說明

### Source Processors (源處理器)
- **功能**: 從 Kafka Topics 讀取資料流
- **特點**: 
  - 每個 Source 對應一個輸入 Topic
  - 自動處理偏移量管理
  - 支援多種序列化格式

### Stream Processors (流處理器)
1. **無狀態處理器**:
   - `Filter`: 過濾符合條件的記錄
   - `Map/MapValues`: 轉換記錄的鍵或值
   - `FlatMap`: 一對多的記錄轉換

2. **有狀態處理器**:
   - `GroupBy`: 按鍵重新分組
   - `Aggregate`: 聚合計算
   - `Join`: 流與流/表的關聯

3. **窗口處理器**:
   - `Window`: 時間窗口聚合
   - 支援滾動、跳躍、會話窗口

### Sink Processors (匯處理器)
- **功能**: 將處理結果寫入 Kafka Topics
- **特點**:
  - 自動序列化輸出資料
  - 支援分區策略配置
  - 處理背壓和錯誤恢復

### State Stores (狀態存儲)
- **本地狀態**: 每個處理任務的本地狀態
- **容錯性**: 透過 changelog topics 實現狀態恢復
- **類型**:
  - Key-Value Store
  - Window Store
  - Session Store

## 處理語義

### 流處理模式
```mermaid
graph LR
    A[Record] --> B[Transform]
    B --> C[Forward]
    C --> D[Next Processor]
```

### 表處理模式
```mermaid
graph LR
    A[Update] --> B[Materialize]
    B --> C[State Store]
    C --> D[Query/Join]
```

## 並行處理模型

### Task 分配
- 每個 Task 處理特定的 Partition 集合
- Task 數量 = 輸入 Topic 的 Partition 數量
- 可在多個執行緒和實例間分配

### 容錯機制
1. **狀態恢復**: 從 changelog topics 恢復本地狀態
2. **重平衡**: 動態調整 Task 分配
3. **恰好一次**: 透過交易保證處理語義

## 實際應用場景

### 即時分析
- 用戶行為分析
- 業務指標計算
- 異常檢測

### 資料轉換
- ETL 處理
- 格式標準化
- 資料豐富化

### 事件驅動架構
- 微服務間通訊
- 事件溯源
- CQRS 模式實現
