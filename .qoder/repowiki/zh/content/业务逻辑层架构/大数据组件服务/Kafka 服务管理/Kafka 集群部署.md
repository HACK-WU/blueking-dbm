# Kafka 集群部署

<cite>
**本文档引用文件**  
- [install_broker.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_broker.go)
- [install_zookeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_zookeeper.go)
- [decompress_kafka_pkg.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/decompress_kafka_pkg.go)
- [install_supervisor.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_supervisor.go)
- [install_kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/install_kafka.go)
- [kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/kafka.go)
- [kafka.py](file://dbm-ui/backend/db_services/bigdata/kafka/views.py)
</cite>

## 目录
1. [简介](#简介)
2. [核心组件分析](#核心组件分析)
3. [Kafka Broker 安装流程](#kafka-broker-安装流程)
4. [ZooKeeper 部署流程](#zookeeper-部署流程)
5. [介质解压与 Supervisor 进程管理](#介质解压与-supervisor-进程管理)
6. [前端调用后端 API 流程](#前端调用后端-api-流程)
7. [三节点 Kafka 集群部署实战](#三节点-kafka-集群部署实战)
8. [配置文件生成逻辑与关键参数](#配置文件生成逻辑与关键参数)
9. [总结](#总结)

## 简介
本文档详细描述了在 BlueKing DBM 系统中部署 Kafka 集群的完整流程。重点分析 `install_broker.go` 中如何解压安装包、配置 Broker 参数并启动服务，以及 `install_zookeeper.go` 对 ZooKeeper 依赖的部署流程。同时说明 `decompress_kafka_pkg.go` 在介质解压中的作用和 `install_supervisor.go` 如何通过 Supervisor 管理进程。结合 `dbm-ui` 的 `views.py` 说明前端调用后端 API 触发部署的完整流程。最后提供一个从零开始部署三节点 Kafka 集群的实战示例，并解释各配置文件的生成逻辑和关键参数。

## 核心组件分析

**Section sources**
- [install_broker.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_broker.go)
- [install_zookeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_zookeeper.go)
- [decompress_kafka_pkg.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/decompress_kafka_pkg.go)
- [install_supervisor.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_supervisor.go)

## Kafka Broker 安装流程

```mermaid
sequenceDiagram
participant 前端 as 前端界面
participant API as 后端API
participant Actuator as DBActuator
participant Kafka as Kafka Broker
前端->>API : 提交部署请求
API->>Actuator : 调用install_broker命令
Actuator->>Actuator : 初始化参数
Actuator->>Actuator : 创建用户和目录
Actuator->>Actuator : 配置环境变量
Actuator->>Kafka : 启动Broker进程
Kafka-->>Actuator : 返回成功状态
Actuator-->>API : 返回执行结果
API-->>前端 : 显示部署成功
```

**Diagram sources**
- [install_broker.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_broker.go)
- [install_kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/install_kafka.go)

**Section sources**
- [install_broker.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_broker.go)
- [install_kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/install_kafka.go)

## ZooKeeper 部署流程

```mermaid
sequenceDiagram
participant 前端 as 前端界面
participant API as 后端API
participant Actuator as DBActuator
participant ZooKeeper as ZooKeeper
前端->>API : 提交ZooKeeper部署请求
API->>Actuator : 调用install_zookeeper命令
Actuator->>Actuator : 检查端口占用
Actuator->>Actuator : 创建软链接
Actuator->>Actuator : 创建数据目录
Actuator->>Actuator : 配置zoo.cfg
Actuator->>Actuator : 写入myid文件
Actuator->>ZooKeeper : 启动ZooKeeper进程
ZooKeeper-->>Actuator : 返回成功状态
Actuator-->>API : 返回执行结果
API-->>前端 : 显示部署成功
```

**Diagram sources**
- [install_zookeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_zookeeper.go)
- [install_kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/install_kafka.go)

**Section sources**
- [install_zookeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_zookeeper.go)
- [install_kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/install_kafka.go)

## 介质解压与 Supervisor 进程管理

```mermaid
flowchart TD
A[开始] --> B[复制安装包]
B --> C[进入目标目录]
C --> D[执行tar解压]
D --> E[验证解压结果]
E --> F{解压成功?}
F --> |是| G[记录成功日志]
F --> |否| H[返回错误信息]
G --> I[结束]
H --> I
```

```mermaid
flowchart TD
A[开始] --> B[检查Supervisor存在]
B --> C[创建Python软链接]
C --> D[创建Supervisord配置软链接]
D --> E[创建supervisorctl软链接]
E --> F[替换环境变量]
F --> G[设置crontab任务]
G --> H[启动Supervisord进程]
H --> I{启动成功?}
I --> |是| J[记录成功日志]
I --> |否| K[返回错误信息]
J --> L[结束]
K --> L
```

**Diagram sources**
- [decompress_kafka_pkg.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/decompress_kafka_pkg.go)
- [install_supervisor.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_supervisor.go)
- [install_kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/install_kafka.go)

**Section sources**
- [decompress_kafka_pkg.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/decompress_kafka_pkg.go)
- [install_supervisor.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_supervisor.go)
- [install_kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/install_kafka.go)

## 前端调用后端 API 流程

```mermaid
sequenceDiagram
participant 用户 as 用户
participant 前端 as 前端界面
participant 后端 as 后端API
participant 执行器 as DBActuator
用户->>前端 : 在UI上选择部署Kafka集群
前端->>后端 : 发送REST API请求
后端->>后端 : 验证请求参数
后端->>执行器 : 调用DBActuator执行部署命令
执行器->>执行器 : 执行安装步骤
执行器-->>后端 : 返回执行结果
后端-->>前端 : 返回响应
前端-->>用户 : 显示部署进度和结果
```

**Diagram sources**
- [views.py](file://dbm-ui/backend/db_services/bigdata/kafka/views.py)
- [install_broker.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_broker.go)

**Section sources**
- [views.py](file://dbm-ui/backend/db_services/bigdata/kafka/views.py)

## 三节点 Kafka 集群部署实战
本节提供一个从零开始部署三节点 Kafka 集群的完整示例。

### 准备工作
1. 确保三台服务器已安装操作系统并配置好网络
2. 确保服务器之间可以互相SSH访问
3. 准备Kafka和ZooKeeper安装包

### 部署步骤
1. **部署ZooKeeper集群**
   - 在每个节点上执行 `install_zookeeper` 命令
   - 配置 `zoo.cfg` 文件中的服务器列表
   - 在每个节点的 `myid` 文件中写入对应的ID（1, 2, 3）

2. **解压Kafka安装包**
   - 在每个节点上执行 `decompress_pkg` 命令
   - 将Kafka安装包解压到 `/data/kafkaenv` 目录

3. **安装Supervisor**
   - 在每个节点上执行 `install_supervisor` 命令
   - 配置Supervisor的crontab任务以确保进程监控

4. **部署Kafka Broker**
   - 在每个节点上执行 `install_broker` 命令
   - 配置 `server.properties` 文件中的关键参数
   - 启动Kafka Broker进程

5. **验证集群状态**
   - 使用Kafka自带的命令行工具检查集群状态
   - 确认所有Broker都已成功加入集群

**Section sources**
- [install_broker.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_broker.go)
- [install_zookeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_zookeeper.go)
- [decompress_kafka_pkg.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/decompress_kafka_pkg.go)
- [install_supervisor.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/kafkacmd/install_supervisor.go)

## 配置文件生成逻辑与关键参数

### server.properties 配置文件
该文件是Kafka Broker的核心配置文件，由 `install_kafka.go` 中的 `InstallBroker` 方法生成。

```mermaid
flowchart TD
A[开始] --> B[获取基础参数]
B --> C[设置retention.hours]
C --> D[设置default.replication.factor]
D --> E[设置num.partitions]
E --> F[设置num.network.threads]
F --> G[设置log.dirs]
G --> H[设置listeners]
H --> I[设置zookeeper.connect]
I --> J[写入server.properties]
J --> K[结束]
```

**关键参数说明：**
- **broker.id**: Broker的唯一标识符
- **listeners**: 监听地址和端口
- **log.dirs**: 日志数据存储目录
- **zookeeper.connect**: ZooKeeper连接字符串
- **log.retention.hours**: 日志保留时间（小时）
- **default.replication.factor**: 默认副本数
- **num.partitions**: 默认分区数

### zoo.cfg 配置文件
该文件是ZooKeeper的核心配置文件，由 `install_kafka.go` 中的 `configZookeeper` 方法生成。

**关键参数说明：**
- **tickTime**: ZooKeeper的基本时间单位（毫秒）
- **initLimit**: Follower初始化连接到Leader的最大心跳数
- **syncLimit**: Follower与Leader之间发送消息的最大时间间隔
- **dataDir**: 内存数据库快照存放位置
- **clientPort**: 客户端连接端口
- **server.x**: 集群中每个服务器的配置

**Diagram sources**
- [install_kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/install_kafka.go)
- [kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/kafka.go)

**Section sources**
- [install_kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/install_kafka.go)
- [kafka.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/kafka/kafka.go)

## 总结
本文档详细介绍了在 BlueKing DBM 系统中部署 Kafka 集群的完整流程。通过分析 `install_broker.go`、`install_zookeeper.go`、`decompress_kafka_pkg.go` 和 `install_supervisor.go` 等核心组件，我们了解了从介质解压到服务启动的全过程。同时，通过分析 `dbm-ui` 的 `views.py` 文件，我们理解了前端如何通过 API 调用触发后端部署流程。最后，我们提供了一个三节点 Kafka 集群的部署实战示例，并解释了关键配置文件的生成逻辑和参数含义。这些内容为运维人员提供了完整的 Kafka 集群部署指南。