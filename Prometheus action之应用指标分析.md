## Prometheus action之应用指标分析

**Google监控的4个黄金指标**

Four Golden Signals是Google针对大量分布式监控的经验总结，4个黄金指标可以在服务级别帮助衡量终端用户体验、服务中断、业务影响等层面的问题。主要关注与以下四种类型的指标：延迟，通讯量，错误以及饱和度：

- 延迟：服务请求所需时间。

记录用户所有请求所需的时间，重点是要区分成功请求的延迟时间和失败请求的延迟时间。 例如在数据库或者其他关键后端服务异常触发HTTP 500的情况下，用户也可能会很快得到请求失败的响应内容，如果不加区分计算这些请求的延迟，可能导致计算结果与实际结果产生巨大的差异。除此以外，在微服务中通常提倡“快速失败”，开发人员需要特别注意这些延迟较大的错误，因为这些缓慢的错误会明显影响系统的性能，因此追踪这些错误的延迟也是非常重要的。

- 通讯量：监控当前系统的流量，用于衡量服务的容量需求。

流量对于不同类型的系统而言可能代表不同的含义。例如，在HTTP REST API中, 流量通常是每秒HTTP请求数；

- 错误：监控当前系统所有发生的错误请求，衡量当前系统错误发生的速率。

对于失败而言有些是显式的(比如, HTTP 500错误)，而有些是隐式(比如，HTTP响应200，但实际业务流程依然是失败的)。

对于一些显式的错误如HTTP 500可以通过在负载均衡器(如Nginx)上进行捕获，而对于一些系统内部的异常，则可能需要直接从服务中添加钩子统计并进行获取。

- 饱和度：衡量当前服务的饱和度。

主要强调最能影响服务状态的受限制的资源。 例如，如果系统主要受内存影响，那就主要关注系统的内存状态，如果系统主要受限与磁盘I/O，那就主要观测磁盘I/O的状态。因为通常情况下，当这些资源达到饱和后，服务的性能会明显下降。同时还可以利用饱和度对系统做出预测，比如，“磁盘是否可能在4个小时候就满了”。



**RED方法**

RED方法是Weave Cloud在基于Google的“4个黄金指标”的原则下结合Prometheus以及Kubernetes容器实践，细化和总结的方法论，特别适合于云原生应用以及微服务架构应用的监控和度量。主要关注以下三种关键指标：

- (Request) **R**ate - 每秒处理的请求数量。
- (Request) **E**rrors - 每秒失败的请求数量。
- (Request) **D**uration - 请求处理时间的分布。

上述指标能让我们清楚地感知微服务应用的运行状况，并能帮助衡量终端用户的体验问题。



**USE方法**

USE方法全称"Utilization Saturation and Errors Method"，主要用于分析系统性能问题，可以指导用户快速识别资源瓶颈以及错误的方法。正如USE方法的名字所表示的含义，USE方法主要关注与资源的：使用率(Utilization)、饱和度(Saturation)以及错误(Errors)。

- 使用率：关注系统资源的使用情况。 这里的资源主要包括但不限于：CPU，内存，网络，磁盘等等。100%的使用率通常是系统性能瓶颈的标志。
- 饱和度：例如CPU的平均运行排队长度，这里主要是针对资源的饱和度(注意，不同于4大黄金信号)。任何资源在某种程度上的饱和都可能导致系统性能的下降。
- 错误：错误计数。例如：“网卡在数据包传输过程中检测到的以太网网络冲突了14次”。





#### 1. kubernetes ingress nginx监控指标分析与直方图

Histogram(直方图)采集指标：

nginx_ingress_controller_request_duration_seconds_bucket

nginx_ingress_controller_request_duration_seconds_sum

nginx_ingress_controller_request_duration_seconds_count



理解Histogram度量指标（如下图）:

后面桶的值需要将前面桶的值累加上得到的值才是<basename>_bucket采集的值，图中并没有这样表示出来。

![histograms](pics\histograms.png)



- 分析在过去5分钟的平均请求时间

```
  rate(nginx_ingress_controller_request_duration_seconds_sum[5m])
/
  rate(nginx_ingress_controller_request_duration_seconds_count[5m])
```





计算RED方法中提到的三个关键指标:

**Rate**

```
round(sum(irate(nginx_ingress_controller_requests[5m])), 0.001)
```

> 参考grafana Panel



**Errors**

```
sum(rate(nginx_ingress_controller_requests{status=~"[4-5].*"}[5m])) / sum(rate(nginx_ingress_controller_requests[5m]))

## 以上表达式得到的是所有请求，我们可以通过分组得到更具体的信息=>
sum(rate(nginx_ingress_controller_requests{status=~"[4-5].*"}[5m])) by (ingress) / sum(rate(nginx_ingress_controller_requests[5m])) by (ingress) * 100
```

> 参考grafana Panel
>
> 第一条规则求的是总的错误率，第二条通过ingress名称进行了聚合分组



统计各个服务请求15分钟内的异常数量:

```
sum by(service, status) (increase(nginx_ingress_controller_requests{status=~"5.+"}[15m]))
或
sum(increase(nginx_ingress_controller_requests{status=~"5.+"}[15m])) by(service, status)
```

> sum(...) by(...)   == sum by(...) (...)



**Duration**

```
histogram_quantile(0.99, sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])) by(le))

histogram_quantile(0.90, sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])) by(le))

histogram_quantile(0.50, sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])) by(le))
```

> 参考grafana Panel





prometheus 告警规则:

Errors - 每秒失败(4xx或5xx)请求失败

```
sum(rate(nginx_ingress_controller_requests{status=~"5.+"}[5m])) / sum(rate(nginx_ingress_controller_requests[5m])) * 100 > 3

sum(rate(nginx_ingress_controller_requests{status=~"4.+"}[5m])) / sum(rate(nginx_ingress_controller_requests[5m])) * 100 > 5
```



Duration - 90百分位请求时间

```
histogram_quantile(0.90, sum(rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])) by(le)) > 0.25
```

> 90%请求时间超过了0.25s就报警



其他Errors:

```
## ingressNGINXConfigFailed
nginx_ingress_controller_config_last_reload_successful

## ingressNGINXCertificateExpiry
(avg(nginx_ingress_controller_ssl_expire_time_seconds) by (host) - time()) < 604800
```

