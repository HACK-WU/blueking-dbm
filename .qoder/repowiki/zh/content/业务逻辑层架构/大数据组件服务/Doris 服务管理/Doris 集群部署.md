# Doris 集群部署

<cite>
**本文档引用的文件**  
- [install_doris.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/doriscmd/install_doris.go)
- [decompress_pkg.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/doriscmd/decompress_pkg.go)
- [install_supervisor.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/doriscmd/install_supervisor.go)
- [init_system_config.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/doriscmd/init_system_config.go)
- [install_doris.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/doris/install_doris.go)
- [decompress_pkg.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/doris/decompress_pkg.go)
- [init_system_config.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/doris/init_system_config.go)
- [views.py](file://dbm-ui/backend/db_proxy/frontend_views/views.py)
</cite>

## 目录
1. [简介](#简介)
2. [Doris 部署流程概览](#doris-部署流程概览)
3. [核心部署组件分析](#核心部署组件分析)
4. [前端视图与 API 触发机制](#前端视图与-api-触发机制)
5. [部署参数示例](#部署参数示例)
6. [日志分析方法](#日志分析方法)
7. [部署失败常见原因及排查步骤](#部署失败常见原因及排查步骤)

## 简介
本文档详细描述了在 BlueKing DBM 系统中部署 Doris 集群的完整流程。重点涵盖 FE（Frontend）和 BE（Backend）节点的安装过程，包括软件包解压、Supervisor 进程管理配置以及初始系统配置等关键步骤。同时，文档解释了前端如何通过 API 触发部署任务，并提供部署参数示例、日志分析方法以及部署失败的常见原因和排查步骤。

## Doris 部署流程概览
Doris 集群的部署由多个有序步骤组成，每个步骤通过独立的命令执行，确保部署过程的模块化和可维护性。主要流程如下：

1. **初始化系统配置**：创建执行用户、初始化安装目录、修改系统配置（如文件句柄数、swap 设置）、写入环境变量。
2. **解压软件包**：将包含 Doris、JDK 和 Supervisor 的压缩包解压到指定目录，并创建软链接。
3. **安装 Supervisor**：配置 Supervisor 作为进程守护工具，确保 Doris 服务的稳定运行。
4. **安装 Doris 节点**：根据角色（FE 或 BE）渲染配置文件，启动并验证服务。

这些步骤通过 `dbactuator` 工具的子命令实现，每个命令对应一个具体的部署任务。

## 核心部署组件分析

### 软件包解压 (decompress_pkg.go)
该组件负责将 Doris 软件包解压到目标目录，并为不同角色（FE/BE）创建相应的软链接。

**功能流程**：
- 验证解压目标目录是否存在。
- 执行 `tar zxf` 命令解压指定版本的 Doris 包。
- 在 Doris 环境目录下创建指向具体角色目录的软链接（如 `fe` 指向 `doris-2.0.4/fe`）。

**关键参数**：
- `Version`：Doris 版本号（如 2.0.4）
- `Role`：节点角色（fe/be）

**Section sources**
- [decompress_pkg.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/doriscmd/decompress_pkg.go#L1-L105)
- [decompress_pkg.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/doris/decompress_pkg.go#L1-L60)

### Supervisor 进程管理配置 (install_supervisor.go)
该组件配置 Supervisor 作为 Doris 服务的守护进程，确保服务异常退出后能自动重启。

**功能流程**：
- 创建 Supervisor 配置文件的软链接（`/etc/supervisord.conf`）。
- 创建 `supervisorctl` 和 `supervisord` 命令的软链接。
- 为 `mysql` 用户配置定时任务（crontab），每分钟检查 Supervisor 是否正常运行。
- 启动 Supervisor 服务。

**关键操作**：
- 使用 `ln -sf` 创建软链接
- 使用 `crontab -u mysql` 为执行用户添加监控任务
- 使用 `su - mysql -c` 以指定用户身份启动 Supervisor

**Section sources**
- [install_supervisor.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/doriscmd/install_supervisor.go#L1-L97)

### 初始系统配置 (init_system_config.go)
该组件负责在部署前对目标主机进行系统级配置，为 Doris 运行提供合适的环境。

**功能流程**：
- **创建执行用户**：检查并创建名为 `mysql` 的系统用户（若不存在）。
- **初始化安装目录**：创建 `/data/dorisenv` 等目录并赋予 `mysql` 用户权限。
- **修改系统配置**：
  - 设置文件句柄数限制（`/etc/security/limits.d/doris-no-file.conf`）
  - 关闭 swap 分区
  - 对于 BE 节点，增加 `vm.max_map_count` 的值
- **写入 profile**：将 Java 和 Doris 的环境变量写入 `/etc/profile`，确保服务能正确加载。

**Section sources**
- [init_system_config.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/doriscmd/init_system_config.go#L1-L115)
- [init_system_config.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/doris/init_system_config.go#L1-L134)

### FE 和 BE 节点安装 (install_doris.go)
该组件是 Doris 集群部署的核心，负责根据节点角色安装和配置 FE 或 BE 服务。

**功能流程**：
- **渲染配置文件**：
  - 对于 FE 节点：生成 `fe.conf`，设置 `priority_networks`、JVM 堆内存大小等。
  - 对于 BE 节点：生成 `be.conf`，设置数据目录路径、介质类型（SSD/HDD）等。
- **启动 FE 服务**：通过 `start_fe.sh` 启动 FE，并通过 HTTP 接口检查服务是否正常启动。
- **启动 BE 服务**：通过 `start_be.sh` 启动 BE，并检查查询端口是否开放。
- **服务注册**：将新节点信息注册到集群中。

**关键参数**：
- `Host`：节点 IP 地址
- `Role`：节点角色（fe/follower/cn/be/hot/cold）
- `FeConf` / `BeConf`：FE/BE 的配置项
- `HttpPort` / `QueryPort`：服务端口
- `MasterFeIp`：主 FE 节点 IP

**Section sources**
- [install_doris.go](file://dbm-services/bigdata/db-tools/dbactuator/internal/subcmd/doriscmd/install_doris.go#L1-L110)
- [install_doris.go](file://dbm-services/bigdata/db-tools/dbactuator/pkg/components/doris/install_doris.go#L1-L358)

## 前端视图与 API 触发机制
前端通过 `views.py` 中的视图类定义 API 接口，用户在界面上发起部署请求后，后端将参数传递给 `dbactuator` 执行具体的部署任务。

**触发流程**：
1. 用户在前端界面填写部署参数（如 IP、角色、版本、配置等）。
2. 前端通过 HTTP 请求调用后端 API（如 `/api/dbm/doris/deploy/`）。
3. 后端视图函数接收参数，进行验证和处理。
4. 后端调用 `dbactuator` 的命令行工具，将参数序列化后传递给 `install_doris.go` 等组件。
5. `dbactuator` 按照预定义的步骤顺序执行部署任务。
6. 执行结果返回给前端，更新部署状态。

虽然具体的 Doris 部署视图未在提供的 `views.py` 文件中展示，但系统架构遵循上述通用模式，通过 RESTful API 实现前后端交互。

**Section sources**
- [views.py](file://dbm-ui/backend/db_proxy/frontend_views/views.py#L1-L80)

## 部署参数示例
以下是一个典型的 Doris 部署参数 JSON 示例：

```json
{
  "host": "192.168.1.100",
  "role": "fe",
  "fe_conf": {
    "http_port": "8030",
    "query_port": "9030",
    "edit_log_port": "9010",
    "rpc_port": "9020"
  },
  "be_conf": {
    "webserver_port": "8040",
    "heartbeat_service_port": "9050",
    "brpc_port": "8060"
  },
  "http_port": 8030,
  "query_port": 9030,
  "root_password": "root_password",
  "admin_password": "admin_password",
  "version": "2.0.4",
  "cluster_name": "doris-cluster",
  "master_fe_ip": "192.168.1.101"
}
```

## 日志分析方法
部署过程中的日志是排查问题的关键依据。主要日志来源包括：

1. **dbactuator 日志**：位于 `/data/dbactuator/logs/` 目录下，记录每个步骤的执行情况。
   - 关键日志：`InstallDorisAct Init`、`decompress_pkg 执行成功`、`install_supervisor successfully`
2. **Doris 服务日志**：
   - FE 日志：`/data/dorisenv/fe/log/fe.log`
   - BE 日志：`/data/dorisenv/be/log/be.log`
3. **Supervisor 日志**：`/var/log/supervisor/supervisord.log`，检查进程守护状态。
4. **系统日志**：`/var/log/messages` 或 `journalctl`，查看系统级错误。

**分析技巧**：
- 使用 `grep -i error` 或 `grep -i fail` 快速定位错误。
- 按时间顺序关联 `dbactuator` 日志和 Doris 服务日志。
- 检查端口监听状态：`netstat -tlnp | grep <port>`。

## 部署失败常见原因及排查步骤

### 常见原因
1. **权限问题**：`mysql` 用户不存在或目录权限不足。
2. **端口冲突**：指定的 HTTP、Query 或 RPC 端口已被占用。
3. **资源不足**：内存或磁盘空间不足，导致 JVM 启动失败。
4. **网络问题**：节点间网络不通，无法加入集群。
5. **软件包问题**：指定版本的 Doris 包不存在或损坏。
6. **系统配置未生效**：`vm.max_map_count` 或文件句柄数限制未正确设置。

### 排查步骤
1. **检查 dbactuator 日志**：确定失败发生在哪个步骤。
2. **验证参数**：确认传入的 IP、端口、版本等参数正确无误。
3. **检查目标主机**：
   - 确认 `mysql` 用户存在：`id mysql`
   - 确认安装目录可写：`ls -ld /data/dorisenv`
   - 检查端口占用：`lsof -i :8030`
4. **检查系统配置**：
   - 查看文件句柄限制：`ulimit -n`
   - 查看 vma 数量：`cat /proc/sys/vm/max_map_count`
5. **检查 Doris 服务日志**：查看 `fe.log` 或 `be.log` 中的详细错误信息。
6. **手动执行关键命令**：尝试手动执行 `tar` 解压、`supervisord` 启动等命令，观察输出。

通过以上系统化的排查流程，可以快速定位并解决 Doris 集群部署过程中的绝大多数问题。