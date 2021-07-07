# Loki的HTTP API

Loki 暴露了一个用于推送、查询和跟踪日志数据的HTTP API。请注意，针对 API 进行[身份验证](https://grafana.com/docs/loki/v2.2.0/operations/authentication/)超出了 Loki 的范围。

HTTP API 包括以下endpoints：

- [Microservices Mode](#微服务模式)
- [Matrix, Vector, And Streams](#Matrix,vector,and streams)
- [`GET /loki/api/v1/query`](#`GET /loki/api/v1/query`)
- [`GET /loki/api/v1/query_range` ](`GET /loki/api/v1/query_range`)
- [`GET /loki/api/v1/labels`](`GET /loki/api/v1/labels`)
- [`GET /loki/api/v1/label/<name>/values`](`GET /loki/api/v1/label/<name>/values`)
- [`GET /loki/api/v1/tail`](`GET /loki/api/v1/tail`)
- [`POST /loki/api/v1/push`](`POST /loki/api/v1/push`)
- [`GET /api/prom/tail`](`GET /api/prom/tail`)
- [`GET /api/prom/query`](`GET /api/prom/query`)
- [`GET /api/prom/label`](`GET /api/prom/label`)
- [`GET /api/prom/label/<name>/values`]()
- [`POST /api/prom/push`](`GET /api/prom/push`)
- [`GET /ready`](https://grafana.com/docs/loki/v2.2.0/api/#get-ready)
- [`POST /flush`](https://grafana.com/docs/loki/v2.2.0/api/#post-flush)
- [`POST /ingester/flush_shutdown`](https://grafana.com/docs/loki/v2.2.0/api/#post-ingesterflush_shutdown)
- [`GET /metrics`](https://grafana.com/docs/loki/v2.2.0/api/#get-metrics)
- [`GET /config`](https://grafana.com/docs/loki/v2.2.0/api/#get-config)
- [Series](https://grafana.com/docs/loki/v2.2.0/api/#series)
- [统计](https://grafana.com/docs/loki/v2.2.0/api/#statistics)
- [`GET /ruler/ring`](https://grafana.com/docs/loki/v2.2.0/api/#ruler-ring-status)
- [`GET /loki/api/v1/rules`](https://grafana.com/docs/loki/v2.2.0/api/#list-rule-groups)
- [`GET /loki/api/v1/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#get-rule-groups-by-namespace)
- [`GET /loki/api/v1/rules/{namespace}/{groupName}`](https://grafana.com/docs/loki/v2.2.0/api/#get-rule-group)
- [`POST /loki/api/v1/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#set-rule-group)
- [`DELETE /loki/api/v1/rules/{namespace}/{groupName}`](https://grafana.com/docs/loki/v2.2.0/api/#delete-rule-group)
- [`DELETE /loki/api/v1/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#delete-namespace)
- [`GET /api/prom/rules`](https://grafana.com/docs/loki/v2.2.0/api/#list-rule-groups)
- [`GET /api/prom/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#get-rule-groups-by-namespace)
- [`GET /api/prom/rules/{namespace}/{groupName}`](https://grafana.com/docs/loki/v2.2.0/api/#get-rule-group)
- [`POST /api/prom/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#set-rule-group)
- [`DELETE /api/prom/rules/{namespace}/{groupName}`](https://grafana.com/docs/loki/v2.2.0/api/#delete-rule-group)
- [`DELETE /api/prom/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#delete-namespace)
- [`GET /prometheus/api/v1/rules`](https://grafana.com/docs/loki/v2.2.0/api/#list-rules)
- [`GET /prometheus/api/v1/alerts`](https://grafana.com/docs/loki/v2.2.0/api/#list-alerts)

## 微服务模式

在微服务模式下部署 Loki 时，每个组件公开的endpoints是不同的。

以下endpoints被所有组件公开：

- [`GET /ready`](https://grafana.com/docs/loki/v2.2.0/api/#get-ready)
- [`GET /metrics`](https://grafana.com/docs/loki/v2.2.0/api/#get-metrics)
- [`GET /config`](https://grafana.com/docs/loki/v2.2.0/api/#get-config)

以下endpoints由querier和前端公开：

- `GET /loki/api/v1/query`
- `GET /loki/api/v1/query_range`
- `GET /loki/api/v1/labels`
- `GET /loki/api/v1/label/<name>/values`
- [`GET /loki/api/v1/tail`](https://grafana.com/docs/loki/v2.2.0/api/#get-lokiapiv1tail)
- `POST /loki/api/v1/push`
- [`GET /api/prom/tail`](https://grafana.com/docs/loki/v2.2.0/api/#get-apipromtail)
- `GET /api/prom/query`
- `GET /api/prom/label`
- `GET /api/prom/label/<name>/values`
- `POST /api/prom/push`
- [Series](https://grafana.com/docs/loki/v2.2.0/api/#series)
- [统计](https://grafana.com/docs/loki/v2.2.0/api/#statistics)

这些endpoints仅由distributor公开：

- [`POST /loki/api/v1/push`](https://grafana.com/docs/loki/v2.2.0/api/#post-lokiapiv1push)

这些endpoints仅由 ingester公开：

- [`POST /flush`](https://grafana.com/docs/loki/v2.2.0/api/#post-flush)
- [`POST /ingester/flush_shutdown`](https://grafana.com/docs/loki/v2.2.0/api/#post-ingesterflush_shutdown)

以`/loki/`开头的API endpoints与[Prometheus API兼容](https://prometheus.io/docs/prometheus/latest/querying/api/)，并且结果格式可以互换使用。

这些endpoints由ruler公开：

- [`GET /ruler/ring`](https://grafana.com/docs/loki/v2.2.0/api/#ruler-ring-status)
- [`GET /loki/api/v1/rules`](https://grafana.com/docs/loki/v2.2.0/api/#list-rule-groups)
- [`GET /loki/api/v1/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#get-rule-groups-by-namespace)
- [`GET /loki/api/v1/rules/{namespace}/{groupName}`](https://grafana.com/docs/loki/v2.2.0/api/#get-rule-group)
- [`POST /loki/api/v1/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#set-rule-group)
- [`DELETE /loki/api/v1/rules/{namespace}/{groupName}`](https://grafana.com/docs/loki/v2.2.0/api/#delete-rule-group)
- [`DELETE /loki/api/v1/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#delete-namespace)
- [`GET /api/prom/rules`](https://grafana.com/docs/loki/v2.2.0/api/#list-rule-groups)
- [`GET /api/prom/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#get-rule-groups-by-namespace)
- [`GET /api/prom/rules/{namespace}/{groupName}`](https://grafana.com/docs/loki/v2.2.0/api/#get-rule-group)
- [`POST /api/prom/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#set-rule-group)
- [`DELETE /api/prom/rules/{namespace}/{groupName}`](https://grafana.com/docs/loki/v2.2.0/api/#delete-rule-group)
- [`DELETE /api/prom/rules/{namespace}`](https://grafana.com/docs/loki/v2.2.0/api/#delete-namespace)
- [`GET /prometheus/api/v1/rules`](https://grafana.com/docs/loki/v2.2.0/api/#list-rules)
- [`GET /prometheus/api/v1/alerts`](https://grafana.com/docs/loki/v2.2.0/api/#list-alerts)

可在clients文档中找到[clients清单](clients.md)。

## Matrix,vector,and streams

一些 Loki API endpoints返回矩阵、向量或流的结果：

- Matrix(矩阵)：一个表类型的值，其中每一行代表一个不同的标签集，列是查询时间内该行的每个样本值。矩阵类型仅在运行计算某些值的查询时返回。
- Instant Vector(瞬时向量)：在类型标识为 `vector`，瞬时向量表示给定标签集的最新计算值。仅在对单个时间点进行查询时才返回即时向量。
- Stream(流)：流是在查询时间范围内给定标签集的所有值（日志）的集合。流是唯一会导致返回日志行的类型。

## `GET /loki/api/v1/query`

`/loki/api/v1/query`允许对单个时间点进行查询。URL 查询参数支持以下值：

- `query`：要执行的[LogQL](https://grafana.com/docs/loki/v2.2.0/logql/)查询
- `limit`：要返回的最大条目数
- `time`：查询的评估时间，单位为纳秒。默认为现在。
- `direction`：确定日志的排序顺序。支持的值为`forward`或`backward`。默认为`backward.`

在微服务模式下，`/loki/api/v1/query`由querier和前端公开。

响应：

```json
{
  "status": "success",
  "data": {
    "resultType": "vector" | "streams",
    "result": [<vector value>] | [<stream value>].
    "stats" : [<statistics>]
  }
}
```

`<vector value>`的响应：

```json
{
  "metric": {
    <label key-value pairs>
  },
  "value": [
    <number: second unix epoch>,
    <string: value>
  ]
}
```

`<stream value>`的响应：

```json
{
  "stream": {
    <label key-value pairs>
  },
  "values": [
    [
      <string: nanosecond unix epoch>,
      <string: log line>
    ],
    ...
  ]
}
```

有关Loki 返回的统计信息的信息，请参阅[统计](https://grafana.com/docs/loki/v2.2.0/api/#Statistics)信息。

### 例子

```bash
$ curl -G -s  "http://localhost:3100/loki/api/v1/query" --data-urlencode 'query=sum(rate({job="varlogs"}[10m])) by (level)' | jq
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {},
        "value": [
          1588889221,
          "1267.1266666666666"
        ]
      },
      {
        "metric": {
          "level": "warn"
        },
        "value": [
          1588889221,
          "37.77166666666667"
        ]
      },
      {
        "metric": {
          "level": "info"
        },
        "value": [
          1588889221,
          "37.69"
        ]
      }
    ],
    "stats": {
      ...
    }
  }
}
```

```bash
$ curl -G -s  "http://localhost:3100/loki/api/v1/query" --data-urlencode 'query={job="varlogs"}' | jq
{
  "status": "success",
  "data": {
    "resultType": "streams",
    "result": [
      {
        "stream": {
          "filename": "/var/log/myproject.log",
          "job": "varlogs",
          "level": "info"
        },
        "values": [
          [
            "1568234281726420425",
            "foo"
          ],
          [
            "1568234269716526880",
            "bar"
          ]
        ],
      }
    ],
    "stats": {
      ...
    }
  }
}
```

## `GET /loki/api/v1/query_range`

`/loki/api/v1/query_range` 用于在一段时间内执行查询，并接受 URL 中的以下查询参数：

- `query`：要执行的[LogQL](https://grafana.com/docs/loki/v2.2.0/logql/)查询
- `limit`：要返回的最大条目数
- `start`：查询的开始时间，单位为纳秒。默认为一小时前。
- `end`：查询的结束时间，单位为纳秒。默认为现在。
- `step`：以`duration`格式或浮点秒数查询解析步长。`duration`指形式为 `[0-9]+[smhdwy]`的 Prometheus 持续时间字符串。例如，5m表示持续时间为5分钟。默认为基于`start`和`end`的动态值。仅适用于产生矩阵响应的查询类型。
- `interval`：**实验性参数，见下文**。仅返回（或大于）指定间隔的条目，可以是`duration`格式或浮点秒数。仅适用于产生流响应的查询。
- `direction`：确定日志的排序顺序。支持的值为`forward`或`backward`。默认为`backward.`

在微服务模式下，`/loki/api/v1/query_range`由querier和前端公开。

##### 步长与间隔

在对 Loki 进行指标查询或返回矩阵响应的查询时使用`step`参数。它的计算方式与 Prometheus 的计算方式完全相同。首先，查询将在`start`处求值，然后在`start + step`和`start + step + step`处再次求值，直到到达`end`。结果是将结果按照每一步长计算的矩阵。

在对 Loki 进行日志查询或返回流响应的查询时使用`interval`参数。在`start`处返回一个日志条目，然后在时间戳>=`start + interval`时返回下一个条目，然后在时间戳>=`start + interval + interval`时再返回下一个条目，以此类推，直到到达`end`。它不会填充缺失的条目。

**请注意 interval 的实验性质** 此参数将来可能会被删除，如果是这样，它可能会支持 LogQL 表达式来执行类似的行为，但目前尚不确定。 创建[问题 1779](https://github.com/grafana/loki/issues/1779)是为了跟踪讨论，如果您正在使用`interval`，请将您的用例和想法添加到该问题。

响应：

```json
{
  "status": "success",
  "data": {
    "resultType": "matrix" | "streams",
    "result": [<matrix value>] | [<stream value>]
    "stats" : [<statistics>]
  }
}
```

`<matrix value>`的响应：

```
{
  "metric": {
    <label key-value pairs>
  },
  "values": [
    <number: second unix epoch>,
    <string: value>
  ]
}
```

`<stream value>`的响应：

```
{
  "stream": {
    <label key-value pairs>
  },
  "values": [
    [
      <string: nanosecond unix epoch>,
      <string: log line>
    ],
    ...
  ]
}
```

有关Loki 返回的统计信息的信息，请参阅[统计](https://grafana.com/docs/loki/v2.2.0/api/#Statistics)信息。

### 例子

```bash
$ curl -G -s  "http://localhost:3100/loki/api/v1/query_range" --data-urlencode 'query=sum(rate({job="varlogs"}[10m])) by (level)' --data-urlencode 'step=300' | jq
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
       "metric": {
          "level": "info"
        },
        "values": [
          [
            1588889221,
            "137.95"
          ],
          [
            1588889221,
            "467.115"
          ],
          [
            1588889221,
            "658.8516666666667"
          ]
        ]
      },
      {
        "metric": {
          "level": "warn"
        },
        "values": [
          [
            1588889221,
            "137.27833333333334"
          ],
          [
            1588889221,
            "467.69"
          ],
          [
            1588889221,
            "660.6933333333334"
          ]
        ]
      }
    ],
    "stats": {
      ...
    }
  }
}
```

```bash
$ curl -G -s  "http://localhost:3100/loki/api/v1/query_range" --data-urlencode 'query={job="varlogs"}' | jq
{
  "status": "success",
  "data": {
    "resultType": "streams",
    "result": [
      {
        "stream": {
          "filename": "/var/log/myproject.log",
          "job": "varlogs",
          "level": "info"
        },
        "values": [
          [
            "1569266497240578000",
            "foo"
          ],
          [
            "1569266492548155000",
            "bar"
          ]
        ]
      }
    ],
    "stats": {
      ...
    }
  }
}
```

## `GET /loki/api/v1/labels`

`/loki/api/v1/labels`查询给定时间范围内的已知标签列表。它接受 URL 中的以下查询参数：

- `start`: 查询的开始时间，单位为纳秒。默认为 6 小时前。
- `end`: 查询的结束时间，单位为纳秒。默认为现在。

在微服务模式下，`/loki/api/v1/labels`由querier公开。

响应：

```json
{
  "status": "success",
  "data": [
    <label string>,
    ...
  ]
}
```

### 例子

```bash
$ curl -G -s  "http://localhost:3100/loki/api/v1/labels" | jq
{
  "status": "success",
  "data": [
    "foo",
    "bar",
    "baz"
  ]
}
```

## `GET /loki/api/v1/label/<name>/values`

`/loki/api/v1/label/<name>/values`查询给定时间范围内给定标签的已知值列表。它接受 URL 中的以下查询参数：

- `start`: 查询的开始时间，单位为纳秒。默认为 6 小时前。
- `end`: 查询的结束时间，单位为纳秒。默认为现在。

在微服务模式下，`/loki/api/v1/label/<name>/values`由querier公开。

回复：

```
{
  "status": "success",
  "data": [
    <label value>,
    ...
  ]
}
```

### 例子

```bash
$ curl -G -s  "http://localhost:3100/loki/api/v1/label/foo/values" | jq
{
  "status": "success",
  "data": [
    "cat",
    "dog",
    "axolotl"
  ]
}
```

## `GET /loki/api/v1/tail`

`/loki/api/v1/tail`是一个 WebSocket 端点，它将根据查询流式传输日志消息。它接受 URL 中的以下查询参数：

- `query`：要执行的[LogQL](https://grafana.com/docs/loki/v2.2.0/logql/)查询
- `delay_for`：延迟检索日志以让慢速记录器赶上的秒数。默认为 0，并且不能大于 5。
- `limit`: 要返回的最大条目数
- `start`: 查询的开始时间，单位为纳秒。默认为一小时前。

在微服务模式下，`/loki/api/v1/tail`由querier公开。

响应：

```json
{
  "streams": [
    {
      "stream": {
        <label key-value pairs>
      },
      "values": [
        [
          <string: nanosecond unix epoch>,
          <string: log line>
        ]
      ]
    }
  ],
  "dropped_entries": [
    {
      "labels": {
        <label key-value pairs>
      },
      "timestamp": "<nanosecond unix epoch>"
    }
  ]
}
```

## `POST /loki/api/v1/push`

`/loki/api/v1/push`是用于向 Loki 发送日志条目的endpoint。默认行为是 POST，body是一个快速压缩的 protobuf 消息：

- [Protobuf 的定义](https://github.com/grafana/loki/tree/v2.2.0/pkg/logproto/logproto.proto)
- [Go 客户端库](https://github.com/grafana/loki/tree/v2.2.0/pkg/promtail/client/client.go)

或者，如果`Content-Type`标头设置为`application/json`，则可以按以下格式发送 JSON post body：

```json
{
  "streams": [
    {
      "stream": {
        "label": "value"
      },
      "values": [
          [ "<unix epoch in nanoseconds>", "<log line>" ],
          [ "<unix epoch in nanoseconds>", "<log line>" ]
      ]
    }
  ]
}
```

您可以设置`Content-Encoding: gzip`请求标头并post 用gzip压缩的 JSON。

> **注意**：每个流发送到 Loki 的日志必须按时间戳升序排列；只有当它们的内容不同时，才允许具有相同时间戳的日志。如果收到的日志行的时间戳比最近收到的日志更旧，则会因出现乱序错误而被拒绝。如果收到的日志与最近的日志具有相同的时间戳和内容，则它会被静默忽略。有关排序规则的更多详细信息，请参阅 [Loki 概述文档](概述.md#timestamp-ordering)。

在微服务模式下，`/loki/api/v1/push`由distributor公开。

### 例子

```bash
$ curl -v -H "Content-Type: application/json" -XPOST -s "http://localhost:3100/loki/api/v1/push" --data-raw \
  '{"streams": [{ "stream": { "foo": "bar2" }, "values": [ [ "1570818238000000000", "fizzbuzz" ] ] }]}'
```

## `GET /api/prom/tail`

> **已弃用**：`/api/prom/tail`已弃用。使用`/loki/api/v1/tail` 来代替。

`/api/prom/tail`是一个 WebSocket endpoint，它将根据查询流式传输日志消息。它接受 URL 中的以下查询参数：

- `query`：要执行的[LogQL](https://grafana.com/docs/loki/v2.2.0/logql/)查询
- `delay_for`：延迟检索日志以让慢速记录器赶上的秒数。默认为 0，并且不能大于 5。
- `limit`: 要返回的最大条目数 
- `start`: 查询的开始时间，单位为纳秒。默认为一小时前。

在微服务模式下，`/api/prom/tail`由querier公开。

响应：

```json
{
  "streams": [
    {
      "labels": "<LogQL label key-value pairs>",
      "entries": [
        {
          "ts": "<RFC3339Nano timestamp>",
          "line": "<log line>"
        }
      ]
    }
  ],
  "dropped_entries": [
    {
      "Timestamp": "<RFC3339Nano timestamp>",
      "Labels": "<LogQL label key-value pairs>"
    }
  ]
}
```

当 tailer 跟不上 Loki 中的流量时，`dropped_entries`将被填充。如果它存在，则表示在流中接收到的条目不是 Loki 中存在的全部日志量。需要注意的是`dropped_entries`中的键会被作为全大写的`Timestamp` 和`Labels`发送，而不是数据流条目中的`ts` 和 `labels`。

随着响应被流式传输，上面的响应格式定义的对象将通过 WebSocket 多次发送。

## `GET /api/prom/query`

> **警告**：`/api/prom/query`已弃用；使用`/loki/api/v1/query_range` 来代替。

`/api/prom/query`支持进行一般查询。URL 查询参数支持以下值：

- `query`：要执行的[LogQL](https://grafana.com/docs/loki/v2.2.0/logql/)查询
- `limit`：要返回的最大条目数
- `start`：查询的开始时间，单位为纳秒。默认为一小时前。
- `end`：查询的结束时间，单位为纳秒。默认为现在。
- `direction`：确定日志的排序顺序。支持的值为`forward`或`backward`。默认为`backward.`
- `regexp`：用于过滤返回结果的正则表达式

在微服务模式下，`/api/prom/query`由querier和前端公开。

请注意，`start`和`end`之间的时间跨度越大，将导致 Loki 和索引存储的负载越大，从而导致查询速度越慢。

响应：

```json
{
  "streams": [
    {
      "labels": "<LogQL label key-value pairs>",
      "entries": [
        {
          "ts": "<RFC3339Nano string>",
          "line": "<log line>"
        },
        ...
      ],
    },
    ...
  ],
  "stats": [<statistics>]
}
```

有关Loki 返回的统计信息的信息，请参阅[统计](https://grafana.com/docs/loki/v2.2.0/api/#Statistics)信息。

### 例子

```bash
$ curl -G -s  "http://localhost:3100/api/prom/query" --data-urlencode '{foo="bar"}' | jq
{
  "streams": [
    {
      "labels": "{filename=\"/var/log/myproject.log\", job=\"varlogs\", level=\"info\"}",
      "entries": [
        {
          "ts": "2019-06-06T19:25:41.972739Z",
          "line": "foo"
        },
        {
          "ts": "2019-06-06T19:25:41.972722Z",
          "line": "bar"
        }
      ]
    }
  ],
  "stats": {
    ...
  }
}
```

## `GET /api/prom/label`

> **警告**：`/api/prom/label`已弃用；用`/loki/api/v1/labels`代替。

`/api/prom/label`检索给定时间范围内的已知标签列表。它接受 URL 中的以下查询参数：

- `start`: 查询的开始时间，单位为纳秒。默认为 6 小时前。
- `end`: 查询的结束时间，单位为纳秒。默认为现在。

在微服务模式下，`/api/prom/label`由querier公开。

响应：

```json
{
  "values": [
    <label string>,
    ...
  ]
}
```

### 例子

```bash
$ curl -G -s  "http://localhost:3100/api/prom/label" | jq
{
  "values": [
    "foo",
    "bar",
    "baz"
  ]
}
```

## `GET /api/prom/label/<name>/values`

> **警告**：`/api/prom/label/<name>/values`已弃用；用`/loki/api/v1/label/<name>/values`代替。

`/api/prom/label/<name>/values`检索给定时间跨度内给定标签的已知值列表。它接受 URL 中的以下查询参数：

- `start`: 查询的开始时间，单位为纳秒。默认为 6 小时前。
- `end`: 查询的结束时间，单位为纳秒。默认为现在。

在微服务模式下，`/api/prom/label/<name>/values`由querier公开。

响应：

```json
{
  "values": [
    <label value>,
    ...
  ]
}
```

### 例子

```bash
$ curl -G -s  "http://localhost:3100/api/prom/label/foo/values" | jq
{
  "values": [
    "cat",
    "dog",
    "axolotl"
  ]
}
```

## `POST /api/prom/push`

> **警告**：`/api/prom/push`已弃用；使用`/loki/api/v1/push` 来代替。

`/api/prom/push`是用于向 Loki 发送日志条目的endpoint。默认行为是 POST，body是一个快速压缩的 protobuf 消息：

- [Protobuf 定义](https://github.com/grafana/loki/tree/v2.2.0/pkg/logproto/logproto.proto)
- [Go 客户端库](https://github.com/grafana/loki/tree/v2.2.0/pkg/promtail/client/client.go)

或者，如果`Content-Type`标头设置为`application/json`，则可以按以下格式发送 JSON post body

```json
{
  "streams": [
    {
      "labels": "<LogQL label key-value pairs>",
      "entries": [
        {
          "ts": "<RFC3339Nano string>",
          "line": "<log line>"
        }
      ]
    }
  ]
}
```

> **注意**：每个流发送到 Loki 的日志必须按时间戳升序排列，这意味着每个日志行必须比上次接收的日志行更新。如果日志不遵循此顺序，Loki 将拒绝日志并出现乱序错误。。

在微服务模式下，`/api/prom/push`由distributor公开。

### 例子

```bash
$ curl -H "Content-Type: application/json" -XPOST -s "https://localhost:3100/api/prom/push" --data-raw \
  '{"streams": [{ "labels": "{foo=\"bar\"}", "entries": [{ "ts": "2018-12-18T08:28:06.801064-04:00", "line": "fizzbuzz" }] }]}'
```

## `GET /ready`

`/ready`当 Loki ingester准备好接受流量时，返回 HTTP 200。如果在 Kubernetes 上运行 Loki，`/ready`可以用作就绪探针。

在微服务模式下，`/ready ` endpoint被所有组件公开。

## `POST /flush`

`/flush`将触发ingesters持有的所有内存中的块刷新到持久存储中。主要用于局部测试。

在微服务模式下，`/flush`端点由ingester公开。

## `POST /ingester/flush_shutdown`

`/ingester/flush_shutdown`会触发ingester的关闭，并总是刷新它持有的所有内存中的块。这有助于缩小启用 WAL 的ingester，我们希望确保旧的 WAL 目录不是孤立的，而是刷新到我们的持久存储块中。

在微服务模式下，`/ingester/flush_shutdown`端点由ingester公开。

## `GET /metrics`

`/metrics`公开 Prometheus 指标。有关导出指标的列表，请参阅 [查看 Loki](操作.md#查看Loki)。

在微服务模式下，`/metrics` endpoint被所有组件公开。

## `GET /config`

`/config`公开当前配置。可选的`mode`查询参数可用于修改输出。如果它具有`diff`属性值，则仅返回默认配置和当前配置之间的差异。有`defaults`属性则返回默认配置。

在微服务模式下，`/config`endpoint被所有组件公开。

## Series

Series API 可在以下位置使用：

- `GET /loki/api/v1/series`
- `POST /loki/api/v1/series`
- `GET /api/prom/series`
- `POST /api/prom/series`

这个endpoint返回与特定标签集匹配的时间序列列表。

支持以下URL查询参数：

- `match[]=<series_selector>`：选择要返回的流的重复日志流选择器参数。必须至少提供一个`match[]`参数。
- `start=<nanosecond Unix epoch>`：开始时间戳。
- `end=<nanosecond Unix epoch>`：结束时间戳。

您可以使用 POST 方法和`Content-Type: application/x-www-form-urlencoded`标头直接在请求正文中对这些参数进行 URL 编码。这在指定可能违反服务器端 URL 字符限制的大量或动态流选择器时很有用。

在微服务模式下，这些端点由querier公开。

### 例子

```bash
$ curl -s "http://localhost:3100/loki/api/v1/series" --data-urlencode 'match={container_name=~"prometheus.*", component="server"}' --data-urlencode 'match={app="loki"}' | jq '.'
{
  "status": "success",
  "data": [
    {
      "container_name": "loki",
      "app": "loki",
      "stream": "stderr",
      "filename": "/var/log/pods/default_loki-stack-0_50835643-1df0-11ea-ba79-025000000001/loki/0.log",
      "name": "loki",
      "job": "default/loki",
      "controller_revision_hash": "loki-stack-757479754d",
      "statefulset_kubernetes_io_pod_name": "loki-stack-0",
      "release": "loki-stack",
      "namespace": "default",
      "instance": "loki-stack-0"
    },
    {
      "chart": "prometheus-9.3.3",
      "container_name": "prometheus-server-configmap-reload",
      "filename": "/var/log/pods/default_loki-stack-prometheus-server-696cc9ddff-87lmq_507b1db4-1df0-11ea-ba79-025000000001/prometheus-server-configmap-reload/0.log",
      "instance": "loki-stack-prometheus-server-696cc9ddff-87lmq",
      "pod_template_hash": "696cc9ddff",
      "app": "prometheus",
      "component": "server",
      "heritage": "Tiller",
      "job": "default/prometheus",
      "namespace": "default",
      "release": "loki-stack",
      "stream": "stderr"
    },
    {
      "app": "prometheus",
      "component": "server",
      "filename": "/var/log/pods/default_loki-stack-prometheus-server-696cc9ddff-87lmq_507b1db4-1df0-11ea-ba79-025000000001/prometheus-server/0.log",
      "release": "loki-stack",
      "namespace": "default",
      "pod_template_hash": "696cc9ddff",
      "stream": "stderr",
      "chart": "prometheus-9.3.3",
      "container_name": "prometheus-server",
      "heritage": "Tiller",
      "instance": "loki-stack-prometheus-server-696cc9ddff-87lmq",
      "job": "default/prometheus"
    }
  ]
}
```

## 统计数据

如`/api/prom/query`，`/loki/api/v1/query`和`/loki/api/v1/query_range`的Query endpoints返回一组有关查询执行的统计信息。这些统计数据使用户能够了解处理的数据量和处理速度。

下面的示例显示了返回的所有可能的统计数据及其各自的描述。

```json
{
  "status": "success",
  "data": {
    "resultType": "streams",
    "result": [],
    "stats": {
     "ingester" : {
        "compressedBytes": 0, // Total bytes of compressed chunks (blocks) processed by ingesters
        "decompressedBytes": 0, // Total bytes decompressed and processed by ingesters
        "decompressedLines": 0, // Total lines decompressed and processed by ingesters
        "headChunkBytes": 0, // Total bytes read from ingesters head chunks
        "headChunkLines": 0, // Total lines read from ingesters head chunks
        "totalBatches": 0, // Total batches sent by ingesters
        "totalChunksMatched": 0, // Total chunks matched by ingesters
        "totalDuplicates": 0, // Total of duplicates found by ingesters
        "totalLinesSent": 0, // Total lines sent by ingesters
        "totalReached": 0 // Amount of ingesters reached.
      },
      "store": {
        "compressedBytes": 0, // Total bytes of compressed chunks (blocks) processed by the store
        "decompressedBytes": 0,  // Total bytes decompressed and processed by the store
        "decompressedLines": 0, // Total lines decompressed and processed by the store
        "chunksDownloadTime": 0, // Total time spent downloading chunks in seconds (float)
        "totalChunksRef": 0, // Total chunks found in the index for the current query
        "totalChunksDownloaded": 0, // Total of chunks downloaded
        "totalDuplicates": 0 // Total of duplicates removed from replication
      },
      "summary": {
        "bytesProcessedPerSecond": 0, // Total of bytes processed per second
        "execTime": 0, // Total execution time in seconds (float)
        "linesProcessedPerSecond": 0, // Total lines processed per second
        "totalBytesProcessed":0, // Total amount of bytes processed overall for this request
        "totalLinesProcessed":0 // Total amount of lines processed overall for this request
      }
    }
  }
}
```

## Ruler

Ruler API endpoint需要配置后端对象存储来存储记录规则和警报。在创建规则组时，ruler API 使用“命名空间”的概念。这是 Prometheus 中规则文件名称的替代。规则组必须在命名空间内唯一命名。

### Ruler环状态

```
GET /ruler/ring
```

显示带有ruler哈希环状态的网页，包括每个ruler的状态、健康度和上次心跳时间。

### 列出规则组

```
GET /loki/api/v1/rules
```

列出经过身份验证的租户配置的所有规则。此endpoint返回一个YAML字典，其中包含每个命名空间的所有规则组和成功时的状态代码`200`。

这个实验性endpoint默认是禁用的，可以通过CLI参数`-experimental.ruler.enable-api`（或其相应的 YAML 配置选项）来启用。

#### 响应示例

```yaml
---
<namespace1>:
- name: <string>
  interval: <duration;optional>
  rules:
  - alert: <string>
      expr: <string>
      for: <duration>
      annotations:
      <annotation_name>: <string>
      labels:
      <label_name>: <string>
- name: <string>
  interval: <duration;optional>
  rules:
  - alert: <string>
      expr: <string>
      for: <duration>
      annotations:
      <annotation_name>: <string>
      labels:
      <label_name>: <string>
<namespace2>:
- name: <string>
  interval: <duration;optional>
  rules:
  - alert: <string>
      expr: <string>
      for: <duration>
      annotations:
      <annotation_name>: <string>
      labels:
      <label_name>: <string>
```

### 按命名空间获取规则组

```
GET /loki/api/v1/rules/{namespace}
```

返回给定命名空间定义的规则组。

这个实验性endpoint默认是禁用的，可以通过CLI参数`-experimental.ruler.enable-api` （或其相应的 YAML 配置选项）来启用。

#### 响应示例

```yaml
name: <string>
interval: <duration;optional>
rules:
  - alert: <string>
    expr: <string>
    for: <duration>
    annotations:
      <annotation_name>: <string>
    labels:
      <label_name>: <string>
```

### 获取规则组

```
GET /loki/api/v1/rules/{namespace}/{groupName}
```

返回与请求命名空间和组名匹配的规则组。

这个实验性endpoint默认是禁用的，可以通过CLI参数`-experimental.ruler.enable-api` （或其相应的 YAML 配置选项）来启用。

### 设置规则组

```
POST /loki/api/v1/rules/{namespace}
```

创建或更新规则组。此endpoint需要请求正文中包含`Content-Type: application/yaml`的标头和请求正文中用YAML定义的规则，并在成功时返回`202`。

这个实验性endpoint默认是禁用的，可以通过CLI参数`-experimental.ruler.enable-api` （或其相应的 YAML 配置选项）来启用。

#### 请求示例

请求头：

- `Content-Type: application/yaml`

请求正文：

```yaml
name: <string>
interval: <duration;optional>
rules:
  - alert: <string>
    expr: <string>
    for: <duration>
    annotations:
      <annotation_name>: <string>
    labels:
      <label_name>: <string>
```

### 删除规则组

```
DELETE /loki/api/v1/rules/{namespace}/{groupName}
```

按命名空间和组名删除规则组。这个endpoint在成功时返回`202`。

### 删除命名空间

```
DELETE /loki/api/v1/rules/{namespace}
```

删除命名空间中的所有规则组（包括命名空间本身）。这个endpoint在成功时返回`202`。

这个实验性endpoint默认是禁用的，可以通过CLI参数 `-experimental.ruler.enable-api` （或其相应的 YAML 配置选项）来启用。

### 列出规则

```
GET /prometheus/api/v1/rules
```

Prometheus 兼容的规则endpoint，用于列出当前加载的警报和记录规则。

有关更多信息，请参阅[Prometheus 规则](https://prometheus.io/docs/prometheus/latest/querying/api/#rules)文档。

这个实验性endpoint默认是禁用的，可以通过CLI参数`-experimental.ruler.enable-api`（或其相应的 YAML 配置选项）来启用。

### 列出警报

```
GET /prometheus/api/v1/alerts
```

Prometheus 兼容的规则endpoint，用于列出所有活动的警报。

有关更多信息，请查看 [Prometheus 警报](https://prometheus.io/docs/prometheus/latest/querying/api/#alerts)文档。

这个实验性endpoint默认是禁用的，可以通过CLI参数`-experimental.ruler.enable-api`（或其相应的 YAML 配置选项）来启用。