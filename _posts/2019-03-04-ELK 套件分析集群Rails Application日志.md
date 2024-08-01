---
layout: post
title: ELK 套件分析集群Rails Application日志
keywords: ELK 日志分析
background: '/assets/images/bg-post.jpg'
---

随着系统访问量的增大，集群化的部署方式使得每个 `rails log` 文件都是写在各自的服务器硬盘上，这给排查错误造成很大的困扰，只能挨个服务器去找日志信息。 这个时候就需要一个工具来将日志收集起来统一处理。ELK 就是一个很好的日志处理分析工具。

## ELK 简介
`ELK` 是 `Elasticsearch`、`Logstash` 和 `Kibana `三种软件产品的首字母缩写。这三者都是开源软件，通常配合使用，而且又先后归于 `Elastic.co` 公司名下，所以被简称为 `ELK Stack`。根据 `Google Trend` 的信息显示，`ELK Stack` 已经成为目前最流行的集中式日志解决方案。

* Elasticsearch：分布式搜索和分析引擎，具有高可伸缩、高可靠和易管理等特点。基于 Apache Lucene 构建，能对大容量的数据进行接近实时的存储、搜索和分析操作。通常被用作某些应用的基础搜索引擎，使其具有复杂的搜索功能；
* Logstash：数据收集引擎。它支持动态的从各种数据源搜集数据，并对数据进行过滤、分析、丰富、统一格式等操作，然后存储到用户指定的位置；
* Kibana：数据分析和可视化平台。通常与 Elasticsearch 配合使用，对其中数据进行搜索、分析和以统计图表的方式展示；
* Filebeat：轻量级开源日志文件数据搜集器。在需要采集日志数据的 server 上安装 Filebeat，并指定日志目录或日志文件后，Filebeat 就能读取数据，迅速发送到 Logstash 进行解析，亦或直接发送到 Elasticsearch 进行集中式存储和分析。

## 常见架构模式

### 最简单的架构
![elk-simple.png](/assets/images/elk-simple.png)
在这种架构中，只有一个  `Logstash`、`Elasticsearch` 和 `Kibana `实例。`Logstash` 通过输入插件从多种数据源（比如日志文件、标准输入 `Stdin` 等）获取数据，再经过滤插件加工数据，然后经 `Elasticsearch` 输出插件输出到 `Elasticsearch`，通过 `Kibana`展示。这种架构适合初学者学习 `ELK` 的工作模式，搭建最简单的工作模式。

### Logstash 作为日志搜索器
![elk-logstash.png](/assets/images/elk-logstash.png)
这种架构是对上面架构的扩展，把一个 `Logstash` 数据搜集节点扩展到多个，分布于多台机器，将解析好的数据发送到 `Elasticsearch server` 进行存储，最后在 `Kibana` 查询、生成日志报表等。

这种架构在日常中应用的比较少；因为需要在各个服务器上部署 Logstash，而它比较消耗 CPU 和内存资源，所以比较适合计算资源丰富的服务器，否则容易造成服务器性能下降，甚至可能导致无法正常工作。

### 使用 `filebeat` 作为日志收集器
![elk-filebeat.png](/assets/images/elk-filebeat.png)
这种架构是对上面架构的优化，`filebeat`比 `logstash` 更轻量，占用资源更少，通过`filebeat`采集日志，经`logstash`处理之后传递给`Elasticsearch`，通过 `Kibana` 展示。

### 引入消息队列,将ES做成集群
![elk-cluster.png](/assets/images/elk-cluster.png)
一些大型项目日志访问频繁，产生的日志文件比较多，对日志的分析要求较高，对稳定性与可靠性要求较高，可以在 `filebeat` 与 `logstash` 之间插入一层队列, 先将数据传递给消息队列，然后通过 `logstash` 拉取消息队列数据, 处理之后传递给ES集群处理。

## 为Rails应用集群增加ELK

### 为项目添加日志解析

* 添加日志解析的Gem

```ruby
gem "lograge"
gem "logstash-event"
```
* 在 `config/initializers/lograge.rb` 配置如下

```ruby
Rails.application.configure do
  config.lograge.enabled = true
  config.lograge.formatter = Lograge::Formatters::Logstash.new
  config.lograge.keep_original_rails_log = true
  config.lograge.logger = ActiveSupport::Logger.new "#{Rails.root}/log/lograge_#{Rails.env}.log"
end
```
如果使用 `grape`，grape的日志并不会写入文件中需要另外处理
* 添加 `gem grape_logging` 到gemfile
* 在 api.rb 中添加如下代码

```ruby
class API < Grape::API
  use GrapeLogging::Middleware::RequestLogger,
      instrumentation_key: 'api',
      include: [
        GrapeLogging::Loggers::Response.new,
        GrapeLogging::Loggers::FilterParameters.new,
        GrapeLogging::Loggers::ClientEnv.new,
        GrapeLogging::Loggers::RequestHeaders.new
      ]
  ...
end
```

* 在 `config/initializers/instrumentation.rb` 配置如下, 将日志按照格式发给Lograge

```ruby
# Subscribe to grape request and log with a logger dedicated to Grape
ActiveSupport::Notifications.subscribe('api') do |name, starts, ends, notification_id, payload|
  info = {
    name: name,
    status: payload[:status],
    method: payload[:method],
    path: payload[:path],
    params: payload[:params],
    host: payload[:host],
    db: payload[:time][:db],
    view: payload[:time][:view],
    total: payload[:time][:total],
    remote_ip: payload[:ip],
    format: "json",
    user_agent: payload[:ua],
    origin: payload[:headers]["Referer"],
    logtime: Time.current,
  }.to_json
  Lograge.logger.info info
end
```

### 在集群的各个节点配置日志发送工具 filebeat
* 安装 filesbeat   [点击教程](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation.html)
* `filebeat.yml` 主要配置如下

```
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/www/xxx/xxx/log/lograge_production.log
  fields:
      appname: appname
      log_type: application
      env: production
  fields_under_root: true
output.logstash:
  hosts: [remote_logstash_ip:5044"]
processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~town-square
```
**path路径为`lograge`根据规则生成的 `json` 日志路径**

#### 在运维服务器安装并配置 `logstash`
* 安装 logstash [点击教程](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html)
* `logstash.yml` 主要配置如下

```json
input {
  beats {

    port => 5044
  }
}

filter {
  if [log_type] == "application" {
    json {
      source => "message"
   }
  }
}

output {
  stdout {
    codec => rubydebug
  }

  elasticsearch {
    index => "%{[appname]}-%{[env]}-%{+YYYY.MM.dd}"
    hosts => ['127.0.0.1:9200']
  }
}
```

### 在运维服务器安装并配置elasticsearch和kibana即可

## 参考文章
* [https://www.ibm.com/developerworks/cn/opensource/os-cn-elk-filebeat/index.html](https://www.ibm.com/developerworks/cn/opensource/os-cn-elk-filebeat/index.html)
* [https://www.elastic.co/guide/index.html](https://www.elastic.co/guide/index.html)
* [https://www.elastic.co/guide/cn/kibana/current/index.html](https://www.elastic.co/guide/cn/kibana/current/index.html)