# Kafka 分區和副本分布圖

## 概述
此圖展示 Kafka Topic 分區在不同 Broker 上的分布，以及 Leader/Follower 副本的配置。

## Mermaid 圖表

```mermaid
graph TB
    %% 定義樣式
    classDef leader fill:#ffcdd2,stroke:#d32f2f,stroke-width:3px,color:#000
    classDef follower fill:#c8e6c9,stroke:#388e3c,stroke-width:2px,color:#000
    classDef broker fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
    classDef topic fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000

    %% Topic 定義
    subgraph "Topics"
        T1[orders-topic<br/>Partitions: 6<br/>Replication: 3]:::topic
        T2[payments-topic<br/>Partitions: 3<br/>Replication: 2]:::topic
    end

    %% Kafka 叢集
    subgraph "Kafka Cluster"
        subgraph "Broker 1 (ID: 1)"
            B1[Broker 1]:::broker
            B1_O0[orders-0<br/>Leader]:::leader
            B1_O3[orders-3<br/>Leader]:::leader
            B1_O1[orders-1<br/>Follower]:::follower
            B1_O4[orders-4<br/>Follower]:::follower
            B1_O2[orders-2<br/>Follower]:::follower
            B1_O5[orders-5<br/>Follower]:::follower
            B1_P0[payments-0<br/>Leader]:::leader
            B1_P1[payments-1<br/>Follower]:::follower
            B1_P2[payments-2<br/>Leader]:::leader
        end

        subgraph "Broker 2 (ID: 2)"
            B2[Broker 2]:::broker
            B2_O1[orders-1<br/>Leader]:::leader
            B2_O4[orders-4<br/>Leader]:::leader
            B2_O0[orders-0<br/>Follower]:::follower
            B2_O2[orders-2<br/>Follower]:::follower
            B2_O3[orders-3<br/>Follower]:::follower
            B2_O5[orders-5<br/>Follower]:::follower
            B2_P1[payments-1<br/>Leader]:::leader
            B2_P0[payments-0<br/>Follower]:::follower
            B2_P2[payments-2<br/>Follower]:::follower
        end

        subgraph "Broker 3 (ID: 3)"
            B3[Broker 3]:::broker
            B3_O2[orders-2<br/>Leader]:::leader
            B3_O5[orders-5<br/>Leader]:::leader
            B3_O0[orders-0<br/>Follower]:::follower
            B3_O1[orders-1<br/>Follower]:::follower
            B3_O3[orders-3<br/>Follower]:::follower
            B3_O4[orders-4<br/>Follower]:::follower
            B3_P0[payments-0<br/>Follower]:::follower
            B3_P1[payments-1<br/>Follower]:::follower
            B3_P2[payments-2<br/>Follower]:::follower
        end
    end

    %% ISR (In-Sync Replicas) 與追隨者抓取方向
    subgraph "副本同步 (Follower Fetch → Leader)"
        B2_O0 -.->|Follower Fetch| B1_O0
        B3_O0 -.->|Follower Fetch| B1_O0

        B1_O1 -.->|Follower Fetch| B2_O1
        B3_O1 -.->|Follower Fetch| B2_O1

        B1_O2 -.->|Follower Fetch| B3_O2
        B2_O2 -.->|Follower Fetch| B3_O2

        B2_O3 -.->|Follower Fetch| B1_O3
        B3_O3 -.->|Follower Fetch| B1_O3

        B1_O4 -.->|Follower Fetch| B2_O4
        B3_O4 -.->|Follower Fetch| B2_O4

        B1_O5 -.->|Follower Fetch| B3_O5
        B2_O5 -.->|Follower Fetch| B3_O5

        B2_P0 -.->|Follower Fetch| B1_P0
        B3_P0 -.->|Follower Fetch| B1_P0

        B1_P1 -.->|Follower Fetch| B2_P1
        B3_P1 -.->|Follower Fetch| B2_P1

        B2_P2 -.->|Follower Fetch| B1_P2
        B3_P2 -.->|Follower Fetch| B1_P2
    end

    class B2_O0,B3_O0,B1_O1,B3_O1,B1_O2,B2_O2,B2_O3,B3_O3,B1_O4,B3_O4,B1_O5,B2_O5,B2_P0,B3_P0,B1_P1,B3_P1,B2_P2,B3_P2 follower
```

## 分區分布策略

### 分區分配原則
1. **均勻分布**: 分區盡可能均勻分布到所有 Broker
2. **副本分離**: 同一分區的副本不在同一 Broker
3. **負載平衡**: Leader 分區均勻分布
4. **機架感知**: 考慮機架拓撲避免單點故障

### 副本配置詳情

#### orders-topic (6 分區, 3 副本)
| 分區 | Leader | Follower 1 | Follower 2 | ISR |
|------|--------|------------|------------|-----|
| 0 | Broker 1 | Broker 2 | Broker 3 | [1,2,3] |
| 1 | Broker 2 | Broker 1 | Broker 3 | [2,1,3] |
| 2 | Broker 3 | Broker 1 | Broker 2 | [3,1,2] |
| 3 | Broker 1 | Broker 2 | Broker 3 | [1,2,3] |
| 4 | Broker 2 | Broker 1 | Broker 3 | [2,1,3] |
| 5 | Broker 3 | Broker 1 | Broker 2 | [3,1,2] |

#### users-topic (3 分區, 2 副本)
| 分區 | Leader | Follower 1 | ISR |
|------|--------|------------|-----|
| 0 | Broker 1 | Broker 2 | [1,2] |
| 1 | Broker 2 | Broker 1 | [2,1] |
| 2 | Broker 1 | Broker 2 | [1,2] |

## 關鍵概念

### Leader 和 Follower
- **Leader**: 處理所有讀寫請求的主副本
- **Follower**: 被動複製 Leader 資料的副本
- **ISR**: In-Sync Replicas，與 Leader 保持同步的副本集合

### 容錯機制
```mermaid
graph LR
    A[Leader 故障] --> B[從 ISR 選舉新 Leader]
    B --> C[更新元數據]
    C --> D[客戶端重新連接]
```

### 效能考量

#### 讀寫分離
- **寫入**: 只能寫入 Leader
- **讀取**: 預設從 Leader 讀取，可配置從 Follower 讀取

#### 分區策略影響
1. **並行度**: 分區數決定最大並行消費者數
2. **儲存**: 分區分布影響儲存均衡
3. **網路**: 副本同步產生網路流量

## 管理操作

### 分區重分配
```bash
# 生成重分配計劃
kafka-reassign-partitions.sh --generate

# 執行重分配
kafka-reassign-partitions.sh --execute

# 驗證重分配
kafka-reassign-partitions.sh --verify
```

### 副本同步監控
- **Lag 監控**: 監控 Follower 落後程度
- **ISR 變化**: 追蹤 ISR 成員變化
- **Under-replicated**: 識別副本不足的分區

## 最佳實踐

### 分區數量規劃
- 考慮預期吞吐量和並行度需求
- 避免過多分區影響元數據管理
- 預留擴展空間但不過度配置

### 副本因子選擇
- **生產環境**: 建議 RF ≥ 3
- **開發環境**: RF = 1 或 2 即可
- **關鍵資料**: 考慮更高的副本因子

### 機架感知配置
```properties
# 配置 Broker 機架資訊
broker.rack=rack1

# 啟用機架感知分配
replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector
```
