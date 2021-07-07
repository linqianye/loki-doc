# Loki文档

![logo_and_name](image/logo_and_name.png)

用于日志的 Prometheus！

Grafana Loki 是一组可以组成一个功能齐全的日志堆栈的组件。

与其他日志系统不同，Loki的构建思想是只为相关日志的元数据建立索引：标签（就像Prometheus标签一样）。然后，日志数据本身被压缩并以块的形式存储在对象存储中（如S3或GCS），或者存储在本地文件系统中。小索引和高度压缩的块简化了操作，并显著降低了Loki的成本。



文档版本 v2.2.0