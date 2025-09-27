# Kafka 部署架構圖

## 概述
此圖展示 Kafka 在實際生產環境中的部署方式，包括多機房部署、容器化部署和雲端部署。

## Mermaid 圖表

### 多機房部署架構

```mermaid
graph TB
    %% 定義樣式
    classDef datacenter fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    classDef broker fill:#ffcdd2,stroke:#d32f2f,stroke-width:2px,color:#000
    classDef zk fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px,color:#000
    classDef client fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#000
    classDef lb fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000
    classDef monitor fill:#fce4ec,stroke:#c2185b,stroke-width:2px,color:#000

    %% 機房 A
    subgraph "Data Center A (Primary)"
        subgraph "Availability Zone A1"
            LB_A1[Load Balancer<br/>HAProxy/Nginx]:::lb
            B1[Kafka Broker 1<br/>rack: rack-a1]:::broker
            B2[Kafka Broker 2<br/>rack: rack-a1]:::broker
            ZK1[ZooKeeper 1<br/>myid: 1]:::zk
        end
        
        subgraph "Availability Zone A2"
            B3[Kafka Broker 3<br/>rack: rack-a2]:::broker
            B4[Kafka Broker 4<br/>rack: rack-a2]:::broker
            ZK2[ZooKeeper 2<br/>myid: 2]:::zk
        end
        
        subgraph "Management Zone A"
            MON_A[Monitoring Stack<br/>Prometheus + Grafana<br/>Kafka Manager]:::monitor
            SCHEMA_A[Schema Registry<br/>Confluent/Apicurio]:::client
        end
    end

    %% 機房 B
    subgraph "Data Center B (DR)"
        subgraph "Availability Zone B1"
            LB_B1[Load Balancer<br/>HAProxy/Nginx]:::lb
            B5[Kafka Broker 5<br/>rack: rack-b1]:::broker
            B6[Kafka Broker 6<br/>rack: rack-b1]:::broker
            ZK3[ZooKeeper 3<br/>myid: 3]:::zk
        end
        
        subgraph "Management Zone B"
            MON_B[Monitoring Stack<br/>Prometheus + Grafana]:::monitor
            MM[MirrorMaker 2.0<br/>Cross-DC Replication]:::client
        end
    end

    %% 應用層
    subgraph "Application Layer"
        PROD[Producer Apps<br/>Spring Boot<br/>Node.js<br/>Python]:::client
        CONS[Consumer Apps<br/>Kafka Streams<br/>Kafka Connect<br/>Flink/Spark]:::client
        STREAM[Streaming Apps<br/>Real-time Analytics<br/>Event Processing]:::client
    end

    %% 連接關係
    PROD --> LB_A1
    PROD --> LB_B1
    LB_A1 --> B1
    LB_A1 --> B2
    LB_B1 --> B5
    LB_B1 --> B6
    
    CONS --> LB_A1
    STREAM --> LB_A1
    
    %% ZooKeeper 叢集
    ZK1 -.-> ZK2
    ZK2 -.-> ZK3
    ZK3 -.-> ZK1
    
    %% Broker 叢集內通訊
    B1 -.-> B2
    B1 -.-> B3
    B1 -.-> B4
    B3 -.-> B4
    
    %% 跨機房複製
    B1 -.->|MirrorMaker| B5
    B2 -.->|MirrorMaker| B6
    
    %% 監控連接
    MON_A -.-> B1
    MON_A -.-> B2
    MON_A -.-> B3
    MON_A -.-> B4
    MON_B -.-> B5
    MON_B -.-> B6
```

### 容器化部署架構 (Kubernetes)

```mermaid
graph TB
    %% 定義樣式
    classDef k8s fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    classDef pod fill:#ffcdd2,stroke:#d32f2f,stroke-width:2px,color:#000
    classDef service fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px,color:#000
    classDef storage fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#000
    classDef ingress fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000

    %% Kubernetes 叢集
    subgraph "Kubernetes Cluster"
        subgraph "kafka Namespace"
            %% Ingress
            ING[Ingress Controller<br/>nginx/traefik]:::ingress
            
            %% Kafka Services
            SVC_KAFKA[Kafka Service<br/>ClusterIP: 9092<br/>External: 9094]:::service
            SVC_ZK[ZooKeeper Service<br/>ClusterIP: 2181]:::service
            
            %% Kafka StatefulSet
            subgraph "Kafka StatefulSet"
                POD_K1[kafka-0<br/>Image: confluentinc/cp-kafka<br/>CPU: 2, Memory: 4Gi]:::pod
                POD_K2[kafka-1<br/>Image: confluentinc/cp-kafka<br/>CPU: 2, Memory: 4Gi]:::pod
                POD_K3[kafka-2<br/>Image: confluentinc/cp-kafka<br/>CPU: 2, Memory: 4Gi]:::pod
            end
            
            %% ZooKeeper StatefulSet
            subgraph "ZooKeeper StatefulSet"
                POD_Z1[zookeeper-0<br/>Image: confluentinc/cp-zookeeper<br/>CPU: 1, Memory: 2Gi]:::pod
                POD_Z2[zookeeper-1<br/>Image: confluentinc/cp-zookeeper<br/>CPU: 1, Memory: 2Gi]:::pod
                POD_Z3[zookeeper-2<br/>Image: confluentinc/cp-zookeeper<br/>CPU: 1, Memory: 2Gi]:::pod
            end
        end
        
        subgraph "monitoring Namespace"
            POD_PROM[Prometheus<br/>Metrics Collection]:::pod
            POD_GRAF[Grafana<br/>Visualization]:::pod
            POD_ALERT[AlertManager<br/>Alert Handling]:::pod
        end
        
        subgraph "Storage"
            PVC_KAFKA[Kafka PVCs<br/>SSD Storage<br/>100Gi each]:::storage
            PVC_ZK[ZooKeeper PVCs<br/>SSD Storage<br/>20Gi each]:::storage
        end
    end

    %% 外部應用
    subgraph "External Applications"
        EXT_PROD[External Producers<br/>REST Proxy<br/>Schema Registry]:::service
        EXT_CONS[External Consumers<br/>Kafka Connect<br/>KSQL Server]:::service
    end

    %% 連接關係
    EXT_PROD --> ING
    EXT_CONS --> ING
    ING --> SVC_KAFKA
    SVC_KAFKA --> POD_K1
    SVC_KAFKA --> POD_K2
    SVC_KAFKA --> POD_K3
    
    SVC_ZK --> POD_Z1
    SVC_ZK --> POD_Z2
    SVC_ZK --> POD_Z3
    
    POD_K1 --> SVC_ZK
    POD_K2 --> SVC_ZK
    POD_K3 --> SVC_ZK
    
    %% 儲存連接
    POD_K1 -.-> PVC_KAFKA
    POD_K2 -.-> PVC_KAFKA
    POD_K3 -.-> PVC_KAFKA
    POD_Z1 -.-> PVC_ZK
    POD_Z2 -.-> PVC_ZK
    POD_Z3 -.-> PVC_ZK
    
    %% 監控連接
    POD_PROM -.-> POD_K1
    POD_PROM -.-> POD_K2
    POD_PROM -.-> POD_K3
    POD_GRAF -.-> POD_PROM
```

### 雲端部署架構 (AWS/Azure/GCP)

```mermaid
graph TB
    %% 定義樣式
    classDef cloud fill:#e3f2fd,stroke:#1565c0,stroke-width:2px,color:#000
    classDef compute fill:#ffcdd2,stroke:#d32f2f,stroke-width:2px,color:#000
    classDef storage fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px,color:#000
    classDef network fill:#fff3e0,stroke:#ef6c00,stroke-width:2px,color:#000
    classDef managed fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px,color:#000

    %% 雲端區域
    subgraph "Cloud Region (us-east-1)"
        subgraph "Availability Zone A"
            %% 計算資源
            EC2_A1[EC2 Instance<br/>m5.2xlarge<br/>Kafka Broker 1]:::compute
            EC2_A2[EC2 Instance<br/>m5.2xlarge<br/>Kafka Broker 2]:::compute
            
            %% 儲存
            EBS_A1[EBS Volume<br/>gp3, 1TB<br/>IOPS: 3000]:::storage
            EBS_A2[EBS Volume<br/>gp3, 1TB<br/>IOPS: 3000]:::storage
        end
        
        subgraph "Availability Zone B"
            EC2_B1[EC2 Instance<br/>m5.2xlarge<br/>Kafka Broker 3]:::compute
            EC2_B2[EC2 Instance<br/>t3.large<br/>ZooKeeper 1]:::compute
            
            EBS_B1[EBS Volume<br/>gp3, 1TB<br/>IOPS: 3000]:::storage
            EBS_B2[EBS Volume<br/>gp3, 100GB<br/>IOPS: 1000]:::storage
        end
        
        subgraph "Availability Zone C"
            EC2_C1[EC2 Instance<br/>m5.2xlarge<br/>Kafka Broker 4]:::compute
            EC2_C2[EC2 Instance<br/>t3.large<br/>ZooKeeper 2]:::compute
            EC2_C3[EC2 Instance<br/>t3.large<br/>ZooKeeper 3]:::compute
            
            EBS_C1[EBS Volume<br/>gp3, 1TB<br/>IOPS: 3000]:::storage
            EBS_C2[EBS Volume<br/>gp3, 100GB<br/>IOPS: 1000]:::storage
            EBS_C3[EBS Volume<br/>gp3, 100GB<br/>IOPS: 1000]:::storage
        end
    end

    %% 網路層
    subgraph "Network Layer"
        VPC[VPC<br/>10.0.0.0/16]:::network
        ALB[Application Load Balancer<br/>Multi-AZ]:::network
        NLB[Network Load Balancer<br/>TCP 9092]:::network
        SG[Security Groups<br/>Port 9092, 2181, 22]:::network
    end

    %% 託管服務
    subgraph "Managed Services"
        MSK[Amazon MSK<br/>Managed Kafka Service<br/>3 Brokers, m5.large]:::managed
        CLOUDWATCH[CloudWatch<br/>Metrics & Logs]:::managed
        S3[S3 Bucket<br/>Backup & Archive]:::managed
        IAM[IAM Roles<br/>Access Control]:::managed
    end

    %% 應用層
    subgraph "Application Services"
        ECS[ECS/Fargate<br/>Containerized Apps]:::compute
        LAMBDA[Lambda Functions<br/>Event Processing]:::compute
        KINESIS[Kinesis Analytics<br/>Stream Processing]:::managed
    end

    %% 連接關係
    ALB --> EC2_A1
    ALB --> EC2_A2
    ALB --> EC2_B1
    ALB --> EC2_C1
    
    NLB --> EC2_A1
    NLB --> EC2_A2
    NLB --> EC2_B1
    NLB --> EC2_C1
    
    EC2_A1 -.-> EBS_A1
    EC2_A2 -.-> EBS_A2
    EC2_B1 -.-> EBS_B1
    EC2_C1 -.-> EBS_C1
    
    EC2_B2 -.-> EBS_B2
    EC2_C2 -.-> EBS_C2
    EC2_C3 -.-> EBS_C3
    
    ECS --> NLB
    LAMBDA --> MSK
    KINESIS --> MSK
    
    CLOUDWATCH -.-> EC2_A1
    CLOUDWATCH -.-> EC2_A2
    CLOUDWATCH -.-> EC2_B1
    CLOUDWATCH -.-> EC2_C1
```

## 部署配置詳解

### 硬體規格建議

#### 生產環境 Broker 規格
```yaml
# 高吞吐量配置
CPU: 16+ cores
Memory: 32-64 GB
Storage: NVMe SSD, 2-10 TB
Network: 10 Gbps+

# 中等負載配置  
CPU: 8-16 cores
Memory: 16-32 GB
Storage: SSD, 1-2 TB
Network: 1-10 Gbps
```

#### ZooKeeper 規格
```yaml
# ZooKeeper 配置
CPU: 4-8 cores
Memory: 8-16 GB
Storage: SSD, 100-500 GB
Network: 1 Gbps
```

### 網路配置

#### 防火牆規則
```bash
# Kafka Broker
9092/tcp  # Client connections
9093/tcp  # Inter-broker communication
9094/tcp  # External connections

# ZooKeeper
2181/tcp  # Client connections
2888/tcp  # Follower connections
3888/tcp  # Leader election

# JMX Monitoring
9999/tcp  # JMX port
```

#### DNS 配置
```yaml
# 內部 DNS
kafka-1.internal.company.com
kafka-2.internal.company.com
kafka-3.internal.company.com

# 外部 DNS
kafka.company.com -> Load Balancer
```

### 容器化配置

#### Docker Compose 範例
```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zk-data:/var/lib/zookeeper/data
      - zk-logs:/var/lib/zookeeper/log

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: false
      KAFKA_LOG_RETENTION_HOURS: 168
    volumes:
      - kafka-data:/var/lib/kafka/data
```

#### Kubernetes Helm Chart
```yaml
# values.yaml
kafka:
  replicaCount: 3
  resources:
    requests:
      memory: "4Gi"
      cpu: "2"
    limits:
      memory: "8Gi"
      cpu: "4"
  
  persistence:
    enabled: true
    size: 100Gi
    storageClass: "fast-ssd"
  
  configurationOverrides:
    "log.retention.hours": "168"
    "log.segment.bytes": "1073741824"
    "num.network.threads": "8"
    "num.io.threads": "16"
```

## 監控和運維

### 監控指標

#### 系統指標
```yaml
# CPU 和記憶體
node_cpu_usage_percent
node_memory_usage_percent
node_disk_usage_percent

# 網路
node_network_receive_bytes
node_network_transmit_bytes
```

#### Kafka 指標
```yaml
# Broker 指標
kafka_server_brokertopicmetrics_messagesinpersec
kafka_server_brokertopicmetrics_bytesinpersec
kafka_controller_kafkacontroller_activecontrollercount

# Consumer 指標
kafka_consumer_lag_sum
kafka_consumer_records_consumed_rate
```

### 備份策略

#### 資料備份
```bash
# 定期快照
aws ec2 create-snapshot --volume-id vol-xxx --description "kafka-backup-$(date)"

# 跨區域複製
kafka-mirror-maker.sh --consumer.config source.properties \
                      --producer.config target.properties \
                      --whitelist ".*"
```

#### 配置備份
```bash
# 備份 ZooKeeper 資料
zkCli.sh -server localhost:2181 ls /
zkCli.sh -server localhost:2181 get /brokers/ids/1

# 備份 Kafka 配置
cp /opt/kafka/config/server.properties /backup/
```

## 災難恢復

### 故障場景處理

#### Broker 故障
```mermaid
flowchart TD
    A[Broker 故障檢測] --> B{是否為 Controller?}
    B -->|是| C[Controller 重新選舉]
    B -->|否| D[分區 Leader 重新選舉]
    C --> E[更新元數據]
    D --> E
    E --> F[客戶端重新連接]
```

#### 機房故障
```mermaid
flowchart TD
    A[機房故障] --> B[切換到 DR 機房]
    B --> C[啟動 MirrorMaker]
    C --> D[更新 DNS 記錄]
    D --> E[客戶端重新路由]
```

### 恢復程序

#### 資料恢復
```bash
# 從快照恢復
aws ec2 create-volume --snapshot-id snap-xxx --availability-zone us-east-1a

# 重建 Broker
kafka-server-start.sh config/server.properties

# 驗證資料完整性
kafka-log-dirs.sh --bootstrap-server localhost:9092 --describe
```

## 最佳實踐

### 部署建議
1. **機架感知**: 配置 `broker.rack` 實現跨機架分布
2. **資源隔離**: 使用專用機器避免資源競爭
3. **網路優化**: 配置專用網路避免頻寬競爭
4. **監控完整**: 部署完整的監控和告警系統

### 安全配置
```properties
# SSL 配置
security.inter.broker.protocol=SSL
ssl.keystore.location=/path/to/keystore.jks
ssl.truststore.location=/path/to/truststore.jks

# SASL 配置
sasl.mechanism.inter.broker.protocol=PLAIN
sasl.enabled.mechanisms=PLAIN,SCRAM-SHA-256

# ACL 配置
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
```

### 效能調優
```properties
# 網路調優
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400

# 日誌調優
log.segment.bytes=1073741824
log.retention.hours=168
log.cleanup.policy=delete
```
