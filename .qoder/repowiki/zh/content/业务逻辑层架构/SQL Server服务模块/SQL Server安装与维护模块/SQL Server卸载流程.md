# SQL Server卸载流程

<cite>
**本文档引用的文件**   
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go)
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/internal/subcmd/sqlservercmd/uninstall_sqlserver.go)
- [clear_config.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/clear/clear_config.go)
- [clear_config.go](file://dbm-services/sqlserver/db-tools/dbactuator/internal/subcmd/sqlservercmd/clear_config.go)
- [sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/util/sqlserver/sqlserver.go)
- [const.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/core/cst/const.go)
- [init_sqlserver.ps1](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/core/staticembed/init_sqlserver.ps1)
- [sysinit.ps1](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/core/staticembed/sysinit.ps1)
</cite>

## 目录
1. [引言](#引言)
2. [核心组件分析](#核心组件分析)
3. [卸载流程架构](#卸载流程架构)
4. [详细组件分析](#详细组件分析)
5. [依赖关系分析](#依赖关系分析)
6. [性能与安全考虑](#性能与安全考虑)
7. [故障排除指南](#故障排除指南)
8. [结论](#结论)

## 引言
本文档深入分析SQL Server实例的卸载逻辑，重点解析`uninstall_sqlserver.go`中的实现细节。文档涵盖服务停止、配置清理、文件删除和注册表项移除等关键步骤，讨论卸载过程中可能遇到的问题及其处理策略，并强调卸载操作的安全性和完整性保障措施。

## 核心组件分析

**本文档引用的文件**   
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go)
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/internal/subcmd/sqlservercmd/uninstall_sqlserver.go)

## 卸载流程架构

```mermaid
graph TD
A[开始卸载] --> B[预检查]
B --> C{实例是否正常运行?}
C --> |是| D[停止MSSQL服务]
C --> |否| E[跳过关闭]
D --> F[禁用服务启动]
F --> G[清理周边配置]
G --> H[移除注册表项]
H --> I[删除文件和目录]
I --> J[完成卸载]
```

**图表来源**
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go#L62-L133)
- [init_sqlserver.ps1](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/core/staticembed/init_sqlserver.ps1#L127-L187)

## 详细组件分析

### 卸载组件分析

#### 对象导向组件：
```mermaid
classDiagram
class UnInstallSQLServerComp {
+GeneralParam *components.GeneralParam
+Params *UnInstallSQLServerParam
+runTimeCtx
+Init() error
+PreCheck() error
+ShutDownMSSQL() error
}
class UnInstallSQLServerParam {
+Host string
+Force bool
+Ports []int
}
class runTimeCtx {
+insObj map[Port]*obj
}
class obj {
+InstanceName string
+IsShutdown bool
}
UnInstallSQLServerComp --> UnInstallSQLServerParam : "包含"
UnInstallSQLServerComp --> runTimeCtx : "包含"
runTimeCtx --> obj : "映射"
```

**图表来源**
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go#L23-L47)

#### API/服务组件：
```mermaid
sequenceDiagram
participant 命令行 as 命令行
participant UninstallSqlServerAct as UninstallSqlServerAct
participant UnInstallSQLServerComp as UnInstallSQLServerComp
命令行->>UninstallSqlServerAct : 执行卸载命令
UninstallSqlServerAct->>UninstallSqlServerAct : 反序列化参数
UninstallSqlServerAct->>UninstallSqlServerAct : 初始化
UninstallSqlServerAct->>UnInstallSQLServerComp : 执行预检查
UnInstallSQLServerComp-->>UninstallSqlServerAct : 检查结果
UninstallSqlServerAct->>UnInstallSQLServerComp : 停止MSSQL服务
UnInstallSQLServerComp-->>UninstallSqlServerAct : 停止结果
UninstallSqlServerAct-->>命令行 : 卸载成功
```

**图表来源**
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/internal/subcmd/sqlservercmd/uninstall_sqlserver.go#L37-L87)
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go#L62-L133)

#### 复杂逻辑组件：
```mermaid
flowchart TD
Start([开始]) --> PreCheck["预检查"]
PreCheck --> CheckConnection{"能否连接实例?"}
CheckConnection --> |否| MarkShutdown["标记为已关闭"]
CheckConnection --> |是| CheckConnections{"存在业务连接?"}
CheckConnections --> |是且非强制| Fail["预检查失败"]
CheckConnections --> |否或强制| GetInstanceName["获取实例名称"]
GetInstanceName --> StopService["停止MSSQL服务"]
StopService --> DisableService["禁用服务启动"]
DisableService --> Success["卸载成功"]
MarkShutdown --> Success
Fail --> End([结束])
Success --> End
```

**图表来源**
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go#L62-L133)

**本文档引用的文件**
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go#L1-L133)
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/internal/subcmd/sqlservercmd/uninstall_sqlserver.go#L1-L88)

### 配置清理组件分析

#### 对象导向组件：
```mermaid
classDiagram
class ClearConfigComp {
+GeneralParam *components.GeneralParam
+Params *ClearConfigParam
+ClearRunTimeCtx
+Init() error
+ClearConfig() error
+ClearJob() error
+ClearLinkServer() error
}
class ClearConfigParam {
+Host string
+Port int
+IsClearJob bool
+IsClearLinkServer bool
}
class ClearRunTimeCtx {
+LocalDB *sqlserver.DbWorker
}
ClearConfigComp --> ClearConfigParam : "包含"
ClearConfigComp --> ClearRunTimeCtx : "包含"
ClearRunTimeCtx --> DbWorker : "引用"
```

**图表来源**
- [clear_config.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/clear/clear_config.go#L22-L35)

#### API/服务组件：
```mermaid
sequenceDiagram
participant 命令行 as 命令行
participant ClearConfigAct as ClearConfigAct
participant ClearConfigComp as ClearConfigComp
命令行->>ClearConfigAct : 执行清理配置命令
ClearConfigAct->>ClearConfigAct : 反序列化参数
ClearConfigAct->>ClearConfigAct : 初始化
ClearConfigAct->>ClearConfigComp : 执行清理配置
ClearConfigComp->>ClearConfigComp : 清理Job任务
ClearConfigComp->>ClearConfigComp : 清理LinkServer
ClearConfigComp-->>ClearConfigAct : 清理结果
ClearConfigAct-->>命令行 : 清理成功
```

**图表来源**
- [clear_config.go](file://dbm-services/sqlserver/db-tools/dbactuator/internal/subcmd/sqlservercmd/clear_config.go#L37-L84)
- [clear_config.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/clear/clear_config.go#L61-L77)

**本文档引用的文件**
- [clear_config.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/clear/clear_config.go#L1-L95)
- [clear_config.go](file://dbm-services/sqlserver/db-tools/dbactuator/internal/subcmd/sqlservercmd/clear_config.go#L1-L84)

## 依赖关系分析

```mermaid
graph TD
A[卸载SQL Server] --> B[预检查]
A --> C[停止服务]
A --> D[清理配置]
B --> E[数据库连接]
C --> F[PowerShell命令]
D --> G[SQL命令执行]
E --> H[DbWorker]
F --> I[StandardPowerShellCommands]
G --> J[ExecMore]
H --> K[sqlserver包]
I --> L[osutil包]
J --> K
K --> M[sqlx]
L --> N[操作系统]
```

**图表来源**
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go#L13-L21)
- [sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/util/sqlserver/sqlserver.go#L23-L27)

**本文档引用的文件**
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go#L13-L21)
- [sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/util/sqlserver/sqlserver.go#L23-L27)

## 性能与安全考虑

**本文档引用的文件**
- [const.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/core/cst/const.go#L75-L95)
- [sysinit.ps1](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/core/staticembed/sysinit.ps1#L1-L32)

## 故障排除指南

**本文档引用的文件**
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go#L62-L111)
- [uninstall_sqlserver.go](file://dbm-services/sqlserver/db-tools/dbactuator/pkg/components/sqlserver/uninstall_sqlserver.go#L115-L133)

## 结论
SQL Server实例的卸载流程是一个复杂而严谨的过程，涉及多个关键步骤和组件的协同工作。通过预检查、服务停止、配置清理等步骤，确保了卸载操作的安全性和完整性。系统提供了强制卸载选项以应对特殊情况，同时通过详细的日志记录和错误处理机制保障了操作的可追溯性和可靠性。