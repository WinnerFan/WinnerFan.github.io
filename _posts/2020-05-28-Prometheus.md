---
layout: post
title: Prometheus
tags: Open Source Software
---

## 客户端改造

### Main配置
暴露一个http get
```
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_httpserver</artifactId>
    <version>0.7.0</version>
</dependency>

// 启动http server
HTTPServer httpServer = new HTTPServer(new InetSocketAddress(3333), CollectorRegistry.defaultRegistry);
```
### Web配置
1. 共用Web项目ip和port
```
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_httpserver</artifactId>
    <version>0.7.0</version>
</dependency>


<servlet>
    <servlet-name>PrometheusServlet</servlet-name>
    <servlet-class>io.prometheus.client.exporter.MetricsServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>PrometheusServlet</servlet-name>
    <!--复用ip:port,对prometheus server暴露的采集接口-->
    <url-pattern>/prometheus</url-pattern>
</servlet-mapping>
```
2. 不共用ip和port

```
public class HTTPServerListener implements ServletContextListener {
    public HTTPServer httpServer;
    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent){
        try {
            httpServer = new HTTPServer(new InetSocketAddress(3333), CollectorRegistry.defaultRegistry);
        }
        catch (Exception e){
            e.printStackTrace();
        }
    }

    @Override
    public void contextDestroyed(ServletContextEvent servletContextEvent) {
        httpServer.stop();
    }
}

<listener>
    <listener-class><包含包名的完整类名></listener-class>
</listener>
```

### 应用埋点
四种统计指标：Counter、Gauge、Summary、Histogram，基本作用为只增计算、可增可减计算、统计分位数（分桶中数量/总数）、统计分位数（分桶中数量）。name指标名，labelNames标签名，help提示，register注册器分组
```
Counter requests = Counter.build()
    .name("requests_total")
    .labelNames("activityId")
    .help("Total requests.")
    .register();
//开始与结束计数  
Histogram.Timer requestTimer = requestHistogram.startTimer();
requestTimer.observeDuration();
```

### Pushgateway
用于push方式在ps客户端和ps服务端之间配置
```
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_pushgateway</artifactId>
    <version>0.7.0</version>
</dependency

CollectorRegistry registry = new CollectorRegistry();
Counter requests = Counter.build()
    .name("requests_total")
    .labelNames("activityId")
    .help("Total requests.")
    .register(registry);
    
PushGateway pushGateway = new PushGateway("<ip>:<port>");   # 创建长连接
pushGateway.push(registry, <jobName>, PushGateway.instanceIPGroupingKey()); # 推送所有指标
pushGateway.push(requests, <jobName>, PushGateway.instanceIPGroupingKey()); # 推送单个指标
```
## 服务端安装

### 流程
1. 通过grafana展示和报警：ps客户端->ps服务端->grafana->alertmanager->post请求或其他
2. 不通过grafana展示和报警：ps客户端->ps服务端->alertmanager->post请求或其他

### 服务端配置
指标默认携带`instance`和`job`标签，表示ip与port和jobname

prometheus.yml
```
global:
  scrape_interval:     15s      # 抓取时间间隔
  evaluation_interval: 15s      # 检查报警规则的时间间隔
  # scrape_timeout is set to the global default (10s).

# Alertmanager配置
alerting:
  alertmanagers:
  - static_configs:
    - targets: ["<ip1>:<port1>", "<ip2>:<port2>"]

# 报警规则
rule_files:
  # - "using_rules.yml"

# 抓取数据的应用节点配置
scrape_configs:
  - job_name: '<job_name>'       # jobname，可被覆盖
    scrape_interval: 15s         # 局部配置，仅针对当前采集节点有效
    metrics_path: '/prometheus'  # 客户端暴露的采集数据接口
    file_sd_configs:             # 文件配置方式支持热更新，启动加入web.enable-lifecycle
    - files:
      - 'targets.json'
    # honor_labels: true         # pushgateway添加参数
    metric_relabel_configs:      # 只是保留test_target指标中cmd正则test_label的指标，服务器处理
      - source_labels: [__name__,cmd]
        regex: 'test_target;test_label'
        action: keep
    params:                      # 只是保留test_target指标，客户端处理
      'name[]':
        - 'test_target'
```
同目录下targets.json
```
[
    {
        "targets": ["<ip1>:<port1>","<ip2>:<port2>"],
        "labels": {            # 可以省略
			"job": "sys_11",   # 指标中的标签，jobname
            "msg": "msg_1"     # 自定义信息，可以有多个
         }
     }
]
```
同目录下using_rules.yml
```
groups:
- name: total_name
  rules:
  - alert: alert_name    # 以此分组
    annotation: {description: '{{$labels.id}}的值为{{$value}}'}
    expr: sum(a{id="1"} - a{id="1"} offset 1m) by("region", "status") < 0
    for: 10s             # 连续持续时间后报警
```
alertmanager.yml
```
global:
  resolve_timeout: 5m        # 持续未收到报警5m后，报警消除
route:
  group_by: ['alertname']    # 分组规则，根据标签配置
  group_wait: 10s            # 第一次等待多久时间发送一组警报的通知
  group_interval: 10s        # 一组group内最快多久通知一次，firing和resolved共享同一个group
  repeat_interval: 1h        # 一个相同的报警发送间隔
  receiver: 'web.hook'       # 接收器名字
receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://127.0.0.1:5001/alert' # 向xx发送POST请求
```
第一次等待group_wait，后面按照group_interval周期判断发送，当组内报警变化了则立即发送；否则判断是否大于等于repeat_interval，是则发送，否则不发送。例如group_interval=6s，repeat_interval=10s，若报警变化了则6s发送下一个报警，若未变化到第二个周期12s>10s时才发送下一个报警
## PromQL
```
sum(a{id="1"} - a{id="1"} offset 1m) by("region", "status")
```
## Q&A
1. `(a-b)/b>300`
    - `a，b`都没值，no data，不报警
    - `a，b`任意有值，会计算表达式
2. expr为`(a-b)/b>300`报警，当`a=1，b=0`时，会得到+Inf的报警错误
    - ` (a-b-1)/(b-1)>300`，注意当`b-1=0`时，仍会报错
    - ` (a-b)/b>300 and  (a-b)/(b)<9999`，繁琐
    - `(a-b)/b>300 and b!=0`，完美
3. `{{$labels.id}}`和`{{$value}}`格式不同
4. `increase`和`rate`PromQL中只用于Counter类型数据，Count只支持incr和reset，例如sum(Count)后reset一个Count不能使全部reset，rate出现尖峰
5. `increase`和`rate`慎用，是一个估算值，线性外插，即线性推测，推测数据在样本点范围外。例如5s采集一次，即0s，5s等有数据，`[3s, 23s]`查询区间时间与采样点不重合，拿到采样点`{5s:10}`和`{20s:30}`，计算增长率为`20/15`，得到增长值`20/15*20s=26.67`
6. `{{$value}}`只能获取第一个表达式左边的值
7. Grafana配置监控可以监控，PS与Grafana之间是否通
8. increase会自动拟合，在label组合比较少的情况下，可以都初始化所有label组合的指标数据为0
9. increase从零开始增长，0到1会错误，解决方案还是无法处理重启的短时间
```
(sum(pay_metrics{pay_way="weixin"} or pay_metrics{pay_way="weixin"}*0) by(payMethod)-
sum(pay_metrics{pay_way="weixin"} offset 30m or pay_metrics{pay_way="weixin"}*0) by(pay_way))
```