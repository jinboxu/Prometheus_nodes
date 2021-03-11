## alertmanager action

重点记录webhook的告警方式

#### 1. Alertmanager之webhook

https://prometheus.io/docs/alerting/latest/configuration/#webhook_config



- 请求格式

The Alertmanager will send HTTP POST requests in the following JSON format to the configured endpoint:

```json
{
  "version": "4",
  "groupKey": <string>,              // key identifying the group of alerts (e.g. to deduplicate)
  "truncatedAlerts": <int>,          // how many alerts have been truncated due to "max_alerts"
  "status": "<resolved|firing>",
  "receiver": <string>,
  "groupLabels": <object>,
  "commonLabels": <object>,
  "commonAnnotations": <object>,
  "externalURL": <string>,           // backlink to the Alertmanager.
  "alerts": [
    {
      "status": "<resolved|firing>",
      "labels": <object>,
      "annotations": <object>,
      "startsAt": "<rfc3339>",
      "endsAt": "<rfc3339>",
      "generatorURL": <string>       // identifies the entity that caused the alert
    },
    ...
  ]
}
```



- 通知模板的数据结构

https://prometheus.io/docs/alerting/latest/notifications/#data-structures



*StartsAt和EndsAt*  (待验证)

StartsAt用于告警触发的时间，EndsAt则用于告警恢复的时间。

如果我们在告警模板中直接使用$alert.StartsAt，得到的时间格式如下，

```bash
1 告警时间：2020-12-16 22:35:33.676515606 +0800 CST
```

这个时间也就是和我们机器上的时间一致，因此如果需要保证机器上的时间是我们需要的时区。

可以通过**tzselect**命令设置时区，完成后最好重启下系统，通过/var/log/messages的时间戳来确定时间是否满足我们的需求，这样我们收到的告警时间才是符合我们所在时区。

不过对于我们的告警不需要精确到纳秒级别，也不需要显示时区，那就需要对这个时间进行格式化，这也是我们模板中使用$alert.StartsAt.Format的原因，至于其中的"2006-01-02 15:04:05"，可以理解为时间格式，而且还必须就是这个时间，不可以修改，就当做是魔术字吧，go的开发者就是这么个性。

在使用邮件告警时，一般使用如下格式，

```bash
1 告警时间: {{ ($alert.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
```

其中Add 28800e9表示在基准时间上添加8小时，28800e9是8小时的纳秒数。这就是从UTC时间转换到北京东八区时间。



- 企业微信机器人使用指南

https://blog.csdn.net/qq_15174755/article/details/96742578

```
curl 'https://oapi.dingtalk.com/robot/send?access_token=95911e13db3074168aeed8b666acff2b0930a5f14ac6907b547d1f041173593e' \
   -H 'Content-Type: application/json' \
   -d '
   {
        "msgtype": "text",
        "text": {
            "content": "hello world"
        }
   }'
```



#### 2. webhook接口

在github下做如下搜索: https://github.com/search?q=wechat+alertmanager 得到:

```
dingtalk:
- go , 灵活度更高，支持template模板
   https://github.com/timonwong/prometheus-webhook-dingtalk
- python
  https://github.com/cnych/alertmanager-dingtalk-hook/blob/master/app.py

wechat:
- python
   https://github.com/daozzg/work_wechat_robot
- go ，配置灵活，支持template模板
   https://github.com/kanliyong/alertmanager-wechatrobot-webhook

adapter:
# https://my.oschina.net/guyongquan/blog/3128025
同时支持dingtalk和wechat，但是token或key定义在接口处，不方便使用
```

> 使用webhook的模板定义在webhook接口处



#### 3. 模板

