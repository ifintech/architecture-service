## docker日志处理

## beats

```shell
chmod 666 /var/run/docker.sock
```

```shell
docker service create --name metricbeat \
  --mode global \
  --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  --mount type=bind,src=/proc,dst=/hostfs/proc,ro=true \
  --mount type=bind,src=/sys/fs/cgroup,dst=/hostfs/sys/fs/cgroup,ro=true \
  --mount type=bind,src=/,dst=/hostfs,ro=true \
  --env ES=10.1.2.4:9200 \
  --network host \
  ifintech/metricbeat
```

### 安装kibana 索引及可视化模板

```shell
docker run -t \
ifintech/metricbeat ./scripts/import_dashboards -es http://10.1.2.5:9200 -url https://artifacts.elastic.co/downloads/beats/beats-dashboards/beats-dashboards-5.5.0.zip
```

### logstash

1. 添加日志解析配置 **/data1/logstash/service.conf**

```shell
input {
  syslog {
    port => '10514'
  }
}
filter {
  dissect {
     mapping => { 
       "message" => "%{?prefix} %{log_time} %{} %{service_name}.%{container_no}.%{container_id} %{?pid} - - %{log_content}"
     }
  }
  date {
    match => ["log_time", "ISO8601"]
  }
  # 判断日志类型
  if [service_name] =~ /^app+/ {
    # app 日志需要进一步解析 去除附加的头信息
    dissect {
      mapping => { 
        "log_content" => "%{?prefix} said into stdout: %{app_log_content}"
      }
    }
    grok {
      match => {
        "app_log_content" => "^\"(?<app_log_content_trim>[\s\S]+)\"$"
      }
    }
    if "_grokparsefailure" not in [tags] {
      mutate {
        update       => ["log_content", "%{app_log_content_trim}"]
      }
    }
    mutate {
      remove_field => ["app_log_content", "app_log_content_trim"]
      add_field => ["log_type", "app"]
    }
  } else if [service_name] =~ /^http+/ {
    mutate {
      add_field => ["log_type", "nginx"]
    }
  } else {
    mutate {
      add_field => ["log_type", "other"]
    }
  }
  # 解析json 格式日志
  if [log_type] in ["app", "nginx"]{
    json {
      source => "log_content"
    }
  } 
  # 获取落地路径
  if [log_type] == "app" {
    # 非json格式的日志 则为php日志
    if "_jsonparsefailure" in [tags] {
      mutate {
        replace   => ["log_type", "php"]
        add_field => ["log_path", "%{service_name}/%{+YYYYMM}/%{+YYYYMMdd}.log"]
      }
    }
  } else if [log_type] == "nginx" {
    # 非json格式的日志 为nginx error日志
    if "_jsonparsefailure" in [tags] {
      mutate {
        add_field => ["app", "error"]
        add_field => ["level", "error"]
        add_field => ["log_path", "error/%{+YYYYMM}/error.%{+YYYYMMdd}.log"]
      }
    }else{
      mutate {
        add_field => ["level", "info"]
        add_field => ["log_path", "%{app}/%{+YYYYMM}/access.%{+YYYYMMdd}.log"]
      }
    }
  } else if [log_type] == "other" {
    mutate {
      add_field => ["log_path", "%{service_name}/%{+YYYYMM}/%{+YYYYMMdd}.log"]
    }
  }
  # 去掉不必要的数据项
  mutate {
    remove_field => ["message", "0", "1", "2", "3", "4", "5", "6", "7", "8", "9"]
  }
}
output {
  file {
    path => "/mnt/logs/%{log_type}/%{log_path}"
    gzip => false #按gzip方式压缩
    flush_interval => 2  #指定刷入间隔(秒数)，0代表实时写入 默认2s
    codec => line {
      format => "%{log_content}"
    }
  }
  if [log_type] in ["app", "nginx"]{
    elasticsearch {
      hosts => ["10.1.2.5:9200"]
      index => "logstash-%{log_type}-%{app}-%{+YYYYMM}"
      document_type => "%{level}"
      flush_size => 1000 #批量上送数据最大条数 不会超过pipeline.batch.size 默认500
      idle_flush_time => 1 #批量上送数据间隔 默认1s
    }
  } 
}
```

```shell
input {
  stdin {}
}   
filter {
  dissect {
    mapping => { 
        "message" => "%{?prefix} said into stdout: %{log_content}"
      }
    }
    grok {
      match => {
        "log_content" => "^\"(?<app_log_content_trim>[\s\S]+)\"$"
      }
    }
    if "_grokparsefailure" not in [tags] {
      mutate {
        update       => ["log_content", "%{app_log_content_trim}"]
      }
    }
    mutate {
      remove_field => ["app_log_content", "app_log_content_trim"]
      add_field => ["log_type", "app"]
    }
}
output {
   stdout { codec => rubydebug }
}
```

```shell
docker config create logstash-service /data1/logstash/service.conf
```

2. 启动

```shell
docker service create --name logstash \
    --network=elknet \
    --with-registry-auth \
    -p 514:10514 \
    --replicas 2 \
    --limit-cpu 1 \
    --limit-memory 2048mb \
    --config source=logstash-app,target=/conf.d/app.conf \
    dockerhub.jrmf360.com/log/logstash-with-dissect:latest
```

## ElasticSearch

```shell
sudo sysctl -w vm.max_map_count=262144
```

```shell
mkdir /tmp/es
chmod 777 -R /tmp/es
```

```shell
docker service create --name elasticsearch --network=elknet \
  --mode global \
  --with-registry-auth \
  --mount type=bind,src=/tmp/es,dst=/usr/share/elasticsearch/data \
  --env SERVICE_NAME=es \
  --publish 9200:9200 \
  --env discovery.zen.minimum_master_nodes=3 \
  --env path.data=/usr/share/elasticsearch/data  \
  dockerhub.jrmf360.com/log/swarm-elasticsearch:latest
```

## Kibana

```shell
docker service create --name kibana2 \
  --replicas 2 \
  --publish 5602:5601 \
  --env  ELASTICSEARCH_URL=http://10.1.2.4:9200 \
  ifintech/kibana
```

## 统一部署

**compose-stack-log.yml**

```yaml
version: "3.3"

services:
  logstash:
    image: logstash:5.4.0
    ports:
      - 514:10514
    networks:
      - servicenet
    configs:
      - source: logstash-service
        target: /conf.d/service.conf
    volumes:
      - type: bind
        source: /tmp/logs
        target: /mnt/logs
    deploy:
      replicas: 2
    command: ["-f", "/conf.d"]
  elasticsearch:
    image: ifintech/swarm-elasticsearch
    ports:
      - 9200:9200
    networks:
      - servicenet
    environment:
      SERVICE_NAME: log_elasticsearch
      discovery.zen.minimum_master_nodes: 3
      path.data: /usr/share/elasticsearch/data
    volumes:
      - type: bind
        source: /tmp/es
        target: /usr/share/elasticsearch/data
    deploy:
      mode: global
  kibana:
    image: kibana
    ports:
      - 5601:5601
    networks:
      - servicenet
    environment:
      ELASTICSEARCH_URL: http://log_elasticsearch:9200
    deploy:
      replicas: 2
configs:
  logstash-service:
    file: /data1/logstash/service.conf
networks:
  servicenet:
    external: true
```

```shell
docker config create wallet-constant /data1/wallet/constant.php
docker config create wallet-security /data1/wallet/security.php
docker config create wallet-server   /data1/wallet/server.php

docker service create --name app-wallet \
  --with-registry-auth \
  --replicas 2 \
  --env PHP_RUN_ENV=product \
  --config source=wallet-constant,target=/data1/htdocs/wallet/conf/constant/product.php \
  --config source=wallet-security,target=/data1/htdocs/wallet/conf/security/product.php \
  --config source=wallet-server,target=/data1/htdocs/wallet/conf/server/product.php \
  -p 30000:9000 \
  dockerhub.jrmf360.com/cash-loan/wallet php-fpm
```

### logtail

**安装配置** 

1. 需要按kibana版本来安装对应版本的插件 [插件地址](https://github.com/sivasamyk/logtrail/releases)

```shell
cd /usr/share/kibana
./bin/kibana-plugin install https://github.com/sivasamyk/logtrail/releases/download/v0.1.17/logtrail-5.4.3-0.1.17.zip
```

1. 修改配置文件 **/usr/share/kibana/plugins/logtrail/package.json**

```json
{
  "index_patterns" : [
    {
      "es": {
        "default_index": "logstash-app-*",
        "allow_url_parameter": true
      },
      "tail_interval_in_seconds": 10,
      "es_index_time_offset_in_seconds": 0,
      "display_timezone": "local",
      "display_timestamp_format": "MM-DD HH:mm:ss",
      "max_buckets": 500,
      "nested_objects" : false,
      "default_time_range_in_days": 10,
      "max_hosts": 100,
      "max_events_to_keep_in_viewer": 5000,
      "fields" : {
        "mapping" : {
            "timestamp" : "@timestamp",
            "display_timestamp" : "@timestamp",
            "hostname" : "server_ip",
            "program": "app",
            "message": "log_content"
        }
      }
    }
  ]
}
```

- default_index 默认索引
- tail_interval_in_seconds 数据刷新间隔时间
- default_time_range_in_days 默认查询时间范围(天)

#### 网络

创建集群私有网络

```shell
docker network create --driver overlay --subnet 192.168.1.0/24 servicenet
```

#### 