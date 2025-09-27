# Apache Kafka 架構研究專案

## 專案目的
深入理解 Apache Kafka 的架構設計，透過視覺化圖表快速掌握各個組件的關係和運作原理。

## 目錄結構
```
kafka-architecture-study/
├── README.md                    # 專案說明
├── mermaid-diagrams/           # Mermaid 圖表集合
│   ├── 01-system-architecture.md      # 系統整體架構圖
│   ├── 02-module-dependency.md        # 模組依賴關係圖
│   ├── 03-data-flow.md               # 資料流程圖
│   ├── 04-streams-topology.md        # Kafka Streams 處理拓撲圖
│   ├── 05-partition-replica.md       # 分區和副本分布圖
│   ├── 06-coordinator-architecture.md # 協調器架構圖
│   ├── 07-network-protocol.md        # 網路協定互動圖
│   ├── 08-class-relationship.md      # 類別關係圖
│   ├── 09-state-machine.md           # 狀態機圖
│   └── 10-deployment-architecture.md  # 部署架構圖
└── notes/                      # 學習筆記 (待建立)
```

## 圖表清單

### ✅ 已完成
1. **系統整體架構圖** - 展示 Kafka 叢集的整體架構和核心組件
2. **模組依賴關係圖** - 專案中各個模組之間的依賴關係
3. **資料流程圖** - 資料在 Kafka 系統中的流動路徑
4. **Kafka Streams 處理拓撲圖** - Kafka Streams 的處理流程
5. **分區和副本分布圖** - Topic 分區在不同 Broker 上的分布
6. **協調器架構圖** - 各種協調器的組織結構
7. **網路協定互動圖** - 客戶端與 Broker 之間的通訊協定
8. **類別關係圖** - 核心類別之間的繼承和組合關係
9. **狀態機圖** - Kafka 組件的狀態轉換
10. **部署架構圖** - Kafka 在實際環境中的部署方式

## 如何使用

### 查看圖表
1. 開啟 `mermaid-diagrams/` 目錄下的 `.md` 檔案
2. 使用支援 Mermaid 的編輯器或瀏覽器擴充功能預覽
3. 推薦工具：
   - VS Code + Mermaid Preview 擴充功能
   - 線上 Mermaid 編輯器：https://mermaid.live/

### 學習建議
1. **初學者**: 從 01-system-architecture 開始，建立整體概念
2. **開發者**: 重點關注 02-module-dependency 和 08-class-relationship
3. **運維人員**: 專注於 10-deployment-architecture 和 05-partition-replica

## 相關資源
- [Apache Kafka 官方文檔](https://kafka.apache.org/documentation/)
- [Kafka 原始碼](https://github.com/apache/kafka)
- [Mermaid 語法文檔](https://mermaid-js.github.io/mermaid/)

## 更新日誌
- 2025-09-27: 建立專案結構，完成系統整體架構圖
