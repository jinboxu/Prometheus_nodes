## swarmprom

#### 1. 安装node_exporter服务

```
[root@manager1 ~]# cat /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=https://prometheus.io

[Service]
Restart=on-failure
ExecStart=/usr/local/node_exporter/node_exporter \
          --collector.textfile.directory=/var/lib/node-exporter \
          --collector.systemd --collector.systemd.unit-whitelist=(docker|ssh|rsyslog).service \
          --collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/) \
          --no-collector.ipvs

[Install]
WantedBy=multi-user.target
```



#### 2. `dockerswarm_sd_config`

官方配置示例:  https://github.com/prometheus/prometheus/blob/release-2.25/documentation/examples/prometheus-dockerswarm.yml



一个最基础的prometheus配置文件:

```yaml
# Docker Swarm.
global:
  scrape_interval: 1m
  scrape_timeout: 10s
  evaluation_interval: 2m

rule_files:
- "rules/*.yml"

scrape_configs:

  # Make Prometheus scrape itself for metrics.
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']

  # Create a job for Docker daemons.
  #
  # This example requires Docker daemons to be configured to expose
  # Prometheus metrics, as documented here:
  # https://docs.docker.com/config/daemon/prometheus/
  #- job_name: 'docker'
  #  dockerswarm_sd_configs:
  #    - host: unix:///var/run/docker.sock # You can also use http/https to connect to the Docker daemon.
  #      role: nodes
  #  relabel_configs:
  #    # Fetch metrics on port 9323.
  #    - source_labels: [__meta_dockerswarm_node_address]
  #      target_label: __address__
  #      replacement: $1:9323

  - job_name: 'dockerswarm-nodes'
    dockerswarm_sd_configs:
      - host: unix:///var/run/docker.sock 
        role: nodes
    relabel_configs:
      - source_labels: [__meta_dockerswarm_node_address]
        target_label: __address__
        replacement: $1:9100

  - job_name: 'dockerswarm-tasks'
    dockerswarm_sd_configs:
      - host: unix:///var/run/docker.sock # You can also use http/https to connect to the Docker daemon.
        role: tasks
    relabel_configs:
      # Only keep containers that should be running.
      - source_labels: [__meta_dockerswarm_task_desired_state]
        regex: running
        action: keep
      # Only keep containers that have a `prometheus-job` label.
      - source_labels: [__meta_dockerswarm_service_label_prometheus_job]
        regex: .+
        action: keep
      - source_labels: [__meta_dockerswarm_service_label_prometheus_job]
        target_label: job
      # Use the task labels that are prefixed by `prometheus-`.
      #- regex: __meta_dockerswarm_service_label_prometheus_(.+)
      #  action: labelmap
      #  replacement: $1

      - action: replace
        regex: (.+)
        source_labels:
        - __meta_dockerswarm_service_label_prometheus_io_path
        target_label: __metrics_path__
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels:
        - __address__
        - __meta_dockerswarm_service_label_prometheus_io_port
        target_label: __address__

alerting:
  alertmanagers:
  - static_configs:
      - targets: ['alertmanager:9093']
```

- 通过role: nodes 采集暴露在节点上服务
- 通过role: tasks 和 服务标签 采集容器服务

通过与官方的示例配置，我这个地方有一些改变：

1. 与k8s的服务发现保持了一致性，如下通过cadvisor服务来示例

```yaml
  cadvisor:
    image: google/cadvisor
    #labels:
    #  prometheus-job: 'cadvisor'
    networks:
      - net
    command: -logtostderr -docker_only
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /:/rootfs:ro
      - /var/run:/var/run
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    deploy:
      mode: global
      labels:
        prometheus-job: 'cadvisor'
        prometheus.io/port: '8080'
      restart_policy:
        condition: on-failure
      resources:
        limits:
          memory: 128M
        reservations:
          memory: 64M
```

> 注意，我们使用的是服务的标签deploy.labels , 而不是容器标签(注释的)



如上，类比kubernetes,通过标签```prometheus.io/port: '8080'```定义了服务的端口，采集路径为默认的'/metrics'，当然我们也可以通过```prometheus.io/path```标签名重定义



2. 重新定义job的名字

我们通过role: tasks 服务发现采集的服务可以有很多，如果不对其进行重写relabel_config，它们的job名字都将保持一样，如上为dockerswarm-tasks, 不利于我们对告警进行细致的分析和处理，所以有了如下标签重写:

```
      - source_labels: [__meta_dockerswarm_service_label_prometheus_job]
        regex: .+
        action: keep
      - source_labels: [__meta_dockerswarm_service_label_prometheus_job]
        target_label: job
```

> - 采集有prometheus_job标签的服务
> - 对这个标签改名，这样我们就手动定义了这个job的名称



注意，官方示例配置下面有个这样的配置被我舍弃了，原因是：如果使用了这个配置，上面定义采集方式的服务标签将在prometheus抓取后生成io-path和io-port这样没有意义的标签。

```
      # Use the task labels that are prefixed by `prometheus-`.
      - regex: __meta_dockerswarm_service_label_prometheus_(.+)
        action: labelmap
        replacement: $1
```



