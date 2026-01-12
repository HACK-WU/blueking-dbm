# DataNode 节点管理

<cite>
**本文档引用的文件**  
- [install_datanode.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/hdfscmd/install_datanode.go)
- [refresh_nodes.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/hdfscmd/refresh_nodes.go)
- [check_decommission.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/hdfscmd/check_decommission.go)
- [update_dfs_host.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/hdfscmd/update_dfs_host.go)
- [install_hdfs.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/hdfs/install_hdfs.go)
</cite>

## 目录
1. [简介](#简介)
2. [DataNode 安装流程](#datanode-安装流程)
3. [节点动态刷新机制](#节点动态刷新机制)
4. [节点退役状态检查](#节点退役状态检查)
5. [HDFS 主机配置更新](#hdfs-主机配置更新)
6. [节点扩缩容与维护操作](#节点扩缩容与维护操作)
7. [与 NameNode 的通信机制](#与-namenode-的通信机制)
8. [数据块报告流程](#数据块报告流程)

## 简介
本文档详细阐述了 HDFS DataNode 节点的全生命周期管理机制，涵盖安装、刷新、退役检查、配置更新等核心功能。系统通过 `dbactuator` 工具实现自动化运维，确保 HDFS 集群的高可用性与弹性伸缩能力。

## DataNode 安装流程

`install_datanode.go` 文件定义了 DataNode 的安装命令和执行流程。该流程通过 `InstallDataNodeAct` 结构体封装，调用 `InstallHdfsService.InstallDataNode()` 方法完成具体操作。

安装流程主要包括：
1. 清理旧的数据目录 `/data/hadoopdata/data`
2. 根据主机映射替换配置文件中的 `{{dn_host}}` 占位符
3. 创建新的数据目录并设置正确的权限
4. 更新 HDFS 数据目录配置，支持多磁盘挂载
5. 配置磁盘预留空间和坏盘容忍数量
6. 更新 Supervisor 配置以管理 DataNode 进程

该流程确保了 DataNode 在新节点上的正确部署和配置初始化。

**节来源**
- [install_datanode.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/hdfscmd/install_datanode.go#L1-L106)
- [install_hdfs.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/hdfs/install_hdfs.go#L239-L278)

## 节点动态刷新机制

`refresh_nodes.go` 实现了 HDFS 节点列表的动态刷新功能。当集群拓扑发生变化（如新增或移除 DataNode）时，需要通知 NameNode 重新加载节点配置。

刷新流程通过 `RefreshNodesAct` 结构体执行，调用 `RefreshNodesService.RefreshNodes()` 方法。该操作会触发 NameNode 重新读取 `dfs.include` 和 `dfs.exclude` 文件，同步最新的节点列表。

在集群扩容或缩容后，此步骤至关重要，确保 NameNode 能够识别新加入的 DataNode 或停止与已移除节点的通信。

**节来源**
- [refresh_nodes.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/hdfscmd/refresh_nodes.go#L1-L100)

## 节点退役状态检查

`check_decommission.go` 提供了对 DataNode 退役状态的检查功能。当需要安全地移除一个 DataNode 时，系统会先将其标记为退役状态，等待其上的数据块复制到其他节点后再下线。

`CheckDecommissionAct` 结构体负责执行检查任务，调用 `CheckDecommissionService.CheckDatanodeDecommission()` 方法。该方法会查询 NameNode，确认指定 DataNode 是否已完成数据迁移，确保在节点下线前所有数据块都有足够的副本。

这一机制保障了数据的完整性和集群的稳定性。

**节来源**
- [check_decommission.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/hdfscmd/check_decommission.go#L1-L102)

## HDFS 主机配置更新

`update_dfs_host.go` 负责管理 HDFS 的主机配置文件（`dfs.hosts` 和 `dfs.hosts.exclude`）。这些文件定义了允许连接到 NameNode 的 DataNode 列表。

`UpdateDfsHostAct` 结构体通过 `UpdateDfsHostService.UpdateDfsHost()` 方法更新配置文件内容。在节点上线时，将其主机名添加到 `dfs.include`；在节点下线或退役时，可将其移入 `dfs.exclude`。

此功能是实现节点准入控制和灰度下线的核心，与 `refresh_nodes` 配合使用，完成节点的全生命周期管理。

**节来源**
- [update_dfs_host.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/hdfscmd/update_dfs_host.go#L1-L102)
- [install_hdfs.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/hdfs/install_hdfs.go#L428-L438)

## 节点扩缩容与维护操作

### 扩容操作
1. 在配置中添加新 DataNode 的 IP 和主机名映射
2. 执行 `update_dfs_host` 将新节点添加到 `dfs.include`
3. 在新节点上执行 `install_datanode` 安装并启动服务
4. 执行 `refresh_nodes` 通知 NameNode 刷新节点列表

### 缩容与维护操作
1. 执行 `update_dfs_host` 将目标节点从 `dfs.include` 移除（或加入 `dfs.exclude`）
2. 执行 `refresh_nodes` 使配置生效
3. 使用 `check_decommission` 检查节点是否已安全退役
4. 确认无数据风险后，可安全停止节点服务并移除

这些操作通过自动化流程编排，确保了集群变更的安全性和一致性。

## 与 NameNode 的通信机制

DataNode 与 NameNode 通过 HDFS 私有协议进行通信，主要包括：
- **心跳机制**：DataNode 定期向 NameNode 发送心跳，报告其存活状态
- **块报告**：DataNode 启动时和周期性地向 NameNode 发送其存储的数据块列表
- **指令响应**：NameNode 可通过心跳响应向 DataNode 下发指令，如创建、删除或复制数据块

NameNode 依赖这些通信来维护集群的元数据视图，确保数据的可靠性和负载均衡。

## 数据块报告流程

DataNode 在以下情况下会向 NameNode 报告其数据块信息：
1. **启动时**：执行全量块报告（Full Block Report）
2. **运行中**：周期性执行增量块报告（Incremental Block Report）
3. **块状态变化**：当创建、删除或接收到新的数据块副本时

NameNode 根据这些报告更新其内存中的块映射表，用于后续的读写请求路由和副本管理。该机制是 HDFS 实现高可用和容错的基础。