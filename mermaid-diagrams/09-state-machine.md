# Kafka 狀態機圖

## 概述
此圖展示 Kafka 各個組件的狀態轉換，包括 Consumer Group、Streams 應用、交易等的生命週期。

## Mermaid 圖表

### Consumer Group 狀態機

```mermaid
stateDiagram-v2
    [*] --> Empty : 建立群組
    
    Empty --> PreparingRebalance : 第一個 Consumer 加入
    PreparingRebalance --> CompletingRebalance : 所有成員已加入
    CompletingRebalance --> Stable : 分區分配完成
    
    Stable --> PreparingRebalance : Consumer 加入/離開
    Stable --> PreparingRebalance : 心跳超時
    Stable --> PreparingRebalance : 分區數變化
    
    PreparingRebalance --> Empty : 所有 Consumer 離開
    CompletingRebalance --> Empty : 重平衡失敗
    Stable --> Empty : 群組過期
    
    Empty --> Dead : 群組刪除
    PreparingRebalance --> Dead : 強制刪除
    CompletingRebalance --> Dead : 強制刪除
    Stable --> Dead : 強制刪除
    
    Dead --> [*]
    
    note right of Empty
        無活躍成員
        等待新成員加入
    end note
    
    note right of PreparingRebalance
        準備重平衡
        收集成員資訊
        等待所有成員響應
    end note
    
    note right of CompletingRebalance
        完成重平衡
        分配分區給成員
        等待同步完成
    end note
    
    note right of Stable
        穩定運行狀態
        正常消費訊息
        定期心跳檢查
    end note
```

### KafkaStreams 應用狀態機

```mermaid
stateDiagram-v2
    [*] --> CREATED : 建立實例
    
    CREATED --> REBALANCING : start() 調用
    REBALANCING --> RUNNING : 拓撲啟動成功
    REBALANCING --> ERROR : 啟動失敗
    
    RUNNING --> REBALANCING : 重平衡觸發
    RUNNING --> PENDING_SHUTDOWN : close() 調用
    RUNNING --> ERROR : 執行時錯誤
    
    REBALANCING --> RUNNING : 重平衡完成
    REBALANCING --> ERROR : 重平衡失敗
    REBALANCING --> PENDING_SHUTDOWN : close() 調用
    
    ERROR --> PENDING_SHUTDOWN : close() 調用
    ERROR --> REBALANCING : 錯誤恢復
    
    PENDING_SHUTDOWN --> NOT_RUNNING : 關閉完成
    
    NOT_RUNNING --> [*]
    
    note right of CREATED
        實例已建立
        尚未啟動
        可配置參數
    end note
    
    note right of REBALANCING
        重新平衡中
        重新分配任務
        恢復狀態存儲
    end note
    
    note right of RUNNING
        正常運行
        處理訊息流
        維護本地狀態
    end note
    
    note right of ERROR
        錯誤狀態
        停止處理
        等待恢復或關閉
    end note
```

### 交易狀態機

```mermaid
stateDiagram-v2
    [*] --> Empty : Producer 初始化
    
    Empty --> Ongoing : beginTransaction()
    
    Ongoing --> PrepareCommit : commitTransaction()
    Ongoing --> PrepareAbort : abortTransaction()
    Ongoing --> PrepareAbort : 異常發生
    
    PrepareCommit --> CompleteCommit : 準備完成
    PrepareAbort --> CompleteAbort : 準備完成
    
    CompleteCommit --> Empty : 提交完成
    CompleteAbort --> Empty : 中止完成
    
    PrepareCommit --> PrepareAbort : 提交失敗
    
    note right of Empty
        無活躍交易
        可開始新交易
    end note
    
    note right of Ongoing
        交易進行中
        可發送訊息
        可提交或中止
    end note
    
    note right of PrepareCommit
        準備提交
        協調器記錄狀態
        通知所有參與者
    end note
    
    note right of PrepareAbort
        準備中止
        協調器記錄狀態
        清理交易資源
    end note
```

### Broker 狀態機

```mermaid
stateDiagram-v2
    [*] --> Starting : Broker 啟動
    
    Starting --> RunningAsBroker : 註冊成功 (傳統模式)
    Starting --> RunningAsController : 選為 Controller (KRaft)
    Starting --> Failed : 啟動失敗
    
    RunningAsBroker --> PendingControlledShutdown : 優雅關閉
    RunningAsController --> PendingControlledShutdown : 優雅關閉
    
    RunningAsBroker --> Failed : 運行時錯誤
    RunningAsController --> Failed : 運行時錯誤
    
    PendingControlledShutdown --> NotRunning : 關閉完成
    Failed --> NotRunning : 強制關閉
    
    NotRunning --> [*]
    
    note right of Starting
        載入配置
        初始化組件
        連接 ZooKeeper/KRaft
    end note
    
    note right of RunningAsBroker
        處理客戶端請求
        管理分區副本
        參與叢集協調
    end note
    
    note right of RunningAsController
        管理叢集元數據
        協調分區分配
        處理 Broker 變化
    end note
```

## 狀態轉換觸發條件

### Consumer Group 狀態轉換

| 當前狀態 | 觸發事件 | 目標狀態 | 說明 |
|----------|----------|----------|------|
| Empty | Consumer 加入 | PreparingRebalance | 第一個成員加入群組 |
| Stable | Consumer 加入/離開 | PreparingRebalance | 成員變化觸發重平衡 |
| Stable | 心跳超時 | PreparingRebalance | 成員被認為已離線 |
| PreparingRebalance | 所有成員響應 | CompletingRebalance | 進入分區分配階段 |
| CompletingRebalance | 分配完成 | Stable | 重平衡成功完成 |

### KafkaStreams 狀態轉換

| 當前狀態 | 觸發事件 | 目標狀態 | 說明 |
|----------|----------|----------|------|
| CREATED | start() | REBALANCING | 開始啟動流程 |
| RUNNING | 成員變化 | REBALANCING | 觸發任務重分配 |
| RUNNING | 異常 | ERROR | 處理過程中發生錯誤 |
| ERROR | 恢復 | REBALANCING | 錯誤恢復後重新平衡 |
| 任何狀態 | close() | PENDING_SHUTDOWN | 開始關閉流程 |

## 狀態監控和診斷

### JMX 指標監控

#### Consumer Group 指標
```java
// 群組狀態監控
"kafka.coordinator.group:type=GroupMetadataManager,name=NumGroups"
"kafka.coordinator.group:type=GroupMetadataManager,name=NumGroupsPreparingRebalance"
"kafka.coordinator.group:type=GroupMetadataManager,name=NumGroupsCompletingRebalance"
"kafka.coordinator.group:type=GroupMetadataManager,name=NumGroupsStable"
```

#### Streams 應用指標
```java
// Streams 狀態監控
"kafka.streams:type=kafka-streams-state,client-id=*"
"kafka.streams:type=stream-thread-metrics,thread-id=*"
```

### 狀態變化日誌

#### Consumer Group 日誌
```log
[GroupCoordinator] Group app-group transitioned from Stable to PreparingRebalance
[GroupCoordinator] Member consumer-1 joined group app-group
[GroupCoordinator] Group app-group transitioned from PreparingRebalance to CompletingRebalance
```

#### Streams 應用日誌
```log
[StreamThread] State transition from RUNNING to REBALANCING
[StreamThread] Partition assignment: {topic-0=[0,1], topic-1=[2,3]}
[StreamThread] State transition from REBALANCING to RUNNING
```

## 異常處理和恢復

### 重平衡異常處理
```mermaid
flowchart TD
    A[重平衡開始] --> B{所有成員響應?}
    B -->|是| C[分配分區]
    B -->|否| D[等待超時]
    D --> E[移除無響應成員]
    E --> C
    C --> F{分配成功?}
    F -->|是| G[進入 Stable 狀態]
    F -->|否| H[重新開始重平衡]
```

### 交易異常處理
```mermaid
flowchart TD
    A[交易進行中] --> B{發生異常?}
    B -->|是| C[自動中止交易]
    B -->|否| D[正常提交]
    C --> E[清理資源]
    D --> F[標記已提交]
```

## 最佳實踐

### 狀態監控配置
```properties
# Consumer Group 監控
group.id=my-app-group
session.timeout.ms=30000
heartbeat.interval.ms=3000
max.poll.interval.ms=300000

# Streams 應用監控
application.id=my-streams-app
commit.interval.ms=30000
state.dir=/tmp/kafka-streams
```

### 異常處理策略
```java
// Consumer 異常處理
consumer.subscribe(topics, new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // 保存處理進度
        commitOffsets();
    }
    
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // 恢復處理狀態
        restoreState();
    }
});

// Streams 異常處理
streams.setUncaughtExceptionHandler((thread, exception) -> {
    logger.error("Streams thread {} encountered error", thread.getName(), exception);
    return StreamsUncaughtExceptionHandler.StreamThreadExceptionResponse.REPLACE_THREAD;
});
```

### 狀態轉換優化
1. **減少重平衡頻率**: 調整心跳和會話超時參數
2. **快速故障檢測**: 縮短檢測時間但避免誤判
3. **優雅關閉**: 實作正確的關閉流程避免資源洩漏
