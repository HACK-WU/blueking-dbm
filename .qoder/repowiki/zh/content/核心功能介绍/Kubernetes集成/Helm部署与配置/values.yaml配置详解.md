# values.yaml配置详解

<cite>
**本文档引用的文件**  
- [values.yaml](file://helm-charts/bk-dbm/values.yaml)
- [dbconfig/values.yaml](file://helm-charts/bk-dbm/charts/dbconfig/values.yaml)
- [dbm/values.yaml](file://helm-charts/bk-dbm/charts/dbm/values.yaml)
- [dbpriv/values.yaml](file://helm-charts/bk-dbm/charts/dbpriv/values.yaml)
- [dbpartition/values.yaml](file://helm-charts/bk-dbm/charts/dbpartition/values.yaml)
- [k8s-dbs/values.yaml](file://helm-charts/bk-dbm/charts/k8s-dbs/values.yaml)
- [Chart.yaml](file://helm-charts/bk-dbm/Chart.yaml)
- [kb_util.go](file://dbm-services/k8s-dbs/core/util/kb_util.go)
- [dev.py](file://dbm-ui/config/dev.py)
- [stag.py](file://dbm-ui/config/stag.py)
- [prod.py](file://dbm-ui/config/prod.py)
</cite>

## 目录
1. [全局配置](#全局配置)
2. [子Chart配置](#子chart配置)
3. [资源限制与请求](#资源限制与请求)
4. [副本数配置](#副本数配置)
5. [环境变量](#环境变量)
6. [数据库连接配置](#数据库连接配置)
7. [不同环境的配置定制](#不同环境的配置定制)
8. [配置继承与覆盖机制](#配置继承与覆盖机制)
9. [命令行动态覆盖配置](#命令行动态覆盖配置)

## 全局配置

`values.yaml`文件中的全局配置定义了整个Helm Chart的基础设置，这些配置会被所有子Chart继承和使用。全局配置位于文件的`global`部分，主要包括镜像仓库、存储类、域名等关键参数。

全局配置中的`imageRegistry`字段指定了所有镜像的默认仓库地址，`storageClass`定义了持久化存储的类型，`bkDomain`设置了蓝鲸主域名。这些配置为整个系统提供了统一的基础设置，确保各个组件能够协调工作。

`k8sWaitFor`配置项定义了Kubernetes等待工具的镜像信息，用于在部署过程中等待依赖服务就绪。这个工具对于确保服务启动顺序和依赖关系的正确性至关重要。

**Section sources**
- [values.yaml](file://helm-charts/bk-dbm/values.yaml#L3-L22)

## 子Chart配置

Helm Chart结构中包含了多个子Chart，每个子Chart对应一个独立的服务组件。通过`Chart.yaml`文件中的dependencies定义，可以看到主要的子Chart包括：`dbm`、`dbconfig`、`dbpriv`、`dbpartition`、`k8s-dbs`等。

每个子Chart都有自己的`values.yaml`文件，定义了该组件特有的配置参数。例如，`dbconfig`子Chart的配置文件定义了该服务的副本数、镜像信息、资源限制等。主Chart的`values.yaml`文件通过同名的顶级键（如`dbconfig:`）来覆盖或补充子Chart的默认配置。

这种分层的配置结构使得系统既具有统一的全局配置，又能为每个组件提供个性化的设置。通过`enabled`字段可以控制各个子Chart的启用状态，实现灵活的服务组合。

**Section sources**
- [Chart.yaml](file://helm-charts/bk-dbm/Chart.yaml#L2-L108)
- [values.yaml](file://helm-charts/bk-dbm/values.yaml#L243-L412)

## 资源限制与请求

资源限制（resources.limits）和请求（resources.requests）是Kubernetes中控制容器资源使用的重要配置。在`values.yaml`文件中，多个组件都定义了这些参数，用于确保服务的稳定运行和资源的合理分配。

对于关键服务如`bkdata-kafka-consumer`，配置了明确的资源限制：
```yaml
resources:
  limits:
    cpu: 1000m
    memory: 1024Mi
  requests:
    cpu: 200m
    memory: 256Mi
```

这种配置确保了容器至少能获得请求的资源量，同时不会超过限制的资源量。CPU以millicores（m）为单位，内存以Mi（Mebibytes）为单位。合理的资源配置可以避免资源争用，提高系统稳定性。

在`k8s-dbs`组件中，资源配置更为严格：
```yaml
resources:
  limits:
    cpu: 2
    memory: 4Gi
  requests:
    cpu: 1
    memory: 2Gi
```

这反映了不同组件对资源需求的差异，数据库相关服务通常需要更多的计算和内存资源。

**Section sources**
- [values.yaml](file://helm-charts/bk-dbm/values.yaml#L538-L544)
- [k8s-dbs/values.yaml](file://helm-charts/bk-dbm/charts/k8s-dbs/values.yaml#L17-L27)

## 副本数配置

副本数（replicaCount）配置决定了服务实例的数量，是实现高可用和负载均衡的关键参数。在`values.yaml`文件中，多个组件都设置了副本数配置。

大多数服务的默认副本数设置为1：
```yaml
replicaCount: 1
```

这在开发和测试环境中是合适的，但在生产环境中通常需要增加副本数以提高可用性。例如，`dbm`组件的`saas`部分为不同服务设置了独立的副本数配置：
```yaml
saas:
  api:
    replicaCount: 1
  backendApi:
    replicaCount: 1
  celeryBeat:
    replicaCount: 1
  celeryWorker:
    replicaCount: 1
```

这种细粒度的配置允许根据各个服务组件的实际负载需求进行优化。对于处理高并发请求的API服务，可以增加副本数；而对于定时任务服务，单个副本通常就足够了。

**Section sources**
- [dbconfig/values.yaml](file://helm-charts/bk-dbm/charts/dbconfig/values.yaml#L4)
- [dbm/values.yaml](file://helm-charts/bk-dbm/charts/dbm/values.yaml#L27-L72)

## 环境变量

环境变量（env）配置用于向容器传递运行时参数，这些参数通常包括服务地址、认证信息、功能开关等。在`values.yaml`文件中，环境变量主要通过`envs`键来定义。

`dbm`组件定义了大量的环境变量：
```yaml
envs:
  djangoSettingsModule: "config.prod"
  runVer: "open"
  bkAppCode: "bk_dbm"
  bkAppToken: "bk_dbm_token"
  bkSaasUrl: "https://bkdbm.example.com/"
  brokerUrl: "redis://localhost:6379/0"
```

这些变量涵盖了Django框架配置、应用标识、消息队列连接等关键信息。特别值得注意的是`djangoSettingsModule`变量，它决定了Django使用的配置文件，从而影响整个应用的行为。

APM（应用性能监控）相关的环境变量也被广泛使用：
```yaml
TRACE_SERVICE_NAME: dbconfig
TRACE_ENABLE: true
TRACE_HOST: 127.0.0.1
TRACE_PORT: 4317
```

这些变量启用了分布式追踪功能，有助于监控和诊断系统性能问题。

**Section sources**
- [values.yaml](file://helm-charts/bk-dbm/values.yaml#L113-L134)
- [values.yaml](file://helm-charts/bk-dbm/values.yaml#L247-L257)

## 数据库连接配置

数据库连接配置是系统稳定运行的关键，`values.yaml`文件中通过`externalDatabase`部分定义了各个组件的数据库连接信息。

每个主要组件都有独立的数据库配置：
```yaml
externalDatabase:
  dbm:
    username: bk-dbm
    password: external-db-pwd-example
    host: external-db-host-example
    port: 3306
    name: bk_dbm
  dbconfig:
    username: bk-dbm
    password: external-db-pwd-example
    host: external-db-host-example
    port: 3306
    name: bk_dbm_dbconfig
```

这种分离的数据库设计遵循了微服务架构的最佳实践，每个服务拥有独立的数据库实例，降低了耦合度。密码等敏感信息不应直接写在配置文件中，而应通过Kubernetes Secrets等安全机制管理。

对于`k8s-dbs`组件，数据库配置更为详细，包括连接池参数：
```yaml
datasource:
  auth:
    host: ""
    port: 
    user: ""
    password: ""
    dbname: ""
    maxOpenConns: 10
    maxIdelConns: 5
    maxLifeTime: "30m"
    maxIdleTime: "30m"
```

这些连接池参数对于数据库性能至关重要，合理的配置可以避免连接泄漏和资源耗尽。

**Section sources**
- [values.yaml](file://helm-charts/bk-dbm/values.yaml#L713-L791)
- [k8s-dbs/values.yaml](file://helm-charts/bk-dbm/charts/k8s-dbs/values.yaml#L61-L85)

## 不同环境的配置定制

根据不同的部署环境（开发、测试、生产），需要对`values.yaml`文件进行相应的定制。虽然`values.yaml`本身是Helm Chart的一部分，但可以通过覆盖配置来适应不同环境的需求。

在开发环境中，通常会启用调试模式，降低资源限制，使用简单的认证配置：
```yaml
global:
  bkDomain: "dev.example.com"
dbm:
  envs:
    djangoSettingsModule: "config.dev"
    runVer: "development"
```

在测试环境中，配置更接近生产环境，但可能保留一些监控和日志功能：
```yaml
global:
  bkDomain: "test.example.com"
dbm:
  envs:
    djangoSettingsModule: "config.stag"
```

在生产环境中，需要最严格的配置：
```yaml
global:
  bkDomain: "prod.example.com"
  storageClass: "ssd-prod"
dbm:
  envs:
    djangoSettingsModule: "config.prod"
    runVer: "production"
  saas:
    api:
      replicaCount: 3
      resources:
        limits:
          cpu: 2000m
          memory: 2048Mi
```

这种环境差异化的配置策略确保了系统在不同阶段的稳定性和安全性。

**Section sources**
- [dev.py](file://dbm-ui/config/dev.py#L1-L77)
- [stag.py](file://dbm-ui/config/stag.py#L1-L27)
- [prod.py](file://dbm-ui/config/prod.py#L1-L20)

## 配置继承与覆盖机制

Helm Chart的配置系统采用了层次化的继承与覆盖机制。主Chart的`values.yaml`文件可以覆盖子Chart中的同名配置，实现了配置的集中管理和个性化定制。

配置继承遵循特定的优先级规则：子Chart的默认值 < 主Chart的values.yaml < 命令行--set参数 < values文件。这种层次结构提供了极大的灵活性。

在代码层面，`kb_util.go`文件中的`MergeValues`函数实现了配置合并逻辑：
```go
func MergeValues(values map[string]interface{}, request *coreentity.Request, isInstall bool) error {
    err := mergeMetaData(values, request)
    if err != nil {
        return err
    }
    
    err = mergeComponentList(values, request.ComponentList, isInstall)
    if err != nil {
        return err
    }
    
    err = mergeDependencies(values, request.Dependencies)
    if err != nil {
        return err
    }
    
    return nil
}
```

这个函数按顺序合并元数据、组件列表和依赖关系，确保了配置的完整性和一致性。`MergeObjectToVal`函数则负责将对象合并到目标值映射中，使用`mergo.WithOverride`选项确保新值覆盖旧值。

这种机制允许在不修改子Chart的情况下，通过主Chart的配置文件来调整其行为，大大提高了配置管理的效率。

**Section sources**
- [kb_util.go](file://dbm-services/k8s-dbs/core/util/kb_util.go#L243-L264)

## 命行动态覆盖配置

除了静态的`values.yaml`文件，Helm还支持通过`--set`参数在命令行中动态覆盖配置，这为部署提供了极大的灵活性。

基本的`--set`语法如下：
```bash
helm install my-release ./chart --set dbm.saas.api.replicaCount=3 --set dbm.resources.limits.memory=2048Mi
```

对于嵌套的配置项，使用点号（.）分隔层级。可以同时设置多个参数，用空格分隔。

对于列表类型的配置，可以使用数组语法：
```bash
helm install my-release ./chart --set ingress.tls[0].hosts[0]=example.com --set ingress.tls[0].secretName=tls-secret
```

还可以设置布尔值和字符串值：
```bash
helm install my-release ./chart --set global.cloudContainer=true --set dbm.envs.bkAppCode="myapp"
```

在CI/CD流水线中，这种动态覆盖能力特别有用，可以根据不同的部署阶段应用不同的配置，而无需维护多个values文件。

**Section sources**
- [values.yaml](file://helm-charts/bk-dbm/values.yaml#L1-L792)