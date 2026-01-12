# API参考

<cite>
**本文档中引用的文件**   
- [swagger.json](file://dbm-services/bigdata/db-tools/dbactuator/docs/swagger.json)
- [swagger.json](file://dbm-services/common/db-config/docs/swagger.json)
- [swagger.yaml](file://dbm-services/common/db-config/docs/swagger.yaml)
- [urls.py](file://dbm-ui/backend/urls.py)
- [bk.py](file://dbm-ui/backend/components/bk.py)
- [views.py](file://dbm-ui/backend/db_services/dbconfig/views.py)
- [definition.yaml](file://dbm-ui/backend/dbm_init/apigw/definition.yaml)
</cite>

## 目录
1. [简介](#简介)
2. [公开的RESTful API端点](#公开的restful-api端点)
3. [db-config API组](#db-config-api组)
4. [db-resource API组](#db-resource-api组)
5. [前端与后端服务之间的内部API调用](#前端与后端服务之间的内部api调用)
6. [认证方法](#认证方法)
7. [API版本控制策略](#api版本控制策略)
8. [错误码](#错误码)
9. [客户端代码示例](#客户端代码示例)

## 简介
本文档旨在为蓝鲸数据库管理平台（BlueKing DBM）提供全面的API参考。该平台通过多个服务组件（如`dbactuator`、`db-config`、`db-resource`）暴露RESTful API端点，这些端点通过Swagger文档进行描述。前端服务（`dbm-ui`）通过内部API调用与后端服务进行交互。API通过API网关进行认证和管理，确保安全性和可扩展性。

## 公开的RESTful API端点
通过分析`dbm-services`目录下的Swagger文档，我们识别出多个公开的RESTful API端点。这些端点主要分为两类：由`dbactuator`工具暴露的底层操作API，以及由`db-config`等服务暴露的配置管理API。

`dbactuator`的Swagger文档（位于`dbm-services/bigdata/db-tools/dbactuator/docs/swagger.json`）定义了以下主要端点：
- `/common/file-server`: 启动一个简单的HTTP文件服务器，用于在机器间共享文件。
- `/common/rm-file`: 限速删除大文件。
- `/download/http`: 通过HTTP协议下载文件，支持BasicAuth认证。
- `/download/scp`: 通过SCP协议下载文件。

`db-config`服务的Swagger文档（位于`dbm-services/common/db-config/docs/swagger.json`）定义了更丰富的数据库配置管理API，包括MySQL、Spider和TBinlogDumper等组件的部署、配置变更和备份恢复操作。

## db-config API组
`db-config` API组提供了对数据库配置的全面管理功能。这些API主要通过HTTP POST方法进行调用，请求体包含详细的配置参数。

### HTTP方法和URL路径
所有`db-config` API端点均使用`POST`方法，其URL路径遵循`/服务名/操作名`的模式。例如：
- `POST /mysql/deploy`: 部署MySQL实例。
- `POST /mysql/change-master`: 建立MySQL主从关系。
- `POST /mysql/restore-dr`: 执行数据库备份恢复。
- `POST /spider/deploy`: 部署Spider实例。

### 请求和响应的JSON Schema
请求体的JSON Schema在Swagger文档的`definitions`部分有详细定义。以`/mysql/deploy`端点为例，其请求体包含一个`extend`对象，该对象引用了`InstallMySQLParams`定义，该定义包含了部署MySQL实例所需的所有参数，如`charset`（字符集）、`host`（主机地址）、`ports`（端口列表）、`pkg`（安装包名）和`mycnf_configs`（my.cnf配置）等。

响应体通常为空，或者在查询操作中返回特定的数据结构。例如，`/mysql/find-local-backup`端点的响应体是一个`FindLocalBackupResp`对象，包含`backups`（备份文件列表）和`latest`（最近一次备份）等字段。

**Section sources**
- [swagger.json](file://dbm-services/common/db-config/docs/swagger.json#L23-L768)

## db-resource API组
`db-resource` API组负责管理数据库资源的分配和使用。该API组的端点定义在前端`dbm-ui`的路由配置中。

### HTTP方法和URL路径
`db-resource` API端点主要通过`dbm-ui`的`/api/dbresource/`路径暴露。根据`urls.py`文件的配置，其子路径包括：
- `GET /api/dbresource/`: 获取资源列表。
- `POST /api/dbresource/`: 创建或分配资源。
- `DELETE /api/dbresource/<id>/`: 删除指定ID的资源。

### 请求和响应的JSON Schema
具体的请求和响应Schema由`dbm-ui`后端的视图和序列化器定义。虽然没有直接的Swagger文档，但可以通过分析`db_services/dbresource/`目录下的代码来推断其结构。典型的请求体可能包含资源类型、数量、所属业务等信息，而响应体则返回分配的资源详情。

**Section sources**
- [urls.py](file://dbm-ui/backend/urls.py#L46)

## 前端与后端服务之间的内部API调用
前端`dbm-ui`通过Django REST framework构建后端API，这些API作为内部服务被前端页面调用。`dbm-ui`的后端服务又会作为客户端，调用`dbm-services`中的具体服务。

### 调用机制
`dbm-ui`通过`backend/components/`目录下的组件模块（如`dbconfig.py`、`dbresource.py`）封装对后端服务的调用。这些组件使用`HttpClient`类发送HTTP请求。

例如，在`bk.py`文件中定义了`BKAuth`类，它实现了`requests.AuthBase`，用于在每个请求中自动添加蓝鲸平台所需的认证信息（`bk_app_code`, `bk_app_secret`, `bk_username`）。`BKClient`类则封装了通用的请求方法，简化了API调用。

### 调用示例
当用户在`dbm-ui`界面上请求部署一个MySQL实例时，调用流程如下：
1.  前端页面调用`dbm-ui`后端的`/api/mysql/deploy/` API。
2.  `dbm-ui`后端的`mysql`服务视图接收到请求。
3.  `mysql`服务视图通过`DBConfigApi`组件，使用`BKClient`向`db-config`服务的`/mysql/deploy`端点发起一个POST请求。
4.  `db-config`服务执行部署逻辑，并返回结果。
5.  `dbm-ui`后端将结果返回给前端。

**Section sources**
- [bk.py](file://dbm-ui/backend/components/bk.py#L1-L102)
- [urls.py](file://dbm-ui/backend/urls.py#L50)

## 认证方法
API的认证主要通过API网关实现。所有外部请求都必须经过API网关，网关负责验证请求的合法性。

### API网关认证
根据`dbm-ui/backend/dbm_init/apigw/definition.yaml`文件的配置，API网关为`dbm`应用定义了网关实例。网关通过`bk_app_code`和`bk_app_secret`来识别和验证调用方的身份。这些凭证在`bk.py`的`BKAuth`类中被添加到请求体中。

此外，网关还支持主动授权机制，可以预先为指定的应用（`bk_app_code`）授予访问所有资源的权限，这在`definition.yaml`的`grant_permissions`部分有定义。

### 内部服务认证
对于`dbm-ui`与`dbm-services`之间的内部调用，除了API网关的认证外，`dbm-services`内部的服务（如`dbha-v2`）也实现了自己的认证逻辑。例如，在`dbha-v2`的`client.go`中，`SendRequest`方法会设置`Content-Type`头，但具体的认证可能依赖于网络隔离和内部信任机制。

**Section sources**
- [definition.yaml](file://dbm-ui/backend/dbm_init/apigw/definition.yaml#L1-L39)
- [bk.py](file://dbm-ui/backend/components/bk.py#L37-L51)

## API版本控制策略
该平台的API版本控制策略体现在多个层面：

1.  **服务版本**: 在`db-config`的Swagger文档中，`info.version`字段为`0.0.1`，这表明该API的初始版本。未来的更新将通过递增此版本号来管理。
2.  **配置版本**: `db-config`服务本身提供了对配置项的版本管理。`ConfigViewSet`中的`list_config_version_history`和`get_config_version_detail`等API允许查询和回滚到特定的配置版本。这确保了配置变更的可追溯性和可恢复性。
3.  **向后兼容性**: 从API的设计来看，平台倾向于通过添加新的端点或参数来扩展功能，而不是修改现有端点的行为，这有助于保持向后兼容性。例如，`dbactuator`为不同的下载方式（`/download/http`, `/download/scp`）提供了独立的端点，而不是在一个端点内通过参数切换。

**Section sources**
- [views.py](file://dbm-ui/backend/db_services/dbconfig/views.py#L220-L244)

## 错误码
系统定义了统一的错误码体系，用于在不同服务间传递错误信息。

### 错误码定义
在`dbm-services/common/go-pubpkg/errno/errno.go`和`dbm-services/common/db-config/internal/pkg/errno/code.go`等文件中，定义了全局的错误码常量。例如：
- `OK.Code = 0`: 操作成功。
- `ErrInputParameter.Code = 10201`: 输入参数错误。
- `ErrInvokeAPI.Code = 15000`: 调用其他服务API时发生错误。
- `ErrRecordNotFound.Code = 50202`: 数据库中未找到记录。

### 错误处理
当API调用失败时，服务会返回一个包含错误码和错误信息的JSON响应。`dbm-ui`的`BKClient`在`common_request`方法中会检查响应的`result`字段，如果为`false`，则会抛出一个包含错误信息的异常，该异常信息会最终传递给前端用户。

**Section sources**
- [code.go](file://dbm-services/common/db-config/internal/pkg/errno/code.go#L21-L49)

## 客户端代码示例
以下提供使用Python和Go调用`db-config`服务的`/mysql/deploy`端点的示例。

### Python客户端示例
```python
import requests
import json

# API网关地址和端点
url = "http://your-api-gateway/db-config/mysql/deploy"

# 蓝鲸认证信息
auth_data = {
    "bk_app_code": "your_app_code",
    "bk_app_secret": "your_app_secret",
    "bk_username": "your_username"
}

# API请求参数
request_data = {
    "extend": {
        "charset": "utf8mb4",
        "host": "192.168.1.100",
        "ports": [3306],
        "pkg": "mysql-5.7.30-linux-glibc2.12-x86_64.tar.gz",
        "pkg_md5": "a1b2c3d4e5f6...",
        "mycnf_configs": {
            "3306": "[mysqld]\ninnodb_buffer_pool_size=1G"
        },
        "mysql_version": "5.7"
    }
}

# 合并认证信息和请求数据
payload = {**auth_data, **request_data}

# 发送POST请求
response = requests.post(url, json=payload)

# 处理响应
if response.status_code == 200:
    print("部署请求已提交")
else:
    print(f"请求失败: {response.status_code}, {response.text}")
```

### Go客户端示例
```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type DeployMySQLRequest struct {
    Extend InstallMySQLParams `json:"extend"`
    BkAppCode string `json:"bk_app_code"`
    BkAppSecret string `json:"bk_app_secret"`
    BkUsername string `json:"bk_username"`
}

type InstallMySQLParams struct {
    Charset string `json:"charset"`
    Host string `json:"host"`
    Ports []int `json:"ports"`
    Pkg string `json:"pkg"`
    PkgMd5 string `json:"pkg_md5"`
    MycnfConfigs map[int]string `json:"mycnf_configs"`
    MysqlVersion string `json:"mysql_version"`
}

func main() {
    url := "http://your-api-gateway/db-config/mysql/deploy"

    request := DeployMySQLRequest{
        Extend: InstallMySQLParams{
            Charset: "utf8mb4",
            Host: "192.168.1.100",
            Ports: []int{3306},
            Pkg: "mysql-5.7.30-linux-glibc2.12-x86_64.tar.gz",
            PkgMd5: "a1b2c3d4e5f6...",
            MycnfConfigs: map[int]string{
                3306: "[mysqld]\ninnodb_buffer_pool_size=1G",
            },
            MysqlVersion: "5.7",
        },
        BkAppCode: "your_app_code",
        BkAppSecret: "your_app_secret",
        BkUsername: "your_username",
    }

    jsonData, _ := json.Marshal(request)
    resp, err := http.Post(url, "application/json", bytes.NewBuffer(jsonData))
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusOK {
        fmt.Println("部署请求已提交")
    } else {
        fmt.Printf("请求失败: %d\n", resp.StatusCode)
    }
}
```