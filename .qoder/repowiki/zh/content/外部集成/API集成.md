# API集成

<cite>
**本文档引用的文件**
- [cc/client.py](file://dbm-ui/backend/components/cc/client.py)
- [job/client.py](file://dbm-ui/backend/components/job/client.py)
- [bkmonitorv3/client.py](file://dbm-ui/backend/components/bkmonitorv3/client.py)
- [base.py](file://dbm-ui/backend/components/base.py)
- [domains.py](file://dbm-ui/backend/components/domains.py)
- [batch_request.py](file://dbm-ui/backend/utils/batch_request.py)
- [bkmonitorbeat.go](file://dbm-services/redis/db-tools/dbmon/pkg/sendwarning/bkmonitorbeat.go)
- [bkmonitorbeat.go](file://dbm-services/mongodb/db-tools/dbmon/pkg/sendwarning/bkmonitorbeat.go)
- [dataclass.py](file://dbm-ui/backend/db_monitor/dataclass.py)
- [db_proxy/views/jobapi/views.py](file://dbm-ui/backend/db_proxy/views/jobapi/views.py)
- [db_monitor/utils.py](file://dbm-ui/backend/db_monitor/utils.py)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构概述](#架构概述)
5. [详细组件分析](#详细组件分析)
6. [依赖分析](#依赖分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)

## 简介
本文档详细说明了平台如何通过HTTP/HTTPS协议与蓝鲸配置平台（CC）、作业平台（JOB）和监控平台（BKMONITOR）进行集成。文档分析了components目录下各组件的客户端代码，解释了RESTful API的调用方式、认证机制（如API Key）、请求/响应格式以及错误处理策略。提供了实际代码示例，展示如何在业务逻辑中调用这些API。文档化了常见的集成场景，如从CC同步主机信息、通过JOB执行远程脚本、向BKMONITOR上报监控数据等。包含性能优化建议和故障排查指南。

## 项目结构
项目结构清晰地组织了各个服务和组件。`dbm-ui/backend/components`目录包含了与各个蓝鲸平台（如CC、JOB、BKMONITOR）集成的客户端组件。`dbm-services`目录包含了具体的服务实现，如`db-resource`、`dbha`等。

```mermaid
graph TD
subgraph "前端"
UI[用户界面]
end
subgraph "后端"
Backend[后端服务]
Components[组件]
Services[服务]
end
subgraph "蓝鲸平台"
CC[配置平台]
JOB[作业平台]
BKMONITOR[监控平台]
end
UI --> Backend
Backend --> Components
Components --> CC
Components --> JOB
Components --> BKMONITOR
Services --> Components
```

**图源**
- [cc/client.py](file://dbm-ui/backend/components/cc/client.py)
- [job/client.py](file://dbm-ui/backend/components/job/client.py)
- [bkmonitorv3/client.py](file://dbm-ui/backend/components/bkmonitorv3/client.py)

**节源**
- [cc/client.py](file://dbm-ui/backend/components/cc/client.py)
- [job/client.py](file://dbm-ui/backend/components/job/client.py)
- [bkmonitorv3/client.py](file://dbm-ui/backend/components/bkmonitorv3/client.py)

## 核心组件
核心组件包括与蓝鲸配置平台（CC）、作业平台（JOB）和监控平台（BKMONITOR）集成的客户端。这些客户端封装了RESTful API的调用，提供了简洁的接口供业务逻辑使用。

**节源**
- [cc/client.py](file://dbm-ui/backend/components/cc/client.py)
- [job/client.py](file://dbm-ui/backend/components/job/client.py)
- [bkmonitorv3/client.py](file://dbm-ui/backend/components/bkmonitorv3/client.py)

## 架构概述
系统架构基于微服务设计，通过HTTP/HTTPS协议与蓝鲸平台进行通信。客户端组件负责封装API调用，提供统一的接口。业务逻辑通过调用这些客户端组件来实现与蓝鲸平台的集成。

```mermaid
graph TB
subgraph "业务逻辑"
BusinessLogic[业务逻辑]
end
subgraph "客户端组件"
CCClient[CC客户端]
JOBClient[JOB客户端]
BKMONITORClient[BKMONITOR客户端]
end
subgraph "蓝鲸平台"
CC[配置平台]
JOB[作业平台]
BKMONITOR[监控平台]
end
BusinessLogic --> CCClient
BusinessLogic --> JOBClient
BusinessLogic --> BKMONITORClient
CCClient --> CC
JOBClient --> JOB
BKMONITORClient --> BKMONITOR
```

**图源**
- [cc/client.py](file://dbm-ui/backend/components/cc/client.py)
- [job/client.py](file://dbm-ui/backend/components/job/client.py)
- [bkmonitorv3/client.py](file://dbm-ui/backend/components/bkmonitorv3/client.py)

## 详细组件分析

### CC客户端分析
CC客户端提供了与蓝鲸配置平台的集成，支持查询业务、主机、模块等信息。

#### 类图
```mermaid
classDiagram
class _CCApi {
+MODULE : str
+BASE : str
+list_hosts_without_biz : DataAPI
+search_business : DataAPI
+search_module : DataAPI
+create_set : DataAPI
+search_set : DataAPI
+create_module : DataAPI
+delete_module : DataAPI
+transfer_host_across_biz : DataAPI
+transfer_host_module : DataAPI
+update_business : DataAPI
+update_host : DataAPI
+batch_update_host : DataAPI
+create_biz_custom_field : DataAPI
+search_object_attribute : DataAPI
+create_object_attribute : DataAPI
+transfer_host_to_idlemodule : DataAPI
+transfer_host_to_recyclemodule : DataAPI
+search_biz_inst_topo : DataAPI
+list_biz_hosts : DataAPI
+list_biz_hosts_topo : DataAPI
+get_biz_internal_module : DataAPI
+find_host_topo_relation : DataAPI
+search_cloud_area : DataAPI
+list_host_total_mainline_topo : DataAPI
+create_service_instance : DataAPI
+list_service_instance : DataAPI
+list_service_instance_by_host : DataAPI
+list_service_instance_detail : DataAPI
+add_label_for_service_instance : DataAPI
+remove_label_from_service_instance : DataAPI
+delete_service_instance : DataAPI
+create_process_instance : DataAPI
+delete_process_instance : DataAPI
+list_process_instance : DataAPI
+update_process_instance : DataAPI
+find_module_with_relation : DataAPI
+find_module_host_relation : DataAPI
+find_host_biz_relations : DataAPI
+find_module_batch : DataAPI
+check_host_event : DataAPI
+batch_find_host_biz_relations(params) : list
}
class CCApi {
+_CCApi
}
_CCApi <|-- CCApi
```

**图源**
- [cc/client.py](file://dbm-ui/backend/components/cc/client.py)

### JOB客户端分析
JOB客户端提供了与蓝鲸作业平台的集成，支持快速执行脚本、分发文件、查询作业状态等。

#### 类图
```mermaid
classDiagram
class _JobApi {
+MODULE : str
+BASE : str
+fast_execute_script : DataAPI
+fast_transfer_file : DataAPI
+push_config_file : DataAPI
+get_job_instance_status : DataAPI
+get_job_instance_ip_log : DataAPI
+batch_get_job_instance_ip_log : DataAPI
+create_credential : DataAPI
+create_file_source : DataAPI
+create_account : DataAPI
+get_account_list : DataAPI
+operate_step_instance : DataAPI
+__format_fast_execute_script(params) : dict
}
class JobApi {
+_JobApi
}
_JobApi <|-- JobApi
```

**图源**
- [job/client.py](file://dbm-ui/backend/components/job/client.py)

### BKMONITOR客户端分析
BKMONITOR客户端提供了与蓝鲸监控平台的集成，支持创建和查询告警策略、上报监控数据等。

#### 类图
```mermaid
classDiagram
class _BKMonitorV3Api {
+MODULE : str
+BASE : str
+query_custom_event_group : DataAPI
+custom_time_series : DataAPI
+get_custom_event_group : DataAPI
+custom_time_series_detail : DataAPI
+create_custom_time_series : DataAPI
+create_custom_event_group : DataAPI
+save_alarm_strategy_v3 : DataAPI
+switch_alarm_strategy : DataAPI
+update_partial_strategy_v3 : DataAPI
+delete_alarm_strategy_v3 : DataAPI
+search_alarm_strategy_v3 : DataAPI
+save_collect_config : DataAPI
+run_collect_config : DataAPI
+query_collect_config : DataAPI
+get_collect_config_list : DataAPI
+query_collect_config_detail : DataAPI
+search_user_groups : DataAPI
+search_user_group_detail : DataAPI
+delete_user_groups : DataAPI
+save_user_group : DataAPI
+save_duty_rule : DataAPI
+search_duty_rules : DataAPI
+delete_duty_rules : DataAPI
+save_rule_group : DataAPI
+search_rule_groups : DataAPI
+delete_rule_group : DataAPI
+search_event : DataAPI
+search_alert : DataAPI
+unify_query : DataAPI
+proxy_host_info : DataAPI
+search_action_config : DataAPI
+save_action_config : DataAPI
+edit_action_config : DataAPI
+add_shield : DataAPI
+disable_shield : DataAPI
+edit_shield : DataAPI
+list_shield : DataAPI
+get_shield : DataAPI
+bulk_save_subscribe : DataAPI
+bulk_delete_subscribe : DataAPI
+list_subscribe : DataAPI
+bulk_save_subscribe_in_batch(bk_biz_id, subscriptions) : void
+list_full_subscribe(bk_biz_id, username) : list
}
class _BKMonitorV3EventApi {
+MODULE : str
+BASE : str
+DATA_ID : int
+ACCESS_TOKEN : str
+send_monitor_event : DataAPI
+__init_api() : void
+__init_conf() : void
+send_event(events) : void
}
class BKMonitorV3Api {
+_BKMonitorV3Api
}
class BKMonitorV3EventApi {
+_BKMonitorV3EventApi
}
_BKMonitorV3Api <|-- BKMonitorV3Api
_BKMonitorV3EventApi <|-- BKMonitorV3EventApi
```

**图源**
- [bkmonitorv3/client.py](file://dbm-ui/backend/components/bkmonitorv3/client.py)

## 依赖分析
系统依赖于蓝鲸平台提供的API，通过HTTP/HTTPS协议进行通信。客户端组件依赖于`base.py`中的`DataAPI`类来封装API调用，`domains.py`中定义了各个平台的API网关域名。

```mermaid
graph TD
subgraph "客户端组件"
CCClient[CC客户端]
JOBClient[JOB客户端]
BKMONITORClient[BKMONITOR客户端]
end
subgraph "基础组件"
BaseAPI[BaseAPI]
DataAPI[DataAPI]
Domains[Domains]
end
CCClient --> BaseAPI
JOBClient --> BaseAPI
BKMONITORClient --> BaseAPI
BaseAPI --> DataAPI
BaseAPI --> Domains
```

**图源**
- [base.py](file://dbm-ui/backend/components/base.py)
- [domains.py](file://dbm-ui/backend/components/domains.py)

**节源**
- [base.py](file://dbm-ui/backend/components/base.py)
- [domains.py](file://dbm-ui/backend/components/domains.py)

## 性能考虑
为了提高性能，系统采用了多线程并发请求。`batch_request`函数用于批量请求API，`request_multi_thread`函数用于多线程并发请求。这些函数可以显著减少请求的总时间。

```mermaid
flowchart TD
Start([开始]) --> PrepareParams["准备请求参数"]
PrepareParams --> CheckCount{"是否有count参数?"}
CheckCount --> |是| CalculatePages["计算页数"]
CheckCount --> |否| UseSync["使用同步请求"]
CalculatePages --> SubmitTasks["提交任务到线程池"]
SubmitTasks --> WaitForResults["等待结果"]
WaitForResults --> CombineResults["合并结果"]
CombineResults --> End([结束])
UseSync --> End
```

**图源**
- [batch_request.py](file://dbm-ui/backend/utils/batch_request.py)

## 故障排除指南
### 常见问题
1. **认证失败**：检查`bk_app_code`和`bk_app_secret`是否正确。
2. **请求超时**：增加超时时间或检查网络连接。
3. **API返回错误**：查看返回的错误信息，根据错误码进行处理。

### 错误处理
客户端组件通过`DataAPI`类封装了错误处理逻辑。当API调用失败时，会抛出`ApiResultError`异常，业务逻辑需要捕获并处理这些异常。

```mermaid
sequenceDiagram
participant Client as "客户端"
participant DataAPI as "DataAPI"
participant API as "API"
Client->>DataAPI : 调用API
DataAPI->>API : 发送请求
API-->>DataAPI : 返回响应
DataAPI->>DataAPI : 检查响应结果
alt 响应成功
DataAPI-->>Client : 返回数据
else 响应失败
DataAPI->>DataAPI : 抛出ApiResultError
DataAPI-->>Client : 抛出异常
end
```

**图源**
- [base.py](file://dbm-ui/backend/components/base.py)

**节源**
- [base.py](file://dbm-ui/backend/components/base.py)
- [cc/client.py](file://dbm-ui/backend/components/cc/client.py)
- [job/client.py](file://dbm-ui/backend/components/job/client.py)
- [bkmonitorv3/client.py](file://dbm-ui/backend/components/bkmonitorv3/client.py)

## 结论
本文档详细介绍了平台如何通过HTTP/HTTPS协议与蓝鲸配置平台（CC）、作业平台（JOB）和监控平台（BKMONITOR）进行集成。通过分析客户端组件的代码，我们了解了RESTful API的调用方式、认证机制、请求/响应格式以及错误处理策略。文档还提供了性能优化建议和故障排查指南，帮助开发者更好地使用这些API。