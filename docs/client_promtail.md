# Promtail

Promtail是一个代理，它将本地日志的内容发送到私有Loki实例或 [Grafana Cloud](https://grafana.com/oss/loki)上。它通常部署到每台需要监控应用程序的机器上。

它主要包括：

- 发现 target
- 将标签附加到日志流
- 将它们推送到 Loki 实例。

目前，Promtail 可以从两个来源跟踪日志：本地日志文件和 systemd 日志（仅在 AMD64 机器上）。

## 日志文件发现

在 Promtail 将任何数据从日志文件发送到Loki之前，它需要收集有关其环境的信息。具体来说就是发现需要被监控的发送日志的应用程序。

Promtail 借鉴了[Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)的 [服务发现机制](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)，尽管它目前仅支持`static`和`kubernetes`服务发现。这种限制是由于`promtail`作为守护进程部署到每个本地机器上，因此不会从其他机器上发现标签。`kubernetes`服务发现从 Kubernetes API 服务器获取所需的标签，同时`static`通常覆盖所有其他用例。

和 Prometheus 一样，`promtail`使用`scrape_configs`进行配置。 `relabel_configs`允许对接收、删除以及附加到日志行的最终元数据进行细粒度控制。有关更多详细信息，请参阅[配置 Promtail](clients/promtail/configuration/)的文档 。

## Loki的推送API

通过使用[loki_push_api](clients/promtail/configuration#loki_push_api_config)来抓取配置公开的[Loki Push API](https://grafana.com/docs/loki/latest/api#post-lokiapiv1push)，Promtail 还可以配置为从另一个 Promtail 或任何 Loki 客户端接收日志。

在某些情况下，这可能会有所帮助：

- 在复杂的网络基础设施中，其中许多机器不可达。
- 使用 Docker 日志驱动程序并希望提供复杂的管道或从日志中提取指标。
- 许多serverless状态的临时日志源想要发送到 Loki，可使用`use_incoming_timestamp`== false发送到 Promtail 实例可以避免出现乱序错误并避免必须使用高基数标签。

## 从Syslog接收日志

当使用 [Syslog Target](https://grafana.com/docs/loki/latest/clients/promtail/configuration#syslog_config) 时，可以使用 syslog 协议将日志写入指定的端口。

## AWS

如果您需要在 Amazon Web Services EC2 实例上运行 Promtail，您可以查看我们的[详细教程](clients/aws/ec2/)。

## 标记和解析

在服务发现期间，确定元数据（pod 名称、文件名等），这些元数据可以作为标签附加到日志行，以便在 Loki 中查询日志时更容易识别。通过`relabel_configs`，可以将发现的标签变为所需的形式。

为了之后进行更复杂的过滤，Promtail 不仅允许从服务发现中设置标签，还允许基于每个日志行的内容设置标签。`pipeline_stages`可用于添加或更新标签，纠正时间戳，或完全重新写日志行。有关更多详细信息，请参阅[管道](clients/promtail/pipelines/)的文档 。

## Shipping

一旦 Promtail 有一组目标（即要读取的内容，如文件）并且所有标签都设置正确，它将开始追踪（连续读取）来自目标的日志。一旦将足够的数据读入内存或在可配置的超时之后，它就会作为单个批次被发送到 Loki。

当 Promtail 从源（文件和 systemd 日志，如果已配置）读取数据时，它将跟踪它在文件中读取的最后一个位置的偏移量。默认情况下，positions文件存储在`/var/log/positions.yaml`. 在 Promtail 实例重新启动的情况下，positions文件可API以帮助 Promtail 从它停止的地方继续读取。

## API

Promtail 具有一个嵌入式 Web 服务器，在`/`以及 API 端点处公开 Web 控制台：

### `GET /ready`

当 Promtail 启动并运行时，此端点返回 200，并且至少有一个工作目标。

### `GET /metrics`

此端点返回 Prometheus 的 Promtail 指标。请参阅“[操作 > 可观察性](https://grafana.com/docs/loki/latest/operations/observability/)”以获取导出的指标列表。

### 配置Promtail Web服务器

Promtail 暴露出来的 Web 服务器可以在 Promtail`.yaml`配置文件中配置：

```yaml
server:
  http_listen_address: 127.0.0.1
  http_listen_port: 9080
```

