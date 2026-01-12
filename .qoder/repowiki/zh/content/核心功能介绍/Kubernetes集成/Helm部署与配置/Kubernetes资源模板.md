# Kubernetes资源模板

<cite>
**本文档引用的文件**  
- [dbm-configmap.yaml](file://helm-charts/bk-dbm/templates/configmaps/dbm-configmap.yaml)
- [dbconfig-configmap.yaml](file://helm-charts/bk-dbm/templates/configmaps/dbconfig-configmap.yaml)
- [backend-api.yaml](file://helm-charts/bk-dbm/charts/dbm/templates/deployments/backend-api/backend-api.yaml)
- [saas-api.yaml](file://helm-charts/bk-dbm/charts/dbm/templates/deployments/saas-api/saas-api.yaml)
- [service.yaml](file://helm-charts/bk-dbm/charts/dbm/templates/deployments/backend-api/service.yaml)
- [values.yaml](file://helm-charts/bk-dbm/values.yaml)
- [_helpers.tpl](file://helm-charts/bk-dbm/templates/_helpers.tpl)
- [dbm/_helpers.tpl](file://helm-charts/bk-dbm/charts/dbm/templates/_helpers.tpl)
</cite>

## 目录
1. [简介](#简介)
2. [ConfigMap模板分析](#configmap模板分析)
3. [Deployment模板分析](#deployment模板分析)
4. [Service模板分析](#service模板分析)
5. [Helm模板语言应用](#helm模板语言应用)
6. [总结](#总结)

## 简介
本文档详细分析Helm模板生成的Kubernetes资源，包括ConfigMap、Deployment和Service。以`templates/configmaps/`目录下的各类ConfigMap为例，说明如何将配置文件注入到容器中。解释Deployment模板中的容器镜像、启动命令、健康检查（liveness/readiness probe）和卷挂载（volumeMounts）的配置。阐述Service模板如何定义服务发现和网络访问策略。结合具体YAML代码，展示Helm模板语言（如`{{ .Values }}`、`{{ include }}`）如何动态生成资源清单。

## ConfigMap模板分析

Helm模板中的ConfigMap用于将配置数据注入到Kubernetes Pod中。在`helm-charts/bk-dbm/templates/configmaps/`目录下，有多个ConfigMap模板文件，每个文件对应不同的服务组件。

### dbm-configmap.yaml分析
`dbm-configmap.yaml`文件定义了`bk-dbm-db-env` ConfigMap，用于存储dbm服务的环境变量。该模板使用Helm的`{{ include }}`函数调用`_helpers.tpl`中的`bk-dbm.database`模板，动态生成数据库连接信息。

```yaml
{{- $dbmDB := fromYaml (include "bk-dbm.database" (list . "dbm")) -}}
```

此行代码通过`include`函数调用`bk-dbm.database`模板，并传入当前上下文和"dbm"参数，然后使用`fromYaml`将返回的YAML字符串转换为对象，存储在`$dbmDB`变量中。

ConfigMap的`data`字段包含了多个环境变量，如数据库名称、主机、端口、用户名和密码等，这些值都来自`values.yaml`文件中的配置。

**ConfigMap来源**
- [dbm-configmap.yaml](file://helm-charts/bk-dbm/templates/configmaps/dbm-configmap.yaml)

### dbconfig-configmap.yaml分析
`dbconfig-configmap.yaml`文件定义了`dbconfig-configmap` ConfigMap，用于存储dbconfig服务的配置文件。与`dbm-configmap.yaml`不同，此ConfigMap不仅包含环境变量，还包含完整的配置文件内容。

```yaml
data:
  config.yaml: |-
    gormlog: true

    http:
      listenAddress: 0.0.0.0:80

    db:
      name:  "{{ $dbconfigDB.name }}"
      addr:  "{{ $dbconfigDB.host }}:{{ $dbconfigDB.port }}"
      username:  "{{ $dbconfigDB.user }}"
      password:  "{{ $dbconfigDB.password }}"
```

此ConfigMap使用`|-`语法定义了多行字符串，直接将配置文件内容嵌入到ConfigMap中。这种方式允许将复杂的配置文件（如YAML、JSON、INI等）作为ConfigMap的一部分，然后通过卷挂载的方式注入到容器中。

**ConfigMap来源**
- [dbconfig-configmap.yaml](file://helm-charts/bk-dbm/templates/configmaps/dbconfig-configmap.yaml)

## Deployment模板分析

Deployment模板定义了Kubernetes中Pod的部署策略，包括容器镜像、启动命令、健康检查和卷挂载等配置。

### backend-api.yaml分析
`backend-api.yaml`文件定义了backend-api服务的Deployment。该模板使用Helm的条件判断、变量引用和模板包含等功能，实现了高度的可配置性。

```yaml
{{- if .Values.saas.backendApi.enabled | default .Values.enabled -}}
```

此行代码使用`if`条件判断，只有当`values.yaml`中`saas.backendApi.enabled`为true时，才会生成此Deployment资源。

容器的镜像配置如下：
```yaml
image: "{{ .Values.global.imageRegistry | default .Values.image.registry }}/{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

此行代码使用了Helm的管道操作符`|`和`default`函数，实现了镜像地址的灵活配置。如果`global.imageRegistry`存在，则使用它作为镜像仓库地址，否则使用`image.registry`的值。

容器的启动命令和参数配置如下：
```yaml
command:
  - /bin/bash
  - -c
args:
  - export SERVICE_ONLY=true && gunicorn wsgi -k gevent -w {{ .Values.saas.backendApi.gunicornWorker }} -b :8000 --access-logfile - --error-logfile - --access-logformat '[%(h)s] %({request_id}i)s %(u)s %(t)s "%(r)s" %(s)s %(D)s %(b)s "%(f)s" "%(a)s"'
```

此配置使用`/bin/bash -c`执行一个复合命令，先设置`SERVICE_ONLY`环境变量，然后启动gunicorn服务器。gunicorn的worker数量通过`{{ .Values.saas.backendApi.gunicornWorker }}`动态配置。

健康检查配置如下：
```yaml
livenessProbe:
  httpGet:
    path: {{ .Values.livenessProbe.path | default "/ping"}}
    port: http
  initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds | default 5}}
  periodSeconds: {{ .Values.livenessProbe.periodSeconds | default 30}}
  timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds | default 5}}
  successThreshold: {{ .Values.livenessProbe.successThreshold | default 1}}
  failureThreshold: {{ .Values.livenessProbe.failureThreshold | default 3}}
```

此配置定义了livenessProbe，用于检测容器是否存活。probe类型为httpGet，检测路径、端口、延迟时间、检测周期等参数都可以通过`values.yaml`文件进行配置。

**Deployment来源**
- [backend-api.yaml](file://helm-charts/bk-dbm/charts/dbm/templates/deployments/backend-api/backend-api.yaml)

### saas-api.yaml分析
`saas-api.yaml`文件定义了saas-api服务的Deployment。与`backend-api.yaml`类似，它也使用了条件判断、变量引用和模板包含等功能。

值得注意的是，`saas-api.yaml`中的健康检查配置与`backend-api.yaml`有所不同：
```yaml
livenessProbe:
  httpGet:
    path: {{ .Values.livenessProbe.path | default "/ping"}}
    port: http
  initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds | default 15}}
  periodSeconds: {{ .Values.livenessProbe.periodSeconds | default 30}}
  timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds | default 15}}
```

与`backend-api.yaml`相比，`initialDelaySeconds`和`timeoutSeconds`的默认值更大，这可能是由于saas-api服务启动时间较长，需要更长的初始化时间。

**Deployment来源**
- [saas-api.yaml](file://helm-charts/bk-dbm/charts/dbm/templates/deployments/saas-api/saas-api.yaml)

## Service模板分析

Service模板定义了Kubernetes中服务的网络访问策略，包括服务类型、端口映射和选择器等配置。

### service.yaml分析
`service.yaml`文件定义了backend-api服务的Service。该模板相对简单，主要配置了服务类型、端口映射和选择器。

```yaml
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 8000
      protocol: TCP
      name: http
  selector:
    {{- include "dbm.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: "{{ $fullName }}"
```

此Service的类型和端口通过`values.yaml`文件中的`service.type`和`service.port`进行配置。`targetPort`固定为8000，这是容器内部gunicorn服务器监听的端口。

选择器（selector）使用了`include`函数包含`dbm.selectorLabels`模板，并添加了`app.kubernetes.io/component`标签，确保Service能够正确选择到对应的Pod。

**Service来源**
- [service.yaml](file://helm-charts/bk-dbm/charts/dbm/templates/deployments/backend-api/service.yaml)

## Helm模板语言应用

Helm模板语言提供了强大的功能，使得Kubernetes资源清单的生成更加灵活和可配置。

### values.yaml中的配置
`values.yaml`文件是Helm模板的配置中心，包含了所有可配置的参数。例如：
```yaml
dbm:
  enabled: true
  service:
    port: 80
    type: ClusterIP
  envs:
    djangoSettingsModule: "config.prod"
    runVer: "open"
    bkAppCode: "bk_dbm"
```

这些配置可以在模板中通过`{{ .Values.dbm.enabled }}`、`{{ .Values.dbm.service.port }}`等方式引用。

### _helpers.tpl中的模板函数
`_helpers.tpl`文件定义了多个模板函数，用于生成重复的配置。例如：
```yaml
{{- define "bk-dbm.database" -}}
{{- $root := first . -}}
{{- $name := last . -}}
{{- $values := $root.Values -}}

{{- if $values.mysql.enabled -}}
name: {{ $name | required "mysql.name is required" }}
user: {{ $values.mysql.auth.username | required "mysql.auth.username is required" }}
password: {{ $values.mysql.auth.password | required "mysql.auth.password is required" }}
host: {{ include "mysql.primary.fullname" (dict "Values" $values.mysql "Chart" $root.Chart "Release" $root.Release) }}
port: {{ $values.mysql.primary.service.port | required "mysql.primary.service.port is required" }}
{{- else -}}
{{- $dbDefault := $values.externalDatabase -}}
{{- $db := index $values.externalDatabase $name -}}
name: {{ $db.name | default $dbDefault.name | required "externalDatabase.name is required" }}
user: {{ $db.username | default $dbDefault.username | required "externalDatabase.username is required" }}
password: {{ $db.password | default $dbDefault.password | required "externalDatabase.password is required" }}
host: {{ $db.host | default $dbDefault.host  | required "externalDatabase.host is required" }}
port: {{ $db.port | default $dbDefault.port | required "externalDatabase.port is required" }}
{{- end -}}
{{- end -}}
```

此模板函数根据`mysql.enabled`的值，决定使用内建的MySQL还是外部的MySQL，并生成相应的数据库连接信息。这种设计使得模板能够适应不同的部署环境。

**Helm模板来源**
- [values.yaml](file://helm-charts/bk-dbm/values.yaml)
- [_helpers.tpl](file://helm-charts/bk-dbm/templates/_helpers.tpl)
- [dbm/_helpers.tpl](file://helm-charts/bk-dbm/charts/dbm/templates/_helpers.tpl)

## 总结
本文档详细分析了Helm模板生成的Kubernetes资源，包括ConfigMap、Deployment和Service。通过分析`templates/configmaps/`目录下的ConfigMap模板，说明了如何将配置文件注入到容器中。通过分析Deployment模板，解释了容器镜像、启动命令、健康检查和卷挂载的配置。通过分析Service模板，阐述了服务发现和网络访问策略的定义。最后，通过分析`values.yaml`和`_helpers.tpl`文件，展示了Helm模板语言如何动态生成资源清单。这些分析有助于理解Helm模板的工作原理，为后续的模板开发和维护提供了参考。