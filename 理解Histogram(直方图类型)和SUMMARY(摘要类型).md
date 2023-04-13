## 理解Histogram(直方图类型)和SUMMARY(摘要类型)

Prometheus将所有采集到的样本数据以[时间序列](https://so.csdn.net/so/search?q=时间序列&spm=1001.2101.3001.7020)（time-series）的方式保存在内存数据库中，并定时保存在硬盘上。时间序列中的每一个样本由以下三部分组成。

- 指标(metric): metric name和描述当前样本特征的labelsets组成，参考格式如 `<metric name>{<label name>=<label value>, ...}；`，其中metric name的命名规则为：应用名称开头_监测对像_数值类型_单位
- 时间截(timestamp):一个精确到毫秒的时间截；
- 样本值(value):一个float64的浮点类型数据表示当前的样本值。



#### 1.  Histogram(直方图类型)

Histogram 由 < basename>\_bucket{le="< upper inclusive bound>"}，< basename>\_bucket{le="+Inf"}, < basename>\_sum，< basename>_count 组成，主要用于表示一段时间范围内对数据进行采样（通常是请求持续时间或响应大小），并能够对其指定区间以及总数进行统计，通常它采集的数据展示为直方图。

以请求响应时间requests_latency_seconds为例，比如我们需要记录http请求响应时间符合在分布范围{0.005，0.01，0.025，0.05，0.075，0.1，0.25，0.5，0.75，1，2.5，5，7.5，10}中的次数时：

- 事件发生的总次数，basename_count。

```
# 实际含义： 当前一共发生了2次http请求
io_namespace_http_requests_latency_seconds_histogram_count{path="/",method="GET",code="200",} 2.0
```

- 所有事件产生值的大小的总和，basename_sum。

```
# 实际含义： 发生的2次http请求总的响应时间为13.107670803000001 秒
io_namespace_http_requests_latency_seconds_histogram_sum{path="/",method="GET",code="200",} 13.107670803000001
```

- 事件产生的值分布在bucket中的次数，basename_bucket{le=“上包含”}

```
# 在总共2次请求当中。http请求响应时间 <=0.005 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.005",} 0.0
# 在总共2次请求当中。http请求响应时间 <=0.01 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.01",} 0.0
# 在总共2次请求当中。http请求响应时间 <=0.025 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.025",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.05",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.075",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.1",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.25",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.5",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="0.75",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="1.0",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="2.5",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="5.0",} 0.0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="7.5",} 2.0
# 在总共2次请求当中。http请求响应时间 <=10 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="10.0",} 2.0
# 在总共2次请求当中。http请求响应时间 10 秒 的请求次数为0
io_namespace_http_requests_latency_seconds_histogram_bucket{path="/",method="GET",code="200",le="+Inf",} 2.0
```



=> 聚合分析

以kubernetes ingress nginx监控指标做分析

> 过去5分钟的平均响应时间

```
  rate(nginx_ingress_controller_request_duration_seconds_sum[5m])
/
  rate(nginx_ingress_controller_request_duration_seconds_count[5m])
```



> 过去5分钟请求处理时间的第99，90，50百分位

```
histogram_quantile(0.99, sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])) by(le))

histogram_quantile(0.90, sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])) by(le))

histogram_quantile(0.50, sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])) by(le))
```

histogram





#### 2 Summary(摘要类型)

摘要和直方图之间的本质区别在于，摘要在客户端计算流式 φ 分位数并直接公开它们，而直方图公开分桶观察计数，并且从直方图的桶中计算分位数发生在服务器端使用 histogram_quantile( ）功能。







```
https://www.cnblogs.com/snow-man/p/15402671.html
https://zhuanlan.zhihu.com/p/104607739
https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile
```

