# Loki clients

Loki 支持以下官方客户端发送日志：

- [Promtail](clients/promtail/)
- [Docker Driver](clients/docker-driver/)
- [Fluentd](clients/fluentd/)
- [Fluent Bit](clients/fluentbit/)
- [Logstash](clients/logstash/)
- [Lambda Promtail](clients/lambda-promtail/)

## 选择client

虽然可以同时使用所有clients来覆盖多个用例，但最初选择哪个client发送日志取决于您的用例。

### Promtail

Promtail 是您运行 Kubernetes 时的首选客户端，因为您可以将其配置为自动从 Promtail 运行所在节点上运行的 pod 中抓取日志。Promtail 和 Prometheus 在 Kubernetes 中一起运行可以实现强大的调试：如果 Prometheus 和 Promtail 使用相同的标签，用户可以使用 Grafana 等工具根据标签集在指标和日志之间切换。

Promtail 也是裸机上的首选客户端，因为它可以配置为从给定主机路径的所有文件中追踪日志。这是从纯文本文件（例如记录到`/var/log/*.log`的内容）向 Loki 发送日志的最简单方法。

最后，如果您想从日志中提取指标（例如计算特定消息的出现次数），Promtail 效果很好。

### Docker 日志驱动程序

当使用 Docker 而不是 Kubernetes 时，应该使用 Loki 的 Docker 日志驱动程序，因为它会自动为正在运行的容器添加适当的标签。

### Fluentd 和 Fluent Bit

当您已经部署了Fluentd并且已经配置了`Parser`和`Filter`插件时，Fluentd和Fluent Bit是最理想的。

Fluentd 在使用 Prometheus 插件时也可以很好地从日志中提取指标。

### Logstash

如果您已经在使用 logstash 或 beats，这将是最简单的开始方式。通过添加我们的output插件，您可以快速试用 Loki，而无需进行大的配置更改。

### Lambda Promtail

这是一个结合 promtail push-api [抓取配置](clients.md#promtail/configuration#loki_push_api_config)和[lambda-promtail](clients.md#lambda-promtail/) AWS Lambda 函数的工作流，它将日志从 Cloudwatch 传输到 Loki。

如果您希望以低占用率的方式测试Loki，或者您希望在Loki中监控AWS lambda日志，那么这是一个不错的选择。



# 非官方client

请注意，Loki API 尚不稳定，因此在使用或编写第三方客户端时可能会发生重大更改。

- [promtail-client](https://github.com/afiskon/promtail-client) (Go)
- [push-to-loki.py](https://github.com/sleleko/devops-kb/blob/master/python/push-to-loki.py) (Python 3)
- [Serilog-Sinks-Loki](https://github.com/JosephWoodward/Serilog-Sinks-Loki) (C#)
- [loki-logback-appender](https://github.com/loki4j/loki-logback-appender) (Java)
- [Log4j2 appender for Loki](https://github.com/tkowalcz/tjahzi) (Java)

