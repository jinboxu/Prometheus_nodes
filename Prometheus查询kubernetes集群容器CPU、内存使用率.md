## Prometheus查询kubernetes集群容器CPU、内存使用率

参考文档:  https://www.lijiaocn.com/%E6%8A%80%E5%B7%A7/2018/09/14/prometheus-compute-kubernetes-container-cpu-usage.html





#### 1. 容器CPU使用率的计算

kubelet中的cadvisor采集的指标与含义，见：[Monitoring cAdvisor with Prometheus](https://github.com/google/cadvisor/blob/2fa6c624a2b22004ef437c0798c0253189b2f01f/docs/storage/prometheus.md)。

| 名称                                      | 类型    | 单位 | 说明                                                         |
| :---------------------------------------- | :------ | :--- | :----------------------------------------------------------- |
| container_cpu_usage_seconds_total         | counter | 秒数 | 该容器服务累计消耗的CPU时间。                                |
| container_cpu_user_seconds_total          | counter | 秒数 | 该容器服务累计消耗的用户（user）CPU时间                      |
| container_cpu_system_seconds_total        | counter | 秒数 | 该容器服务累计消耗的系统（system）CPU时间                    |
| container_cpu_cfs_throttled_seconds_total | counter | 秒数 | cfs 是完全公平调度器(Completely Fair Scheduler)的缩写，是Linux的一种控制CPU资源占用的机制，可以按指定比例分配调度CPU 的使用时间。这个指标指的是该容器服务被限制使用的CPU时间 |
| container_cpu_cfs_throttled_periods_total | counter | 个数 | 文档注释是：“Number of throttled period intervals.”，解释为被限制/节流的CPU时间周期数。 |
| container_cpu_cfs_periods_total           | counter | 个数 | 文档注释是：“Number of elapsed enforcement period intervals。”，应该解释为已经执行的CPU时间周期数。 |
| container_cpu_load_average_10s            | gauge   |      | 文档注释是：“Value of container cpu load average over the last 10 seconds.”应该解释为过去10秒内的CPU负载的平均值。 |

> 针对不同的版本有些参数的意义可能不太一样。比如**container_cpu_usage_seconds_total在之前的版本可能就是"该容器服务针对每个CPU累计消耗的CPU时间。如果有多个CPU，则总的CPU时间需要把各个CPU耗费的时间相加"**





```
rate(container_cpu_usage_seconds_total{image!="",container_name!="POD"}[5m]) / on(id,pod_name,container_name,namespace) (container_spec_cpu_quota/100000) *100
```

> - 标签过滤
>
> 通过image!="",container_name!="POD"得到所需向量（各个容器累计消耗的cpu时间），如下通过一个pod_name会得到多个向量：
>
> ![](pics\label.jpg)
>
> - on(id,pod_name,container_name,namespace) , on(id)可得到同样的结果，但是没有其他标签
>
> - container_spec_cpu_quota：container的配额，为容器指定的CPU个数*100000
>
> 比如kubernetes下限制容器的cpu为200m ，200m / 1000m = 200 00 /  100 000 = 0.2个





#### 2. 容器内存使用率

参考文档:      https://blog.csdn.net/palet/article/details/82889493

| 名称                               | 类型    | 单位        | 说明                                                         |
| :--------------------------------- | :------ | :---------- | :----------------------------------------------------------- |
| container_memory_rss               | gauge   | 字节数bytes | RSS内存，即常驻内存集（Resident Set Size），是分配给进程使用实际物理内存，而不是磁盘上缓存的虚拟内存。RSS内存包括所有分配的栈内存和堆内存，以及加载到物理内存中的共享库占用的内存空间，但不包括进入交换分区的内存。 |
| container_memory_usage_bytes       | gauge   | 字节数bytes | 当前使用的内存量，包括所有使用的内存，不管有没有被访问。     |
| container_memory_max_usage_bytes   | gauge   | 字节数bytes | 最大内存使用量的记录。                                       |
| container_memory_cache             | gauge   | 字节数bytes | 高速缓存（cache）的使用量。cache是位于CPU与主内存间的一种容量较小但速度很高的存储器，是为了提高cpu和内存之间的数据交换速度而设计的。 |
| container_memory_swap              | gauge   | 字节数bytes | 虚拟内存使用量。虚拟内存（swap）指的是用磁盘来模拟内存使用。当物理内存快要使用完或者达到一定比例，就可以把部分不用的内存数据交换到硬盘保存，需要使用时再调入物理内存 |
| container_memory_working_set_bytes | gauge   | 字节数bytes | 当前内存工作集（working set）使用量。                        |
| container_memory_failcnt           | counter | 次          | 申请内存失败次数计数                                         |
| container_memory_failures_total    | counter | 次          | 累计的内存申请错误次数                                       |

> container_memory_max_usage_bytes > container_memory_usage_bytes >= container_memory_working_set_bytes > container_memory_rss
>
> 



```
container_memory_rss{image!="",container_name!="POD"}/container_spec_memory_limit_bytes*100 != +inf
```

> +inf表示无穷大，即当没有设置limits内存限制



#### 3. 文件系统使用量



https://blog.csdn.net/qq_41674452/article/details/106210971