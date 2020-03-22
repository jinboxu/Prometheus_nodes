## k8s监控

![1](pics\1.png)



#### 1. 在k8s中部署Prometheus

**https://github.com/kubernetes/kubernetes/tree/v1.15.7/cluster/addons/prometheus**



#### 2. 监控指标

通过各种exporter采集不同维度的监控指标，并通过Prometheus支持的数据格式暴露出来，Prometheus定期pull数据并用Grafana展示，异常情况使用AlertManager告警。

- 通过cadvisor采集容器、Pod相关的性能指标数据，并通过暴露的/metrics接口用prometheus抓取

- 通过prometheus-node-exporter采集主机的性能指标数据，并通过暴露的/metrics接口用prometheus抓取

- 应用侧自己采集容器中进程主动暴露的指标数据（暴露指标的功能由应用自己实现，并添加平台侧约定的annotation，平台侧负责根据annotation实现通过Prometheus的抓取）

- 通过kube-state-metrics采集k8s资源对象的状态指标数据，并通过暴露的/metrics接口用prometheus抓取

- 通过etcd、kubelet、kube-apiserver、kube-controller-manager、kube-scheduler自身暴露的/metrics获取节点上与k8s集群相关的一些特征指标数据。



| **监控指标** | **具体实现**       | **举例**               |
| ------------ | ------------------ | ---------------------- |
| Pod性能      | cAdvisor           | 容器CPU，内存利用率    |
| Node性能     | node-exporter      | 节点CPU，内存利用率    |
| K8S资源对象  | kube-state-metrics | Pod/Deployment/Service |



由于容器隔离原因，使用容器NodeExporter并不能正确获取到宿主机磁盘信息，故将NodeExporter部署到宿主机。如下为在宿主机上部署NodeExporter的脚本：

```shell
#!/bin/bash

wget https://github.com/prometheus/node_exporter/releases/download/v0.17.0/node_exporter-0.17.0.linux-amd64.tar.gz

tar zxf node_exporter-0.17.0.linux-amd64.tar.gz
mv node_exporter-0.17.0.linux-amd64 /usr/local/node_exporter

cat <<EOF >/usr/lib/systemd/system/node_exporter.service
[Unit]
Description=https://prometheus.io

[Service]
Restart=on-failure
ExecStart=/usr/local/node_exporter/node_exporter --collector.systemd --collector.systemd.unit-whitelist=(docker|kubelet|kube-proxy|flanneld).service

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable node_exporter
systemctl restart node_exporter
```





#### 3. Prometheus基本配置

- 在Prometheus中创建报警规则

```shell
# vi prometheus.yml
rule_files:
   - "rules/*.yml"

# vi rules/node.yml
groups:
- name: example   # 报警规则组名称
  rules:
  # 任何实例5分钟内无法访问发出告警
  - alert: InstanceDown
    expr: up == 0
    for: 5m  #持续时间 ， 表示持续一分钟获取不到信息，则触发报警
    labels:
      severity: page  # 自定义标签
    annotations:
      summary: "Instance {{ $labels.instance }} down" # 自定义摘要
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes." # 自定义具体描述
```



*prometheus stafulset资源对象的卷挂载内容*

```shell
[k8s@k8s-master monitoring]$ cat prometheus-statefulset.yaml
......//略          
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
            - name: prometheus-data
              mountPath: /data
              subPath: ""
            - name: prometheus-rules
              mountPath: /etc/config/rules
      terminationGracePeriodSeconds: 300
      volumes:
        - name: config-volume            //configmap卷挂载之配置文件
          configMap:
            name: prometheus-config
        - name: prometheus-rules         //configmap卷挂载之报警规则定义
          configMap:
            name: prometheus-rules
  volumeClaimTemplates:                   //定义pvc卷挂载模板,数据存储
  - metadata:
      name: prometheus-data
    spec:
      storageClassName: managed-nfs-storage
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: "16Gi"

```

> 配置文件的configmap资源对象修改后应用会自动生效(configmap-reload容器)，该配置文件定义了
>
> - 采集周期和计算周期(scrape_interval和evaluation_interval)
>
> - target监控对象
> - 与alertmanager通信配置
> - 报警规则配置文件路径



- 告警分配（Alertmanager）

```shell
route:
  receiver: 'default-receiver'  
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  group_by: [cluster, alertname]
# 所有不匹配以下子路由的告警都将保留在根节点，并发送到“default-receiver”
  routes:
# 所有service=mysql或者service=cassandra的告警分配到数据库接收端
  - receiver: 'database-pager'
    group_wait: 10s
    match_re:
      service: mysql|cassandra
# 所有带有team=frontend标签的告警都与此子路由匹配
# 它们是按产品和环境分组的，而不是集群
  - receiver: 'frontend-pager'
    group_by: [product, environment]
    match:
      team: frontend 
```

> **主要工作流程:**
>
> 1. 接收到Alert，根据labels判断属于哪些Route（可存在多个Route，一个Route有多个Group，一个Group有多个Alert）。
> 2. 将Alert分配到Group中，没有则新建Group。
> 3. 新的Group等待group_wait指定的时间（等待时可能收到同一Group的Alert），根据resolve_timeout判断Alert是否解决，然后发送通知。
> 4. 已有的Group等待group_interval指定的时间，判断Alert是否解决，当上次发送通知到现在的间隔大于repeat_interval或者Group有更新时会发送通知。




#### 4. prometheus in k8s细节分析

- 监控K8S Node节点

*使用k8s元数据指定node，而不是static_configs*

```shell
    - job_name: kubernetes-nodes            //job1
      scrape_interval: 30s
      static_configs:
      - targets:
        - 192.168.43.3:9100
    - job_name: kubernetes-nodes2           //job2
      scrape_interval: 30s
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels:
        - __address__
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace
```

> 如上，job1和job2监控的目标是一样的，但是job2的更适合于k8s下的配置，因为它的通用性更强



- K8S中部署Alertmanager

1. 部署Alertmanager

2. 配置Prometheus与Alertmanager通信

3. 配置告警

​      -  prometheus指定rules目录

​      - configmap存储告警规则

​      - configmap挂载到容器rules目录

​      - 增加alertmanager告警配置

(下面是配置Prometheus与Alertmanager通信，2和3请看上面的内容)



*Prometheus与Alertmanager通信*

```shell
    alerting:                                  //配置1,默认配置
      alertmanagers:
      - kubernetes_sd_configs:
          - role: pod
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace]
          regex: monitoring
          action: keep
        - source_labels: [__meta_kubernetes_pod_label_k8s_app]
          regex: alertmanager
          action: keep
        - source_labels: [__meta_kubernetes_pod_container_port_number]
          regex:
          action: drop

    alerting:                                //配置2
      alertmanagers:
      - static_configs:
          - targets: ["alertmanager:80"]
```

> 这两个配置都可



*rules配置的加载*

**参考文档: https://www.cnblogs.com/wurijie/p/11795711.html**

```shell
curl -X POST http://IP:port/-/reload
```



