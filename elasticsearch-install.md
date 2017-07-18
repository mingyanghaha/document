---
title: Elasticsearch & plugins 安装教程

---

##一、Elasticsearch 安装文档
###环境准备
```
CentOS7.1
JDK1.8
elasticsearch-5.4.3
```
###1.1 JDK安装配置
wget -c http://download.oracle.com/otn-pub/java/jdk/8u111-b14/jdk-8u111-linux-x64.tar.gz

###1.2 elasticsearch安装

####1.2.1 下载解压
```
cd 指定目录 #指定下载目录
wget -N https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.4.3.tar.gz
tar -zxf elasticsearch-5.4.3.tar.gz
ln -s elasticsearch-5.4.3 elasticsearch
mkdir -p /var/data/elasticsearch
mkdir -p /var/logs/elasticsearch
```

####1.2.2 修改配置文件
```
vi elasticsearch/config/elasticsearch.yml

cluster.name: elasticstack
path.data: /var/data/elasticsearch
path.logs: /var/logs/elasticsearch
network.host: 0.0.0.0
http.port: 11200
transport.tcp.port: 11300
discovery.zen.ping.unicast.hosts: ["10.213.162.77", "10.213.162.78", "10.213.162.79"]
discovery.zen.minimum_master_nodes: 3
http.cors.enabled: true
http.cors.allow-origin: "*"
```
####1.2.3 创建elasticsearch用户
elasticsearch不能用root用户启动

```
groupadd elasticsearch
useradd  elasticsearch -g elasticsearch -p elasticsearch
```
####1.2.4 修改服务器相关参数
```
修改vm.map 限制
sysctl -w vm.max_map_count=262144
或
vi /etc/sysctl.conf
vm.max_map_count=262144

修改文件限制
ulimit -n 102400
或
vi /etc/security/limits.conf
elasticsearch hard nofile 102400
elasticsearch soft nofile 102400
```
####1.2.5 切换到elasticsearch用户下启动
```
su elasticsearch
cd /var/wd/elasticsearch
./bin/elasticsearch
```
验证是否启动是否成功
curl  'http://10.213.162.77:11200'

##二、head插件安装文档
###2.1 安装node
```
cd /usr/local
wget -N https://nodejs.org/dist/v7.2.0/node-v7.2.0-linux-x64.tar.gz
tar -zxf node-v7.2.0-linux-x64.tar.gz
ln -s node-v7.2.0-linux-x64 node
vi /etc/profile
export PATH=$PATH:/usr/local/node/bin
source /etc/profile
```
###2.2 安装grunt
```
npm install -g grunt-cli
```
###2.3 安装head
####2.3.1 下载head插件源码

```
git clone git://github.com/mobz/elasticsearch-head.git

```
####2.3.2 修改Gruntfile.js
```
connect: {
    server: {
        options: {
            port: 11100,
            hostname: '*',
            base: '.',
            keepalive: true
        }
    }
}
```
####2.3.3 修改app.js
```
this.base_uri = this.config.base_uri || this.prefs.get("app-base_uri") || "http://10.213.162.77:11200";

```
####2.2.4 运行head
```
npm install 
grunt server

```
##三、Kibana安装文档

###3.1 下载及安装
注：Kibana必须保证和elasticsearch版本一致

###3.1.1 下载
```
cd /var/wd
wget -N https://artifacts.elastic.co/downloads/kibana/kibana-5.4.3-linux-x86_64.tar.gz
tar -zxf kibana-5.4.3-linux-x86_64.tar.gz
ln -s kibana-5.4.3 kibana
```

###3.1.2 修改配置
```
vi kibana/config/kibana.yml

server.port: 11201
server.host: "0.0.0.0"
elasticsearch.url: "http://10.213.162.78:11200"
elasticsearch.username: "elastic"
elasticsearch.password: "elastic"
```

##四、Logstash安装
    Logstash 是一款强大的数据处理工具，它可以实现数据传输，格式处理，格式化输出，还有强大的插件功能，常用于日志处理。

###4.1 logstash版本要求
| Kafka Client Version | Logstash Version | Plugin Version | Why? |
|----------|-----------|----------|-------------------------------|
|   0.8  |2.0.0 - 2.x.x     | <3.0.0         | Legacy, 0.8 is still popular|       
|   0.9  |2.0.0 - 2.3.x     | 3.x.x         | Works with the old Ruby Event API (event[‘product’][‘price’] = 10)|     
|   0.9  |2.4.x - 5.x.x     | 4.x.x          | Works with the new getter/setter APIs (event.set(‘[product][price]’, 10)) | 
|   0.10.0.x  |2.4.x - 5.x.x     | 5.x.x          | Not compatible with the ⇐ 0.9 broker | 

###4.2 logstash下载安装
```
cd /var/wd/
wget -c https://artifacts.elastic.co/downloads/logstash/logstash-5.4.3.tar.gz
tar -xzvf logstash-5.4.3.tar.gz
ln -s logstash-5.4.3 logstash
cd logstash
```

###4.3 日志采集filebeat配置
```
mkdir plugin-config
vi plugin-config/filebeat.conf
```
###4.4 日志采集filebeat配置

```
input {
    beats {
        port => "10044"
    }
}
# The filter part of this file is commented out to indicate that it is
# optional.
filter {
	#grok根据日志格式配置
   grok {
        match => ["message", "%{TIMESTAMP_ISO8601:timestamp} %{WORD:trace_id} \[.*\] %{LOGLEVEL:level}"]
        remove_field => [ "beat","tags"]
     }
  }
output {
	 #logstash直接输出到es
     elasticsearch {
        hosts => ["10.213.131.131:11200","10.213.131.132:11200","10.213.131.134:11200"]
        index => "%{[@metadata][beat]}-%{+YYYY.MM}"
        document_type => "%{[@metadata][type]}"
     }
}
```

###4.5 启动logstash

  控台启动，观察错误日志，没问题在后台启动

```
bin/logstash -f plugin-config/filebeat.conf --config.reload.automatic

```

##五、Filebeat安装
###5.1 Filebeat介绍
<p>Beats 平台是 Elastic.co 从 packetbeat 发展出来的数据收集器系统。beat 收集器可以直接写入 Elasticsearch，也可以传输给 Logstash。其中抽象出来的 libbeat，提供了统一的数据发送方法，输入配置解析，日志记录框架等功能。也就是说，所有的 beat 工具，在配置上，除了 input 以外，在output、filter、shipper、logging、run-options 上的配置规则都是完全一致的 ，filebeat是beat中的一员。
</p>

###5.2 Filebeat下载
```
cd /var/wd/
wget -c https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.4.3-linux-x86_64.tar.gz
tar -xzvf filebeat-5.4.3-linux-x86_64.tar.gz
cd filebeat-5.4.3-linux-x86_64
```
<p>这里安装的是bit版，也可以选择rpm版本安装</p>
```
wget -c https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.4.3-x86_64.rpm
```

###5.3 Filebeat配置
```
vi filebeat.yml
```
####5.3.1 采集数据源
```
- input_type: log

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /var/wd/feeds_api/logs/feeds_api_info.log
```

####5.4.2 添加产生日志应用名
```
fields:
     app: feeds-api
```
####5.4.3 指定输出
```
output.logstash:
  # The Logstash hosts
  hosts: ["10.213.131.132:10044","10.213.131.131:10044"]
  worker: 2
  loadbalance: true
  index: feeds-log
```
####5.4.3 启动关闭脚本
+ 启动脚本

```
vi startup.sh

#!/bin/bash
nohup ./filebeat -e -c filebeat.yml -d publish  &
```

+ 关闭脚本

```
vi shutdown.sh

#!/bin/bash
runningPID=`pgrep -f "./filebeat -e -c filebeat.yml -d publish"`
if [ "$runningPID" ]; then
      echo "filebeat pid: $runningPID"
      kill -15 $runningPID
fi
sleep 2
```

