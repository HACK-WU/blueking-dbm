# RBAC模型

<cite>
**本文档引用的文件**   
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py)
- [actions.py](file://dbm-ui/backend/iam_app/dataclass/actions.py)
- [resources.py](file://dbm-ui/backend/iam_app/dataclass/resources.py)
- [client.py](file://dbm-ui/backend/iam_app/handlers/client.py)
- [exceptions.py](file://dbm-ui/backend/iam_app/exceptions.py)
- [notify_group_provider.py](file://dbm-ui/backend/iam_app/views/notify_group_provider.py)
- [ticket_group_provider.py](file://dbm-ui/backend/iam_app/views/ticket_group_provider.py)
- [grafana/permissions.py](file://dbm-ui/backend/bk_dataview/grafana/permissions.py)
</cite>

## 目录
1. [引言](#引言)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构概述](#架构概述)
5. [详细组件分析](#详细组件分析)
6. [依赖分析](#依赖分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)

## 引言
本文档详细阐述了蓝鲸智云-DB管理系统（BlueKing-BK-DBM）中的基于IAM（身份与访问管理）的RBAC（基于角色的访问控制）模型。该模型通过定义角色、权限和用户组，实现了对数据库管理系统的精细化权限控制。文档深入解析了角色与权限的绑定关系、权限继承规则以及用户组的层级结构，并详细说明了权限策略的存储结构、查询机制和缓存策略。此外，文档还涵盖了权限校验的性能优化方法、创建自定义角色和权限集的配置示例、细粒度权限控制在不同数据库类型中的实现差异、权限冲突解决策略以及权限变更审计日志的实现方式。

## 项目结构
蓝鲸DBM系统的RBAC模型主要实现在`dbm-ui/backend/iam_app`目录下，其核心模块包括权限处理、动作定义、资源定义和客户端接口。权限处理模块负责权限的校验和申请，动作定义模块定义了系统中的各种操作权限，资源定义模块定义了系统中的各种资源类型，客户端接口模块提供了与IAM系统交互的API。

```mermaid
graph TD
subgraph "权限管理模块"
permission[permission.py]
actions[actions.py]
resources[resources.py]
client[client.py]
exceptions[exceptions.py]
end
subgraph "视图提供者"
notify_group_provider[notify_group_provider.py]
ticket_group_provider[ticket_group_provider.py]
end
subgraph "其他权限相关"
grafana_permissions[grafana/permissions.py]
end
permission --> actions
permission --> resources
permission --> client
permission --> exceptions
notify_group_provider --> permission
ticket_group_provider --> permission
```

**图源**
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py)
- [actions.py](file://dbm-ui/backend/iam_app/dataclass/actions.py)
- [resources.py](file://dbm-ui/backend/iam_app/dataclass/resources.py)
- [client.py](file://dbm-ui/backend/iam_app/handlers/client.py)
- [exceptions.py](file://dbm-ui/backend/iam_app/exceptions.py)
- [notify_group_provider.py](file://dbm-ui/backend/iam_app/views/notify_group_provider.py)
- [ticket_group_provider.py](file://dbm-ui/backend/iam_app/views/ticket_group_provider.py)
- [grafana/permissions.py](file://dbm-ui/backend/bk_dataview/grafana/permissions.py)

**节源**
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py)
- [actions.py](file://dbm-ui/backend/iam_app/dataclass/actions.py)
- [resources.py](file://dbm-ui/backend/iam_app/dataclass/resources.py)
- [client.py](file://dbm-ui/backend/iam_app/handlers/client.py)
- [exceptions.py](file://dbm-ui/backend/iam_app/exceptions.py)
- [notify_group_provider.py](file://dbm-ui/backend/iam_app/views/notify_group_provider.py)
- [ticket_group_provider.py](file://dbm-ui/backend/iam_app/views/ticket_group_provider.py)
- [grafana/permissions.py](file://dbm-ui/backend/bk_dataview/grafana/permissions.py)

## 核心组件
RBAC模型的核心组件包括权限处理类`Permission`、动作枚举类`ActionEnum`、资源枚举类`ResourceEnum`和IAM客户端类`IAM`。`Permission`类封装了权限校验和无权限申请的通用逻辑，`ActionEnum`类定义了系统中的所有操作权限，`ResourceEnum`类定义了系统中的所有资源类型，`IAM`类提供了与IAM系统交互的API。

**节源**
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py#L47-L670)
- [actions.py](file://dbm-ui/backend/iam_app/dataclass/actions.py#L1-L2777)
- [resources.py](file://dbm-ui/backend/iam_app/dataclass/resources.py#L1-L758)
- [client.py](file://dbm-ui/backend/iam_app/handlers/client.py#L1-L67)

## 架构概述
RBAC模型的架构基于IAM系统，通过定义动作和资源来实现权限控制。系统中的每个操作都被定义为一个动作，每个可操作的对象都被定义为一个资源。用户通过角色获得对特定资源的特定动作的权限。权限校验时，系统会检查用户是否具有执行该动作的权限。

```mermaid
graph TD
User[用户] --> Role[角色]
Role --> Action[动作]
Action --> Resource[资源]
Permission[权限校验] --> IAM[IAM系统]
IAM --> Action
IAM --> Resource
```

**图源**
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py)
- [actions.py](file://dbm-ui/backend/iam_app/dataclass/actions.py)
- [resources.py](file://dbm-ui/backend/iam_app/dataclass/resources.py)

## 详细组件分析

### 权限处理类分析
`Permission`类是权限处理的核心，它提供了权限校验、权限申请、权限字段插入等功能。该类通过`is_allowed`方法进行权限校验，通过`get_apply_url`方法生成权限申请链接，通过`insert_permission_field`方法在数据返回时插入权限字段。

```mermaid
classDiagram
class Permission {
+username : str
+bk_token : str
+is_superuser : bool
+_iam : IAM
+__init__(username : str, request : Request)
+get_iam_client() IAM
+get_system_info() dict
+setup_meta() None
+make_resource_instance(resource_type : str, instance_id : str) Resource
+check_resource_is_local(resources : List[Resource]) bool
+batch_make_resource_instance(resources : List[Dict]) List[Resource]
+make_request(action : Union[ActionMeta, str], resources : List[Resource]) Request
+make_multi_request(actions : List[Union[ActionMeta, str]], resources : List[Resource]) MultiActionRequest
+is_allowed(action : Union[ActionMeta, str], resources : List[Resource], is_raise_exception : bool) bool
+multi_actions_is_allowed(actions : List[Union[ActionMeta, str]], resources : List[Resource], is_raise_exception : bool) Dict[str, bool]
+batch_is_allowed(actions : List[Union[ActionMeta, str]], resources_list : List[List[Resource]], is_raise_exception : bool) Dict[str, Dict[str, bool]]
+policy_query(action : Union[ActionMeta, str], obj_list : List[Union[int, str]]) List
+make_application(action_ids : List[str], resources_list : List[List[Resource]], system_id : str) Application
+get_apply_url(action_ids : List[str], resources_list : List[List[Resource]], system_id : str) str
+_get_topo_resource(resource : Resource) List[Resource]
+get_apply_data(actions : List[Union[ActionMeta, str]], resources_list : List[List[Resource]]) Tuple[Any, str]
+_grant_actions(resource : Resource, application : Any, grant_func : Callable, raise_exception : bool) Any
+grant_creator_actions(resource : Resource, creator : str) Any
+grant_creator_actions_attr(resource : Resource, creator : str) Any
+insert_permission_field(response : Any, actions : List[ActionMeta], resource_meta : ResourceMeta, id_field : Callable, data_field : Callable, always_allowed : Callable, many : bool) Any
+insert_external_permission_field(response : Any, actions : List[ActionMeta], resource_meta : Union[ResourceMeta, List[ResourceMeta], None], resource_id : Any) Any
+decorator_permission_field(actions : List[ActionMeta], action_filed : Callable, resource_meta : ResourceMeta, id_field : Callable, data_field : Callable, always_allowed : Callable, many : bool) Callable
+decorator_external_permission_field(actions : List[ActionMeta], action_filed : Callable, param_field : Callable, resource_meta : Union[ResourceMeta, List[ResourceMeta], None]) Callable
}
class IAM {
+__init__(app_code : str, app_secret : str, bk_iam_host : str, bk_paas_host : str, bk_apigateway_url : str, api_version : str)
+is_allowed(request : Request) bool
+resource_multi_actions_allowed(multi_request : MultiActionRequest) Dict[str, bool]
+batch_resource_multi_actions_allowed(multi_request : MultiActionRequest, resources_list : List[List[Resource]]) Dict[str, Dict[str, bool]]
+_do_policy_query(request : Request) List
+_eval_expr(expression : Any, iam_obj : ObjectSet) bool
+get_apply_url(application : Application, bk_token : str, username : str) Tuple[bool, str, str]
+grant_resource_creator_actions(application : Any, bk_token : str, username : str) Any
+grant_resource_creator_action_attributes(application : Any, bk_token : str, username : str) Any
}
Permission --> IAM : "使用"
```

**图源**
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py#L47-L670)
- [client.py](file://dbm-ui/backend/iam_app/handlers/client.py#L61-L67)

### 动作与资源定义分析
`ActionEnum`和`ResourceEnum`类分别定义了系统中的所有动作和资源。每个动作和资源都有一个唯一的ID和名称，并且可以关联其他资源类型和动作。通过这些定义，系统可以精确地控制用户对特定资源的特定操作权限。

```mermaid
classDiagram
class ActionMeta {
+id : str
+name : str
+name_en : str
+type : str
+related_resource_types : List[ResourceMeta]
+related_actions : List
+version : str
+hidden : bool
+group : str
+subgroup : str
+common_labels : List[str]
+is_ticket_action : bool
+__post_init__() None
+__ticket_tool_action_init__() None
+to_json() Dict
+__hash__() int
+__eq__(other : Any) bool
}
class ResourceMeta {
+system_id : str
+id : str
+name : str
+selection_mode : str
+attribute : str
+attribute_display : str
+lookup_field : str
+display_fields : list
+parent : ResourceMeta
+for_select : bool
+select_id : str
+__post_init__() None
+Field(value : Any) field
+_create_simple_instance(instance_id : str, attr : Any) Resource
+create_instance(instance_id : str, attr : Any) Resource
+batch_create_instances(instance_ids : list, attr : Any) List[Resource]
+create_model_instance(model : models.Model, instance_id : str, instance : models.Model, attr : Any) Tuple[Resource, models.Model]
+batch_create_model_instances(model : models.Model, instance_ids : list, instance_queryset : models.QuerySet, attr : dict) List[Tuple[Resource, models.Model]]
+batch_create_with_iam_path(model : models.Model, instance_ids : list, instance_queryset : models.QuerySet, attr : dict) List[Tuple[Resource, models.Model]]
+to_json() Dict
}
class ActionEnum {
+DB_MANAGE : ActionMeta
+GLOBAL_MANAGE : ActionMeta
+TICKET_VIEW : ActionMeta
+GLOBAL_TICKET_CONFIG_SET : ActionMeta
+BIZ_TICKET_CONFIG_SET : ActionMeta
+BIZ_ASSISTANCE_VARS_CONFIG : ActionMeta
+BIZ_NOTIFY_CONFIG : ActionMeta
+RESOURCE_MANAGE : ActionMeta
+FLOW_DETAIL : ActionMeta
+PLATFORM_MANAGE : ActionMeta
+PLATFORM_TICKET_VIEW : ActionMeta
+PLATFORM_TASKFLOW_VIEW : ActionMeta
+PLATFORM_HEALTHY_REPORT_VIEW : ActionMeta
+PLATFORM_ALERT_EVENT_VIEW : ActionMeta
+PLATFROM_RISK_MEMO_VIEW : ActionMeta
+MYSQL_DBCONSOLE : ActionMeta
+TENDBCLUSTER_DBCONSOLE : ActionMeta
+SQLSERVER_DBCONSOLE : ActionMeta
+DBCONFIG_VIEW : ActionMeta
+DBCONFIG_EDIT : ActionMeta
+GLOBAL_DBCONFIG_EDIT : ActionMeta
+GLOBAL_DBCONFIG_CREATE : ActionMeta
+GLOBAL_DBCONFIG_DESTROY : ActionMeta
+MYSQL_APPLY : ActionMeta
+MYSQL_VIEW : ActionMeta
+MYSQL_EDIT : ActionMeta
+MYSQL_SUBSCRIBE_MONITOR : ActionMeta
+MYSQL_IMPORT_SQLFILE : ActionMeta
+MYSQL_INSTANCE_CLONE_RULES : ActionMeta
+MYSQL_DUMP_DATA : ActionMeta
+MYSQL_WEBCONSOLE : ActionMeta
+MYSQL_ADMIN_PWD_MODIFY : ActionMeta
+MYSQL_ENABLE_DISABLE : ActionMeta
+MYSQL_CLIENT_CLONE_RULES : ActionMeta
+MYSQL_DESTROY : ActionMeta
+MYSQL_CREATE_ACCOUNT : ActionMeta
+MYSQL_DELETE_ACCOUNT : ActionMeta
+MYSQL_ADD_ACCOUNT_RULE : ActionMeta
+MYSQL_ACCOUNT_RULES_VIEW : ActionMeta
+MYSQL_AUTHORIZE_RULES : ActionMeta
+MYSQL_EXCEL_AUTHORIZE_RULES : ActionMeta
+MYSQL_PARTITION_CREATE : ActionMeta
+MYSQL_PARTITION_UPDATE : ActionMeta
+MYSQL_PARTITION_DELETE : ActionMeta
+MYSQL_PARTITION_ENABLE_DISABLE : ActionMeta
+MYSQL_PARTITION : ActionMeta
+MYSQL_FLASHBACK : ActionMeta
+MYSQL_HA_TRUNCATE_DATA : ActionMeta
+MYSQL_HA_DB_TABLE_BACKUP : ActionMeta
+MYSQL_HA_RENAME_DATABASE : ActionMeta
+MYSQL_SINGLE_TRUNCATE_DATA : ActionMeta
+MYSQL_SINGLE_RENAME_DATABASE : ActionMeta
+MYSQL_RENAME_DATABASE : ActionMeta
+MYSQL_OPEN_AREA : ActionMeta
+MYSQL_OPENAREA_CONFIG_CREATE : ActionMeta
+MYSQL_OPENAREA_CONFIG_UPDATE : ActionMeta
+MYSQL_OPENAREA_CONFIG_DESTROY : ActionMeta
+DUMPER_CONFIG_VIEW : ActionMeta
+DUMPER_CONFIG_UPDATE : ActionMeta
+DUMPER_CONFIG_DESTROY : ActionMeta
+TBINLOGDUMPER_INSTALL : ActionMeta
}
class ResourceEnum {
+BUSINESS : BusinessResourceMeta
+TASKFLOW : TaskFlowResourceMeta
+TICKET : TicketResourceMeta
+MYSQL : MySQLResourceMeta
+TENDBCLUSTER : TendbClusterResourceMeta
+REDIS : RedisResourceMeta
+ES : EsResourceMeta
+DORIS : DorisResourceMeta
+KAFKA : KafkaResourceMeta
+HDFS : HdfsResourceMeta
+PULSAR : PulsarResourceMeta
+RIAK : RiakResourceMeta
+MONGODB : MongoDBResourceMeta
+SQLSERVER : SQLServerResourceMeta
+ORACLE : OracleResourceMeta
+DBTYPE : DBTypeResourceMeta
+TICKET_GROUP : TicketGroupResourceMeta
+MONITOR_POLICY : MonitorPolicyResourceMeta
+GLOBAL_MONITOR_POLICY : GlobalMonitorPolicyResourceMeta
+NOTIFY_GROUP : NotifyGroupResourceMeta
+GLOBAL_NOTIFY_GROUP : GlobalNotifyGroupResourceMeta
+OPENAREA_CONFIG : OpenareaConfigResourceMeta
+DUMPER_SUBSCRIBE_CONFIG : DumperSubscribeConfigResourceMeta
+MYSQL_ACCOUNT : MySQLAccountResourceMeta
+SQLSERVER_ACCOUNT : SQLServerAccountResourceMeta
+MONGODB_ACCOUNT : MongoDBAccountResourceMeta
+TENDBCLUSTER_ACCOUNT : TendbClusterAccountResourceMeta
+VM : VmResourceMeta
+get_resource_by_id(resource_id : Union[ResourceMeta, str]) ResourceMeta
+cluster_type_to_resource_meta(cluster_type : str) ResourceMeta
+instance_type_to_resource_meta(instance_role : str) ResourceMeta
}
ActionEnum --> ActionMeta
ResourceEnum --> ResourceMeta
```

**图源**
- [actions.py](file://dbm-ui/backend/iam_app/dataclass/actions.py#L26-L800)
- [resources.py](file://dbm-ui/backend/iam_app/dataclass/resources.py#L27-L758)

### 用户组管理分析
用户组管理通过`Client`类提供的API实现，包括创建用户组、用户组授权、添加用户组成员、查询用户组和更新用户组名字和描述。这些API通过HTTP请求与IAM系统交互，实现了对用户组的全生命周期管理。

```mermaid
sequenceDiagram
participant User as "用户"
participant Permission as "Permission"
participant Client as "Client"
participant IAM as "IAM系统"
User->>Permission : 请求创建用户组
Permission->>Client : 调用create_user_groups
Client->>IAM : 发送POST请求
IAM-->>Client : 返回创建结果
Client-->>Permission : 返回结果
Permission-->>User : 返回用户组创建结果
User->>Permission : 请求用户组授权
Permission->>Client : 调用grant_user_group_actions
Client->>IAM : 发送POST请求
IAM-->>Client : 返回授权结果
Client-->>Permission : 返回结果
Permission-->>User : 返回用户组授权结果
User->>Permission : 请求添加用户组成员
Permission->>Client : 调用add_user_group_members
Client->>IAM : 发送POST请求
IAM-->>Client : 返回添加结果
Client-->>Permission : 返回结果
Permission-->>User : 返回添加成员结果
```

**图源**
- [client.py](file://dbm-ui/backend/iam_app/handlers/client.py#L17-L60)
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py)

**节源**
- [client.py](file://dbm-ui/backend/iam_app/handlers/client.py#L17-L60)
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py)

## 依赖分析
RBAC模型的实现依赖于IAM系统提供的API，通过`IAM`类和`Client`类与IAM系统进行交互。此外，权限处理还依赖于Django框架的用户模型和数据库模型，通过`ResourceMeta`类中的`create_model_instance`方法与数据库进行交互。

```mermaid
graph TD
Permission --> IAM
IAM --> Client
Client --> HTTP
Permission --> Django
Django --> Database
ResourceMeta --> Database
```

**图源**
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py)
- [client.py](file://dbm-ui/backend/iam_app/handlers/client.py)
- [resources.py](file://dbm-ui/backend/iam_app/dataclass/resources.py)

**节源**
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py)
- [client.py](file://dbm-ui/backend/iam_app/handlers/client.py)
- [resources.py](file://dbm-ui/backend/iam_app/dataclass/resources.py)

## 性能考虑
为了提高权限校验的性能，系统采用了多种优化策略。首先，通过`batch_is_allowed`方法批量校验多个动作和资源的权限，减少了与IAM系统的交互次数。其次，通过`policy_query`方法批量判断业务资源关联动作是否有权限，利用IAM系统的策略查询功能提高查询效率。此外，系统还通过缓存机制减少重复的权限校验请求。

**节源**
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py#L242-L290)
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py#L292-L325)

## 故障排除指南
在使用RBAC模型时，可能会遇到权限校验失败、权限申请链接生成失败等问题。对于权限校验失败，可以通过检查用户角色和权限配置来解决；对于权限申请链接生成失败，可以通过检查IAM系统的配置和网络连接来解决。此外，系统还提供了详细的日志记录，可以帮助定位和解决问题。

**节源**
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py#L197-L207)
- [permission.py](file://dbm-ui/backend/iam_app/handlers/permission.py#L380-L395)
- [exceptions.py](file://dbm-ui/backend/iam_app/exceptions.py)

## 结论
蓝鲸DBM系统的RBAC模型通过基于IAM的权限管理机制，实现了对数据库管理系统的精细化权限控制。该模型具有良好的可扩展性和灵活性，能够满足不同场景下的权限管理需求。通过本文档的详细说明，用户可以更好地理解和使用RBAC模型，提高系统的安全性和管理效率。