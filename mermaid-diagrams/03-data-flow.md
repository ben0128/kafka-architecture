# Kafka 資料流程圖

## 概述
此圖展示資料在 Kafka 系統中的完整流動路徑，從 Producer 發送到 Consumer 接收的整個過程。

## Mermaid 圖表

```mermaid
sequenceDiagram
    participant P as Producer
    participant MB as Metadata Broker
    participant LB as Leader Broker
    participant FB1 as Follower Broker 1
    participant FB2 as Follower Broker 2
    participant CG as Consumer Group
    participant C1 as Consumer 1
    participant C2 as Consumer 2

    %% 1. Producer 啟動和元數據獲取
    Note over P,MB: 1. Producer 初始化
    P->>MB: 請求叢集元數據
    MB-->>P: 回傳 Topics、Partitions、Leader 資訊

    %% 2. 訊息發送流程
    Note over P,FB2: 2. 訊息發送流程
    P->>P: 序列化訊息 (Key, Value)
    P->>P: 選擇 Partition (根據 Key 或輪詢)
    P->>LB: 發送 ProduceRequest 到 Leader
    
    %% 3. Leader 處理和副本同步
    Note over LB,FB2: 3. Leader 處理和副本同步
    LB->>LB: 寫入本地日誌
    LB->>FB1: 同步副本 (ISR)
    LB->>FB2: 同步副本 (ISR)
    FB1-->>LB: ACK 確認
    FB2-->>LB: ACK 確認
    LB-->>P: ProduceResponse (成功)

    %% 4. Consumer 啟動和群組協調
    Note over CG,C2: 4. Consumer 群組初始化
    C1->>CG: 加入 Consumer Group
    C2->>CG: 加入 Consumer Group
    CG->>CG: 分配 Partitions 給 Consumers
    CG-->>C1: 分配 Partition 0, 2
    CG-->>C2: 分配 Partition 1, 3

    %% 5. Consumer 元數據獲取
    Note over C1,MB: 5. Consumer 元數據獲取
    C1->>MB: 請求 Partition 元數據
    MB-->>C1: 回傳 Leader Broker 資訊
    C2->>MB: 請求 Partition 元數據
    MB-->>C2: 回傳 Leader Broker 資訊

    %% 6. 訊息消費流程
    Note over C1,LB: 6. 訊息消費流程
    C1->>LB: FetchRequest (Partition 0, offset)
    LB-->>C1: FetchResponse (訊息批次)
    C1->>C1: 反序列化並處理訊息
    C1->>CG: 提交 Offset (異步)

    C2->>LB: FetchRequest (Partition 1, offset)
    LB-->>C2: FetchResponse (訊息批次)
    C2->>C2: 反序列化並處理訊息
    C2->>CG: 提交 Offset (異步)

    %% 7. 錯誤處理和重試
    Note over P,C2: 7. 錯誤處理場景
    P->>LB: 發送訊息 (網路錯誤)
    LB--xP: 連接失敗
    P->>MB: 重新獲取元數據
    MB-->>P: 更新的 Leader 資訊
    P->>LB: 重試發送訊息
    LB-->>P: 成功回應
```

## 詳細流程說明

### 階段 1: Producer 初始化
1. **元數據獲取**: Producer 連接任一 Broker 獲取叢集元數據
2. **分區策略**: 根據 Key 或配置的分區器選擇目標 Partition
3. **序列化**: 將 Key 和 Value 序列化為位元組陣列

### 階段 2: 訊息發送
1. **批次處理**: Producer 將訊息加入批次緩衝區
2. **壓縮**: 可選的訊息壓縮 (gzip, snappy, lz4, zstd)
3. **網路傳輸**: 發送 ProduceRequest 到 Partition Leader

### 階段 3: Broker 處理
1. **Leader 寫入**: Leader Broker 將訊息寫入本地日誌
2. **副本同步**: 同步到 ISR (In-Sync Replicas) 中的 Follower
3. **確認回應**: 根據 `acks` 配置決定何時回應成功

### 階段 4: Consumer 群組協調
1. **群組加入**: Consumer 加入指定的 Consumer Group
2. **分區分配**: 群組協調器分配 Partitions 給各個 Consumer
3. **重平衡**: 當 Consumer 加入/離開時觸發重平衡

### 階段 5: 訊息消費
1. **拉取請求**: Consumer 主動向 Leader 發送 FetchRequest
2. **批次讀取**: 一次性讀取多個訊息以提高效率
3. **偏移量管理**: 追蹤和提交消費進度

## 關鍵特性

### 可靠性保證
- **至少一次**: 預設的訊息傳遞語義
- **恰好一次**: 透過冪等性和交易實現
- **副本機制**: 透過多副本保證資料持久性

### 效能優化
- **批次處理**: Producer 和 Consumer 都支援批次操作
- **零拷貝**: 使用 sendfile() 系統調用優化網路傳輸
- **壓縮**: 減少網路頻寬和儲存空間使用

### 容錯機制
- **自動重試**: Producer 和 Consumer 的自動重試機制
- **故障轉移**: Leader 故障時自動選舉新 Leader
- **元數據更新**: 動態更新叢集拓撲變化
