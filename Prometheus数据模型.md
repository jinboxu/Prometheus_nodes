## Prometheus数据模型 

#### 1. 数据模型

每个时间序列都是由度量标准名称和一组键值对（也称为标签）组成唯一标识。

- **标签**，通过一组标签表示特定的实例
- **度量名称**，指定被测量的系统的特征（例如：http_requests_total-接收到的HTTP请求的总数）



**标签**启用Prometheus的维度数据模型：对于相同度量标准名称，任何给定的标签组合都标识该度量标准的特定维度实例。查询语言允许基于这些维度进行筛选和聚合。更改任何标签值（包括添加或删除标签）都会创建新的时间序列。

```
node_filesystem_free_bytes{device="/dev/vda1",fstype="ext4",instance="k8s-master-01",job="kubernetes-nodes",mountpoint="/"}   //1
node_filesystem_free_bytes{device="/dev/vda1",fstype="ext4",instance="k8s-node-02",job="kubernetes-nodes",mountpoint="/"}  //2

node_filesystem_size_bytes{device="/dev/vda1",fstype="ext4",instance="k8s-master-01",job="kubernetes-nodes",mountpoint="/"}   //1
node_filesystem_size_bytes{device="/dev/vda1",fstype="ext4",instance="k8s-node-02",job="kubernetes-nodes",mountpoint="/"}   //2
```

> node_filesystem_free_bytes，节点文件系统空闲空间大小
>
> node_filesystem_size_bytes，节点文件系统总大小
>
> **标签组合不同说明是不同的实例**(相同度量标准名称，任何给定的标签组合都标识该度量标准的特定维度实例)
>
> **相同标签组合的不同度量名称说明了同一实例的不同指标**



节点文件系统使用率:

```
(node_filesystem_free_bytes{fstype=~"ext4|xfs"} / node_filesystem_size_bytes{fstype=~"ext4|xfs"})*100
```

> 得到各个匹配实例指标1与指标2的比值的百分数，即文件系统格式为ext4或xfs的文件分区，它们各个当前的卷空间剩余百分比。



```
100  - (node_filesystem_free_bytes{fstype=~"ext4|xfs"} / node_filesystem_size_bytes{fstype=~"ext4|xfs"}* 100) > 80
```

> 得到各个实例使用的百分比





节点内存使用率:

```
100
  - (node_memory_MemFree_bytes + node_memory_Cached_bytes + node_memory_Buffers_bytes)
  / node_memory_MemTotal_bytes * 100 > 80
```



#### 2. 度量类型

Prometheus 客户端库主要提供Counter、Gauge、**Histogram和Summery**四种主要的 metric 类型：

- **Counter(计算器)：Counter是**一种累加的度量，它的值只能增加或在重新启动时重置为零。例如，您可以使用计数器来表示提供的请求数，已完成的任务或错误的数量。不要使用计数器来表达可减少的值。例如，不要使用Counter来计算当前正在运行的进程的数量，而是使用Gauge。
- ***Gauge(测量)：***Gauge表示单个数值，表达可以任意地上升和下降的度量。Gauge通常用于测量值，例如温度或当前的内存使用情况，但也可以表达上升和下降的“计数”，如正在运行的goroutines的数量。

- Histogram(直方图)：Histogram样本观测（例如：请求持续时间或响应大小），并将它们计入配置的桶中。它也提供所有观测值的总和。具有<basename>基本度量标准名称的histogram的在获取数据期间会显示多个时间序列：

  - 观察桶的累计计数器，暴露为 <basename>_bucket{le=”<upper inclusive bound>”}
  - 所有观察值的总和，暴露为<basename>_sum
  - 已观察到的事件的计数，暴露为<basename>\_count（等同于<basename>_bucket{le=”+Inf”}）

  > 官方手册:    https://prometheus.io/docs/practices/histograms/
>
  > 参考手册:    https://blog.csdn.net/wtan825/article/details/94616813

  **Summery**：**类似于Histogram**，*Summery*样本观察（通常是请求持续时间和响应大小）。虽然它也提供观测总数和所有观测值的总和，但它计算滑动时间窗内的可配置分位数。在获取数据期间，具有<basename>基本度量标准名称的Summery会显示多个时间序列：

  - 流动φ分位数（0≤φ≤1）的观察事件，暴露为<basename>{quantile=”<φ>”}
  - 所有观察值的总和，暴露为<basename>_sum
  - 已经观察到的事件的计数，暴露为<basename>_count



#### 3. 操作符

官方文档:    https://prometheus.io/docs/prometheus/latest/querying/operators/

->  翻译文档:    https://www.cnblogs.com/wayne-liu/p/9273452.html（以官方文档为准）

**向量匹配:**

如下实例，单独将向量匹配拿出来解释一下==>

```
container_memory_working_set_bytes{id="/"}
=>
container_memory_working_set_bytes{beta_kubernetes_io_arch="amd64",beta_kubernetes_io_os="linux",id="/",instance="k8s-master-01",job="kubernetes-nodes-cadvisor",kubernetes_io_arch="amd64",kubernetes_io_hostname="k8s-master-01",kubernetes_io_os="linux"}
container_memory_working_set_bytes{beta_kubernetes_io_arch="amd64",beta_kubernetes_io_os="linux",id="/",instance="k8s-master-02",job="kubernetes-nodes-cadvisor",kubernetes_io_arch="amd64",kubernetes_io_hostname="k8s-master-02",kubernetes_io_os="linux"}
...
```

> 通过id="/"得到容器总的在各个物理节点的内存资源占用



```
machine_memory_bytes
=>
machine_memory_bytes{beta_kubernetes_io_arch="amd64",beta_kubernetes_io_os="linux",instance="k8s-master-01",job="kubernetes-nodes-cadvisor",kubernetes_io_arch="amd64",kubernetes_io_hostname="k8s-master-01",kubernetes_io_os="linux"}   
```

> 该向量标签相比于container_memory_working_set_bytes向量少了id标签



所以，**各个节点下容器资源的占比为**:

```
container_memory_working_set_bytes{id="/"}/ ignoring(id) machine_memory_bytes*100
```





#### 3. ProQL

###### 3.1 标签合并、标签添加

- label_join()

函数可以将向量 v 中多个标签 `src_label` 的值，通过 `separator` 作为连接符写入到一个新的标签 `dst_label` 中。可以有多个 src_label 标签。

> ```
> label_join(v instant-vector, dst_label string, separator string, src_label_1 string, src_label_2 string, ...)
> ```



例如，以下表达式返回的向量多了一个 `foo` 标签，标签值为 `etcd,etcd-k8s`：

```
up{endpoint="api",instance="192.168.123.248:2379",job="etcd",namespace="monitoring",service="etcd-k8s"}
=> up{endpoint="api",instance="192.168.123.248:2379",job="etcd",namespace="monitoring",service="etcd-k8s"} 1
 
label_join(up{endpoint="api",instance="192.168.123.248:2379",job="etcd",namespace="monitoring",service="etcd-k8s"}, "foo", ",", "job", "service")
=> up{endpoint="api",foo="etcd,etcd-k8s",instance="192.168.123.248:2379",job="etcd",namespace="monitoring",service="etcd-k8s"} 1
```



- label_replace()

该函数会依次对 v 中的每个向量进行处理，通过 `regex` 匹配 src_label 的值，并将匹配部分 `relacement` 写入到 dst_label 标签中。

> ```
> label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)
> ```



如下所示：

```
label_replace(up, "host", "$1", "instance", "(.*):.*")
函数处理后，时间序列将包含一个 host 标签，host 标签的值为 Exporter 实例的 IP 地址：

up{host="localhost",instance="localhost:8080",job="cadvisor"} 1
up{host="localhost",instance="localhost:9090",job="prometheus"} 1
up{host="localhost",instance="localhost:9100",job="node"} 1
```



