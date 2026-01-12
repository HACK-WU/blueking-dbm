# Pulsar 服务管理

<cite>
**本文档引用文件**  
- [install_broker.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_broker.go)
- [install_bookkeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_bookkeeper.go)
- [install_zookeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_zookeeper.go)
- [init_cluster.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/init_cluster.go)
- [install_pulsar_manager.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_pulsar_manager.go)
- [check_under_replicated.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/check_under_replicated.go)
- [check_ledger_metadata.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/check_ledger_metadata.go)
- [decommission_bookie.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/decommission_bookie.go)
- [set_bookie_readonly.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/set_bookie_readonly.go)
- [unset_bookie_readonly.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/unset_bookie_readonly.go)
- [install_pulsar.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/pulsar/install_pulsar.go)
- [check_shrink.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/pulsar/check_shrink.go)
- [pulsar.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/pulsar/pulsar.go)
</cite>

## 目录
1. [引言](#引言)
2. [核心组件部署](#核心组件部署)
3. [集群初始化流程](#集群初始化流程)
4. [管理控制台集成](#管理控制台集成)
5. [健康检查机制](#健康检查机制)
6. [Bookie 节点退役与数据迁移](#bookie-节点退役与数据迁移)
7. [Bookie 只读模式管理](#bookie-只读模式管理)
8. [部署与扩容实例](#部署与扩容实例)

## 引言
本文档详细阐述了 Pulsar 服务管理系统的实现机制，重点分析了 `pulsarcmd` 模块中各核心功能组件的作用与交互。通过深入解析代码结构与执行流程，为 Pulsar 集群的部署、维护和管理提供全面的技术指导。

## 核心组件部署

`pulsarcmd` 模块提供了部署 Pulsar 集群核心组件的命令行接口，包括 Broker、BookKeeper 和 ZooKeeper 三个关键服务。

`install_broker.go` 文件定义了 `InstallPulsarBrokerAct` 结构体和 `InstallPulsarBrokerCommand` 函数，实现了 Pulsar Broker 服务的安装功能。该命令通过调用 `pulsar.InstallPulsarComp` 组件的 `InstallBroker` 方法来执行具体的安装步骤。

`install_bookkeeper.go` 文件中的 `InstallPulsarBookkeeperAct` 结构体负责 BookKeeper 服务的部署。其 `Run` 方法通过执行 `InstallBookkeeper` 步骤完成 BookKeeper 节点的安装配置。

`install_zookeeper.go` 文件实现了 ZooKeeper 集群的安装功能，通过 `InstallPulsarZookeeperAct` 结构体和 `InstallPulsarZookeeperCommand` 命令，调用底层组件完成 ZooKeeper 服务的部署。

这些安装命令遵循统一的执行模式：首先进行参数校验，然后初始化服务参数，最后按步骤执行安装流程。所有操作均支持回滚机制，确保部署过程的可靠性。

**本节来源**
- [install_broker.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_broker.go#L1-L105)
- [install_bookkeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_bookkeeper.go#L1-L105)
- [install_zookeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_zookeeper.go#L1-L105)
- [install_pulsar.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/pulsar/install_pulsar.go)

## 集群初始化流程

`init_cluster.go` 文件定义了 `InitPulsarClusterAct` 结构体和 `InitPulsarClusterCommand` 命令，用于执行 Pulsar 集群的初始化操作。该流程是集群部署的关键步骤，确保所有组件正确配置并能协同工作。

集群初始化过程通过调用 `pulsar.InstallPulsarComp` 组件的 `InitCluster` 方法实现。该方法负责设置集群级别的配置，包括元数据初始化、命名空间创建、租户配置等。初始化操作确保了 Pulsar 集群在投入使用前处于一致且可用的状态。

初始化命令的执行流程与其他安装命令类似，包含参数校验、初始化和执行三个阶段。成功执行后，系统会记录操作日志并返回成功状态。

**本节来源**
- [init_cluster.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/init_cluster.go#L1-L105)
- [install_pulsar.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/pulsar/install_pulsar.go)

## 管理控制台集成

`install_pulsar_manager.go` 文件实现了 Pulsar Manager 管理控制台的安装功能。`InstallPulsarManagerAct` 结构体封装了安装操作，通过 `InstallPulsarManagerCommand` 命令提供接口。

Pulsar Manager 是一个 Web 界面，用于监控和管理 Pulsar 集群。其安装过程包括部署管理服务、配置与 Pulsar 集群的连接、设置访问权限等步骤。该功能通过调用 `pulsar.InstallPulsarComp` 组件的 `InstallPulsarManager` 方法完成安装。

集成管理控制台极大地简化了集群的日常运维工作，提供了直观的监控指标、主题管理、租户配置等功能。

**本节来源**
- [install_pulsar_manager.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_pulsar_manager.go#L1-L105)
- [install_pulsar.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/pulsar/install_pulsar.go)

## 健康检查机制

Pulsar 集群的健康检查由两个关键命令实现：`check_under_replicated.go` 和 `check_ledger_metadata.go`。

`check_under_replicated.go` 文件中的 `CheckUnderReplicatedAct` 结构体提供了检查未充分复制的 ledger 的功能。该检查对于确保数据高可用性至关重要，能够识别出副本数不足的 ledger，提示管理员采取补救措施。

`check_ledger_metadata.go` 文件定义了 `CheckLedgerMetadataAct` 结构体，用于检查 ledger 的元数据完整性。该功能通过验证 ledger 的元数据信息，确保数据存储的一致性和可靠性。

这两个健康检查命令均使用 `pulsar.CheckPulsarShrinkComp` 组件，通过 `CheckUnderReplicated` 和 `CheckLedgerMetadata` 方法执行具体的检查逻辑。检查结果可用于评估集群的健康状况和数据安全性。

**本节来源**
- [check_under_replicated.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/check_under_replicated.go#L1-L105)
- [check_ledger_metadata.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/check_ledger_metadata.go#L1-L103)
- [check_shrink.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/pulsar/check_shrink.go)

## Bookie 节点退役与数据迁移

`decommission_bookie.go` 文件实现了 Bookie 节点的退役功能。`DecommissionBookieAct` 结构体通过 `DecommissionBookieCommand` 命令提供接口，用于安全地从集群中移除 Bookie 节点。

当需要缩容或更换硬件时，此功能至关重要。退役过程会自动触发数据迁移，将待退役 Bookie 上的所有 ledger 数据重新复制到其他健康的 Bookie 节点上。只有在所有数据成功迁移后，该 Bookie 才会被标记为可移除状态。

该操作通过调用 `pulsar.CheckPulsarShrinkComp` 组件的 `DecommissionBookie` 方法实现，确保了数据迁移过程的安全性和完整性。

**本节来源**
- [decommission_bookie.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/decommission_bookie.go#L1-L105)
- [check_shrink.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/pulsar/check_shrink.go)

## Bookie 只读模式管理

为了支持安全维护操作，系统提供了 Bookie 只读模式的切换功能，由 `set_bookie_readonly.go` 和 `unset_bookie_readonly.go` 两个文件实现。

`set_bookie_readonly.go` 文件中的 `SetBookieReadOnlyAct` 结构体实现了将 Bookie 设置为只读状态的功能。在此模式下，Bookie 不再接受新的写入请求，但仍能处理读取操作。这为执行维护任务（如磁盘清理、软件升级）提供了安全窗口。

`unset_bookie_readonly.go` 文件中的 `UnsetBookieReadOnlyAct` 结构体则负责取消只读状态，使 Bookie 恢复正常的读写能力。

这两个操作均通过 `pulsar.CheckPulsarShrinkComp` 组件的相应方法实现，确保了状态切换的原子性和一致性。

**本节来源**
- [set_bookie_readonly.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/set_bookie_readonly.go#L1-L97)
- [unset_bookie_readonly.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/unset_bookie_readonly.go#L1-L97)
- [check_shrink.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/pulsar/check_shrink.go)

## 部署与扩容实例

一个完整的 Pulsar 集群部署与 Bookie 节点扩容操作可以按以下步骤进行：

1. **部署 ZooKeeper 集群**：使用 `install_zookeeper` 命令部署作为 Pulsar 元数据存储的 ZooKeeper 集群。
2. **部署 BookKeeper 集群**：使用 `install_bookkeeper` 命令部署初始的 BookKeeper 节点。
3. **部署 Broker 服务**：使用 `install_broker` 命令部署 Pulsar Broker 服务。
4. **初始化集群**：使用 `init_cluster` 命令完成集群的初始化配置。
5. **安装管理控制台**：使用 `install_pulsar_manager` 命令部署 Pulsar Manager。
6. **扩容 Bookie 节点**：当需要增加存储容量时，使用 `install_bookkeeper` 命令在新节点上部署 BookKeeper 服务，系统会自动将其加入集群并开始数据均衡。

在整个生命周期中，可使用 `check_under_replicated` 和 `check_ledger_metadata` 命令定期检查集群健康状况。在进行维护时，可先使用 `set_bookie_readonly` 将目标节点设为只读，完成维护后再用 `unset_bookie_readonly` 恢复其正常状态。当需要永久移除节点时，使用 `decommission_bookie` 命令安全地退役 Bookie 节点。

**本节来源**
- [install_broker.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_broker.go)
- [install_bookkeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_bookkeeper.go)
- [install_zookeeper.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_zookeeper.go)
- [init_cluster.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/init_cluster.go)
- [install_pulsar_manager.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/install_pulsar_manager.go)
- [check_under_replicated.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/check_under_replicated.go)
- [check_ledger_metadata.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/check_ledger_metadata.go)
- [decommission_bookie.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/decommission_bookie.go)
- [set_bookie_readonly.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/set_bookie_readonly.go)
- [unset_bookie_readonly.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/pulsarcmd/unset_bookie_readonly.go)