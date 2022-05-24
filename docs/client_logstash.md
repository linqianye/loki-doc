# Logstash

Loki 有一个名为`logstash-output-loki`的 [ Logstash](https://www.elastic.co/logstash) output 插件 ，可以将日志传送到 Loki 实例或[Grafana Cloud](https://grafana.com/products/cloud/)。

## 安装

### 本地安装

如果您需要手动安装 Loki 输出插件，您可以使用以下命令简单地安装：

```bash
$ bin/logstash-plugin install logstash-output-loki
```

这将下载输出插件的最新 gem 并将其安装在 logstash 中。

### Docker

我们还在[docker hub](https://hub.docker.com/r/grafana/logstash-output-loki)上提供了 docker 镜像。该镜像包含已预安装 Loki 输出插件的 logstash。

例如，如果您想在docker中使用`loki.conf`作为配置运行logstash，您可以使用以下命令：

```bash
docker run -v `pwd`/loki-test.conf:/home/logstash/ --rm grafana/logstash-output-loki:1.0.1 -f loki-test.conf
```

### Kubernetes

我们还提供了默认的 helm，用于使用 Filebeat 抓取日志，并在  `loki-stack` 图中使用logstash将它们转发给Loki。您可以使用以下命令从 Promtail 切换到 logstash：

```bash
helm upgrade --install loki loki/loki-stack \
    --set filebeat.enabled=true,logstash.enabled=true,promtail.enabled=false \
    --set loki.fullnameOverride=loki,logstash.fullnameOverride=logstash-loki
```

这将自动抓取集群中的所有 pod 日志，并将它们发送给 Loki，并将 Kubernetes 元数据作为标签附加。您可以将 [values.yaml](https://github.com/grafana/helm-charts/blob/main/charts/loki-stack/values.yaml) 文件用作您自己配置的起点。

## 使用和配置

要将 Logstash 配置为将日志转发到 Loki，只需将`loki`输出添加到您的[Logstash 配置文件中](https://www.elastic.co/guide/en/logstash/current/configuration-file-structure.html)，如下所述：

```conf
output {
  loki {
    [url => "" | default = none | required=true]

    [tenant_id => string | default = nil | required=false]

    [message_field => string | default = "message" | required=false]

    [batch_wait => number | default = 1(s) | required=false]

    [batch_size => number | default = 102400(bytes) | required=false]

    [min_delay => number | default = 1(s) | required=false]

    [max_delay => number | default = 300(s) | required=false]

    [retries => number | default = 10 | required=false]

    [username => string | default = nil | required=false]

    [password => secret | default = nil | required=false]

    [cert => path | default = nil | required=false]

    [key => path | default = nil| required=false]

    [ca_cert => path | default = nil | required=false]

    [insecure_skip_verify => boolean | default = fasle | required=false]
  }
}
```

默认情况下，Loki 将从它收到的事件字段中创建条目。如下所示的 logstash 事件。

```json
{
  "@timestamp" => 2017-04-26T19:33:39.257Z,
  "src"        => "localhost",
  "@version"   => "1",
  "host"       => "localhost.localdomain",
  "pid"        => "1",
  "message"    => "Apr 26 12:20:02 localhost systemd[1]: Starting system activity accounting tool...",
  "type"       => "stdin",
  "prog"       => "systemd",
}
```

包含`message`和`@timestamp`字段，分别用于构成 Loki 条目日志行和时间戳。

> 您可以使用配置属性[`message_field`](https://grafana.com/docs/loki/latest/clients/logstash/#message_field)为日志行提供不同的属性。如果您还需要更改时间戳值，请使用 Logstash 的`date`过滤器来更改`@timestamp`字段。

所有其他字段（嵌套字段除外）将形成附加到日志行的标签集（键值对）。这意味着您负责改变和删除高基数标签，例如客户端 IP。您通常可以通过使用[`mutate`](https://www.elastic.co/guide/en/logstash/current/plugins-filters-mutate.html)过滤器来实现。

例如下面的配置：

```conf
input {
  ...
}

filter {
  mutate {
    add_field => {
      "cluster" => "us-central1"
      "job" => "logstash"
    }
    replace => { "type" => "stream"}
    remove_field => ["src"]
  }
}
output {
  loki {
    url => "http://myloki.domain:3100/loki/api/v1/push"
  }
}
```

将添加静态标签`cluster`和`job`，删除`src`字段并替换`type`为 `stream`。

如果要包含嵌套字段或元数据字段（以`@`开头），则需要重命名它们。

例如，当 Filebeat 与[`add_kubernetes_metadata`](https://www.elastic.co/guide/en/beats/filebeat/current/add-kubernetes-metadata.html)处理器一起使用时，它会将 Kubernetes 元数据附加到事件中，如下所示：

```json
{
  "kubernetes" : {
    "labels" : {
      "app" : "MY-APP",
      "pod-template-hash" : "959f54cd",
      "serving" : "true",
      "version" : "1.0",
      "visualize" : "true"
    },
    "pod" : {
      "uid" : "e20173cb-3c5f-11ea-836e-02c1ee65b375",
      "name" : "MY-APP-959f54cd-lhd5p"
    },
    "node" : {
      "name" : "ip-xxx-xx-xx-xxx.ec2.internal"
    },
    "container" : {
      "name" : "istio"
    },
    "namespace" : "production",
    "replicaset" : {
      "name" : "MY-APP-959f54cd"
    }
  },
  "message": "Failed to parse configuration",
  "@timestamp": "2017-04-26T19:33:39.257Z",
}
```

下面的过滤器将向您展示如何将Kubernetes字段提取到标签中（`container_name`，`namespace`，`pod`和`host`）：

```conf
filter {
  if [kubernetes] {
    mutate {
      add_field => {
        "container_name" => "%{[kubernetes][container][name]}"
        "namespace" => "%{[kubernetes][namespace]}"
        "pod" => "%{[kubernetes][pod][name]}"
      }
      replace => { "host" => "%{[kubernetes][node][name]}"}
    }
  }
  mutate {
    remove_field => ["tags"]
  }
}
```

### 配置属性

#### URL

要将日志发送到的 Loki 服务器的 URL。发送数据时，还需要提供推送路径，例如：`http://localhost:3100/loki/api/v1/push`.

如果您想发送到[GrafanaCloud，](https://grafana.com/products/cloud/)您可以使用：`https://logs-prod-us-central1.grafana.net/loki/api/v1/push`.

#### username / password

如果 Loki 服务器需要身份验证，请指定用户名和密码。如果使用[GrafanaLab 托管的 Loki](https://grafana.com/products/cloud/)，则需要将用户名设置为您的 instance/user ID，密码应为 Grafana网站 api的密钥。

#### message_field

用于日志行的消息字段。您可以使用logstash键访问器语言来获取嵌套属性，例如：`[log][message]`。

#### batch_wait

在将一批日志推送到 Loki 之前等待的时间间隔（以秒为单位）。这意味着即使在`batch_wait`后未达到批处理大小，也会发送部分批处理，这是为了确保数据的新鲜度。

#### batch_size

在推送到 loki 之前要累积的最大批量的大小。默认为102400字节（100K）

#### 回退配置

##### min_delay => 1(1s)

两次重试之间的最小回退时间

##### max_delay => 300(5m)

两次重试之间的最大回退时间

##### 重试 => 10

最大重试次数

#### tenant_id

Loki是一个多租户日志存储平台，发送的所有请求都必须包含租户。对于某些安装的程序，租户将由身份验证代理自动设置。也可以定义要传递的租户。租户可以是任何字符串值。

#### 客户端证书验证

如果在Loki前面配置了具有客户端证书验证的反向代理，则需要使用`cert` 和 `key`来指定一对客户端证书和私钥。如果服务器使用自定义证书颁发机构，也可以通过 `ca_cert` 指定。

### insecure_skip_verify

禁用服务器证书验证的标志。默认情况下设置为`false`。

### 完整配置示例

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [kubernetes] {
    mutate {
      add_field => {
        "container_name" => "%{[kubernetes][container][name]}"
        "namespace" => "%{[kubernetes][namespace]}"
        "pod" => "%{[kubernetes][pod][name]}"
      }
      replace => { "host" => "%{[kubernetes][node][name]}"}
    }
  }
  mutate {
    remove_field => ["tags"]
  }
}

output {
  loki {
    url => "https://logs-prod-us-central1.grafana.net/loki/api/v1/push"
    username => "3241"
    password => "REDACTED"
    batch_size => 112640 #112.64 kilobytes
    retries => 5
    min_delay => 3
    max_delay => 500
    message_field => "message"
  }
  # stdout { codec => rubydebug }
}
```