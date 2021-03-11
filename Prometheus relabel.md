## kubernetes服务发现-Prometheus relabel

参考手册:

```
## 官方文档rebabel
https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config 
## prometheus手册
https://yunlzheng.gitbook.io/prometheus-book/part-ii-prometheus-jin-jie/sd/service-discovery-with-relabel#prometheus-de-relabeling-ji-zhi
```

默认情况下，当Prometheus加载Target实例完成后，这些Target时候都会包含一些默认的标签：

- `__address__`：当前Target实例的访问地址`:`
- `__scheme__`：采集目标服务访问地址的HTTP Scheme，HTTP或者HTTPS
- `__metrics_path__`：采集目标服务访问地址的访问路径
- `__param_`：采集任务目标服务的中包含的请求参数

一般来说，Target以`__`作为前置的标签是在系统内部使用的，因此这些标签不会被写入到样本数据中。不过这里有一些例外，例如，我们会发现所有通过Prometheus采集的样本数据中都会包含一个名为instance的标签，该标签的内容对应到Target实例的`__address__`。 这里实际上是发生了一次标签的重写处理。





内置函数：

```
https://prometheus.fuckcloudnative.io/di-san-zhang-prometheus/di-4-jie-cha-xun/functions   内置函数
```

