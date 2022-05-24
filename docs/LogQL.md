# LogQL：日志查询语言

Loki为查询提供了自己的PromQL风格的语言LogQL。LogQL 可以被认为是聚合日志源的分布式`grep`。LogQL 使用标签和运算符进行过滤。

LogQL 的查询有两种类型：

- 日志查询：返回日志行的内容。
- 指标查询：扩展了日志查询，并根据日志查询中的日志内容计算样本值。

## 日志查询

一个基本的日志查询由两部分组成：

- **日志流选择器**
- **日志管道**

由于 Loki 的设计，所有 LogQL 查询都必须包含一个**日志流选择器**。

日志流选择器确定将搜索多少个日志流（日志内容的唯一来源，例如文件）。然后更细粒度的日志流选择器将搜索流的数量减少到可管理的数量。这意味着传递给日志流选择器的标签将影响查询执行的相对性能。

**可选项**，日志流选择器后面可以跟一个**日志管道**。日志管道是一组链接在一起并应用于所选日志流的阶段表达式。每个表达式都可以过滤、解析和改变日志行及其各自的标签。

以下示例显示了一个完整的日志查询：

```logql
{container="query-frontend",namespace="tempo-dev"} |= "metrics.go" | logfmt | duration > 10s and throughput_mb < 500
```

查询由以下部分组成：

- 日志流选择器 `{container="query-frontend",namespace="loki-dev"}`，其目标是`tempo-dev`命名空间中的 `query-frontend`容器。
- 日志管道`|= "metrics.go" | logfmt | duration > 10s and throughput_mb < 500`，它将过滤出包含单词`metrics.go`的日志，然后解析每个日志行以提取更多标签并使用它们进行过滤。

> 为了避免转义特殊字符，您可以在引用字符串时使用（反勾号）而不是`"`。例如``\w+``与`"\\w+"`. 这在编写包含多个需要转义的反斜杠的正则表达式时特别有用。

### 日志流选择器

日志流选择器确定哪些日志流应该包含在查询结果中。流选择器由一个或多个键值对组成，其中每个键是一个**日志标签**，每个值是该**标签的值**。

日志流选择器是通过将键值对包装在一对花括号中来编写的：

```logql
{app="mysql",name="mysql-backup"}
```

在这个示例中，所有被包括在查询结果中的日志流都有一个值是`mysql`的标签`app`和值是`mysql-backup`的标签`name`。请注意，这将匹配标签至少包含`mysql-backup`名称标签的任何日志流；如果有多个流包含该标签，则所有匹配流的日志将显示在结果中。

标签名称后的`=`运算符是标签匹配运算符。支持以下标签匹配运算符：

- `=`: 完全一样。
- `!=`: 不相等。
- `=~`: 正则匹配。
- `!~`: 正则表达式不匹配。

例子：

- `{name=~"mysql.+"}`
- `{name!~"mysql.+"}`
- `{name!~`mysql-\d+`}`

适用于[Prometheus 标签选择器](https://prometheus.io/docs/prometheus/latest/querying/basics/#instant-vector-selectors)的相同规则也适用于 Loki 日志流选择器。

**重要提示：**`=~`正则表达式的操作是完全锚定的，这意味着正则表达式必须与整个字符串（包括换行符）匹配。默认情况下，正则表达式的`.`字符不匹配换行符。如果您希望正则表达式的`.`字符匹配换行符，您可以使用单行标志，如下所示：`(?s)search_term.+`匹配 `search_term\n`。

### 日志管道

日志管道可以附加到日志流选择器以进一步处理和过滤日志流。它通常由一个或多个表达式组成，每个表达式对每个日志行按顺序执行。如果表达式过滤掉了日志行，管道将在此时停止并开始处理下一行。

一些表达式可以改变日志内容和相应的标签（例如`| line_format "{{.status_code}}"`），然后这些标签可用于进一步过滤和处理以下表达式或指标查询。

日志管道可以由以下部分组成：

- [行过滤器表达式](#行过滤器表达式)。
- [解析器表达式](#解析器表达式)
- [标签过滤器表达式](#标签过滤器表达式)
- [行格式表达式](#行格式表达式)
- [标签格式表达式](#标签格式表达式)
- [展开表达式](https://grafana.com/docs/loki/v2.2.0/logql/#unwrapped-range-aggregations)

[展开表达式](https://grafana.com/docs/loki/v2.2.0/logql/#unwrapped-range-aggregations)是一个特殊的表达式，只应在指标查询中使用。

#### 行过滤器表达式

行过滤器表达式用于对来自匹配日志流的聚合日志进行分布式`grep`。

写入日志流选择器后，可以使用搜索表达式进一步过滤生成的日志集。搜索表达式可以只是文本或正则表达式：

- `{job="mysql"} |= "error"`
- `{name="kafka"} |~ "tsdb-ops.*io:2003"`
- `{name="cassandra"} |~ `error=\w+``
- `{instance=~"kafka-[23]",name="kafka"} != "kafka.server:type=ReplicaManager"`

在前面的示例中，`|=`，`|~`、 和`!=`充当**过滤器运算符**。支持以下过滤器运算符：

- `|=`: 日志行包含字符串。
- `!=`: 日志行不包含字符串。
- `|~`: 日志行匹配正则表达式。
- `!~`: 日志行与正则表达式不匹配。

过滤器操作符可以被链接起来并按顺序过滤表达式 - 生成的日志行必须满足*每个*过滤器：

```logql
{job="mysql"} |= "error" != "timeout"
```

使用`|~`and `!~`时，可以使用Go（如在[Golang 中](https://golang.org/)）[RE2 语法](https://github.com/google/re2/wiki/Syntax)正则表达式。默认情况下，匹配区分大小写，并且可以在正则表达式前加上`(?i)`切换为不区分大小写。

尽管可以将行过滤器表达式放置在管道中的任何位置，但将它们放在开头几乎总是更好。这种方式将提高查询的性能，仅在行匹配时进行进一步处理。

例如，虽然结果相同，但以下查询的`{job="mysql"} |= "error" | json | line_format "{{.err}}"`运行速度总是比  `{job="mysql"} | json | line_format "{{.message}}" |= "error"`快。行过滤器表达式是在日志流选择器之后过滤日志的最快方式。

#### 解析器表达式

解析器表达式可以从日志内容中解析和提取标签。然后，这些提取的标签可用于使用[标签过滤器表达式](#标签过滤器表达式)进行过滤或用于[指标聚合](#指标查询)。

提取的标签键由所有解析器自动清理，以遵循 Prometheus 指标名称约定。（它们只能包含 ASCII 字母和数字，以及下划线和冒号。它们不能以数字开头。）

例如，管道`| json`将生成以下映射：

```json
{ "a.b": {c: "d"}, e: "f" }
```

->

```
{a_b_c="d", e="f"}
```

如果出现错误，例如如果该行不是预期的格式，日志行将不会被过滤，而是会添加一个新标签`__error__`。

如果原始日志流中已经存在提取出的标签键名，则提取出的标签键后会加上`_extracted`关键字，以区分两个标签。您可以使用[标签格式化表达式](#标签格式化表达式)强制覆盖原始标签。但是，如果提取的键出现两次，则只会保留最新的标签值。

我们目前支持[json](#JSON) , [logfmt](#logfmt) , [regexp](#regexp)和[unpack](#unpack)解析器。

如果可以，使用`json`，`logfmt`等预定义的解析器会更容易，当日志行具有不寻常的结构时可以使用`regexp`日志行具有异常结构的时候。可以在同一个日志管道中使用多个解析器，这对于分析复杂日志非常有用（[见例子](https://grafana.com/docs/loki/v2.2.0/logql/#multiple-parsers)）。

##### JSON

**JSON**解析器有两种操作模式：

1. 不带参数：

   如果日志行是有效的 json 文档，则添加`| json`到您的管道将提取所有 json 属性作为标签。使用`_`分隔符将嵌套属性展平为标签键。

   注意：**跳过数组**。

   例如，json 解析器将从以下文档中提取：

   ```json
   {
       "protocol": "HTTP/2.0",
       "servers": ["129.0.1.1","10.2.1.3"],
       "request": {
           "time": "6.032",
           "method": "GET",
           "host": "foo.grafana.net",
           "size": "55",
           "headers": {
             "Accept": "*/*",
             "User-Agent": "curl/7.68.0"
           }
       },
       "response": {
           "status": 401,
           "size": "228",
           "latency_seconds": "6.031"
       }
   }
   ```

   以下是提取的标签列表：

   ```kv
"protocol" => "HTTP/2.0"
   "request_time" => "6.032"
   "request_method" => "GET"
   "request_host" => "foo.grafana.net"
   "request_size" => "55"
   "response_status" => "401"
   "response_size" => "228"
   "response_size" => "228"
   ```
   
2. 带参数：

   在您的管道中使用`| json label="expression", another="expression"`只会将指定的 json 字段提取到标签。您可以通过这种方式指定一个或多个表达式，与[`label_format`](#标签格式表达式)相似；所有表达式都必须加上引号。

   目前，我们仅支持字段访问 ( `my.field`, `my["field"]`) 和数组访问 ( `list[0]`)，以及在任何嵌套级别 ( `my.list[0]["field"]`) 中的这些的任意组合。

   例如，`| json first_server="servers[0]", ua="request.headers[\"User-Agent\"]`将从以下文档中提取：

   ```json
   {
       "protocol": "HTTP/2.0",
       "servers": ["129.0.1.1","10.2.1.3"],
       "request": {
           "time": "6.032",
           "method": "GET",
           "host": "foo.grafana.net",
           "size": "55",
           "headers": {
             "Accept": "*/*",
             "User-Agent": "curl/7.68.0"
           }
       },
       "response": {
           "status": 401,
           "size": "228",
           "latency_seconds": "6.031"
       }
   }
   ```

   以下是提取的标签列表：

   ```kv
"first_server" => "129.0.1.1"
   "ua" => "curl/7.68.0"
   ```
   
   如果是数组或表达式返回的对象，则会以json格式分配给标签。

   例如，`| json server_list="servers", headers="request.headers`将提取：

   ```kv
"server_list" => `["129.0.1.1","10.2.1.3"]`
   "headers" => `{"Accept": "*/*", "User-Agent": "curl/7.68.0"}`
   ```
```
   

##### logfmt

可以使用`| logfmt`添加**logfmt**解析器，并且将从[logfmt](https://brandur.org/logfmt)格式的日志行提取所有的键和值。

例如以下日志行：

​```logfmt
at=info method=GET path=/ host=grafana.net fwd="124.133.124.161" connect=4ms service=8ms status=200
```

将提取这些标签：

```kv
"at" => "info"
"method" => "GET"
"path" => "/"
"host" => "grafana.net"
"fwd" => "124.133.124.161"
"service" => "8ms"
"status" => "200"
```

##### regexp

与 logfmt 和 json 不同，regexp解析器隐式提取所有值且不接收参数，**regexp**解析器采用单个参数`| regexp "<re>"`，即使用[Golang ](https://golang.org/)[RE2 语法](https://github.com/google/re2/wiki/Syntax)的正则表达式。

正则表达式必须包含至少一个命名子匹配（例如`(?P<name>re)`），每个子匹配将提取不同的标签。

例如，解析器`| regexp "(?P<method>\\w+) (?P<path>[\\w|/]+) \\((?P<status>\\d+?)\\) (?P<duration>.*)"`将从以下行中提取：

```log
POST /api/prom/api/v1/query_range (200) 1.5s
```

提取出的标签：

```kv
"method" => "POST"
"path" => "/api/prom/api/v1/query_range"
"status" => "200"
"duration" => "1.5s"
```

##### unpack

`unpack`解析器将分析JSON日志行，并通过[`pack`](https://grafana.com/docs/loki/v2.2.0/clients/promtail/stages/pack/)阶段解包所有嵌入的标签。 特殊属性`_entry`被用来替换原始日志行。

例如，在以下日志行使用`| unpack`：

```json
{
  "container": "myapp",
  "pod": "pod-3223f",
  "_entry": "original log message"
}
```

允许提取`container`和`pod`标签以及`original log message`，作为新的日志行。

> 如果原始嵌入的日志行是特定格式，您可以将`unpack`与`json`解析器（或任何其他解析器）结合使用。

#### 标签过滤器表达式

标签过滤器表达式允许使用原始标签和提取的标签过滤日志行。它可以包含多个谓语。

谓语包含**标签标识符**、**操作**和与标签进行比较的**值**。

例如使用`cluster="namespace"`时，集群是标签标识符，操作是`=`，值是“namespace”。标签标识符始终位于操作的右侧。

我们支持从查询输入中自动推断出的多种**值**类型。

- **String** 是双引号或反引号，例如`"200"`或``us-central1``。
- **[Duration](https://golang.org/pkg/time/#ParseDuration)** 是一串十进制数字，每个数字都有可选的分数和单位后缀，例如“300ms”、“1.5h”或“2h45m”。有效时间单位为“ns”、“us”（或“µs”）、“ms”、“s”、“m”、“h”。
- **Number **是（64 位）浮点数，如`250`, `89.923`。
- **Bytes** 是十进制数的序列，每个数都有可选的小数和单位后缀，例如“42MB”、“1.5Kib”或“20b”。有效字节单位为“b”、“kib”、“kb”、“mib”、“mb”、“gib”、“gb”、“tib”、“tb”、“pib”、“pb”、“eib” ”、“eb”。

字符串类型的工作方式与 Prometheus 标签匹配器在[日志流选择](https://grafana.com/docs/loki/v2.2.0/logql/#log-stream-selector)器中使用的完全一样。这意味着您可以使用相同的操作（`=`、`!=`、`=~`、`!~`）。

> 字符串类型是唯一可以过滤出带有`__error__`标签的日志行的类型。

使用 Duration、Number 和 Bytes 将在比较之前转换标签值，并支持以下比较器：

- `==`或`=` ：相等。
- `!=` ：不平等。
- `>`和`>=`：大于和大于或等于。
- `<`和`<=`：小于和小于或等于。

例如， `logfmt | duration > 1m and bytes_consumed > 20MB`

如果标签值转换失败，则不过滤日志行并添加`__error__`标签。要过滤这些错误，请参阅[管道错误](#管道错误)部分。

您可以使用`and`和`or`链接多个谓词，分别表示`and`和`or`的二进制操作。`and`可以用逗号、空格或其他管道等价表示。标签过滤器可以放置在日志管道中的任何位置。

这意味着以下所有表达式都是等价的：

```logql
| duration >= 20ms or size == 20kb and method!~"2.."
| duration >= 20ms or size == 20kb | method!~"2.."
| duration >= 20ms or size == 20kb , method!~"2.."
| duration >= 20ms or size == 20kb  method!~"2.."
```

默认情况下，多个谓词的优先级是从右到左。您可以用括号将谓词括起来以强制从左到右具有不同的优先级。

例如，以下是等价的：

```logql
| duration >= 20ms or method="GET" and size <= 20KB
| ((duration >= 20ms or method="GET") and size <= 20KB)
```

它将先评估`duration >= 20ms or method="GET"`。要先评估`method="GET" and size <= 20KB`，请确保使用正确的括号，如下所示。

```logql
| duration >= 20ms or (method="GET" and size <= 20KB)
```

日志

> 标签过滤器表达式是[展开表达式](https://grafana.com/docs/loki/v2.2.0/logql/#unwrapped-range-aggregations)之后唯一允许的[表达式](https://grafana.com/docs/loki/v2.2.0/logql/#unwrapped-range-aggregations)。这主要是为了允许从指标中过滤错误（请参阅[错误](https://grafana.com/docs/loki/v2.2.0/logql/#pipeline-errors)）。

#### 行格式表达式

行格式表达式可以使用[文本/模板](https://golang.org/pkg/text/template/)格式重写日志行内容。它采用单个字符串参数`| line_format "{{.label_name}}"`，即模板格式。所有标签都被注入到模板中，并且可以与`{{.label_name}}`符号一起使用。

例如下面的表达式：

```logql
{container="frontend"} | logfmt | line_format "{{.query}} {{.duration}}"
```

将提取并重写日志行以使其仅包含查询和请求的持续时间。

您可以使用双引号或反引号字符串作为模板，以避免转义特殊字符。

请参阅[模板函数](https://grafana.com/docs/loki/v2.2.0/logql/template_functions/)以了解模板格式中的可用函数。

#### 标签格式表达式

`| label_format`表达式可以重命名、修改或添加标签。它以逗号分隔的相等操作列表作为参数，同时启用多个操作。

例如`dst=src`，当双方都是标签标识符时，操作会将`src`标签重命名为`dst`.

左侧也可以是模板字符串（双引号或反引号），例如`dst="{{.status}} {{.query}}"`，在这种情况下，`dst`标签值将替换为[文本/模板](https://golang.org/pkg/text/template/)评估的结果。这是与`| line_format`表达式相同的模板引擎，这意味着标签可用作变量，您可以使用相同的[函数](https://grafana.com/docs/loki/v2.2.0/logql/functions/)列表。

在这两种情况下，如果目标标签不存在，则会创建一个新标签。

重命名表单`dst=src`将在将其重新映射到`dst`标签后删除`src`标签。但是，模板表单将保留引用的标签，这样 `dst="{{.src}}"`的结果在`dst`和`src`中具有相同的值。

> 单个标签名称在每个表达式中只能出现一次。这意味着不允许使用`| label_format foo=bar,foo="new"`，但您可以使用两个表达式来达到所需的效果：`| label_format foo=bar | label_format foo="new"`

### 日志查询示例

#### 多重过滤

首先使用标签匹配器进行过滤，然后使用行过滤器（如果可能），最后使用标签过滤器。以下查询证明了这一点。

```logql
{cluster="ops-tools1", namespace="loki-dev", job="loki-dev/query-frontend"} |= "metrics.go" !="out of order" | logfmt | duration > 30s or status_code!="200"
```

#### 多个解析器

提取以下 logfmt 日志行的方法和路径：

```log
level=debug ts=2020-10-02T10:10:42.092268913Z caller=logging.go:66 traceID=a9d4d8a928d8db1 msg="POST /api/prom/api/v1/query_range (200) 1.5s"
```

您可以像这样使用多个解析器（logfmt 和 regexp）。

```logql
{job="cortex-ops/query-frontend"} | logfmt | line_format "{{.msg}}" | regexp "(?P<method>\\w+) (?P<path>[\\w|/]+) \\((?P<status>\\d+?)\\) (?P<duration>.*)"`
```

这是可能的，因为`| line_format`将日志行重新格式化为`POST /api/prom/api/v1/query_range (200) 1.5s`，然后可以用`| regexp ...`解析器解析它。

#### 格式化

以下的查询显示了如何重新格式化日志行以使其更易于在屏幕上阅读。

```logql
{cluster="ops-tools1", name="querier", namespace="loki-dev"}
  |= "metrics.go" != "loki-canary"
  | logfmt
  | query != ""
  | label_format query="{{ Replace .query \"\\n\" \"\" -1 }}"
  | line_format "{{ .ts}}\t{{.duration}}\ttraceID = {{.traceID}}\t{{ printf \"%-100.100s\" .query }} "
```

标签格式用于清理查询，而行格式可减少信息量并创建表格输出。

对于那些给定的日志行：

```log
level=info ts=2020-10-23T20:32:18.094668233Z caller=metrics.go:81 org_id=29 traceID=1980d41501b57b68 latency=fast query="{cluster=\"ops-tools1\", job=\"cortex-ops/query-frontend\"} |= \"query_range\"" query_type=filter range_type=range length=15m0s step=7s duration=650.22401ms status=200 throughput_mb=1.529717 total_bytes_mb=0.994659
level=info ts=2020-10-23T20:32:18.068866235Z caller=metrics.go:81 org_id=29 traceID=1980d41501b57b68 latency=fast query="{cluster=\"ops-tools1\", job=\"cortex-ops/query-frontend\"} |= \"query_range\"" query_type=filter range_type=range length=15m0s step=7s duration=624.008132ms status=200 throughput_mb=0.693449 total_bytes_mb=0.432718
```

结果将是：

```log
2020-10-23T20:32:18.094668233Z	650.22401ms	    traceID = 1980d41501b57b68	{cluster="ops-tools1", job="cortex-ops/query-frontend"} |= "query_range"
2020-10-23T20:32:18.068866235Z	624.008132ms	traceID = 1980d41501b57b68	{cluster="ops-tools1", job="cortex-ops/query-frontend"} |= "query_range"
```

## 指标查询

LogQL 还支持使用允许从日志中创建指标的函数包装日志查询。

指标查询可用于计算错误消息率或过去 3 小时内日志量最多的前 N 个日志源等内容。

结合日志[解析器](#解析器表达式)，指标查询还可用于从日志行内的样本值计算指标，例如延迟或请求大小。此外，所有标签，包括提取的标签，都可用于聚合和生成新系列。

### 范围向量聚合

LogQL与 Prometheus共享相同的[范围向量](https://prometheus.io/docs/prometheus/latest/querying/basics/#range-vector-selectors)概念，但所选样本范围是所选日志或标签值的范围。

Loki 支持两种类型的范围聚合。日志范围聚合和展开的范围聚合。

#### 日志范围聚合

日志范围是一个日志查询（有或没有日志管道）后跟范围符号，例如 [1m]。应该注意的是，范围符号`[5m]`可以放在日志管道的末尾或日志流匹配器之后。

日志范围聚合使用日志条目来计算值，支持的操作函数有：

- `rate(log-range)`：计算每秒的条目数
- `count_over_time(log-range)`：计算给定范围内每个日志流的条目。
- `bytes_rate(log-range)`：计算每个流每秒的字节数。
- `bytes_over_time(log-range)`：计算给定范围内每个日志流使用的字节数。
- `absent_over_time(log-range)`：如果传递给它的范围向量有任何元素，则返回一个空向量，如果传递给它的范围向量没有元素，则返回一个值为 1 的一元向量。（`absent_over_time`对于在一定时间内没有标签组合的时间序列和日志流存在时发出警报非常有用。）

##### 日志示例

```logql
count_over_time({job="mysql"}[5m])
```

此示例计算 MySQL 作业最近五分钟内的所有日志行数。

```logql
sum by (host) (rate({job="mysql"} |= "error" != "timeout" | json | duration > 10s [1m]))
```

此示例演示了一个包含过滤器和解析器的 LogQL 聚合。它返回 MySQL 作业在每台主机最后几分钟内每秒发生的所有非超时错误的数目，并且仅包括持续时间超过 10 秒的错误。

#### 展开的范围聚合

展开的范围聚合使用提取的标签作为样本值而不是日志行。但是，要选择在聚合中使用哪个标签，日志查询必须以展开表达式和可选的标签过滤器表达式结束来丢弃[错误](https://grafana.com/docs/loki/v2.2.0/logql/#pipeline-errors)。

展开表达式被标注为`| unwrap label_identifier`，其中标签标识符是用于提取样本值的标签名称。

由于标签值是字符串，默认情况下将尝试转换为浮点数（64 位），万一失败，则会将`__error__`标签添加到样本中。可选地，标签标识符可以由转换函数包装，函数`| unwrap <function>(label_identifier)`将尝试从特定格式转换标签值。

我们目前支持以下功能：

- `duration_seconds(label_identifier)`（或其短等效项`duration`），它将以秒为单位从[go 持续时间格式](https://golang.org/pkg/time/#ParseDuration)（例如`5m`，`24s30ms`）转换标签值。
- `bytes(label_identifier)`它将应标签值转换为原始字节应用到字节单元（例如`5 MiB`, `3k`, `1G`）。

支持在展开范围内操作的函数有：

- `rate(unwrapped-range)`：计算指定时间间隔内所有值的每秒速率。
- `sum_over_time(unwrapped-range)`：指定区间内所有值的总和。
- `avg_over_time(unwrapped-range)`：指定区间内所有点的平均值。
- `max_over_time(unwrapped-range)`：指定区间内所有点的最大值。
- `min_over_time(unwrapped-range)`：指定区间内所有点的最小值
- `stdvar_over_time(unwrapped-range)`：指定区间内值的总体标准方差。
- `stddev_over_time(unwrapped-range)`：指定区间内值的总体标准差。
- `quantile_over_time(scalar,unwrapped-range)`：指定区间内值的 φ 分位数 (0 ≤ φ ≤ 1)。
- `absent_over_time(unwrapped-range)`：如果传递给它的范围向量有任何元素，则返回一个空向量，如果传递给它的范围向量没有元素，则返回一个值为 1 的 一元向量。（`absent_over_time`对于在一定时间内没有标签组合的时间序列和日志流存在时发出警报非常有用。）

除了`sum_over_time`,`absent_over_time`和`rate`，展开的范围聚合支持分组。

```logql
<aggr-op>([parameter,] <unwrapped-range>) [without|by (<label list>)]
```

通过包含`without`或`by`子句，可用于聚合不同的标签维度。

`without`从结果向量中删除列出的标签，而所有其他标签都保留在输出中。`by`执行相反的操作，删除`by`子句中未列出的标签，即使它们的标签值在向量的所有元素之间都相同。

#### 展开的范围聚合示例

```logql
quantile_over_time(0.99,
  {cluster="ops-tools1",container="ingress-nginx"}
    | json
    | __error__ = ""
    | unwrap request_time [1m])) by (path)
```

此示例按路径计算 nginx-ingress 延迟的 99%。

```logql
sum by (org_id) (
  sum_over_time(
  {cluster="ops-tools1",container="loki-dev"}
      |= "metrics.go"
      | logfmt
      | unwrap bytes_processed [1m])
  )
```

这将计算每个组织 ID 处理的字节数。

### 聚合运算符

与[PromQL](https://prometheus.io/docs/prometheus/latest/querying/operators/#aggregation-operators)一样，LogQL 支持内置聚合运算符的子集，这些运算符可用于聚合单个向量的元素，从而生成一个元素较少但具有聚合值的新向量：

- `sum`：计算标签的总和
- `min`：在标签上选择最小值
- `max`：在标签上选择最大值
- `avg`：计算标签的平均值
- `stddev`：计算标签上的总体标准差
- `stdvar`：计算标签上的总体标准方差
- `count`：计算向量中元素的数量
- `bottomk`：通过样本值选择最小的k个元素
- `topk`：按样本值选择最大的k个元素

聚合运算符可用于聚合所有标签值或通过包含一个`without`或一个`by`子句来聚合一组不同的标签值：

```logql
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

`parameter`仅在使用`topk`和`bottomk`时才需要。 `topk`和`bottomk`与其他聚合器的不同之处在于输入样本的子集（包括原始标签）在结果向量中返回。

`by`和`without`仅用于对输入向量进行分组。`without`子句从结果向量中删除列出的标签，保留所有其他标签。`by`子句执行相反的操作，删除子句中未列出的标签，即使它们的标签值在向量的所有元素之间都相同。

#### 向量聚合示例

获取日志吞吐量最高的前 10 个应用程序：

```logql
topk(10,sum(rate({region="us-east1"}[5m])) by (name))
```

获取过去五分钟的日志计数，按级别分组：

```logql
sum(count_over_time({job="mysql"}[5m])) by (level)
```

按区域从 NGINX 日志中获取 /home 请求的 HTTP GET 速率：

```logql
avg(rate(({job="nginx"} |= "GET" | json | path="/home")[10s])) by (region)
```

### 函数

Loki 支持多种函数来操作数据。这些在表达式语言[函数](https://grafana.com/docs/loki/v2.2.0/logql/functions/)页面中有详细描述。

### 二元运算符

#### 算术运算符

Loki 中存在以下二元算术运算符：

- `+` （加）
- `-` （法）
- `*` （乘）
- `/` （除）
- `%` （模）
- `^` （幂）

二元算术运算符在两个数值（标量）、一个数值和一个向量，以及两个向量之间定义。

在两个数值之间，行为很明显：它们计算除另一个数值，该数值是应用于两个标量操作数 ( `1 + 1 = 2`)的运算符的结果。

在向量和数值之间，运算符应用于向量中每个数据样本的值，例如，如果时间序列向量乘以 2，则结果是另一个向量，其中原始向量的每个样本值都乘以2.

在两个向量之间，二元算术运算符应用于左侧向量中的每个条目及其右侧向量中的匹配元素。结果传播到结果向量中，分组标签成为输出标签集。在右侧向量中找不到匹配条目的条目不是结果的一部分。

链接算术运算符时要特别注意[运算符顺序](https://grafana.com/docs/loki/v2.2.0/logql/#operator-order)。

##### 算术示例

使用简单的查询实现健康检查：

```logql
1 + 1
```

将日志流条目的速率加倍：

```logql
sum(rate({app="foo"})) * 2
```

获取`foo`应用程序的警告日志与错误日志的比例

```logql
sum(rate({app="foo", level="warn"}[1m])) / sum(rate({app="foo", level="error"}[1m]))
```

#### 逻辑/段运算符

这些逻辑/集合二元运算符仅在两个向量之间定义：

- `and` （交集）
- `or` （并集）
- `unless` （非集）

`vector1 and vector2` 结果是一个由 vector1 的元素组成的向量，其中 vector2 中的元素具有完全匹配的标签集。其他元素被丢弃。

`vector1 or vector2`  生成一个包含 vector1 的所有原始元素（标签集 + 值）以及 vector2 中所有在 vector1 中没有匹配标签集的元素的向量。

`vector1 unless vector2` 结果是一个由 vector1 的元素组成的向量，其中 vector2 中没有具有完全匹配标签集的元素。两个向量中的所有匹配元素都被删除。

##### 二元运算符示例

这查询将有效地返回这些查询的交集`rate({app="bar"})`：

```logql
rate({app=~"foo|bar"}[1m]) and rate({app="bar"}[1m])
```

#### 比较运算符

- `==` （等于）
- `!=` （不等于）
- `>` （大于）
- `>=` （大于或等于）
- `<` （小于）
- `<=` （小于或等于）

比较运算符在标量/标量、向量/标量和向量/向量值对之间定义。默认情况下，它们会过滤。它们的行为可以通过在运算符之后提供`bool`来修改，该运算符的返回值是 0 或 1，而不是过滤。

在两个标量之间，根据比较结果，这些运算符会产生另一个为 0（假）或 1（真）的标量。不能提供`bool`修饰符。

`1 >= 1` 的结果是 `1`

在向量和标量之间，这些运算符应用于向量中每个数据样本的值，比较结果为false的向量元素将从结果向量中删除。如果提供了`bool`修饰符，则将被删除的向量元素的值为 0，而将保留的向量元素的值为 1。

过滤在最后一分钟记录至少 10 行的流：

```logql
count_over_time({foo="bar"}[1m]) > 10
```

将 `0`/`1`附加到记录少于/多于 10 行的流：

```logql
count_over_time({foo="bar"}[1m]) > bool 10
```

在两个向量之间，这些运算符默认充当过滤器，应用于匹配条目。表达式非真或在表达式的另一侧找不到匹配项的向量元素将从结果中删除，而其他元素则传播到结果向量中。如果提供了`bool`修饰符，则将被删除的向量元素的值为 0，而将保留的向量元素的值为 1，分组标签再次成为输出标签集。

返回最后一分钟内匹配`app=foo`但没有app标签的流，并且这些流在最后一分钟内的计数高于匹配`app=bar`但没有app标签的流：

```logql
sum without(app) (count_over_time({app="foo"}[1m])) > sum without(app) (count_over_time({app="bar"}[1m]))
```

与上面相同，但是如果向量通过比较，则将其值设置为`1`；如果比较失败则将其值设置为`0`，并且会被过滤掉：

```logql
sum without(app) (count_over_time({app="foo"}[1m])) > bool sum without(app) (count_over_time({app="bar"}[1m]))
```

#### 操作员顺序

在链接或组合运算符时，您必须考虑运算符优先级：通常，您可以假设常规[数学约定](https://en.wikipedia.org/wiki/Order_of_operations)与相同优先级的运算符是左结合的。

更多细节可以在[Golang 语言文档中找到](https://golang.org/ref/spec#Operator_precedence)。

`1 + 2 / 3`等于`1 + ( 2 / 3 )`。

`2 * 3 % 2`等于`(2 * 3) % 2`。

### 注释

使用`#`字符可以将LogQL 查询注释：

```logql
{app="foo"} # anything that comes after will not be interpreted in your query
```

对于多行 LogQL 查询，查询解析器可以使用`#`来排除整行或部分行：

```logql
{app="foo"}
    | json
    # this line will be ignored
    | bar="baz" # this checks if bar = "baz"
```

### 管道错误

导致管道处理错误的原因有多种，例如：

- 数字标签过滤器可能无法将标签值转换为数字
- 标签的度量转换可能会失败。
- 日志行不是有效的 json 文本。
- 等等…

当这些错误发生时，Loki 不会过滤掉这些日志行。相反，它们通过名为`__error__`的新系统标签传递到管道的下一阶段。过滤错误的唯一方法是使用标签过滤器表达式。该`__error__`标签不能通过语言来重命名。

例如要删除 json 错误：

```logql
  {cluster="ops-tools1",container="ingress-nginx"}
    | json
    | __error__ != "JSONParserErr"
```

或者，您可以使用抓取所有匹配器（例如`__error__ = ""`）删除所有错误，或者使用`__error__ != ""`仅显示错误。

过滤器应放置在产生此错误的阶段之后。这意味着如果您需要从展开表达式中删除错误，则需要将其放置在展开之后。

```logql
quantile_over_time(
	0.99,
	{container="ingress-nginx",service="hosted-grafana"}
	| json
	| unwrap response_latency_seconds
	| __error__=""[1m]
	) by (cluster)
```

> 指标查询不能包含错误，如果在执行过程中发现错误，Loki 将返回错误和相应的状态代码。



# 函数

## label_replace()

对于`v`中的每个时间序列，`label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)`将使用正则表达式`regex`再次与标签`src_label`进行匹配。如果匹配，则返回时间序列，其中标签`dst_label`替换为`replacement`的扩展。`$1`被替换为第一个匹配的子组，`$2`被替换为第二个匹配的子组，以此类推。如果正则表达式不匹配，则返回不变的时间序列。

此示例将返回一个向量，其中每个时间序列都有一个添加了`a`的`foo`标签：

```logql
label_replace(rate({job="api-server",service="a:c"} |= "err" [1m]), "foo", "$1", "service", "(.*):.*")
```



# 模板函数

`| line_format`和`| label_format` 中使用的[文本模板](https://golang.org/pkg/text/template)格式支持函数的使用。

所有标签都作为变量添加到模板引擎中。可以使用以`.`为前缀的标签名称来引用它们（例如`.label_name`）。例如，以下模板将输出路径标签的值：

```template
{{ .path }}
```

您可以利用[管道](https://golang.org/pkg/text/template/#hdr-Pipelines)将多个函数连接在一起。在链式管道中，每个命令的结果都作为下一个命令的最后一个参数传递。

例子：

```template
{{ .path | replace " " "_" | trunc 5 | upper }}
```

## ToLower 和 ToUpper

此函数将整个字符串转换为小写或大写。

- `ToLower(string) string`
- `ToUpper(string) string`

示例：

```template
"{{.request_method | ToLower}}"
"{{.request_method | ToUpper}}"
`{{ToUpper "This is a string" | ToLower}}`
```

> **注意：**在 Loki 2.1 中，您还可以分别使用[`lower`](https://grafana.com/docs/loki/v2.2.0/logql/template_functions/#lower)和[`upper`](https://grafana.com/docs/loki/v2.2.0/logql/template_functions/#upper) 短格式，例如`{{.request_method | lower }}`。

## 替换字符串

> **注意：**在 Loki 2.1 中[`replace`](https://grafana.com/docs/loki/v2.2.0/logql/template_functions/#replace)（与`Replace`相对）具有不同的语法，但更容易在管道中链接。

使用此函数执行简单的字符串替换。

```
Replace(s, old, new string, n int) string
```

它需要四个参数：

- `s`：源字符串
- `old`：要替换的字符串
- `new`：要替换的字符串
- `n`：最大替换量（-1 为所有）

示例：

```template
`{{ Replace "This is a string" " " "-" -1 }}`
```

结果为：`This-is-a-string`.

## Trim、TrimLeft、TrimRight 和 TrimSpace

> **注意：**在 Loki 2.1中， [trim](https://grafana.com/docs/loki/v2.2.0/logql/template_functions/#trim)、[trimAll](https://grafana.com/docs/loki/v2.2.0/logql/template_functions/#trimAll)、[trimSuffix](https://grafana.com/docs/loki/v2.2.0/logql/template_functions/#trimSuffix)和[trimPrefix](https://grafana.com/docs/loki/v2.2.0/logql/template_functions/trimPrefix)已添加不同的签名以实现更好的管道链接。

`Trim` 返回删除了 cutset 中包含的所有前导和尾随 Unicode 代码点的字符串 s 的切片。

签名： `Trim(value, cutset string) string`

`TrimLeft`和`TrimRight`与`Trim`相同，但是它只修整前导和尾随的字符。

```template
`{{ Trim .query ",. " }}`
`{{ TrimLeft .uri ":" }}`
`{{ TrimRight .path "/" }}`
```

`TrimSpace` TrimSpace 返回删除所有前导和尾随空格的字符串 s，如 Unicode 定义的那样。

签名： `TrimSpace(value string) string`

```template
{{ TrimSpace .latency }}
```

`TrimPrefix`和`TrimSuffix`分别修剪提供的前缀或后缀。

签名：

- `TrimPrefix(value string, prefix string) string`
- `TrimSuffix(value string, suffix string) string`

```template
{{ TrimPrefix .path "/" }}
```

## regexReplaceAll 和 regexReplaceAllLiteral

`regexReplaceAll`返回输入字符串的副本，用替换字符串替换替换正则表达式的匹配项。在字符串替换中，$ 符号被解释为在 Expand 中，因此例如 $1 表示第一个子匹配的文本。有关更多示例，请参阅 golang [Regexp.replaceAll 文档](https://golang.org/pkg/regexp/#Regexp.ReplaceAll)。

```template
`{{ regexReplaceAllLiteral "(a*)bc" .some_label "${1}a" }}`
```

`regexReplaceAllLiteral`函数返回输入字符串的副本，并用替换字符串替换替换正则表达式的匹配项。替换字符串被直接替换，不使用Expand。

```template
`{{ regexReplaceAllLiteral "(ts=)" .timestamp "timestamp=" }}`
```

您可以使用管道组合多个函数。例如，要去掉空格并使请求方法大写，您可以编写以下模板：`{{ .request_method | TrimSpace | ToUpper }}`.

## lower

> 在 Loki 2.1 中添加

使用此函数可转换为小写。

签名：

```
lower(string) string
```

例子：

```template
"{{ .request_method | lower }}"
`{{ lower  "HELLO"}}`
```

最后一个示例将返回`hello`.

## upper

> 在 Loki 2.1 中添加

使用此函数转换为大写。

签名：

```
upper(string) string
```

例子：

```template
"{{ .request_method | upper }}"
`{{ upper  "hello"}}`
```

这导致`HELLO`.

## title

> **注意：**在 Loki 2.1 中添加。

转换为标题式的大小写。

签名：

```
title(string) string
```

例子：

```template
"{{.request_method | title}}"
`{{ title "hello world"}}`
```

最后一个示例将返回`Hello World`.

## trunc

> **注意：**在 Loki 2.1 中添加。

截断字符串并且不添加后缀。

签名：

```
trunc(count int,value string) string
```

例子：

```template
"{{ .path | trunc 2 }}"
`{{ trunc 5 "hello world"}}`   // output: hello
`{{ trunc -5 "hello world"}}`  // output: world
```

## substr

> **注意：**在 Loki 2.1 中添加。

从字符串中获取子字符串。

签名：

```
substr(start int,end int,value string) string
```

如果 start < 0，则调用 value[:end]。如果 start >= 0 且 end < 0 或 end 大于 s 长度，则调用 value[start:] 否则，调用 value[start, end]。

例子：

```template
"{{ .path | substr 2 5 }}"
`{{ substr 0 5 "hello world"}}`  // output: hello
`{{ substr 6 11 "hello world"}}` // output: world
```

## replace

> **注意：**在 Loki 2.1 中添加。

此函数执行简单的字符串替换。

签名： `replace(old string, new string, src string) string`

它需要三个参数：

- `old`：要替换的字符串
- `new`：要替换的字符串
- `src`：源字符串

例子：

```template
{{ .cluster | replace "-cluster" "" }}
{{ replace "hello" "world" "hello world" }}
```

最后一个示例将返回`world world`.

## trim

> **注意：**在 Loki 2.1 中添加。

修剪函数从字符串的两侧删除空格。

签名： `trim(string) string`

例子：

```template
{{ .ip | trim }}
{{ trim "   hello    " }} // output: hello
```

## trimAll

> **注意：**在 Loki 2.1 中添加。

使用此函数从字符串的前面或后面删除给定的字符。

签名： `trimAll(chars string,src string) string`

例子：

```template
{{ .path | trimAll "/" }}
{{ trimAll "$" "$5.00" }} // output: 5.00
```

## trimSuffix

> **注意：**在 Loki 2.1 中添加。

使用此函数仅修剪字符串中的后缀。

签名： `trimSuffix(suffix string, src string) string`

例子：

```template
{{  .path | trimSuffix "/" }}
{{ trimSuffix "-" "hello-" }} // output: hello
```

## trimPrefix

> **注意：**在 Loki 2.1 中添加。

使用此函数仅修剪字符串中的前缀。

签名： `trimPrefix(suffix string, src string) string`

例子：

```template
{{  .path | trimPrefix "/" }}
{{ trimPrefix "-" "-hello" }} // output: hello
```

## indent

> **注意：**在 Loki 2.1 中添加。

indent 函数将给定字符串中的每一行缩进到指定的缩进宽度。这在对齐多行字符串时很有用。

签名： `indent(spaces int,src string) string`

```template
{{ indent 4 .query }}
```

这会将包含在`.query`中的每一行缩进 4 个空格。

## nindent

> **注意：**在 Loki 2.1 中添加。

nindent 函数与 indent 函数相同，但在字符串的开头添加了一个新行。

签名： `nindent(spaces int,src string) string`

```template
{{ nindent 4 .query }}
```

这将使每行文本缩进 4 个空格字符，并在开头添加一个新行。

## repeat

> **注意：**在 Loki 2.1 中添加。

使用此函数可多次重复一个字符串。

签名： `repeat(c int,value string) string`

```template
{{ repeat 3 "hello" }} // output: hellohellohello
```

## contains

> **注意：**在 Loki 2.1 中添加。

使用此函数来测试一个字符串是否包含在另一个字符串中。

签名： `contains(s string, src string) bool`

例子：

```template
{{ if .err contains "ErrTimeout" }} timeout {{end}}
{{ if contains "he" "hello" }} yes {{end}}
```

## hasPrefix 和 hasSuffix

> **注意：**在 Loki 2.1 中添加。

`hasPrefix`和`hasSuffix`功能测试字符串是否具有给定的前缀或后缀。

签名：

- `hasPrefix(prefix string, src string) bool`
- `hasSuffix(suffix string, src string) bool`

例子：

```template
{{ if .err hasSuffix "Timeout" }} timeout {{end}}
{{ if hasPrefix "he" "hello" }} yes {{end}}
```

