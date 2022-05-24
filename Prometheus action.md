#### 1. Prometheus警报

Prometheus服务器也可能出现问题

p121

```yaml
  prometheus.yml: |
    groups:
    - name: prometheus_alerts
      rules:
      - alert: PrometheusConfigReloadFailed
        expr: prometheus_config_last_reload_successful == 0
        for: 10m
        labels:
          severity: warning
        annotations:
          description: Reloading Prometheus configuration has failed .
      - alert: PrometheusNotConnectedToAlertmanagers
        expr: prometheus_notifications_alertmanagers_discovered < 1
        labels:
          severity: warning
        annotations:
          description: Prometheus is not connected to any Alertmanagers.
```

> - Prometheus配置重新加载是否失败
> - 确保Prometheus服务器可以发现Alertmanager. 由于没有任何Alertmanager，因此它只会显示在Prometheus控制台的/alerts页面上



热加载更新配置:  https://songjiayang.gitbooks.io/prometheus/content/qa/hotreload.html

> - kill -HUP pid
> - curl -X POST http://IP/-/reload





```Node Export```

---

#### 2. node cpu 

###### 2.0 可用性up指标

对于每个实例抓取，Prometheus都会在以下时间序列中存储样本:

```yaml
up{job="<job name>", instance="<instance id>"}
```

> 如果实例是健康的，则指标设置为1，即数据抓取成功返回。如果抓取失败，则设置为0.



此外，许多exporter都有特定的指标，旨在确定最后一次成功的数据抓取。例如，cAdvisor指标 container_last_seen ，它提供容器列表以及它们最近一次活动的时间。Mysql Export 返回一个指标 mysql_up ,如果SELECT查询在数据库服务器上成功执行，则将该指标设置为1.



但是，它并不完美，因为我们真正监控的是作业对目标的抓取是否成功。例如，如果从服务发现中删除我们的目标，那么它的指标将不再更新，即没有任何记录，因此up警报不会被触发。

**通过确定目标何时消失，我们可以使up警报稍微健壮一些。Prometheus有一个功能叫absent，可检测是否存在缺席的指标。**

```yaml
      - alert: InstanceGone
        expr: absent(up{job="kubernetes-nodes"})
        for: 10s
        labels:
          severity: critical
        annotations:
          summary: Host {{ $labels.instance }} is no longer reporting!
          description: '找不到我们的主机了，快去看看吧！'
```



###### 2.1 cpu使用率

P70

```yaml
      - alert: NodeCPUUsage
        expr: 100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100) > 60
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "High Node CPU for 30 minutes"
          description: "{{ $labels.instance }}"
```





###### 2.2 cpu饱和度

p72



```yaml
count by (instance) (node_cpu_seconds_total{mode="idle"})
```

> 计算主机上的CPU数量，使用count聚合实现



\*

```yaml
node_load1  >  on(instance) 2 * count by (instance) (node_cpu_seconds_total{mode="idle"})
```

> 1分钟的平均负载超过主机CPU数量两倍的结果



rule:

```yaml
      - alert: NodeCPUUsage
        expr: 100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100) > 60
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "High Node CPU for 30 minutes"
          description: "{{ $labels.instance }}"
      
      - alert: NodeCPULoad
        expr: node_load1  >  on(instance) 2 * count by (instance) (node_cpu_seconds_total{mode="idle"})
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Node Load CPU for 5 minutes"
          description: "{{ $labels.instance }} CPU High Load"
```



#### 3. node memory

###### 3.1 节点内存使用率

```yaml
100 - (node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes) / node_memory_MemTotal_bytes * 100 
```



###### 3.2 内存饱和度

p74

通过检查内存和磁盘的读写来监控内存饱和度。可以使用从/proc/vmstat收集的两个Node Exporter指标:

- node_vmstat_pswpin : 系统每秒从磁盘读到内存的字节数
- node_vmstat_pswpout :  系统每秒从内存写到磁盘的字节数

两者都是自上次启动以来的字节数, 以KB为单位

```yaml
1024 * sum by (instance) (rate(node_vmstat_pswpin[1m]) + rate(node_vmstat_pswpout[1m]))
```

> 取消这里的聚合sum by也可

为了获取饱和度，我们对每个指标计算每一分钟的速率，将两个速率相加，然后乘以1024以获取字节数。我们对此设置图形化展示或警报，以识别行为不当的应用程序主机。



#### 4. 磁盘

###### 4.1 磁盘使用率

p76  p120



```
100 - (node_filesystem_free_bytes{fstype=~"ext4|xfs"} / node_filesystem_size_bytes{fstype=~"ext4|xfs"} * 100)
```

> 当前文件系统磁盘使用的百分比



许多情况下，上面的数据很多情况下是无用的。我们通常需要回答的问题是：**考虑到现在磁盘的使用情况，以及它的增长，我们会在什么时候耗尽磁盘空间**

实际上Prometheus提供了一种机制，通过一个名为predict_linear的函数，我们可以构造一个查询来回答这个问题。

```
predict_linear(node_filesystem_free_bytes{fstype=~"ext4|xfs",mountpoint !~ ".*/kubelet/pods/.*"}[1h], 4*3600)
```

> 我们选择两小时的时间窗口[1h]，并将此时间序列快照放在predict_linear的函数中。该函数使用简单的线性回归，根据以前的增长情况来确定文件系统何时会耗尽空间。
>
> 通过mountpoint !~ ".\*/kubelet/pods/.\*"标签过滤掉pod容器文件系统



rule:

```yaml
      - alert: NodeFilesystemUsage
        expr: 100 - (node_filesystem_free_bytes{fstype=~"ext4|xfs"} / node_filesystem_size_bytes{fstype=~"ext4|xfs"} * 100) > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "分区使用率过高"
          description: "{{ $labels.instance }}: {{ $labels.mountpoint }} 分区使用大于80% (当前值: {{ $value }})"

      - alert: DiskWillFillIn4Hours
        expr: predict_linear(node_filesystem_free_bytes{fstype=~"ext4|xfs",mountpoint !~ ".*/kubelet/pods/.*"}[1h], 4*3600) < 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: Disk on {{ $labels.instance }} will fill in approximately 4 hours.
```

> 这里，如果线性回归预测磁盘空间将在4小时内耗尽，则触发警报



###### 4.2 [磁盘IO](https://www.ipcpu.com/2021/04/prometheus-node_exporter/)

对于磁盘问题，我们主要关注以下几个指标：

> 磁盘空间使用率、磁盘inode使用率（df -h和df -i命令）
> 磁盘读写次数IOPS (iostat中的r/s、w/s)
> 磁盘读写带宽 (iostat中的rkB/s、wkB/s)
> 磁盘IO利用率%util (iostat中的%util)
> 磁盘队列数 (iostat中的avgqu-sz)
> 磁盘读写的延迟时间 (iostat中的r_await、w_await)



record rule:

```yaml
- name: node_disk_io
  rules:
  - record: node_disk_iops_read
    expr: rate(node_disk_reads_completed_total[5m])
  - record: node_disk_iops_write
    expr: rate(node_disk_writes_completed_total[5m])
  - record: node_disk_util_percent
    expr: rate(node_disk_io_time_seconds_total[5m])
  - record: node_disk_io_queue
    expr: rate(node_disk_io_time_weighted_seconds_total[5m])
  - record: node_disk_r_await
    expr: rate(node_disk_read_time_seconds_total[5m]) / rate(node_disk_reads_completed_total[5m]) * 1000
  - record: node_disk_w_await
    expr: rate(node_disk_write_time_seconds_total[5m]) / rate(node_disk_writes_completed_total[5m]) * 1000
```



grafana Query:

```
# IOPS
node_disk_iops_read{kubernetes_node=~"$instance",device=~"$device"}
node_disk_iops_write{kubernetes_node=~"$instance",device=~"$device"}
# 磁盘IO利用率%util
node_disk_util_percent{kubernetes_node=~"$instance",device=~"$device"} * 100
# 磁盘IO队列数
node_disk_io_queue{kubernetes_node=~"$instance",device=~"$device"}
# 读写延迟
node_disk_r_await{kubernetes_node=~"$instance",device=~"$device"}
node_disk_w_await{kubernetes_node=~"$instance",device=~"$device"}
```

> 通过节点标签和设备标签对应的grafana变量值过滤





#### 5. 节点上的服务

p122

```yaml
      - alert: NodeServiceDown
        expr: node_systemd_unit_state{state="active"} != 1
        for: 60s
        labels:
          severity: critical
        annotations:
          summary: service {{ $labels.name }} on {{ $labels.instance }} is no longer active!
          description: 我们的服务呢？快去看看吧!
```



---



```Cadvisor```

#### 6. pod容器CPU使用率

我们先从一个具体的pod出发，在Graph下查看prometheus服务pod本身：

- container_cpu_usage_seconds_total{pod_name="prometheus-0"}
- container_spec_cpu_quota{pod_name="prometheus-0"}

> 分别代表"容器服务累计消耗的cpu时间" 和 "container的配额"
>
> prometheus一个pod下面有两个容器，prometheus-server 和 prometheus-server-configmap-reload



第一个查询可以发现有4个时间序列:

pause容器、prometheus-server容器、prometheus-server-configmap-reload容器、pod(所以容器之和)  总共4个时间序列

第二个查询可以发现有3个时间序列:

prometheus-server容器、prometheus-server-configmap-reload容器、pod(所以容器之和)  总共3个时间序列，pause容器是每个pod必须的，其没有资源限额



对应pod cpu和内存使用率的计算，我们只需要关注pod里面每个容器的cpu和内存的使用率即可,所以我们PromQL应该这样:

- 去掉pause容器，因为container_spec_cpu_quota指标没有pause容器的时间序列，所以这个直接忽略
- **向量匹配同时去掉针对整个pod cpu或内存资源使用率**，我们有以下两种方法:

```
# 1 
## 向量匹配1: container_spec_cpu_quota比container_cpu_usage_seconds_total向量只多了cpu标签，我们可以使用ignoring忽略这个标签
   向量匹配2: 使用on操作符
   排除瞬时向量(pod中所以cpu或内存使用率)，通过描述pod的向量没有image标签
rate(container_cpu_usage_seconds_total{image!="", pod_name="prometheus-0"}[5m]) / ignoring(cpu)  (container_spec_cpu_quota/100000) *100

rate(container_cpu_usage_seconds_total{image!="", pod_name="prometheus-0"}[5m]) / on(id,pod_name,container_name,namespace) (container_spec_cpu_quota/100000) *100
```

查询结果:

<img src="pics\prometheus-pod.jpg" style="zoom:150%;" />



\*

pod容器cpu使用率:

```yaml
rate(container_cpu_usage_seconds_total{image!=""}[5m]) / ignoring(cpu)  (container_spec_cpu_quota/100000) *100 
```



#### 7. pod容器内存使用率

与pod cpu一样，我们从一个具体的pod出发:

- container_memory_rss{pod_name="prometheus-0"}
- container_spec_memory_limit_bytes{pod_name="prometheus-0"}

> 分别表示 "分配给进程使用实际物理内存"和“容器内存的限额”
>
> 不同的是，这两个向量的维度(标签匹配)是一致的,同时没有设置容器内存限制的时间序列值为0(cpu限制没有设置就没有该时间序列)，比如pause容器的container_spec_memory_limit_bytes默认都为0



\*

pod容器内存使用率:

```
container_memory_rss{image!="", container_name!="POD"} / container_spec_memory_limit_bytes * 100 != +inf 
```

> - 通过```image!=""```排除pod下容器的内存之和(整个pod内存使用和限制)
>
> - 通过```container_name!="POD"```排除pause容器
>
> - 通过表达式```!= +inf```排除没有设置内存限制的容器 ，因为pause容器默认内存限制为0，这个表达式同时可以排除pause容器，但是很明显，加上```container_name!="POD"```效率会更好





rule:

```yaml
      rules:
      - alert: PodMemoryUsage
        expr: container_memory_rss{image!="",container_name!="POD"}/container_spec_memory_limit_bytes * 100 != +inf > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "POD Container 内存使用率过高"
          description: "{{ $labels.namespace }} {{ $labels.pod }} {{ $labels.container_name }}内存使用大于80% (当前值: {{ $value }})"

      - alert: PodCpuUsage
        expr: rate(container_cpu_usage_seconds_total{image!=""}[5m]) / ignoring(cpu)  (container_spec_cpu_quota/100000) *100 > 70
        for: 20m
        labels:
          severity: warning
        annotations:
          summary: "High Container CPU for 20 minutes"
          description: "{{ $labels.namespace }} {{ $labels.pod }} {{ $labels.container_name }}"
```



___

p196

```kubernetes-service-endpoints```

```
- job_name: kubernetes-service-endpoints
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    regex: "true"
    action: keep
  ......(其他重新标记等)
```

> 关于重新标记见另一份文档



#### 8. 监控部署在容器中的Node Exporter

p194 



#### 9. kube-stata-metrics

p201

kube-state-metrics基于 client-go 开发，轮询 Kubernetes API，并将 Kubernetes的结构化信息转换为metrics。



- 检测部署是否成功

```yaml
      - alert: DeploymentGenerationMismatch
        expr: kube_deployment_status_observed_generation != kube_deployment_metadata_generation
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: Deployment is outdated
          description: Observed deployment generation done not match expected one for \ 
                       deployment {{ $labels.namespace }} / {{ $labels.deployment }}
```

> 检测部署是否成功。它将部署的运行版本与元数据中的版本进行比较。如果两者在5分钟内不相等，则会发出警报，指示部署失败。



- 检查部署副本

```yaml
      - alert: DeploymentReplicasNotUpdated
        expr: (kube_deployment_status_replicas_updated != kube_deployment_spec_replicas)
              or (kube_deployment_status_replicas_available != kube_deployment_spec_replicas)
              unless (kube_deployment_spec_paused == 1)
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: Deployment replicas are outdated
          description: Replicas are not updated and available for deployment \
                       {{ $labels.namespace }} / {{ $labels.deployment }}
```

> 假设部署没有暂停，确认更新或可用的副本是否与部署规划中的副本数相匹配。



- 检查pod的重启情况

```yaml
      - alert: PodFrequentlyRestarting
        expr: increase(kube_pod_container_status_restarts_total[1h]) > 5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: Pod is restarting frequently
          description: Pod {{ $labels.namespace }} / {{ $labels.pod }} was restarted \
                       {{ $value }} times within the last hour
```

> 如果数值持续10分钟超过5，则会触发警告。



- 检查pod是否调度成功

```
      - alert: PodStatusUnschedulable
        expr: kube_pod_status_unschedulable == 1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Pod is unschedulable
          description: Pod {{ $labels.namespace }} / {{ $labels.pod }} was unschedulable
                       within the last two minutes.
```



- 检查cronjob

```
      - alert: CronjobStatusActive
        expr: kube_cronjob_status_active > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: parallel job?
          description: cronjob {{ $labels.namespace }} / {{ $labels.cronjob }} 活动数量为 {{ $value }}.
      - alert: JobStatusFailed
        expr: kube_job_failed > 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: Job is failed
          description: Failed job {{ $labels.namespace }} / {{ $labels.job_name }}
```

> - ```kube_cronjob_status_active``` 表示当前活动的job数量，当我们的定时任务不是一个需要并发执行的任务的时候，很可能上一个任务已经无法正常结束了
> - ```kube_job_status_failed``` 是一个counter类型，并且已经执行过的任务也会存在（所有不能直接使用kube_job_status_failed > 1 表达式获取，否则会有很多重复无关的告警）



- 检查endpoint

上面我们通过检查副本数、pod重启情况、pod是否调度， 通过这些指标来判断pod是否已经在我们期望的状态下运行。但是还不够，比如在 就绪探针检查失败的情况下，endpoint会被从 Service 的负载均衡器中剔除。

```yaml
- alert: EndpointsAddressNotReady
  expr: kube_endpoint_address_not_ready > 0
  for: 1m
  labels:
    severity: warning
  annotations:
    description: endpoint {{ $labels.namespace }} / {{ $labels.endpoint }} not ready number is {{ $value }}
    summary: kube endpoint not ready 
- alert: EndpointsAddressAvailable
  expr: kube_endpoint_address_available == 0
  labels:
    severity: critical
  annotations:
    description: endpoint {{ $labels.namespace }} / {{ $labels.endpoint }} is not available
    summary: kube endpoint available number is 0 ! 
```

kube_endpoint_address_not_ready  #Gauge类型，endpoint中not ready的addresses数
kube_endpoint_address_available  #Gauge类型，endpoint中可用的addresses数。



- 第一个告警表示有不可用的endpoint，比如就绪探针检查失败，会被从 Service 的负载均衡器中剔除，但是在多副本集的情况下，可能还有其他pod是正常的，这样就不会很影响应用功能，所以设置其告警级别为 warning ，持续时间为 1分钟 触发告警
- 第二个告警表示service后端没有可用的endpoint了，这个应该马上告警，因为没有任何可用的pod了。



---



p202

```kubernetes-apiservers```

```yaml
    - job_name: kubernetes-apiservers
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - action: keep
        regex: default;kubernetes;https
        source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
```



#### 10. kube API

p203

与API相关的指标将构成Kubernetes监控的核心，使我们能够监控集群的延迟、错误率和可用性。



- APIserver请求延迟时间

```
  kubernetes_apiservices.yml: |
    groups:
    - name: kubernetes_apiservers
      rules:
      - record: apiserver_lantency_seconds:quantile
        expr: histogram_quantile(0.99, rate(apiserver_request_latencies_bucket[5m]))  / 1e+06
        labels:
          quantile: "0.99"
      - record: apiserver_lantency_seconds:quantile
        expr: histogram_quantile(0.9, rate(apiserver_request_latencies_bucket[5m]))  / 1e+06
        labels:
          quantile: "0.9"
      - record: apiserver_lantency_seconds:quantile
        expr: histogram_quantile(0.5, rate(apiserver_request_latencies_bucket[5m]))  / 1e+06
        labels:
          quantile: "0.5"

      - alert: APIHighLatency
        expr: apiserver_lantency_seconds:quantile{quantile="0.99", verb!~"^(?:WATCH|WATCHLIST|PROXY|CONNECT)$"} > 4
        for: 10m
        labels:
          severity: critical
        annotations:
          description: the API server {{ $labels.instance }} has a 99th percentile latency of {{ $value }} senconds \
                       for {{ $labels.resource }} {{ $labels.verb }}
```

> **rate(apiserver_request_latencies_bucket[5m]) 计算得到了一个新桶，表示“最近5分钟各个请求的平均个数(每秒的请求次数)”，然后使用histogram_quantile函数计算第50、90和99百分位数的延迟**
>
> 使用标签来选择第99百分位数，任何不是WATCH、WATCHLIST、PROXY 或 CONNECT 的谓词，如果指标的延迟持续10分钟超过4，则会触发报警.



**我们通过对api延时请求进一步研究，来深入理解histogram的数据类型：如何得到各个apiserver(如果有多个的话)的请求延迟？**

>  histogram官方文档： https://prometheus.io/docs/practices/histograms/

```
histogram_quantile(0.99, sum by (job,instance, le) (rate(apiserver_request_latencies_bucket{verb!~"^(?:WATCH|WATCHLIST|PROXY|CONNECT)$"}[5m])))  / 1e+06
```

> 首先还是排除这些谓词: 我们这次不能像上面一样放在外面排除，因为指标马上要被聚合sum by
>
> 然后使用sum by (job,instance, le)对指标进行聚合得到新的桶，当对桶进行聚合是le标签是必须的.
>
> 最后跟上面一样，我们得到了apiserver各个请求延迟之和的99百分位



- 检测API服务器中的高错误率

```yaml
      - alert: APIServerErrorsHigh
        expr: rate(apiserver_request_count{code=~"^(?:5..)$"}[5m]) / rate(apiserver_request_count[5m]) * 100 > 5
        for: 10m
        labels:
          severity: critical
        annotations:
          description: the API server {{ $labels.instance }} returns errors for {{ $value }}% of requests within the last 10m
```

> 此警报计算API请求的错误率，使用正则表达式来匹配任何以5xx开头的错误。如果5分钟向量的百分比超过5%，则触发警报。





#### 11. 

